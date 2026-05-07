---
title: 给 lan-mouse 加上 uinput 后端 + Win11 风格越界停留：dde-kwin Wayland 下的软 KVM 折腾记
date: 2026-05-07 16:00:00
permalink: 2026/05/07/lan-mouse-dde-kwin-uinput-stickycorners/
tags:
  - lan-mouse
  - KVM
  - Rust
  - Wayland
  - dde-kwin
  - uinput
  - Win32
  - 剪贴板
categories:
  - 技术笔记
---

> 把一台 ARM64 UOS（deepin v23 / dde-kwin 5.15 / Wayland）和一台 Windows 11 通过热点连起来，用同一套鼠标键盘办公。Deskflow 在 Qt6 缺失下装不上，Synergy 收费，RustDesk 是远程桌面不是软 KVM。最后选了开源 Rust 项目 [lan-mouse](https://github.com/feschber/lan-mouse) v0.10，但发现它在 dde-kwin Wayland 上完全推不动光标——只好动手改源码。本文记录从 0 到能用的全部踩坑：uinput 后端、Win11 风格的 sticky-corners 越界视觉、双向剪贴板同步守护进程，以及 Wayland 协议层无法绕开的限制。
>
> Fork: <https://github.com/kegechen/lan-mouse/tree/feature/dde-kwin-uinput-stickycorners>

<!-- more -->

## 背景：为什么不用现成方案

需求很朴素：

- Windows 主机 + UOS ARM64 副屏（开发机），鼠标越过屏幕边缘自动切到对端
- 两端共享文本剪贴板
- 不要 RDP / VNC 那种全屏远程桌面，副屏是物理显示器，不是窗口
- 不要 Tailscale 这种带 UI 托盘的"账号入口"产品，越无感越好
- 网络通过 Windows 个人热点

候选评估：

| 方案 | 结论 |
|------|------|
| Deskflow（Synergy 开源继任者） | 依赖 Qt6，UOS deepin v23 仓库里 Qt6 缺失，编译装一套又风险高 |
| Synergy 1 商业版 | 收费 + ARM64 Linux 支持稀烂 |
| Barrier | 已停止维护，输入安全协议有坑 |
| RustDesk | 是远程桌面，不是 KVM，副屏不需要镜像 |
| **lan-mouse v0.10**（Rust） | 设计目标对口、开源、纯命令行，第一选择 |

lan-mouse 的设计很优雅：捕获端（Windows）用低层钩子拦下"越过屏幕边缘"的鼠标和键盘事件，通过局域网 UDP 发到模拟端（Linux），由对端注入到内核。第一次跑就 ssh 上 UOS 启动，Windows 客户端连上，鼠标过界——

**没动**。

## 问题一：Wayland 下 X11 后端推不动光标

lan-mouse 0.10 在 Linux 上默认走 X11 backend：通过 XTest 扩展把模拟事件投到 X server。但 dde-kwin 5.15 跑的是 Wayland session，`DISPLAY` 指向的是 Xwayland——一个 X11 兼容层。

XTest 注入的事件确实到了 Xwayland，但只在 Xwayland 维护的虚拟光标状态里转了一圈，**没穿透到 wayland compositor 的真实指针**。表现就是：lan-mouse 日志一切正常、`xdotool getmouselocation` 显示坐标在变，但屏幕上的光标纹丝不动。

试过的死路：

- `wl-data-control` / `virtual-keyboard-manager` Wayland 协议：dde-kwin 5.15 不支持
- `libei` (Emulated Input)：需要 compositor 主动 grant 端口，dde-kwin 这一代没接入
- `ydotool`：本质也是 uinput，但守护进程跑得不太行

正路只剩一条：**绕开 Wayland，直接走 Linux 内核的 `/dev/uinput`**。uinput 是内核虚拟输入设备接口，从内核视角创建一个"假键盘鼠标"，事件经由 evdev → libinput → compositor，和真实硬件路径一致——dde-kwin 没法拒绝。

### 实现：新增 uinput emulation 后端

lan-mouse 的 `input-emulation` crate 已经把后端抽象成 trait，只要新加一个 `UinputEmulation` 实现 `Emulation` 即可。核心代码大约 150 行：

```rust
// input-emulation/src/uinput.rs
pub(crate) struct UinputEmulation {
    device: uinput::Device,
}

impl UinputEmulation {
    pub(crate) fn new() -> Result<Self, UinputEmulationCreationError> {
        let device = uinput::default()?
            .name("lan-mouse-uinput")?
            .event(controller::Mouse::All)?
            .event(controller::Mouse::Left)?
            .event(controller::Mouse::Right)?
            .event(controller::Mouse::Middle)?
            .event(controller::Mouse::Side)?
            .event(controller::Mouse::Extra)?
            .event(keyboard::Keyboard::All)?
            .event(relative::Position::X)?
            .event(relative::Position::Y)?
            .event(relative::Wheel::Vertical)?
            .event(relative::Wheel::Horizontal)?
            .create()?;
        Ok(Self { device })
    }

    fn motion(&mut self, dx: i32, dy: i32) {
        if dx != 0 {
            let _ = self.device.write(EV_REL as i32, REL_X as i32, dx);
        }
        if dy != 0 {
            let _ = self.device.write(EV_REL as i32, REL_Y as i32, dy);
        }
        let _ = self.device.synchronize();
    }
    // button / scroll / key 同理
}
```

然后在 `input-emulation/src/lib.rs` 里把 uinput 挂进 `Backend` 枚举，并把它放到自动探测优先级第一位：

```rust
pub enum Backend {
    Uinput,    // 新增 - 直接走内核
    Wlroots,   // wlr-data-control（dde-kwin 不支持）
    Libei,     // 需要 compositor grant
    X11,       // Xwayland 假转发
    Xdg,
    Remote(...),
}

// 自动探测时
const PRIORITY: &[Backend] = &[
    Backend::Uinput,   // <- 优先
    Backend::Libei,
    Backend::Wlroots,
    Backend::X11,
    Backend::Xdg,
];
```

`Cargo.toml` 加 feature 门：

```toml
[features]
uinput_emulation = ["input-emulation/uinput"]
```

构建：`cargo build --release --no-default-features --features=uinput_emulation`。运行需要 `/dev/uinput` 写权限——一次性 udev 规则解决：

```
# /etc/udev/rules.d/99-uinput.rules
KERNEL=="uinput", GROUP="input", MODE="0660", OPTIONS+="static_node=uinput"
```

把用户加 input 组，重新登录。鼠标终于过去了。

## 问题二：边缘没有"阻塞"，误触一次就找不回来

光标能越界后，新的体验问题来了：物理鼠标只要靠近屏幕右边缘瞬间就触发跳转。打字、拖拽、关闭右侧窗口——任何贴边动作都会把光标弹到对端。

更糟的是，跳过去的瞬间 Windows 的低层钩子停止处理本机鼠标，**对端没有连接成功的话光标就回不来了**——只能 ssh 重启 lan-mouse。

Windows 11 / macOS 处理这种情况都用 **sticky corners**：贴边后停留 N 毫秒才跳转。我抄了这个体验，但加了视觉提示，免得用户不知道在等什么。

### 设计：sticky corners + V 形指针

放在 `input-capture/src/windows.rs`：

```rust
// 触发条件：低层鼠标钩子里探测到光标贴边
//   PENDING_BARRIER: Option<(Position, (i32,i32), Instant)>
//
// 双路径检查（关键）：
//   路径 A: 鼠标移动事件回调里检查 elapsed >= dwell_ms
//   路径 B: SetTimer 触发的 WM_TIMER 兜底（用户停在边上不动时）
//
// 视觉：在贴边屏幕上画一个矢量 V 形箭头，指向对端方向
//   - 用 GDI Polygon + 双缓冲，不闪
//   - WS_EX_LAYERED + LWA_COLORKEY 抠掉背景，鼠标穿透
//   - LoadCursorW(IDC_ARROW) 防止 hCursor=NULL 被系统画成 BUSY
//
// 越界瞬间：
//   - SetCursor(NULL) + ShowCursor(FALSE) 隐藏本机光标
//   - 释放热键时 ShowCursor(TRUE) 还原
```

#### 坑 1：`SetTimer` 触发饥饿

最初只用 `SetTimer(hwnd, 1, dwell_ms, NULL)` 等贴边时间，结果发现：用户保持鼠标贴边但**有微小抖动**时，dwell timer 永远不到——因为 `WM_TIMER` 的优先级在消息队列里低于 `WM_MOUSEMOVE`，连续移动事件持续触发会把 timer 饿死。

解法：双路径检查。在低层鼠标钩子的回调里，每次都对比 `Instant::now() - PENDING_BARRIER.start_at` 和 `dwell_ms`，达到阈值立刻触发越界，不依赖 timer。timer 只用作"用户停下不动"的兜底路径。

#### 坑 2：指示器闪烁

第一版 V 形箭头窗口直接 `BeginPaint` + `Polygon` 画，看起来一直在抖。原因是 layered window 的 paint 默认走系统背景色擦除——magenta colorkey 被擦出来一瞬间才被 polygon 覆盖。

解法：**GDI 双缓冲**。`CreateCompatibleDC` 创个内存 DC，先画到内存位图，然后一次性 `BitBlt` 到屏幕。配合 `WM_ERASEBKGND` 返回 1 跳过系统擦除。

#### 坑 3：`prev_pos` clamp panic

低层钩子拿到的 `prev_pos` 可能是上一帧屏幕外的负坐标（Windows 多屏 + 缩放下常见），`clamp_to_display_bounds` 直接 panic。安全 fallback：prev 不在任何 display 内时回落用 `curr_pos`。

#### 坑 4：演示模式触发剪贴板"句柄无效"

加了一个 `LAN_MOUSE_INDICATOR_DEMO=1` 环境变量，把 V 形画成醒目红用于截图调试。结果发现 demo 模式下 PowerShell `Get-Clipboard` 报"句柄无效"——`hide_cursor` 时去 `GetDesktopWindow` 拿不到合法句柄。

解法：demo 模式下跳过 `hide_cursor`，反正只是看视觉，不需要光标真的消失。

最后默认值是 `dwell_ms=300`，配合 V 形指针：贴边 → 300ms 内出现指向对端的红色 V → 越界 → 本机光标隐藏。设 `dwell_ms=0` 直接关闭整套机制，老实人模式。

## 问题三：剪贴板：Wayland 协议层的硬限制

光标键盘搞定了，剪贴板更头疼。

### 选型

- 复用 lan-mouse 的连接通道：lan-mouse 0.10 的协议没有为 file/clipboard 留位，改协议要动太多东西
- 走另一个端口的独立 daemon：解耦、好测、出 bug 不影响主链路

选后者，命名 `clipsync`，~280 行 Rust。

### 协议：TLV + FNV-1a 防回环

```
[u8 type][u32 BE length][N bytes payload]

type:
  0x01 = Text(UTF-8)
  0x02 = Image(预留)
  0x03 = Files(预留)
```

防止双向同步形成回环（A 写 → B 收到 → B 写到本地 → 钩子回 A → ...）：每端都维护一个 `latest_received_hash`，本地剪贴板新内容计算 FNV-1a 64 位 hash，等于 latest 就跳过发送。

```rust
fn fnv1a64(bytes: &[u8]) -> u64 {
    let mut h: u64 = 0xcbf29ce484222325;
    for b in bytes {
        h ^= *b as u64;
        h = h.wrapping_mul(0x100000001b3);
    }
    h
}
```

500ms poll 间隔，够低延迟也不会爆 CPU。

### 平台读写

- **Windows**：`arboard` crate，Win32 clipboard API 一把梭
- **Linux 写**：双写 `xclip -selection clipboard` + `wl-copy`，让 Xwayland 应用和原生 Wayland 应用都能粘到
- **Linux 读**：优先 `wl-paste`，失败回退 `xclip -selection clipboard -o`

### Wayland 协议层：读不到 Qt Wayland 应用的剪贴板

最坑的一节。`wl-paste` 在 dde-kwin 上可以读到 X11 应用（Xwayland 路径）和原生 wayland-native 应用（gtk 系）的剪贴板，但**读不到 Qt6 wayland 应用**（如 deepin-editor）。

原因：Wayland 的剪贴板协议（`wl_data_device`）规定，**只有持有键盘焦点的客户端才能读 selection**。`wl-paste` 是个无窗口客户端，永远拿不到键盘焦点，dde-kwin 的实现严格遵守了协议——除非通过 `wlr-data-control` 协议绕过，但 dde-kwin 5.15 不支持该扩展。

GTK 系应用没踩这个坑是因为 dde-kwin 内部对它们走了 X11 桥兼容路径（gtk wayland backend 会主动 register 一个 fake input region）。Qt6 wayland backend 严格按协议来，反而被卡住。

最终解法是**承认限制 + 文档化 workaround**：

```bash
# 让特定 Qt 应用强制走 xcb（X11）后端，绕开 Wayland 限制
QT_QPA_PLATFORM=xcb deepin-editor
```

把这条写进了 `USAGE.md` 的"Wayland 限制"章节。Windows → UOS 单向完整可用，UOS → Windows 在 X11 应用 + Qt-xcb 应用上完整可用，原生 Qt-Wayland 应用需要用户主动配 `QT_QPA_PLATFORM=xcb`。

### 坑：tokio current_thread 死锁

clipsync 一开始写 `#[tokio::main(flavor = "current_thread")]`，跑起来发现剪贴板一变化整个进程就 hang。原因：`std::process::Command::output()` 是同步阻塞 syscall，把单线程 runtime 直接堵死。

解法：换成 `flavor = "multi_thread", worker_threads = 2`，或者用 `tokio::process::Command`。前者更稳，因为 arboard / xclip / wl-paste 链路里还有别的隐式 blocking。

## 问题四：连接编排脚本

每次手动 `ssh uos systemctl --user start lan-mouse`、本地启 `lan-mouse-gui --no-gui`、检查 `clipsync` 跑没跑——太烦。写了个 PowerShell 一键脚本 `connect.ps1`：

- 状态机：**未安装 / 已安装未运行 / 运行中**，对每种状态做不同动作
- 首次运行：交互式向导问对端 IP / 用户名 / 上下左右方位
- ssh stdin 用 `ProcessStartInfo` 显式管理，避免 PowerShell `$str | & ssh.exe` 不关闭 stdin 导致挂住
- 支持参数：`-DwellMs 300`、`-IndicatorDemo`、`-RestartUosService`、`-Reconfigure`

最后还写了个 `build.ps1`，clone 完直接跑：

- 自动安装 rustup（如果没有）
- 写 `~/.cargo/config.toml` 配 rsproxy.cn 镜像 + Windows 主机代理
- 编译 lan-mouse + clipsync
- 部署 binary + 脚本到 `D:\tools\lan-mouse\`，同时打包 patches/ 归档

整个体验现在是：

```
git clone -b feature/dde-kwin-uinput-stickycorners https://github.com/kegechen/lan-mouse
cd lan-mouse/contrib/dde-kwin-uos
.\build.ps1
cd D:\tools\lan-mouse
.\connect.ps1   # 首次问向导，之后一条命令直接连
```

## 总结

整套方案的关键决策：

1. **uinput 而不是 X11/Wayland 协议**：在 dde-kwin 这种"半官方"compositor 上，绕开协议层走内核是最稳的路径，副作用是要配 udev 权限
2. **sticky-corners 双路径检查**：Win32 消息队列优先级会让 timer 饿死，鼠标移动钩子里同步检查 elapsed 才靠谱
3. **clipsync 独立守护进程**：不动 lan-mouse 协议，单独端口、TLV、FNV-1a hash 防环
4. **Wayland 剪贴板硬限制不要硬刚**：wl-paste 读不到 Qt-Wayland 是协议规定，工程上不如配文档让用户给特定应用切 xcb

完整 patch 在 fork：

- 仓库：<https://github.com/kegechen/lan-mouse>
- 分支：`feature/dde-kwin-uinput-stickycorners`
- 关键提交：`0c0be3f`（uinput 后端）、`5ed118a`（sticky corners + V 形指针）、`607499e`（contrib/dde-kwin-uos 部署套件）、`a14a5fc`（build.ps1 一键化）

`contrib/dde-kwin-uos/` 目录是自包含部署套件，包含 `README.md`、`USAGE.md`、`connect.ps1`、`build.ps1` 和 `clipsync/` 完整源码。下次换台机器或者重装系统，clone 跑 build 就能用。
