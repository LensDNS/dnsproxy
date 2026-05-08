# 测试与依赖网络的用例

Language: [English](../testing.md) | **中文**

依赖公网/上游的测试默认开启。在受限或离线环境中可关闭：

- `DNSPROXY_ENABLE_NETWORK_TESTS=0`

在受限网络中，可通过 `DNSPROXY_TEST_*` 覆写测试端点，相关定义见：

- `upstream/resolver_test.go`
- `upstream/upstream_internal_test.go`

## 可选：QUIC / HTTP3 上游定向测试

修改 `upstream/doq.go` 或 `upstream/doh.go` 后建议执行：

```shell
go test ./upstream -run "TestDNSOverQUIC_0RTT|TestUpstreamDoH_0RTT|TestUpstreamDoH_raceReconnect" -count=1
```
