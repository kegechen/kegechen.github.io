---
title: Claude Code && OpenClaw 初体验
date: 2026-02-25 19:30:00
permalink: 2026/02/25/claude-openclaw-guide/
tags:
  - OpenClaw
  - Claude
  - AI
  - 教程
categories:
  - 技术笔记
---

## 前言

最近体验了一下 Anthropic 的 Claude Code 和 OpenClaw，过程中踩了一些坑，记录下来分享给大家。OpenClaw 是一个开源的 Agent 框架，可以接入各种 AI 模型（Claude、DeepSeek、Kimi 等），并支持在飞书、Telegram 等平台使用。

## 准备工作

### 1. 配置 npm 镜像（可选）

如果 npm 安装速度慢或报错，可以使用国内镜像：

```bash
npm config set registry https://registry.npmmirror.com
```

### 2. 安装 Git

确保已安装 Git，OpenClaw 需要使用 Git 进行版本控制。

### 3. 安装 Node.js

**重点：不要下载最新的 v24.x 版本！**

OpenClaw 要求 Node >= 22.12.0，但最新版本的 Node.js（v24.x）会导致 OpenClaw 插件安装失败（经典的 `[openclaw] Failed to start CLI: Error: spawn EINVAL` 错误）。

建议使用 nvm（Node Version Manager）来安装和管理 Node.js 版本：

#### Windows 上安装 nvm

1. 访问 [nvm-windows Releases](https://github.com/coreybutler/nvm-windows/releases)
2. 下载最新版本的 `nvm-setup.exe`（例如 `nvm-setup-1.1.12.exe`）
3. 双击运行安装程序，按照提示完成安装
4. **重要：安装完成后，关闭并重新打开终端（PowerShell 或 CMD）**

5. 验证 nvm 安装是否成功：

```bash
nvm version
```

**如果遇到 `nvm` 命令无法识别的问题**，请尝试以下解决方案：

**问题：PowerShell 执行策略限制**

在某些 Windows 系统上，PowerShell 默认执行策略可能限制脚本运行，导致 nvm 命令无法识别。解决方法：

```powershell
# 设置 PowerShell 执行策略为 RemoteSigned（允许本地脚本运行）
Set-ExecutionPolicy RemoteSigned -Scope CurrentUser
```

执行后，重新打开终端并再次验证：

```bash
nvm version
```

#### 安装 Node.js 22 LTS 版本

```bash
# 安装 Node.js 22 LTS 版本
nvm install 22

# 切换到 Node.js 22
nvm use 22

# 设置默认版本（每次打开终端自动使用此版本）
nvm alias default 22
```

验证安装：

```bash
node -v
# 应该显示 v22.x.x
```

## 安装 Claude Code

### 方式一：使用官方版本

```bash
npm install -g @anthropic-ai/claude-code
```

注册和使用说明：
- 注册页面：https://www.aicodemirror.com/register?invitecode=QKVHMJ
- 需要配置 Anthropic API Key

### 方式二：使用国产模型

考虑到成本和访问速度，可以使用国产 AI 模型。阮一峰老师有详细的教程：

> [豆包 Seed Code：国产版 Claude Code](https://www.ruanyifeng.com/blog/2025/11/doubao-seed-code.html)

**推荐方案：方舟 Coding Plan**

方舟 Coding Plan 支持多种国产模型：
- Doubao（豆包）
- GLM
- DeepSeek（深度求索）
- Kimi（月之暗面）

价格：8.9 元/月，性价比很高

注册链接：https://volcengine.com/L/FrblCtDSpcE/

## 安装 OpenClaw 接入飞书

OpenClaw 是一个功能强大的 Agent 框架，支持：
- 多种 AI 模型
- 多平台接入（飞书、Telegram、Discord 等）
- 丰富的技能和插件
- 本地文件操作、代码编写、任务调度等

### 安装步骤

```bash
# 安装 OpenClaw
npm install -g openclaw

# 初始化配置
openclaw init

# 配置飞书接入（需要飞书企业自建应用）
openclaw gateway start
```

### 常见问题：插件安装失败

如果你遇到以下错误：

```
[openclaw] Failed to start CLI: Error: spawn EINVAL
```

**原因：** Node.js 版本问题。最新版本的 Node.js（v24.x）与某些原生模块不兼容，导致插件安装失败。

**解决方案：** 降级到 Node.js 22 LTS 版本

```bash
nvm install 22
nvm use 22
nvm alias default 22

# 重新安装 OpenClaw
npm uninstall -g openclaw
npm install -g openclaw
```

参考文章：[OpenClaw 安装踩坑记录](https://cloud.tencent.com/developer/article/2626160)

## 后续使用

安装完成后，你就可以在飞书中与 OpenClaw 进行交互了。它可以帮你：

- 📝 写文章、写代码
- 🔍 搜索信息、整理资料
- 📅 管理日程、发送提醒
- 🎯 自动化任务、定时执行
- 🤖 编写自定义技能

## 总结

OpenClaw + Claude（或国产模型）是一个非常强大的 AI Agent 组合，特别适合需要本地文件操作、任务自动化等场景。安装过程中的主要坑点就是 Node.js 版本选择，记得使用 v22 LTS 版本，不要用最新的 v24.x。

希望这篇分享能帮助到大家！

## 参考链接

- [Claude Code 官方文档](https://docs.anthropic.com/claude-code)
- [OpenClaw GitHub](https://github.com/openclaw/openclaw)
- [阮一峰：豆包 Seed Code](https://www.ruanyifeng.com/blog/2025/11/doubao-seed-code.html)
- [方舟 Coding Plan](https://volcengine.com/L/FrblCtDSpcE/)
- [OpenClaw 安装踩坑记录](https://cloud.tencent.com/developer/article/2626160)
