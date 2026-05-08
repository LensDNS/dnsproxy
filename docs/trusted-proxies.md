# Trusted proxies and client address trust

Language: **English** | [中文](zh/trusted-proxies.md)

Client address semantics depend on **who constructs `proxy.Config`** and on **which protocol path** (DoH vs TCP/DoT with PPv2) is in use.

## Who builds `proxy.Config`?

- **Standalone `dnsproxy` binary** (`internal/cmd/proxy.go` → `createProxyConfig`): today `TrustedProxies` may be set very broadly (e.g. all IPv4/IPv6). Treat this as a **known operational footgun** for production: tighten trust at the load balancer and in config when you expose DoH or PPv2.
- **Embedded integrations** (e.g. products that construct `proxy.Config` themselves): `TrustedProxies` is often **narrow** (e.g. loopback only for a local reverse proxy). Do **not** assume standalone defaults apply; use the **host application’s** configuration when debugging “wrong client IP” issues.

## DoH (`proxy/serverhttps.go`)

For DoH, the effective client address may come from headers such as `CF-Connecting-IP` or `X-Forwarded-For` when the **direct TCP peer** is in `TrustedProxies`. Otherwise the implementation falls back to the **direct peer** (headers are not trusted).

If `TrustedProxies` is too wide in standalone mode, a client could spoof forwarding headers and affect ECS, private-address logic, and logging. Production deployments should restrict trust to real reverse-proxy subnets (or adjust defaults via product/config design).

## PPv2 (`proxy/servertcp.go`)

PPv2 is accepted only when the **direct TCP peer** is in `TrustedProxies`. If `TrustedProxies` is too wide, any peer that can connect may influence the apparent client address via PPv2.

When changing trust policy, consider **both** DoH header trust and PPv2 acceptance.

## Maintainer reference

The full trust model and QUIC/DoH notes for maintainers live in [AGENTS.md](../AGENTS.md) (section on `TrustedProxies`, DoH, and PPv2).
