# TCP/TLS behavior

Language: **English** | [中文](zh/tcp-tls-behavior.md)

This document summarizes TCP and TLS-oriented behavior beyond upstream dnsproxy, as implemented in this fork.

## TCP RST: protocol-level behavior correction

In complex LB + encrypted DNS deployments, this project applies a dedicated connection lifecycle state machine to avoid request loss caused by abnormal RST behavior at close phase under high concurrency.

## TLS timeout behavior (DoT)

- Scope: DoT inbound connections (`--tls-port` / `ProtoTLS`).
- Default: `defaultTLSTimeout = 600s` (compile-time default, not exposed as CLI/YAML yet).
- Behavior:
  - DoT idle window uses read deadline (not full deadline).
  - TLS handshake has explicit deadline to avoid indefinite connection occupancy.

## TCP Keep-Alive (TCP/DoT)

- Scope: DNS-over-TCP and DoT inbound connections.
- Default: `defaultTCPKeepAlive = 30s` (compile-time default).
- Goal: reduce connection churn and repeated handshake cost.

## TCP Fast Open (TFO)

- Scope: DoT listener socket path only.
- Platform:
  - Unix (except OpenBSD): attempts `TCP_FASTOPEN` (queue length 256).
  - Windows: no-op.
- Unsupported kernel/platform is non-fatal (logged and continued).

## TLS session resumption

- Server side (DoT/HTTPS): session ticket key initialized for resumption.
- Upstream client side (DoH/DoQ/DoT): `ClientSessionCache` enabled to reduce repeated handshakes.
