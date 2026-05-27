# ClaudeCode-DeepSeek-Proxy

![CI](https://img.shields.io/github/actions/workflow/status/HiChat-fog/claude-deepseek-proxy/ci.yml?branch=master&style=flat-square&label=CI)
![Python](https://img.shields.io/badge/Python-3.9%2B-blue?style=flat-square)
![License](https://img.shields.io/github/license/HiChat-fog/claude-deepseek-proxy?style=flat-square)

A local HTTPS proxy that fixes **thinking block incompatibility** between Claude Code and DeepSeek V4 Pro.

一个本地 HTTPS proxy，解决 **Claude Code + DeepSeek V4 Pro** 的 thinking block 兼容性问题。

---

## What's the problem? / 啥问题？

When using DeepSeek V4 Pro with Claude Code and thinking mode enabled, multi-turn conversations blow up:

用 DeepSeek V4 Pro 跑 Claude Code，开着 thinking 模式，多轮对话直接炸：

1. **400: content[].thinking must be passed back** — DeepSeek returns thinking blocks in responses, but Claude Code doesn't pass them back in the next request. DeepSeek then rejects it. This is essentially a Claude Code bug — it wasn't designed for DeepSeek.

   **400: content[].thinking must be passed back** — DeepSeek 返回了 thinking 块，Claude Code 下一轮没传回去，DeepSeek 就报错了。本质是 Claude Code 的 bug，它不是给 DeepSeek 设计的。

2. **400: thinking cannot be disabled when reasoning_effort is set** — Can you just disable thinking? Nope. Claude Code's `effortLevel` sends `reasoning_effort`, and DeepSeek mandates thinking must be enabled when reasoning_effort is set.

   **400: thinking cannot be disabled when reasoning_effort is set** — 你说那我关 thinking 不行吗？不行。Claude Code 的 `effortLevel` 会发 `reasoning_effort`，DeepSeek 规定设了 reasoning_effort 就必须开 thinking。

Can't enable it, can't disable it — you're stuck. This proxy fixes that.

开也不行关也不行，卡死了。这个 proxy 就是来解决这个的。

---

## How it works / 怎么解决的

```
Claude Code  ──HTTPS──▶  Proxy (127.0.0.1:9191)  ──HTTPS──▶  DeepSeek API
                           │
                           ├─ Request: assistant msg missing thinking blocks?
                           │           Inject them from cache!
                           │           Ensure thinking: enabled
                           │
                           └─ Response: cache thinking blocks
                                         Pass through to Claude Code as-is
```

| Direction | What it does | Why |
|-----------|-------------|-----|
| Request | Inject cached thinking blocks into assistant messages that are missing them | DeepSeek requires them, Claude Code doesn't send them |
| Request | Ensure `thinking: enabled` | Required by reasoning_effort |
| Request | Convert `redacted_thinking` back to `thinking` | DeepSeek only accepts `thinking` type |
| Response | Cache thinking blocks | Needed for future request injection |
| Response | Pass through as-is | Claude Code stores them normally |

TL;DR: **Claude Code doesn't pass back thinking blocks? No worries — the proxy remembers them and auto-injects every request.**

简单说就是：**Claude Code 不传 thinking 块？没关系，proxy 帮你记着，每次请求自动补上。**

---

## Quick Start / 快速开始

### Install dependencies / 装依赖

```bash
pip install cryptography
```

### Clone & install cert / clone & 安装证书

```bash
git clone https://github.com/HiChat-fog/claude-deepseek-proxy.git
cd claude-deepseek-proxy

# Generate cert + install to system trust store (one-time setup)
# 生成证书 + 装到系统信任区（只需跑一次）
python claude_deepseek_proxy.py --install
```

### Start the proxy / 启动 proxy

```bash
# Default port 9191 / 默认 9191 端口
python claude_deepseek_proxy.py

# Or specify a port / 或者指定端口
python claude_deepseek_proxy.py 8443
```

### Configure Claude Code / 配置 Claude Code

Edit `~/.claude/settings.json`:

编辑 `~/.claude/settings.json`：

```json
{
  "env": {
    "ANTHROPIC_BASE_URL": "https://127.0.0.1:9191",
    "ANTHROPIC_AUTH_TOKEN": "your-deepseek-api-key",
    "ANTHROPIC_MODEL": "DeepSeek-V4-pro[1m]"
  },
  "alwaysThinkingEnabled": true
}
```

Restart Claude Code and you're good to go. 重启 Claude Code，搞定。

### Uninstall / 卸载

```bash
python claude_deepseek_proxy.py --uninstall
```

Removes the CA cert from trust store and deletes cert files. Clean exit.

删证书、清文件，干干净净。

---

## Environment Variables / 环境变量

| Variable | Default | Description |
|----------|---------|-------------|
| `DEEPSEEK_API_URL` | `https://api.deepseek.com/anthropic` | Upstream API URL / 上游 API 地址 |
| `THINKING_BUDGET` | `10000` | Thinking token budget / thinking token 预算 |

---

## About the Certificate / 关于证书

The proxy generates a self-signed CA certificate and installs it to your system trust store, so Claude Code won't complain about SSL errors over HTTPS. The CA-issued server cert is for `127.0.0.1`, using `IPAddress` in SAN (not `DNSName` — using DNSName for an IP causes hostname mismatch, that's a gotcha).

proxy 会生成一个自签名 CA 证书装到你系统信任区，这样 Claude Code 走 HTTPS 就不会报 SSL 错误。CA 签发的 server 证书给 `127.0.0.1` 用，SAN 用的是 `IPAddress`（不是 `DNSName`，用 DNSName 会 hostname mismatch，这是个坑）。

---

## Troubleshooting / 踩坑记录

### SSL certificate hostname mismatch

The certificate SAN must use `IPAddress` for IP addresses, not `DNSName`. If you hit this, regenerate:

证书 SAN 里 IP 地址必须用 `IPAddress`，不能用 `DNSName`。遇到这个就重新生成：

```bash
python claude_deepseek_proxy.py --uninstall
python claude_deepseek_proxy.py --install
```

### Proxy not receiving requests / Proxy 收不到请求

Claude Code only connects to `https://` endpoints — it ignores `http://`. Make sure `ANTHROPIC_BASE_URL` is `https://127.0.0.1:PORT`.

Claude Code 只走 `https://`，`http://` 的 endpoint 它不理。确保 `ANTHROPIC_BASE_URL` 是 `https://127.0.0.1:PORT`。

### 400: thinking cannot be disabled when reasoning_effort is set

This is exactly what the proxy fixes. Make sure the proxy is running and `ANTHROPIC_BASE_URL` points to it.

这正是这个 proxy 解决的问题。确保 proxy 在跑，`ANTHROPIC_BASE_URL` 指向它。

---

## License

MIT
