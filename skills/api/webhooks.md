# Webhook Traps

## Delivery

- Webhook timeout 5-30s, long process = timeout = retry = duplicates
- Provider retry = same event multiple times, handler MUST be idempotent
- Delivery order not guaranteed, event B can arrive before A
- Provider IP changes, IP whitelist breaks

## Verification

- Signature with timestamp, replay attack if you don't verify the timestamp is recent
- HMAC comparison without constant-time = timing attack possible
- Signature in a custom header (`X-Hub-Signature`) is non-standard, each provider differs
- Body modified by middleware (parsing) before verifying = invalid signature

## Processing

- Response 200 before processing = provider thinks it's OK but the process fails afterward
- Response 500 = provider retries = you process twice if the first attempt did work
- Webhook queue full = new events lost
- Async processing without durability = crash = event lost

## Payload

- Schema change without notice = parser fails in production
- New fields ignored if the parser is strict
- Removed fields your code expects = null pointer / undefined
- Encoding issues, JSON payload with special characters badly encoded

## Development

- Localhost isn't reachable by the provider, you need a tunnel (ngrok)
- Tunnel URL changes each session, reconfigure the webhook every time
- Provider has no manual retry, you must wait for the next event
- Webhook logs on the provider expire quickly, debugging is hard

## Security

- Public endpoint without verification = anyone can send fake events
- Secret shared across environments = staging can affect production
- Webhook handler that makes external calls = potential SSRF
- Detailed error message in response = info leak to the provider/attacker
