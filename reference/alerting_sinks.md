# Alerting Sinks

Alert routing is available via `cracktrader.alerts`.

## Event Shape

`AlertEvent` fields:

- `alert_type`
- `severity`
- `message`
- `timestamp`
- `trace_id` (optional)
- `context` (object)

## Sink Types (v1 baseline)

- `stdout`: prints JSON lines
- `webhook`: HTTP POST JSON payload
- `email`: SMTP message delivery
- `signal`: HTTP relay to Signal gateway/service

## Registry + Fanout

- Register/build sinks using config dictionaries.
- Route one alert to many sinks with `FanoutAlertSink`.
- Fanout defaults to best-effort delivery (`continue_on_error=True`).

## Webhook Retry/Backoff

`WebhookAlertSink` retry controls:

- `max_retries` (default `2`)
- `backoff_s` (default `0.25`)
- `timeout_s` (default `5.0`)

Backoff is linear (`backoff_s * attempt_number`) and retries occur on delivery exceptions.

## Example

```python
from cracktrader.alerts import AlertEvent, build_alert_sinks_from_config

router = build_alert_sinks_from_config(
    [
        {"type": "stdout", "prefix": "RUNTIME"},
        {"type": "webhook", "url": "https://example.test/hook", "max_retries": 3},
    ]
)

router.emit(AlertEvent(alert_type="risk_trigger", severity="critical", message="kill-switch"))
```

## Config Examples

```python
stdout_cfg = {"type": "stdout", "prefix": "RUNTIME"}
webhook_cfg = {"type": "webhook", "url": "https://example.test/hook", "max_retries": 3}
email_cfg = {
    "type": "email",
    "smtp_host": "localhost",
    "smtp_port": 25,
    "sender_email": "alerts@example.com",
    "recipient_email": "ops@example.com",
}
signal_cfg = {
    "type": "signal",
    "endpoint": "https://signal.example/send",
    "default_recipient": "+15550000000",
    "route_map": {"risk_trigger": "+15551111111"},
    "token": "SIGNAL_GATEWAY_TOKEN",
}
```

For Signal sink secrets, pass `token` from secure config/environment and avoid hardcoding in code.
