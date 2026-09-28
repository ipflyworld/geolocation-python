# IPFly Python SDK

A dependency-free Python client for the [IPFly](https://ipfly.world) IP geolocation API. One file, standard library only — it handles caching, retries, rate limiting, request de-duplication, and batch lookups for you.

- **Zero third-party dependencies** — only `urllib`, `threading`, `ipaddress`, and `json` from the standard library
- **TTL + LRU caching** — in memory, or persisted to a JSON file across process runs
- **Resilient** — automatic retry with exponential backoff + jitter on network/5xx failures
- **Thread-safe de-duplication** — two threads looking up the same IP at the same time share one HTTP request
- **Efficient batching** — a bounded `ThreadPoolExecutor` for looking up many IPs at once
- **Typed errors** — a single `IPFlyError` exception with `.status` and `.code`
- **Context-manager friendly** — `with IPFlyClient(...) as client:`
- **Python 3.7+**

---

## Where this SDK belongs

This client is meant to run **server-side** — in a script, a web backend, a worker, a notebook. It is not meant to be shipped inside a distributed application (a desktop app, a compiled mobile app, anything an end user's machine executes) where your API token could be extracted from the binary. If you need geolocation results in a browser-facing app, put this SDK behind your own backend endpoint and call that endpoint from the client, or use the [IPFly JavaScript SDK](./javascript-sdk.html) with a domain-restricted token for prototyping.

---

## Installation

No PyPI package is published — copy `ipfly_sdk.py` into your project (or host it internally and pull it in your build step).

```bash
# from your project root
curl -O https://ipfly.world/libs/python-sdk/ipfly_sdk.py
# or
wget https://ipfly.world/libs/python-sdk/ipfly_sdk.py
```

```python
from ipfly_sdk import IPFlyClient, IPFlyError, is_valid_ip

client = IPFlyClient(token="YOUR_TOKEN")
```

---

## Quick start

```python
from ipfly_sdk import IPFlyClient, IPFlyError

client = IPFlyClient(
    token="YOUR_TOKEN",
    include="security",  # optional: request the security/ASN field set on every call
)

# Look up a specific IP
data = client.lookup("8.8.8.8")
print(data["city"], data["country_name"], data["security"]["is_vpn"])

# Look up the caller's own IP (omit the ip argument)
me = client.lookup_self()
print("You appear to be in", me["city"])

# Handle errors explicitly
try:
    client.lookup("not-an-ip")
except IPFlyError as err:
    print(err.code, err.message)

# Use as a context manager to clean up cache/state automatically
with IPFlyClient(token="YOUR_TOKEN") as c:
    print(c.lookup("1.1.1.1")["country_name"])
```

---

## Configuration reference

All options are passed to `IPFlyClient(...)`.

| Parameter | Type | Default | Description |
|---|---|---|---|
| `token` | `str` | **required** | Your IPFly API token. |
| `base_url` | `str` | `https://ipfly.world/api` | Override to point at your own backend proxy. |
| `include` | `str` | `None` | Default `include` value sent with every request (e.g. `"security"`). Overridable per call. |
| `timeout` | `float` | `8.0` | Per-request timeout in seconds. |
| `retries` | `int` | `2` | Retry attempts for network errors and `5xx` responses. `4xx` is never retried. |
| `retry_delay` | `float` | `0.3` | Base delay (seconds) for exponential backoff between retries (jitter added automatically). |
| `rate_limit` | `float` | `0` (unlimited) | Max requests/second this client will issue. |
| `concurrency` | `int` | `6` | Default max worker threads for `batch_lookup`. |
| `cache` | `dict` | see below | Response cache settings (see next table). |
| `opener` | `urllib.request.OpenerDirector` | `None` | Custom opener — for corporate proxies, custom SSL contexts, etc. |
| `debug` | `bool` | `False` | Print internal lifecycle events to stdout (tokens are redacted). |
| `on_request` | `(ip, url) -> None` | `None` | Hook fired right before a network request is sent. |
| `on_response` | `(ip, data, from_cache) -> None` | `None` | Hook fired after a successful lookup. |
| `on_error` | `(ip, error) -> None` | `None` | Hook fired when a lookup ultimately fails (after retries). |
| `on_cache_hit` | `(ip, data) -> None` | `None` | Hook fired when a cached response is served instead of a network call. |

### `cache` dict keys

| Key | Type | Default | Description |
|---|---|---|---|
| `enabled` | `bool` | `True` | Set `False` to disable caching entirely. |
| `ttl` | `float` | `300` | How long a cached response stays valid, in **seconds**. |
| `max_size` | `int` | `500` | Max cached entries before least-recently-used entries are evicted. |
| `storage` | `'memory' \| 'file'` | `'memory'` | `'file'` persists the cache to a JSON file across process runs (good for CLI scripts and cron jobs; not built for high write concurrency). |
| `path` | `str` | `.ipfly_cache.json` | File path used when `storage='file'`. |

```python
client = IPFlyClient(
    token="YOUR_TOKEN",
    cache={"storage": "file", "path": "/tmp/ipfly_cache.json", "ttl": 3600},
)
```

---

## API

### `client.lookup(ip=None, include=None, skip_cache=False, timeout=None)`
Look up a single IP. Omit `ip` (or pass `None`) to geolocate the caller.

```python
client.lookup(
    "1.1.1.1",
    include="security",  # overrides the client-level `include` for this call only
    skip_cache=True,       # force a fresh network request, bypassing the cache
    timeout=3.0,            # override the client's default timeout for this call
)
```
Returns the parsed JSON response (a `dict`) — see [Response shape](#response-shape). Raises `IPFlyError` on failure.

### `client.lookup_self(**kwargs)`
Shorthand for `client.lookup(None, **kwargs)` — geolocates the caller's own IP.

### `client.batch_lookup(ips, concurrency=None, **kwargs)`
Looks up a list of IPs using a bounded thread pool (`concurrency` overrides the client default). This call **never raises** for an individual IP failure — it returns a list of `BatchResult` in the same order as `ips`:

```python
from ipfly_sdk import BatchResult

results = client.batch_lookup(["8.8.8.8", "1.1.1.1", "not-an-ip"])
for r in results:
    if r.ok:
        print(r.ip, "->", r.data["city"])
    else:
        print(r.ip, "failed:", r.error.code)
```

`BatchResult` is a small dataclass: `ip: str`, `ok: bool`, `data: Optional[dict]`, `error: Optional[IPFlyError]`.

### `client.clear_cache()`
Empties the response cache immediately.

### `client.cache_stats()`
Returns `{"enabled": ..., "size": ..., "ttl": ..., "max_size": ...}` — useful for logging or a health-check endpoint.

### `client.close()`
Clears the cache and any in-flight request bookkeeping. Called automatically when used as a context manager (`with IPFlyClient(...) as client:`).

### `is_valid_ip(ip: str) -> bool`
Module-level helper — validates an IPv4 or IPv6 string without making a request.

```python
from ipfly_sdk import is_valid_ip

is_valid_ip("8.8.8.8")               # True
is_valid_ip("2001:4860:4860::8888")  # True
is_valid_ip("not-an-ip")             # False
```

---

## Response shape

Successful lookups return the same JSON structure the IPFly API returns, unchanged, as a Python `dict` — see the [IPFly API docs](https://ipfly.world) for the full field reference. Shape depends on your plan and the `include` value requested (`security` and ASN/company fields require Pro or higher).

```python
{
    "ip": "8.8.8.8",
    "hostname": "dns.google",
    "country_name": "United States",
    "city": "Mountain View",
    "latitude": "37.4056",
    "longitude": "-122.0775",
    "time_zone": {"name": "America/New_York", "...": "..."},
    "asn": {"asn": "AS15169", "name": "Google LLC", "...": "..."},
    "security": {"is_vpn": False, "is_tor": False, "...": "..."},
}
```

---

## Error handling

All failures raise `IPFlyError`, a plain `Exception` subclass with:

| Attribute | Description |
|---|---|
| `message` | Human-readable description. |
| `status` | HTTP status code, if the request reached the server (`None` for network/timeout errors). |
| `code` | Machine-readable code — see table below. |
| `cause` | The underlying exception or response body, when available. |

| `code` | Meaning |
|---|---|
| `MISSING_TOKEN` | No token was supplied when creating the client. |
| `INVALID_IP` | The IP string failed local validation before any request was sent. |
| `TIMEOUT` | The request exceeded `timeout`. |
| `NETWORK_ERROR` | The request failed before a response was received (offline, DNS, connection refused, etc). |
| `HTTP_ERROR` / server-provided code | The API responded with a non-2xx status. Check `status` (401 = bad token, 403 = plan/permission issue, 404 = private/bogon IP, 429/5xx = retry-worthy). |
| `BAD_RESPONSE` | The server responded but the body wasn't valid JSON. |

```python
from ipfly_sdk import IPFlyClient, IPFlyError

client = IPFlyClient(token="YOUR_TOKEN")

try:
    client.lookup("8.8.8.8")
except IPFlyError as err:
    if err.code == "TIMEOUT":
        ...  # retry later, log, fall back to a default
    elif err.status == 401:
        ...  # token is invalid — surface a config error, don't retry
    print(err.message)
```

---

## Performance notes

- **Caching** avoids re-querying the same IP within the TTL window — handy for a web app looking up the same visitor repeatedly, or a batch job re-processing overlapping log files.
- **De-duplication**: if two request-handling threads call `lookup("1.2.3.4")` at the same moment, only one HTTP request goes out; both get the same result.
- **Rate limiting** is a courtesy limiter on the *client* side — it smooths bursts (e.g. a bulk-enrichment script) so you don't blow through your plan's requests/second in a tight loop. It doesn't discover your actual plan limits from IPFly.
- **Retries** only apply to transient failures (network errors, timeouts, `5xx`). A `401`/`403`/`404` fails fast since retrying won't fix a bad token or a private IP.
- **Batching** uses a bounded `ThreadPoolExecutor`, not raw `asyncio` — good for I/O-bound HTTP calls without pulling in an async HTTP dependency. If your codebase is already `asyncio`-based, wrap calls with `loop.run_in_executor(None, client.lookup, ip)`.

---

## Examples

### Enriching a CSV of visitor IPs

```python
import csv
from ipfly_sdk import IPFlyClient

client = IPFlyClient(token="YOUR_TOKEN", cache={"storage": "file", "path": "geo_cache.json"})

with open("visitors.csv") as infile, open("visitors_enriched.csv", "w", newline="") as outfile:
    reader = csv.DictReader(infile)
    writer = csv.DictWriter(outfile, fieldnames=reader.fieldnames + ["country", "city"])
    writer.writeheader()

    ips = [row["ip"] for row in reader]

for result in client.batch_lookup(ips, concurrency=8):
    print(result.ip, "->", result.data["country_name"] if result.ok else f"error: {result.error.code}")
```

### A minimal Flask proxy (keep the token server-side)

```python
from flask import Flask, request, jsonify
from ipfly_sdk import IPFlyClient, IPFlyError
import os

app = Flask(__name__)
client = IPFlyClient(token=os.environ["IPFLY_TOKEN"])

@app.get("/api/geo")
def geo():
    ip = request.args.get("ip") or request.remote_addr
    try:
        return jsonify(client.lookup(ip, include="security"))
    except IPFlyError as err:
        return jsonify({"error": err.code, "message": err.message}), err.status or 502
```

Your frontend (browser JS, mobile app, etc.) calls `/api/geo` on your own server — the IPFly token stays in this process and is never sent to the client.

### Command-line usage

The module ships a tiny CLI for quick checks:

```bash
export IPFLY_TOKEN=your_token_here
python -m ipfly_sdk 8.8.8.8
python -m ipfly_sdk               # looks up your own IP
python -m ipfly_sdk 8.8.8.8 --include security
```

---

## Python version support

Requires Python 3.7+ (uses `dataclasses` and `from __future__ import annotations`). No third-party packages — works anywhere the standard library is available, including minimal containers and locked-down environments.

## License

MIT — use it, fork it, ship it.
