# 测试与依赖网络的用例

Language: [English](../testing.md) | **中文**

依赖公网/上游的测试默认开启。在受限或离线环境中可关闭：

- `DNSPROXY_ENABLE_NETWORK_TESTS=0`

在受限网络（例如部分国内网络）中，可通过 `DNSPROXY_TEST_*` 覆写测试端点，相关定义见：

- `upstream/resolver_test.go`
- `upstream/upstream_internal_test.go`

国内友好测试环境变量组合见仓库根目录 [AGENTS.md](../../AGENTS.md) 中的说明。
