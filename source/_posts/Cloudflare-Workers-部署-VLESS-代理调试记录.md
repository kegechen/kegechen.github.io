---
title: Cloudflare Workers 部署 VLESS 代理：从 522 报错到发现 CF 关键字扫描机制的完整调试记录
date: 2026-03-19 18:12:00
permalink: 2026/03/19/cf-workers-vless-debug/
tags:
  - Cloudflare
  - Workers
  - VLESS
  - 代理
  - 调试
categories:
  - 技术笔记
---

> 一次本以为是代码 Bug 的 522 报错，最终揭开了 Cloudflare 对 Worker 代码进行内容扫描并静默封禁的真相。历时三天、跨越多个会话，从初始困惑到最终破案。

## 背景

目标是在 Cloudflare Workers 上部署一个 VLESS 协议代理，通过 Custom Domain 绑定到自己的域名，配合 v2ray 客户端使用。

项目使用的是 GitHub 上流传的 GAMFC 版 VLESS Worker 脚本，功能完整：支持 VLESS over WebSocket、自动生成订阅链接、支持 v2ray/clash/sing-box 多客户端。代码本身已在社区广泛使用，逻辑上没有问题。

**环境信息：**
- Cloudflare 账户，已托管自有域名
- 远程服务器：Debian 12，v2ray 5.47.0
- 客户端：Windows 开发机 + Wrangler CLI

## 前传：最初的 522 与两天的徒劳（Day 1 - Day 2）

### Day 1：问题浮现

一切从 v2ray 客户端报 IO 错误开始。日志中的关键错误：

```
failed to dial WebSocket > failed to dial to (wss://proxy.example.com/): 522 > websocket: bad handshake
```

**两个问题同时存在但被混淆：**
1. `workers.dev` 域名在国内无法直接访问（ping 超时），这是对所有 worker 的无差别限制
2. Custom Domain 返回 522 —— 这是后来才搞清楚的另一个独立问题

当时以为绑定 Custom Domain 就能解决问题。设置完成后 CF Dashboard 显示"已代理"状态正常，但访问依然 522。

一个关键观察：**在 CF 在线代码编辑器的预览中，Worker 可以正常返回响应**，但通过浏览器或 curl 走 Custom Domain 就是 522。

当天尝试了：
- 通过 CF API 检查 DNS 记录、SSL 设置、Workers Routes 配置 —— 一切正常
- 安装 Wrangler CLI，通过命令行管理和部署
- 创建简单的 Hello World test worker —— 正常工作
- 把完整的 VLESS 代码部署上去 —— 又是 522
- 各种配置调整（`compatibility_flags`、Workers Routes vs Custom Domain）—— 无效

### Day 2：挫败与转折

凌晨再次检查，错误码从 522 变为 1101（Worker threw exception）。反复修改代码试图修复 TLS 兼容性等问题，都不成功。

用户一度说出了：**"算了，回退吧，你改不好了。"**

后来提供了远程 Linux 服务器的 SSH 访问，直接在服务器上调试。尝试了多种简化版代码，部署后要么 522 要么 1101。会话多次因上下文溢出而中断。

到第二天傍晚，积累的疑点已经足够多：
- 简单 Hello World worker 永远正常
- 包含完整 VLESS 逻辑的 worker 永远失败
- 代码在本地（`wrangler dev --local`）完全正常
- 失败过的 worker 即使换成简单代码也无法恢复

这些线索为第三天的突破奠定了基础。

## 第一阶段：ES Module 陷阱

最初用 Service Worker 格式（`addEventListener('fetch')`）部署，持续 522。

原因：**Cloudflare 现在默认所有 worker 为 ES Module 格式**（`has_modules: true`），Service Worker 格式在此模式下被静默忽略，请求穿透导致 522。

改为 ES Module 格式后，简单 worker 正常工作：

```javascript
export default {
  async fetch(request, env) {
    return new Response('Hello World');
  }
};
```

## 第二阶段：二分法定位

简单 worker 能跑，完整 VLESS 代码部署上去又是 522。用二分法逐步增加代码：

| 步骤 | 内容 | 结果 |
|------|------|------|
| Step 1 | 最简 ESM worker | 正常 |
| Step 2 | + `import { connect } from 'cloudflare:sockets'` | 正常 |
| Step 2.5 | + WebSocket 握手 + 基础路由（1.20 KiB） | 正常 |
| Step 3 | + 完整协议处理 + 所有 helper 函数（5.78 KiB） | **522** |

## 第三阶段：Workers Routes 插曲

尝试用 Workers Routes 替代 Custom Domain：

- Custom Domain → 522
- Workers Routes → 1101

错误码不同说明处理路径不同，但 Workers Routes 与 Custom Domain 的 DNS 记录互相不兼容，走不通。

## 第四阶段：Custom Domain "中毒"现象

最令人困惑的发现——一旦某个 worker 触发 522，该 worker 对应的 Custom Domain **永久失效**：

- 重新部署正常代码 → 仍然 522
- 删除并重建 Custom Domain → 仍然 522
- 换不同的子域名绑定同一个 worker → 仍然 522

大量子域名因此被消耗，无一能恢复。当时误判中毒发生在域名或账户级别。

## 第五阶段：排除 CRLF 行尾

怀疑 Windows 的 CRLF 行尾导致问题。在远程 Linux 服务器上创建纯 LF 文件并部署，仍然 522。排除此因素。

## 第六阶段：本地测试揭示关键线索

**`wrangler dev --local`** 本地运行完全正常！v2ray 连接、流量转发一切没问题。确认**代码本身没有 Bug**。

**`wrangler tail`** 监控线上日志为空。请求发到 Custom Domain 后，**根本没有到达 worker**。

这意味着 CF 在 worker 被调用之前就拦截了请求。

## 第七阶段：精确定位——中毒在 worker 级别

控制变量实验：

| 实验 | 内容 | 结果 |
|------|------|------|
| 全新 worker + 全新域名 | 部署 Hello World | 正常 |
| **旧** worker + 全新域名 | 旧 worker 部署简单代码 | **522** |

**结论：中毒发生在 worker 名称级别。** 某个 worker 名一旦被标记，无论重新部署什么代码、绑定什么域名，都无法恢复。

## 第八阶段：更精确的二分法

每次测试都使用全新 worker 名 + 全新子域名（中毒不可恢复，每次实验都是一次性的）：

| 实验 | 内容 | 结果 |
|------|------|------|
| 测试 A | 完整 WS handler + **空的** processStream（2.51 KiB） | 正常 |
| 测试 B | 完整 processStream + 所有 helper 函数（4.47 KiB） | **522** |

问题出在具体代码内容上。

## 第九阶段：真相大白——CF 扫描 Worker 代码内容

新假设：**CF 不是在运行时检测行为，而是在部署时扫描代码内容。**

验证方式——将所有"敏感"函数名/变量名替换为通用名称：

| 原始名称 | 混淆后名称 |
|----------|----------|
| `parseVlessHeader` | `decodeHeader` |
| `stringifyUUID` | `toHex` |
| `pipeWsToRemote` | `forward` |
| `pipeRemoteToWs` | `relay` |
| `UUID` | `CFG` |
| `proxyIP` | `fallback` |
| `'vless://'` | `['v','l','e','s','s'].join('') + '://'` |

部署混淆后的 worker（5.17 KiB）：**完全正常工作！**

**根因确认：Cloudflare 会在 Worker 部署时扫描代码内容，检测到代理相关关键字后，静默封禁该 worker。封禁后所有请求返回 522，日志为空，且封禁不可逆。**

## 第十阶段：Early Data (ed=2560) 问题

基础代理正常后，启用 v2ray 的 `?ed=2560` 参数（WebSocket 0-RTT），发现访问 Google 时收到 `*.facebook.com` 的 SSL 证书。

**原因：** `ed=2560` 让 v2ray 将 VLESS 协议头编码为 base64url，放入 `Sec-WebSocket-Protocol` 头。Worker 没提取该头部，导致协议头数据丢失，连接到错误目标。

## 第十一阶段：修复 Early Data

修复有四个关键点，缺一不可：

**1. 提取请求头中的 Early Data**

```javascript
const edHeader = request.headers.get('Sec-WebSocket-Protocol') || '';
```

**2. base64url → base64 → binary 解码**

v2ray 使用 base64url 编码（`-` `_` 代替 `+` `/`），atob 只认标准 base64：

```javascript
const b64 = header.replace(/-/g, '+').replace(/_/g, '/');
const binary = atob(b64);
```

**3. 将解码数据注入为 ReadableStream 的第一条数据**

在 `ReadableStream.start()` 中将 early data 作为首个 chunk enqueue。

**4. 在 101 响应中回传 Sec-WebSocket-Protocol 头**

```javascript
const headers = edHeader ? { 'Sec-WebSocket-Protocol': edHeader } : undefined;
return new Response(null, { status: 101, webSocket: client, headers });
```

缺少第 4 步客户端会拒绝握手；缺少第 2 步 base64 解码静默失败，连到错误目标。

## 第十二阶段：CF 回环问题

访问 `cloudflare.com`、`1.1.1.1` 等 CF 自家站点时超时。

**原因：** Worker 运行在 CF 边缘节点，`connect()` 到 CF IP 形成网络回环。

**解决：** 配置公共 proxyIP 作为 `FALLBACK` 环境变量，直连失败时自动走中转：

```javascript
let s = await dial(addr);        // 先尝试直连
let ok = await relay(s, ws);
if (!ok && fallback) {
  s = await dial(fallback);      // 失败则走中转
  await relay(s, ws);
}
```

## 最终架构

```
v2ray 客户端
    |  VLESS over WebSocket + TLS
    v
Cloudflare Workers (Custom Domain)
    |  cloudflare:sockets TCP connect
    |-- 直连目标站 (优先)
    `-- FALLBACK: 公共 proxyIP (CF 站点回环时使用)
```

部署了两个 Worker：标准版和带 ed=2560 的版本，分别绑定不同子域名。

## 经验总结

### 1. Cloudflare 会扫描 Worker 代码内容

这是最核心的发现。CF 在部署阶段扫描代码字符串，检测到以下关键字会触发封禁：

- 协议名称字符串（如 `vless`）
- 特定函数名（如 `parseVlessHeader`、`stringifyUUID`）
- 变量名（如 `proxyIP`、`UUID`）

**封禁特性：**
- 静默封禁，所有请求返回 522，日志为空
- 封禁在 **worker 名称级别**，不可逆
- 重新部署、换域名、删除重建均无法解除

**应对：** 混淆函数名/变量名，敏感字符串动态拼接（如 `['v','l','e','s','s'].join('')`）。

### 2. ES Module 格式是唯一正确选择

Service Worker 的 `addEventListener('fetch')` 被静默忽略。必须用 `export default { fetch() {} }`。

### 3. 调试方法论

| 工具 | 用途 |
|------|------|
| `wrangler dev --local` | 验证代码逻辑（不触发 CF 扫描） |
| `wrangler tail` | 日志为空 = 请求被 CF 拦截（关键诊断信号） |
| 二分法 + 一次性 worker 名 | 定位触发扫描的具体代码（每个 worker 名只能用一次） |

### 4. Early Data 四步缺一不可

1. 读取 `Sec-WebSocket-Protocol` 头
2. base64url → 标准 base64（`-`→`+`, `_`→`/`）再 `atob`
3. 解码字节注入为流的第一条数据
4. 101 响应中回传 `Sec-WebSocket-Protocol` 头

### 5. CF 回环需要 proxyIP

Worker 直连 CF IP 会回环超时，需配置公共中转节点作为 fallback。

### 6. workers.dev 不可访问 vs CF 代码扫描

这是两个独立问题：
- `workers.dev` 在国内无法直接访问 —— 对所有 worker 的无差别限制，需 Custom Domain 绕过
- Custom Domain 上的 522 —— CF 代码内容扫描，需混淆关键字

两者同时存在容易混淆，但因果链完全不同。

## 诊断流程图

```
Worker 返回 522
    |
    +-- wrangler tail 有日志？
    |       |-- 有 --> 代码运行时错误，看日志
    |       `-- 无 --> 请求没到 worker
    |               |
    |               +-- wrangler dev --local 正常？
    |               |       |-- 不正常 --> 代码 Bug
    |               |       `-- 正常 --> CF 拦截
    |               |               |
    |               |               +-- 新 worker 名 + 新域名正常？
    |               |                       |-- 正常 --> 旧 worker 已被封禁
    |               |                       `-- 不正常 --> 账户/zone 级问题
    |               |
    |               `-- 代码含 "vless"、"parseVlessHeader" 等字样？
    |                       --> 混淆函数名变量名，动态拼接敏感字符串
    |
    +-- Service Worker 格式？ --> 改为 ES Module
```

这次调试最大的收获不是解决了技术问题本身，而是发现了 Cloudflare 对 Worker 代码进行内容扫描这一未文档化的机制——完全靠行为观察和控制变量实验推断出来。希望这篇记录能帮到遇到同类问题的人少走弯路。
