# Proxy Protocol v2（PPv2）

Language: [English](../ppv2.md) | **中文**

生产环境中，DNS 常部署在 L4/L7 代理之后（HAProxy / Nginx / Caddy / 云负载均衡等）。若缺少传输层的客户端身份传递，后端只能看到 LB 的源地址，导致按客户端维度的限流、滥用归因、GeoDNS 与安全观测整体退化。

本项目在 **DNS-over-TCP** 与 **DNS-over-TLS（DoT）** 监听端实现了 PPv2。

## 开关与严格语义

启用后为**严格模式**：必须携带 PPv2 头；缺失则拒绝连接。

- **命令行**
  - `--tcp-proxy-protocol-v2`：在 DNS-over-TCP 监听端要求 PPv2。
  - `--tls-proxy-protocol-v2`：在 DoT 监听端要求 PPv2（在 TLS 握手前解析 PPv2）。
  - `--proxy-protocol-v2-read-timeout=duration`：新建 TCP/DoT 连接上读取 PPv2 前缀与负载的超时（默认 `3s`）。
- **YAML**（见 `config.yaml.dist`）
  - `tcp-proxy-protocol-v2: true|false`
  - `tls-proxy-protocol-v2: true|false`
  - `proxy-protocol-v2-read-timeout: 3s`

建议基线与调优空间：

- 多数 **LB → dnsproxy** 部署保持 `proxy-protocol-v2-read-timeout=3s`。
- 低时延可信内网可降至 `1s`–`2s`，以更快淘汰慢连接。
- 仅当跨地域或 LB 抖动导致误杀时再考虑约 `5s`。
- PPv2 场景下 `max-go-routines=32` 作为默认基线，再按 CPU/内存调优。

## 架构：未启用与启用 PPv2

未启用 PPv2：

```text
+--------+      +-------------------+      +----------+
| Client | ---> | HAProxy/Nginx/LB  | ---> | dnsproxy |
+--------+      +-------------------+      +----------+

说明：dnsproxy 看到的 remote addr 为 LB 地址。
```

启用 PPv2：

```text
+--------+      +-------------------+   send-proxy-v2 / PPv2   +----------+      +--------------+
| Client | ---> | HAProxy/Nginx/LB  | -----------------------> | dnsproxy | ---> | Upstream DNS |
+--------+      +-------------------+     (require PPv2)       +----------+      +--------------+

说明：Client → LB 不携带 PPv2；PPv2 由 LB 在 LB → dnsproxy 的后端连接上注入。
```

## 信任边界（必须落实）

- 将 `TrustedProxies` 配置为仅包含可信 LB/代理网段。
- 通过网络访问控制，使仅有 LB/代理能到达启用 PPv2 的监听端。
- 不要在公网边界接受由客户端提供的 PPv2（例如边界 listener 上滥用 `accept-proxy`），否则可能导致源地址伪造。

DoH 请求头、独立可执行文件与嵌入宿主对 `proxy.Config` 的差异，见 [trusted-proxies.md](trusted-proxies.md)。

## 最小 HAProxy 片段（DoT 示例）

```haproxy
frontend ft_dot_853
  bind :853
  default_backend bk_dnsproxy_dot_853

backend bk_dnsproxy_dot_853
  server dnsproxy 127.0.0.1:853 send-proxy-v2
```

在 dnsproxy 侧启用严格模式：`--tls-proxy-protocol-v2`（若需 TCP 则再加 `--tcp-proxy-protocol-v2`）。

## DoQ/UDP 与 PPv2

DoQ/UDP 上的身份传递不能简单照搬 TCP/DoT（数据报语义、地址验证、连接迁移、代理注入的取舍均不同）。

当前实现范围：**TCP/DoT 上的 PPv2**。DoQ/UDP 处于评估范围，将依据真实拓扑与可执行验收标准推进。讨论见：[Proxy Protocol v2 support（Discussion #1）](https://github.com/LensDNS/dnsproxy/discussions/1)。

## 生产部署反馈

若你在生产中以负载均衡后置方式运行，欢迎反馈真实部署形态。

跟踪：

- [Tracking] PPv2 高并发优化：缓解连接 backlog 丢弃（[Issue #2](https://github.com/LensDNS/dnsproxy/issues/2)）
- 实现进展见 [PR #3](https://github.com/LensDNS/dnsproxy/pull/3) 及 Issue #2 评论区。

特别关注：

- 基于 HAProxy / Nginx / Caddy / Envoy 的拓扑
- 流量规模（QPS 量级）
- 需要真实客户端身份的场景（限速、滥用治理、GeoDNS）

- Discussions: <https://github.com/LensDNS/dnsproxy/discussions>
- Issues: <https://github.com/LensDNS/dnsproxy/issues>
