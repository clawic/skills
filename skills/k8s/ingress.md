# Ingress and Gateway — Getting Outside Traffic To A Pod

**Read `~/Clawic/data/domains/domains.md` before debugging a hostname**: what that name is supposed to point at, and who issues its certificate, is recorded there and settles half the DNS-shaped questions without touching the cluster.

The chain is: DNS → cloud load balancer → controller pod → Service → EndpointSlice → pod. Six links, and the HTTP status code tells you which one broke. The status decoder and the failure modes are universal; the syntax for every knob follows `ingress_controller` (translation table below — examples elsewhere in this file are ingress-nginx unless marked).

## Status Code Decoder

| Response | Broken link | First check |
|---|---|---|
| DNS does not resolve | External DNS record | The LB hostname/IP versus the record; external-dns logs if it manages records |
| Connection refused / timeout at the edge | LB or its health checks | LB target health; with `externalTrafficPolicy: Local`, nodes without a pod fail by design (`networking.md`) |
| 404 from the controller's default backend | No rule matched | Host header (including port), `pathType`, and `ingressClassName` |
| 503 | Service has no ready endpoints | `kubectl get endpointslices` (`networking.md`) |
| 502 | Backend closed the connection or spoke the wrong protocol | Backend on HTTPS or gRPC without the matching `backend-protocol`; keepalive race (below) |
| 504 | Backend slower than the controller's read timeout | Per-route timeout annotation; check whether the app is throttled (`resources.md`) |
| 413 | Request body over the controller limit | ingress-nginx defaults `proxy-body-size` to 1m — uploads need it raised explicitly |
| 400 with a TLS error | SNI/certificate mismatch | The TLS Secret must live in the SAME namespace as the Ingress |
| Intermittent 502 during deploys | Endpoint removal lag | `preStop` sleep + `readinessGates` (`probes.md`) |

## The Same Five Knobs, Per Controller

Every controller exposes the same five levers under a different name. Prefixes: ingress-nginx `nginx.ingress.kubernetes.io/`, HAProxy `haproxy.org/`, ALB `alb.ingress.kubernetes.io/`; Traefik attaches behavior as `Middleware` CRDs referenced by `traefik.ingress.kubernetes.io/router.middlewares: <ns>-<name>@kubernetescrd`.

| Knob | ingress-nginx | Traefik | HAProxy | ALB (AWS LB Controller) |
|---|---|---|---|---|
| Max body size | `proxy-body-size` (default 1m) | `buffering` middleware, `maxRequestBodyBytes` | `max-body-size` in jcmoraisjr's controller; haproxytech's has no per-route equivalent — cap in the app | No ALB-level cap for instance/IP targets (Lambda targets: 1 MB) |
| Backend read timeout | `proxy-read-timeout` (default 60s) | Static config, not per-Ingress: entryPoint `respondingTimeouts` and `ServersTransport.forwardingTimeouts` | `timeout-server` | `load-balancer-attributes: idle_timeout.timeout_seconds` (default 60) |
| Path rewrite | `rewrite-target` + `use-regex: "true"` | `stripPrefix` / `replacePathRegex` middleware | `path-rewrite` | None — actions do forward/redirect/fixed-response only; rewrite in the app |
| Rate limit | `limit-rps` (per controller replica) | `rateLimit` middleware (`average`, `burst`) | `rate-limit-requests` + `rate-limit-period` | None on the ALB — AWS WAF rate-based rule via `wafv2-acl-arn` |
| Backend protocol | `backend-protocol: HTTPS \| GRPC` | `service.serversscheme: h2c \| https` | `server-proto: h2` (+ `server-ssl`) | `backend-protocol: HTTPS` + `backend-protocol-version: GRPC \| HTTP2` |

- **istio**: not annotation-driven at all. Timeout and rewrite are typed fields on `VirtualService` (`http[].timeout`, `http[].rewrite.uri`), protocol comes from the Service port name or `appProtocol` (`grpc`, `http2`), and body limits plus rate limiting need Envoy config (local rate limit, or an external rate-limit service). Treat `ingress_controller: istio` as "mesh objects, not Ingress annotations".
- **gateway-api**: typed spec fields — `HTTPRoute.rules[].timeouts.request` and `.backendRequest`, the `URLRewrite` filter (`ReplacePrefixMatch`), `appProtocol: kubernetes.io/h2c` on the Service. Body limits and rate limiting are still implementation-specific policy attachment (→ Gateway API below).
- **none**: no controller in the path — traffic arrives via `Service type=LoadBalancer` or NodePort, so all five knobs belong to the cloud LB or the app (`networking.md`).
- Names drift across versions and forks — two HAProxy ingress controllers, and `nginx.ingress.kubernetes.io/` (ingress-nginx) is a different project from `nginx.org/` (NGINX Inc). Confirm against the installed controller's annotation reference before shipping; the failure modes in the decoder above are identical either way.

## Ingress Rules That Bite

- `pathType: Prefix` matches whole path segments: `/api` matches `/api` and `/api/v1`, never `/apiv1`. `Exact` matches one string. `ImplementationSpecific` means "ask the controller", so anything portable pins Prefix or Exact.
- Longest-match wins across rules in most controllers, but overlap between two Ingress objects for the same host is resolved controller-specifically — keep one Ingress per host where you can.
- `ingressClassName` is required when more than one controller runs. Without it, either nobody claims the Ingress (silent 404s) or two controllers both do.
- An Ingress can only reference Services in its own namespace. Cross-namespace routing needs Gateway API with a `ReferenceGrant`, or a Service of type ExternalName as a bridge.
- Rewrites are where regex leaks in: ingress-nginx `rewrite-target` with `$1` requires the path to be a capture group and `use-regex: "true"`. A rewrite that works for `/app` and breaks for `/app/` is almost always a missing `(/|$)(.*)`.
- Default backend catches everything unmatched. Pointing it at a real 404 page is worth the five minutes: an unstyled controller 404 is indistinguishable from an outage during triage.

## TLS

- The `tls` block references a Secret of type `kubernetes.io/tls` with `tls.crt` and `tls.key`, in the Ingress's namespace. Wildcard certificates shared across namespaces must be replicated (a secret-replication controller or per-namespace issuance).
- cert-manager debugging is a strict chain: `Certificate` → `CertificateRequest` → `Order` → `Challenge`. Describe them in that order; the failure message lives at the deepest level, never on the Certificate.
- HTTP-01 challenges need the ACME path reachable from the internet — a global redirect-to-HTTPS rule or an auth annotation on `/` breaks them. DNS-01 avoids that entirely and is the only option for wildcards.
- Certificate renewal failures are silent until expiry. Alert on `certmanager_certificate_expiration_timestamp_seconds` minus now, not on renewal errors.
- TLS termination point matters for the backend: terminating at the LB means the controller and pods see plain HTTP, and any application redirect to HTTPS creates an infinite loop unless `X-Forwarded-Proto` is honored.

## Timeouts, Keepalive, and the Mysterious 502

- Three timeouts stack: cloud LB idle timeout, controller `proxy-read-timeout` (ingress-nginx default 60s), and the application's own. The longest useful request must fit inside the smallest of the three.
- The keepalive race: if the backend's idle timeout is shorter than the proxy's, the backend closes a pooled connection exactly as the proxy sends a request on it → intermittent 502s under low traffic, worse at night. Rule: backend keepalive timeout > proxy keepalive timeout > LB idle timeout.
- WebSockets and SSE need the read timeout raised on those routes specifically; the default kills long-lived streams at exactly 60s, which users report as "it disconnects every minute".
- gRPC needs the backend-protocol knob for your controller (table above) and HTTP/2 end to end; without it the controller downgrades and the client sees mangled trailers rather than an obvious error.

## Client IP and Rate Limiting

- Behind an LB, the pod sees the LB's IP unless `externalTrafficPolicy: Local` or PROXY protocol is enabled. Rate limits and audit logs keyed on the observed source IP will then bucket the entire internet into one client.
- Trust `X-Forwarded-For` only for the hops you control, counting from the right. A misconfigured trust depth is a spoofable identity.
- Controller-level rate limiting (ingress-nginx `limit-rps`) is per controller replica, not global: three replicas mean three times the configured rate. Global limits need a shared store or a mesh.

## Gateway API

- The successor design, GA since v1.0, splitting responsibility across three objects: `GatewayClass` (infrastructure provider), `Gateway` (the listener, owned by the platform team), and `HTTPRoute`/`GRPCRoute`/`TCPRoute` (owned by app teams).
- What it fixes over Ingress: cross-namespace routing with explicit `ReferenceGrant` permission, header and query matching, weighted traffic splitting in the API instead of controller annotations, and typed policy attachment.
- Migration reality: run both. Ingress and Gateway can serve the same cluster on different hostnames; move host by host and keep DNS as the cutover switch.
- Traffic splitting via `backendRefs` weights is the cleanest canary available without a mesh — percentage-accurate per request, unlike replica-ratio canaries (`rollouts.md`).

## When Nothing Else Explains It

Bisect the chain from the inside out; each step eliminates everything below it:

1. `kubectl exec` from another pod to the Service → app and Service are fine.
2. `kubectl port-forward svc/<svc>` from your laptop → the Service layer is fine; the fault is the controller or above.
3. `kubectl exec` into the controller pod and curl the Service directly → controller-to-backend path is fine; the fault is routing config or the LB.
4. `curl -H 'Host: real.example.com' http://<controller-pod-ip>/` → tests rule matching without DNS or the LB in the way.
5. Controller logs with the request ID → the only place that shows which rule matched and which upstream was chosen.

When a hostname starts being served, changes what it points at, or gets a new certificate issuer, write its row in the shared `~/Clawic/data/domains/domains.md`: `Domain | Registrar | Expires | Points to | Notes`, with the cluster and ingress controller in `Points to` and the certificate expiry in `Notes` prefixed `TLS:`. Identity is the hostname — update your fields in place and never overwrite `Registrar` or the domain's own `Expires`, which belong to whichever skill manages the registrar (a certificate expiry is not a domain expiry). If the file already has different columns, match them and never rewrite its header. Renewal dates that matter also go in the `## Due` table of `~/Clawic/data/k8s/memory.md`, because certificate renewal failures are silent until expiry.
