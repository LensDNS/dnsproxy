# Proxy Protocol v2 (PPv2)

Language: **English** | [中文](zh/ppv2.md)

In production, DNS is often deployed behind L4/L7 proxies (HAProxy/Nginx/Caddy/cloud LB).
Without transport-level client identity propagation, backend services only see the LB source address, which degrades per-client rate limiting, abuse attribution, GeoDNS decisions, and security observability.

This project implements PPv2 on DNS-over-TCP and DNS-over-TLS listeners.

## Flags and strict semantics

When enabled, strict mode requires PPv2 headers; missing headers are rejected.

- CLI
  - `--tcp-proxy-protocol-v2`: require PPv2 on DNS-over-TCP listeners.
  - `--tls-proxy-protocol-v2`: require PPv2 on DoT listeners (parsed before TLS handshake).
  - `--proxy-protocol-v2-read-timeout=duration`: timeout for reading PPv2 preface and payload on new TCP/DoT connections (default: `3s`).
- YAML (`config.yaml.dist`)
  - `tcp-proxy-protocol-v2: true|false`
  - `tls-proxy-protocol-v2: true|false`
  - `proxy-protocol-v2-read-timeout: 3s`

Recommended baseline with tuning headroom:

- Keep `proxy-protocol-v2-read-timeout=3s` for most LB -> dnsproxy deployments.
- Reduce to `1s-2s` in low-latency trusted networks for faster slow-connection eviction.
- Increase to around `5s` only when cross-region/LB jitter causes false timeout drops.
- Keep `max-go-routines=32` as the default baseline for PPv2 deployments; tune based on CPU/memory.

## Architecture: disabled vs enabled PPv2

PPv2 disabled:

```text
+--------+      +-------------------+      +----------+
| Client | ---> | HAProxy/Nginx/LB  | ---> | dnsproxy |
+--------+      +-------------------+      +----------+

Note: dnsproxy sees remote addr = LB address.
```

PPv2 enabled:

```text
+--------+      +-------------------+   send-proxy-v2 / PPv2   +----------+      +--------------+
| Client | ---> | HAProxy/Nginx/LB  | -----------------------> | dnsproxy | ---> | Upstream DNS |
+--------+      +-------------------+     (require PPv2)       +----------+      +--------------+

Note: Client -> LB does not carry PPv2.
PPv2 is injected by LB on the LB -> dnsproxy backend connection.
```

## Trust boundary (mandatory)

- Configure `TrustedProxies` to only trusted LB/proxy CIDRs.
- Restrict network access so only LB/proxy can reach PPv2-enabled listeners.
- Do not accept client-provided PPv2 at the public edge (e.g. `accept-proxy` at boundary listeners), otherwise source address spoofing becomes possible.

For DoH headers and standalone vs embedded `proxy.Config`, see [trusted-proxies.md](trusted-proxies.md).

## Minimal HAProxy fragment (DoT example)

```haproxy
frontend ft_dot_853
  bind :853
  default_backend bk_dnsproxy_dot_853

backend bk_dnsproxy_dot_853
  server dnsproxy 127.0.0.1:853 send-proxy-v2
```

Enable strict mode in dnsproxy with `--tls-proxy-protocol-v2` (and `--tcp-proxy-protocol-v2` if needed).

## DoQ/UDP and PPv2

DoQ/UDP identity propagation cannot be treated as a direct TCP/DoT extension because of different semantics (datagram model, address validation, connection migration, and proxy injection trade-offs).

Current implementation scope is TCP/DoT PPv2.
DoQ/UDP is in evaluation scope and will be driven by real deployment topologies plus executable acceptance criteria.
Discussion: [Proxy Protocol v2 support (Discussion #1)](https://github.com/LensDNS/dnsproxy/discussions/1)

## Real-world deployments

If you are running this behind a load balancer in production,
feedback on real deployment patterns is valuable.

Tracking notice:

- [Tracking] PPv2 high-concurrency optimization: mitigating connection backlog drops ([Issue #2](https://github.com/LensDNS/dnsproxy/issues/2))
- Implementation progress is tracked in [PR #3](https://github.com/LensDNS/dnsproxy/pull/3) and follow-up comments under Issue #2.

Particularly interested in:

- HAProxy / Nginx / Caddy / Envoy based setups
- Traffic scale (QPS range)
- Scenarios requiring real client identity (rate limiting, abuse control, GeoDNS)

- Discussions: <https://github.com/LensDNS/dnsproxy/discussions>
- Issues: <https://github.com/LensDNS/dnsproxy/issues>
