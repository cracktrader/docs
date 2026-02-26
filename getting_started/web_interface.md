# Using the Web Interface

The local web interface is served directly by the FastAPI runtime and is intended for local workstation use.

## Start the server

```python
from cracktrader.web import start_web_server

start_web_server(host="127.0.0.1", port=8080)
```

Then open:

- `http://127.0.0.1:8080/ui`

## What v1 UI supports

- Start backtest/paper/sandbox/live runs
- Stop and reset runs
- Live status updates (WebSocket + polling)
- Health and result payload visibility
- Strategy/exchange selection with config inputs
- Visible API error banner for actionable failures

## Local-only threat model

Authentication is intentionally out-of-scope for v1. Treat this UI/API as local-trusted only.

- Bind to loopback (`127.0.0.1`) by default.
- Do not expose this endpoint publicly without adding auth and transport hardening.
- Keep reverse-proxy/auth integration as the extension seam for future secured deployments.

## Related references

- [Web API Reference](../reference/web_api.md)
- [Research Pipeline Reference](../reference/research_pipeline.md)
