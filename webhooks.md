# Web Hooks

Cracktrader ships a lightweight webhook surface for sending external exchange
events into your running strategies. The functionality lives inside the core
package, but it is only enabled when you install the optional `[web]`
dependencies.

```bash
pip install "cracktrader[web]"
```

## Running the local webhook server

The CLI bundles a helper that starts a FastAPI application with a default
`NoOpSink`. This mode is useful for local testing and for piping events into
custom sinks without boilerplate.

```bash
cracktrader webhooks serve --host 0.0.0.0 --port 9000
```

By default the sink simply logs payloads; plug in your own implementation by
instantiating `make_router()` manually:

```python
from fastapi import FastAPI
from cracktrader.webhooks import EventSink, make_router

class StrategySink:
    async def on_trade(self, payload: dict[str, object]) -> None:
        ...  # do something with the payload

    def on_order_update(self, payload: dict[str, object]) -> None:
        ...

app = FastAPI()
app.include_router(make_router(StrategySink()))
```

## Accepted endpoints

Every router exposes the following POST endpoints, each returning a `202` status
when the sink accepts the payload:

- `/webhooks/trade`
- `/webhooks/order`
- `/webhooks/position`
- `/webhooks/balance`
- `/webhooks/heartbeat`

Each endpoint expects a JSON document. A minimal `curl` example:

```bash
curl -X POST http://localhost:9000/webhooks/trade \
     -H "Content-Type: application/json" \
     -d '{"exchange": "polymarket", "symbol": "2345", "side": "buy"}'
```

## Deploying behind your own sink

For production flows you will likely want to register a sink that forwards the
events to your monitoring stack or into a queue. Implement the `EventSink`
protocol and include the router inside your existing FastAPI application. The
hooks are intentionally unopinionated so you can compose them in any Starlette
stack.

Remember to secure the endpoints with your preferred authentication layer before
exposing them on the public internet.
