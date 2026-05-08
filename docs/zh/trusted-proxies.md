# 可信代理与客户端地址信任

Language: [English](../trusted-proxies.md) | **中文**

客户端地址语义取决于 **谁构造 `proxy.Config`**，以及当前走的是 **DoH** 还是 **带 PPv2 的 TCP/DoT**。

## 谁构建 `proxy.Config`？

- **独立 `dnsproxy` 可执行文件**（`internal/cmd/proxy.go` → `createProxyConfig`）：当前 `TrustedProxies` 可能被设得很宽（例如全网 IPv4/IPv6）。生产上应视为**已知运维风险**：在暴露 DoH 或 PPv2 时，应在 LB 与配置中收紧信任。
- **嵌入集成**（宿主自行构造 `proxy.Config`）：`TrustedProxies` 往往**较窄**（例如仅环回上的本地反代）。**不要**假定独立二进制默认值适用于宿主；排查「客户端 IP 不对」时以**宿主应用**的配置为准。

## DoH（`proxy/serverhttps.go`）

对 DoH，当 **直连 TCP 对端** 在 `TrustedProxies` 内时，有效客户端地址可来自 `CF-Connecting-IP`、`X-Forwarded-For` 等头；否则回退为 **直连对端**（不信任上述头）。

独立模式下若 `TrustedProxies` 过宽，客户端可能伪造转发头，影响 ECS、私网判定与日志。生产环境应将信任限制在真实反代网段（或通过产品与配置设计收紧默认）。

## PPv2（`proxy/servertcp.go`）

仅当 **直连 TCP 对端** 在 `TrustedProxies` 内时才接受 PPv2。若信任范围过宽，任意能连上的对端都可能通过 PPv2 影响「所见」的客户端地址。

调整信任策略时，请**同时**考虑 DoH 头信任与 PPv2 接受逻辑。

## 维护者参考

完整信任模型及 QUIC/DoH 维护说明见仓库根目录 [AGENTS.md](../../AGENTS.md)（`TrustedProxies`、DoH 与 PPv2 相关章节）。
