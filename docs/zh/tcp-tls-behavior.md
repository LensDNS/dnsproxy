# TCP/TLS 行为

Language: [English](../tcp-tls-behavior.md) | **中文**

本文归纳本 fork 相对上游 dnsproxy 在 TCP 与 TLS 方向上的行为差异。

## TCP RST：协议层行为纠正

在复杂的 LB + 加密 DNS 部署中，本项目通过专用连接生命周期状态机，减轻高并发下关闭阶段异常 RST 导致的请求丢失。

## DoT 的 TLS 超时行为

- **作用域**：DoT 入站连接（`--tls-port` / `ProtoTLS`）。
- **默认值**：`defaultTLSTimeout = 600s`（当前为编译期默认，尚未暴露为 CLI/YAML）。
- **行为要点**：
  - DoT 空闲窗口使用 **read deadline**（非 full deadline）。
  - TLS 握手设有显式 deadline，避免连接长期占用却无进展。

## TCP Keep-Alive（TCP/DoT）

- **作用域**：DNS-over-TCP 与 DoT 入站连接。
- **默认值**：`defaultTCPKeepAlive = 30s`（编译期默认）。
- **目的**：降低连接抖动与重复握手成本。

## TCP Fast Open（TFO）

- **作用域**：仅 DoT 监听 socket 路径。
- **平台**：
  - Unix（不含 OpenBSD）：尝试 `TCP_FASTOPEN`（队列长度 256）。
  - Windows：不启用（no-op）。
- 不支持的内核/平台：记录日志后继续运行，非致命错误。

## TLS 会话恢复

- **服务端（DoT/HTTPS）**：初始化 session ticket key 以支持恢复。
- **上游客户端（DoH/DoQ/DoT）**：启用 `ClientSessionCache`，减少与上游的重复握手。
