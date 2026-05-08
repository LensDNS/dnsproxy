# Testing and network-dependent tests

Language: **English** | [中文](zh/testing.md)

Network-dependent tests are enabled by default.
Disable in constrained/offline environments with:

- `DNSPROXY_ENABLE_NETWORK_TESTS=0`

For restricted networks, test endpoints can be overridden via `DNSPROXY_TEST_*` in:

- `upstream/resolver_test.go`
- `upstream/upstream_internal_test.go`

See also [AGENTS.md](../AGENTS.md) for a domestic (CN-friendly) test profile using the same variables.
