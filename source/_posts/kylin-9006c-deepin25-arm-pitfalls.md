---
title: 在 ARM 笔记本上给 deepin 25 换心不换骨：麒麟 9006C 驱动兼容踩坑全记录
date: 2026-11-25 10:30:00
updated: 2026-11-25 14:00:00
permalink: 2026/11/25/kylin-9006c-deepin25-arm-pitfalls/
tags:
  - deepin 25
  - ARM64
  - 麒麟 9006C
  - treeland
  - Mali
  - Wayland
  - glvnd
  - NetworkManager
  - Linux 内核
categories:
  - 技术笔记
---

> 一台华为擎云 L540（麒麟 9006C / ARM64）出厂是 UOS。想装 deepin 25，ISO 启动后却看不到硬盘、点不亮屏幕。最后我们把 **deepin 25 的用户态装进 Rootb，沿用 UOS 的厂商内核和 initrd**，一路打通到 ddm + treeland 桌面；随后又修掉了 NetworkManager 100% CPU、控制中心卡死、黑屏、500% CPU 鼠标卡顿、动态壁纸选不上等问题，补上了厂商 Mali 的私有 Wayland 协议让系统应用也用上 GPU，最后写了个自检脚本，防止这些补丁被 apt 升级悄悄覆盖。
>
> 这篇不是“一键安装教程”，而是一份排障笔记：每个症状背后是什么、怎么验证、怎么修、哪些做法**不能**照搬。对任何“通用发行版 ISO + 厂商私有驱动 ARM 机器”都有参考价值。
>
> 文中的垫片源码、drop-in、脚本和说明都整理在仓库 **[kegechen/kirin9006c-deepin25](https://github.com/kegechen/kirin9006c-deepin25)**（厂商闭源组件不在仓库里，由脚本从本机 UOS 分区复制）。

<!-- more -->

## 一、先说结论：这台机器为什么不能常规安装

擎云 L540-001 的 SoC 是麒麟 9006C（`hisilicon,baltimore`）。厂商内核是 `5.4.96-arm64-desktop`，关键的 UFS（`ufshcd-kirin`）、显示（`hisi-drm`）、GPU（`mali_kbase`，Mali-G78 22 核）、Hi1103 Wi‑Fi、声卡基本都**内建在厂商内核里**。

deepin 25 的 ARM64 ISO 自带 6.6 通用内核：启动后找不到 UFS、framebuffer 也不工作，安装程序自然走不下去。把厂商的 `.ko` 直接搬到通用内核也不现实——ABI、设备树、私有固件、内核内建初始化流程全都不一样。

所以最终方案是“**换用户态，不换内核**”：

1. 在原有 UOS 的 Rootb 分区里装 deepin 25 的根文件系统；
2. 沿用 UOS 的内核、initrd、固件和 `/lib/modules`；
3. 通过一次性 `grub-reboot` 进入 deepin；默认启动项仍是 Roota 的 UOS，出问题时断电/重启就回原系统。

> 这条经验最重要：厂商闭源驱动强绑定的 ARM 机器，别跟 ISO 死磕，先判断能不能保留厂商内核。

## 二、坑 1：systemd 启动后 execve 全部卡住——elfverify LSM

**症状**：内核和 initrd 都起来了，根文件系统也挂上了，但 systemd 启动过程中每次 exec 都像卡死，服务大面积起不来。

**根因**：厂商内核内建了 `elfverify` LSM。UOS 有 deepin-elf-verify 守护进程负责关掉它；deepin 25 没有这个组件，于是**每一次 execve 都被验证逻辑堵住**。

**修复**：在 `init=` 指向的自定义脚本里，在 `exec systemd` 之前写入：

```sh
printf 0 > /sys/kernel/security/elfverify/status
exec /usr/lib/systemd/systemd "$@"
```

注意必须是 `printf`，不能用 `echo 0`——多出来的换行会让内核解析失败。这是整套方案的第一块基石：如果 exec 都卡死，后面什么都别谈。

## 三、坑 2：Wi‑Fi 连上但全网卡死——Hi1103 的三个私有行为

这台机器的 Wi‑Fi 不是普通的标准网卡，调试时踩了三个互相叠加的坑。

### 3.1 p2p 虚拟接口把驱动卡死

deepin 25 自带的 wpa_supplicant 2.10 会创建 `p2p-dev-wlan0`。Hi1103 驱动碰上这个接口会卡死。

**修复**：改用 UOS 出厂的 wpa_supplicant（带 `+hisi` 补丁），把它放在 `/usr/local/sbin/wpa_supplicant-hisi`，UOS 依赖的 OpenSSL 1.1 放在 `/usr/local/lib/hisi-wpa`，再用 systemd drop-in 替换 `wpa_supplicant.service`。

### 3.2 nl80211 station dump 永远不返回

即使 Wi‑Fi 连上了，NetworkManager 主线程也会 100% CPU：`nmcli` 显示“未运行”，D-Bus 假死，控制中心点开就没响应。

抓包/系统调用跟踪后会发现，Hi1103 对 `NL80211_CMD_GET_STATION + NLM_F_DUMP` 的请求**不会终止**。NM 1.44 的 device 刷新逻辑就在等这个 dump。

**修复**：写一个 LD_PRELOAD 垫片（`/usr/local/lib/hisi-nmfix/libnm-station-fix.so`），拦截 `sendmsg`：识别 generic netlink 上的 `GET_STATION | NLM_F_DUMP`，改成单次 `GET_STATION`，补上 `NLM_F_ACK` 和从 `SIOCGIWAP` 拿到的 BSSID。通过 `NetworkManager.service.d/hisi-station-fix.conf` 只给 NetworkManager 加载。

关键是**只给需要的进程加载**，不要全局注入。

### 3.3 热点 DNS 代理不响应

现象很像“没网”：能 ping 通网关和公网 IP，但浏览器报 DNS 错误，控制中心下载动态壁纸报：

```text
getaddrinfo (16: Device or resource busy)
```

根因不在 ARM：Windows 热点（ICS / SharedAccess）的 DNS 代理只回应某个虚拟网卡，对热点接口不回应。只开关热点没用，必须管理员 `Restart-Service SharedAccess -Force`，再重新打开热点。

不想每次都去 Windows 上折腾的话，可以在 ARM 端让这个连接不用 DHCP 下发的 DNS，并给 systemd-resolved 配一组兜底：

```sh
nmcli connection modify <连接名> ipv4.ignore-auto-dns yes ipv4.dns "<可达的 DNS 1>,223.5.5.5"
# /etc/systemd/resolved.conf.d/fallback-dns.conf
# [Resolve]
# DNS=<可达的 DNS 1> 223.5.5.5
```

顺带一个小坑：`nmcli device reapply wlan0` 在 Hi1103 上会因为 `802-11-wireless.powersave` 属性报错，改完直接 `resolvectl dns wlan0 ...` 临时生效即可。

**教训**：`getaddrinfo (16)` 这种错误不要只看应用层，先测 `dig @网关` 和 `@223.5.5.5` 区分 NAT 与 DNS 故障。

## 四、坑 3：控制中心卡死、X11 黑屏——会话层选错

一开始沿用 lightdm + kwin_x11。dde-session-daemon 的 display 模块会因为 hisi-drm 的 “scaling mode” 属性**除零 panic**，Display1 等一堆 D-Bus 服务缺失，控制中心自然卡死；Xwayland/glamor 路径也反复 `DRI2: failed to create screen`。

deepin 25 本来就是 treeland 的 Wayland 会话，于是改成：

- display-manager 指向 `ddm.service`；
- lightdm disable；
- 老的 `/etc/X11/default-display-manager` 备份后不再使用；
- 用 `treeland` 的 wlroots 合成器。

**X11 这条路上的黑屏不要继续投入时间**：厂商 DRM 驱动和通用发行版 Xorg/glamor 的兼容成本太高，直接走发行版原生的 Wayland 会话更省事。

## 五、坑 4：treeland 起不来——实时调度和 DRM render 节点

到了 treeland 又有两处不兼容。

### 5.1 `CPUSchedulingPolicy=rr` 启动失败

treeland 原单元想用 SCHED_RR，但厂商内核开了 `RT_GROUP_SCHED`，普通用户服务拿不到这个策略，直接 EPERM。

**修复**：drop-in 里清空：

```ini
[Service]
CPUSchedulingPolicy=
```

### 5.2 renderD128 不是 GPU

`/dev/dri/renderD128` 看起来像 GPU 节点，实际只是 hisi 显示控制器；deepin 的 Mesa 又没有 Mali-G78 驱动。wlroots 默认用 render node 会黑屏，或者 `mmap` 报 EACCES/SEGV。

**修复**：`libtl-norender.so` 垫片让 `drmGetRenderDeviceNameFromFd()` 返回 NULL，逼 wlroots 用 `card0`；同时给 `drmPrimeHandleToFD()` 导出的 fd 补上 `O_RDWR`（`DRM_RDWR == O_RDWR`）。

这一阶段可以显示，但 CPU 爆炸——见下一节。

## 六、坑 5：500% CPU、鼠标卡顿——其实是 llvmpipe 软件渲染

`top` 里 treeland 占 500%，进程里能看到 8 个 `llvmpipe-*` 线程。所谓“GPU 加速”根本没发生：Mesa 不认识 Mali-G78，wlroots 退化成软件光栅化。

### 6.1 从 UOS 借厂商 libmali

UOS 里有闭源 `libmali.so.0.23.0`（Valhall r23p0-01rel0，EGL 1.4 + GLES 3.2，自带 gbm）。复制到 `/usr/local/lib/hisi-mali`，**不要放进全局 ld.so 搜索路径**，先只在 treeland 的 drop-in 里给合成器用（客户端怎么接入见第七节）：

```ini
Environment=LD_LIBRARY_PATH=/usr/local/lib/hisi-mali
Environment="LD_PRELOAD=/usr/local/lib/hisi-mali/libmali-compat.so /usr/local/lib/hisi-mali/libmali.so.0 /usr/local/lib/hisi-treeland/libtl-norender.so"
# 后来又加了 libtl-malibuf.so，给客户端提供 mali_buffer_sharing，见第七节
```

验证数据：

- `gbm_create_device(card0)` 成功，backend `armsoc`；
- EGL vendor `ARM`，GL renderer `Mali-G78`；
- 2160×1440 全屏 8 层混合，约 3.94 ms/帧；
- 切到 GPU 后 treeland CPU 从约 500% 降到 3–6%，鼠标不再卡。

### 6.2 libmali-compat 要补什么

厂商库是按 UOS 用户态编译的，放到 deepin 25 会缺这些：

1. `wl_proxy_ref/unref/get_refcount`：UOS 魔改 libwayland-client 的符号，上游没有；treeland 是服务端，最初给空实现就够了，后来客户端也要用 libmali，才改成按 libwayland 私有布局实现的真实引用计数；
2. `gbm_bo_get_fd_for_plane()`：单平面时用 `gbm_bo_get_fd()` 代替；
3. `eglCreateImage/eglDestroyImage`：libmali 只有 KHR 版本，做一次 EGLAttrib → EGLint 转换；
4. `GL_EXT_unpack_subimage`：ES3 里是核心功能，但 Mali 不在扩展串里宣告，wlroots 的 GLES2 渲染器强制检查，需要补上。

还有一个很隐蔽的坑：如果 `LD_PRELOAD/LD_LIBRARY_PATH` 一直往下传，Xwayland、壁纸工厂等子进程也会误用 libmali，而它们依赖标准 libwayland。垫片在**真正的 `/usr/bin/treeland` 本体**里清理自己的环境变量——不能早清理，否则中间的 `env`、`sh`、脚本进程传不下去。

### 6.3 调试时千万别用 root 跑 treeland 二进制

即使只是 `treeland --version`：正在运行的会话会收到 ddm 断开、丢 DRM master，dock（`dde-shell@DDE`）随之 SIGSEGV 并触发 start-limit。恢复：

```sh
systemctl --user reset-failed dde-shell@DDE.service
systemctl --user start dde-shell@DDE.service
```

### 6.4 两个远程调试的小习惯

- **测试进程用完就收**：我在桌面上跑过一个 Mesa 版的 `qml` 测试窗口，忘了关，它在后台吃了 400%+ CPU，后面所有“谁占 CPU”的数据都被它带偏了。
- **`pkill -f` 小心匹配到自己**：`ssh host "pkill -f gltest.qml"` 会把远端那个命令行里也带 `gltest.qml` 的 `bash -c` 一起杀掉，ssh 直接返回 255。写成 `pkill -f '[g]ltest.qml'` 就不会匹配自己。

## 七、让系统应用也用上 GPU：补上厂商私有协议

合成器走 GPU 之后，dde-shell、控制中心这些 Wayland **客户端**还是 Mesa/llvmpipe。控制中心开着个性化页面时能吃到 100%~350% CPU，成了最大的耗电项。

### 7.1 卡在哪：libmali 在找一个 treeland 没有的 global

直接给 Qt 客户端注入 libmali，要先过两道差异：

- deepin 的 Qt6 是桌面 GL 版，EGL config 要求 `EGL_OPENGL_BIT`，libmali 只有 GLES，得在 `eglChooseConfig` 里换成 `EGL_OPENGL_ES2_BIT`；
- Qt Wayland 的 `wayland-egl` 插件直接链接 `eglGetPlatformDisplay` 等 EGL 1.5 函数，libmali 只有 `EXT` 版本，也要补。

补完之后客户端的 `eglInitialize` 仍然失败（0x3001）。用 `WAYLAND_DEBUG=1` 看得很清楚：libmali 枚举了一遍 registry，**一个 global 都没 bind** 就放弃了。它在找合成器提供的私有协议 `mali_buffer_sharing`，treeland 没有。只在客户端伪造这个 global 也不行，`wl_registry.bind` 到合成器那里会直接报协议错误。

好在这个协议有人公开逆向过：panfork（Mesa 的一个分支）里有 XML 和服务端实现，和这台机器上 r23p0 libmali 内嵌的接口逐字一致。v5 版本只有一个事件、两个请求：

```text
event   alloc_device(name: string)      bind 后立刻发，客户端据此打开 DRM 设备
request create_buffer(id: new_id<wl_buffer>, width, height, stride, format, unk1, unk2, fd)
request auth(magic: uint)               转给 drmAuthMagic，没有回复
```

buffer 由客户端自己分配，通过 `create_buffer` 把 dma-buf fd 传过来，合成器只要像 `wl_drm` 那样把它包成 `wlr_buffer`，不需要任何 Mali ioctl。（新版 Rockchip libmali 有 `MALI_WAYLAND_DMABUF_PROTOCOL` 开关可以改走标准 linux-dmabuf，这台机器的 r23p0 里没有，这条捷径走不通。）

### 7.2 合成器侧：libtl-malibuf，不改 treeland 源码

`libtreeland.so` 是动态调用 `wlr_drm_create` / `wlr_allocator_autocreate` 的，所以可以像 `libtl-norender` 一样用 LD_PRELOAD 挂钩，最后写了 400 来行 C：

- 挂 `wlr_allocator_autocreate` 记下 backend；挂 `wlr_drm_create`，在原函数之后多注册一个 `mali_buffer_sharing` global；
- `create_buffer` 照抄 wlroots `types/wlr_drm.c` 的 prime buffer，包成带 `get_dmabuf` 的 `wlr_buffer`，后面的纹理导入直接走 wlroots 现成的 dmabuf 路径；
- `unk1`/`unk2` 其实是 modifier 的高、低 32 位，默认是 ARM AFBC（`0x0800000000000071`）；
- 最隐蔽的坑在 `auth`：必须用**当前**的 DRM master fd。启动早期拿到的 fd 后来会被关掉、编号被复用，用错 fd 时客户端分配显存的 `DRM_IOCTL_HISI_GEM_CREATE` 会报 EACCES。最后改成扫描 `/proc/self/fd`，用 `drmIsMaster()` 找出真正的 master。

代价是结构体布局绑定了 wlroots 0.20.2，wlroots 一升级就得复核。这也是第九节要写自检脚本的原因之一。

### 7.3 客户端：LD_PRELOAD 会被安全模式清掉

第一版客户端方案是给 dde-shell、控制中心的用户服务加 drop-in 设 `LD_PRELOAD`，再用 PATH 优先的包装脚本处理 mpv 和壁纸进程。dde-shell 这种普通服务没问题，控制中心却怎么都不生效：它经 `deepin-security-loader` 启动，后者带 file capability，进程处于 `AT_SECURE` 安全模式，glibc 会直接清掉 `LD_PRELOAD` 和 `LD_LIBRARY_PATH`。

就算没有这一层，这种方案也很难维护：每个程序都要单独接入（D-Bus 激活和应用管理器是两条启动路径），漏一个就是 llvmpipe；变量还会被子进程继承。

### 7.4 换条路：glvnd 的 EGL vendor 垫片

deepin 的 `libEGL.so.1` 来自 libglvnd，它按 `/etc/glvnd/egl_vendor.d/*.json` 加载 vendor 库，不依赖环境变量，自然也不受安全模式影响。于是我写了一个 vendor 库 `libEGL_hisi_mali.so.0`（约 860 行 C），配置文件名叫 `10_hisi_mali.json`，排在 Mesa 的 `50_mesa.json` 前面，glvnd 会先问它：

```json
{
    "file_format_version" : "1.0.0",
    "ICD" : {
        "library_path" : "/usr/local/lib/hisi-mali/libEGL_hisi_mali.so.0"
    }
}
```

关键是**只认领自己能处理的显示**，其余原样交回 Mesa：

- 只认领 Wayland 平台，而且合成器必须提供 `mali_buffer_sharing`（另开一个私有事件队列枚举 registry 来判断）。GBM、X11、device 平台一律不碰，所以 Xwayland 的 glamor 不受影响；
- 程序在黑名单里、libmali 初始化失败等情况下返回 `EGL_NO_DISPLAY`，glvnd 会自动去问 Mesa，Qt 不需要自己处理回退；
- libmali 只在真正认领时才 `dlopen`（`RTLD_LOCAL`）。

垫片里要抹平的差异：

- libmali 的 `wl_egl_window` 是自己私有的 96 字节结构，和 libwayland-egl 的不兼容。建窗口表面时用 libmali 自己的 `wl_egl_window_create` 另建一个，挂上 resize/destroy 回调做转发；
- 前面说的 `EGL_OPENGL_BIT` 换 `EGL_OPENGL_ES2_BIT`，Qt 会据此改走 GLES；当前绑定的是桌面 OpenGL API 时拒绝建上下文，GTK4、mpv 会自己回退到 GLES；
- EGL 1.5 的 `eglCreateImage`、`eglCreateSync` 等转调 KHR 版本；UOS 魔改 libwayland 的那几个符号也由垫片提供。

出问题时有几个开关可以退回 Mesa：

| 开关 | 作用 |
|---|---|
| `HISI_MALI_DISABLE=1 程序名` | 单个程序不用 libmali |
| `/etc/hisi-mali/egl-deny` | 黑名单，每行一个程序名或完整路径 |
| `touch /etc/hisi-mali/egl-off` | 全局关闭 |
| `HISI_MALI_DEBUG=1 程序名` | 打印认领决策 |

确认一个进程用的是哪套 GL：`grep -oE 'libmali|libgallium|libEGL_hisi' /proc/<pid>/maps | sort -u`。

### 7.5 效果和边界

| 场景 | llvmpipe | libmali |
|---|---|---|
| treeland（合成器） | 约 500% | 3%~6% |
| dde-shell（任务栏等） | 没单独测 | 0%~8% |
| 控制中心壁纸页（带动态壁纸预览） | 66% | 30% |
| mpv 全屏播放 4K 视频 | 约 500% | 110%~120% |

没覆盖到的：Xwayland 程序仍是软件渲染；deepin-terminal、系统监视器本来就是 raster 绘制，不走 EGL；视频**解码**仍是 CPU（UOS 的 ffmpeg 有能硬解的 `h264_omx_dec`，deepin 的 ffmpeg 6.1 没有 OMX 解码器），mpv 剩下的 110% 基本都是解码，处理办法见 8.5。控制中心的占用和页面上正在预览的内容关系很大，表里是同一页面在两种渲染方式下的采样对比。

## 八、动态壁纸：选不上、选上了不动，其实是三件事

控制中心里的动态壁纸先是点下去没反应，修好之后又“选上了但不动”。拆开看是三个独立问题。

### 8.1 在线壁纸下载失败

控制中心只是发起 lastore/apt 安装 `deepin-livewallpaper-*` 包，DNS 坏掉时报的是 `getaddrinfo (16: Device or resource busy)`，用户看到的就是“点了没反应”。修好热点 DNS 后，日志里能看到这些包其实已成功下载并安装；之前没装上时才需要重装。

### 8.2 已安装的壁纸被判定 `AlreadyUsed`

点本地 `default.mp4` 时，treeland 的日志是：

```text
wallpaper failed: ".../default.mp4" - source already configured
```

对照源码（`treeland/src/wallpaper/wallpapermanager.cpp`）：`onVideoChanged()` 先用 `globalValidWallpaper()` 收集当前所有输出/工作区在用壁纸；如果新文件已出现在其中，就回 `AlreadyUsed` 而不是 `sendAdd`。也就是说，这个错误通常是**配置里已经引用了同一路径**，不是 mp4 解码失败。

### 8.3 选上了但“不动”：treeland 0.11 本来就这么设计

选上视频壁纸后桌面是一张静止画面。我一开始以为是播放器坏了，翻源码才发现是有意为之（treeland 0.11.0）：

- 视频由 treeland 拉起的子进程 `treeland-wallpaper-factory` 用 libmpv 解码，再通过私有协议 `treeland_wallpaper_shell_v1` 把画面作为壁纸 surface 交给合成器；合成器用 `play` / `pause` / `slow_down` 事件控制它。
- **锁屏**（`plugins/lockscreen/qml/Greeter.qml`）：锁屏界面显示时 `wallpaper.play = true`，会一直循环播放。
- **桌面**（`core/qml/SourceOutput.qml`）：只有解锁后“显示桌面”那一刻才 `play = true` 并紧接着 `slowDown()`；工厂端用 `OutExpo` 曲线减速，速度低于 0.5 就暂停——按公式大约 0.3 秒后就停了。平时切换工作区或换壁纸时（`wallpaperitem.cpp`），只要没锁屏也会直接 `setPlay(false)`。

所以“桌面上不动”是正常现象：**动态壁纸在锁屏上播放，回到桌面时减速定格**。这也合理——这台机器上视频只能软件解码，4K 原片全屏播放要吃掉 110%~120% CPU（这已经是换成 GPU 渲染之后的数字，剩下的基本是解码），常驻播放会很费电。后来把壁纸重编码成 1440p，降到 40% 左右，见 8.5。

### 8.4 千万别手动杀壁纸工厂

排查时我看它线程全在睡，以为是卡死了，就 `kill -9` 掉了工厂进程。treeland 会立刻重新拉起它，新进程也开始解码，但**画面不会重新挂上去**：

- `WallpaperSwitcherItem::handleWallpaperUpdate()` 和 `WallpaperItem::updateSurface()` 只比较壁纸的**路径**；
- 新工厂报上来的 surface 路径和原来一样，于是被当成“没变化”直接忽略，桌面和锁屏还挂着已经销毁的旧 surface；
- 结果是新工厂在没人显示的情况下持续解码，白白吃掉 120% CPU，也没有任何 `pause` 会发给它。

恢复方法：在控制中心先换成任意一张静态壁纸，再切回动态壁纸，让路径变一次就能重新绑定；重启（或重启 treeland）当然也行。教训是：**线程空闲可能只是被暂停了**，先看源码里的播放控制，再决定要不要动进程。

还遇到过一次“切换动态壁纸后桌面全黑”：当时旧的壁纸进程已经跑了两个小时，还在用 llvmpipe（GPU 垫片装上之前启动的）。重启 treeland 后恢复，新进程走的是 libmali。根因没确认，我怀疑是 surface 一就绪桌面就 `setPlay(false)`，而 mpv 还没渲染出第一帧。再遇到时先试“换静态壁纸再切回”。

### 8.5 软解扛不住 4K：把壁纸重编码成屏幕分辨率

deepin 自带的动态壁纸是 4K H.264，屏幕只有 2160×1440。GPU 只解决了渲染，解码还是 CPU 的活，与其硬扛 4K，不如把视频转成屏幕分辨率。为此写了个 `hisi-wallpaper-reencode`：

- 用 `dpkg-divert` 把包里的原文件挪到 `/usr/local/share/hisi-wallpapers/orig`，再编码出 1440p 版本放回原路径。以后壁纸包升级，新文件会落到 divert 的位置，不会覆盖重编码版本，重跑一次脚本就能更新；
- 编码参数：`libx264 -preset slow -crf 20 -tune fastdecode`，音轨原样复制。`fastdecode`（CAVLC、关掉去块滤波）在这颗 CPU 上比默认参数再省约 30% 的解码时间；
- `--check` 只检查不改动（给自检脚本调用），`--undo` 恢复原文件。

| 测试 | 4K 原片 | 1440p 重编码 |
|---|---|---|
| mpv 全屏播放（libmali 渲染） | 111% | 42% |
| 控制中心式小窗预览（QtMultimedia） | 90% | 41% |
| 纯解码 300 帧 | 85% | 36% |

## 九、防止补丁被 apt 升级悄悄覆盖：hisi-patch-check

到这里，系统里已经散落着几十处本地改动：`/usr/local` 下的垫片和从 UOS 借来的二进制、一堆 systemd drop-in、NetworkManager 配置、glvnd 的 JSON、`/etc/fstab` 里的几行，还有 UOS Boot 分区上的 grub 入口。deepin 25 更新很勤，随便一次 `apt upgrade` 就可能：

- 换掉被 drop-in 依赖的单元文件，或者改变 wpa_supplicant、NetworkManager 的行为；
- 把 wlroots 升到新版本。`libtl-malibuf` 是按 0.20.2 的结构体布局写的，布局一变，轻则客户端退回 llvmpipe，重则合成器崩溃；
- 装上 deepin 的内核包，`/etc/kernel/postinst.d/zz-update-grub` 顺手改写 UOS Boot 分区的启动项。

所以写了一个自检脚本 `hisi-patch-check`（bash，约 800 行）。思路是“黄金副本 + 基线”：

- **文件**：`files.list` 里列出所有补丁文件（支持 glob），`snapshot` 时把它们连同权限、符号链接目标复制到 `golden/`，sha256 写进 `manifest.tsv`。前缀 `unowned:` 表示只纳入 dpkg 不拥有的文件，`noauto:` 表示只检测、不自动恢复；
- **包版本**：40 多个敏感包（treeland、ddm、NetworkManager、wpasupplicant、libwayland、Mesa/glvnd、grub、内核……）记录版本基线，变了就提示复核；
- **系统状态**：fstab 和 hosts 里受保护的行、各服务的 enable 状态；UOS Boot 分区平时不挂载，用 `debugfs` 只读校验 `custom.cfg` 的哈希；
- **运行时**：elfverify 是否已关、wlroots 的版本和 `libtl-malibuf` 挂钩的 9 个符号是否还在、treeland 和 dde-shell 进程里是否真的加载了 libmali、动态壁纸是否还是重编码版本、box64 的 binfmt 注册是否还在。

触发时机有三个：apt 的 `DPkg::Post-Invoke` 钩子（每次装完包，带 `|| true`，绝不影响 apt 本身）、开机后跑一次、每天定时跑一次。有问题时给桌面用户发通知；装敏感包之前，`Pre-Install-Pkgs` 钩子还会先在终端里醒目提示一下。

```sh
sudo hisi-patch-check            # 全量检查；退出码 0=正常 1=补丁被改/丢失 2=仅警告 3=未初始化
sudo hisi-patch-check --fix      # 先备份现场到 /var/backups/hisi-patch/<时间戳>/，再恢复文件、
                                 # enable 状态、fstab 缺行、binfmt 注册；只列出需要重启的服务，不自动重启
sudo hisi-patch-check snapshot   # 有意修改补丁后重新抓基线
```

`--fix` 刻意做得很保守：图形和网络服务一律不自动重启，Boot 分区只在确认未挂载时才写入。另外有个演练模式 `HISI_PATCH_ROOT=/tmp/xxx`，让文件检查和修复指向一份临时副本，调试脚本时不会碰到线上文件。现在这台机器上的结果是 146 项检查全部通过，跟踪 69 个文件、43 个包。

## 十、几个零碎问题

### 10.1 注销后再登录，锁屏只剩一张不动的壁纸

开机后第二次走登录流程（手动注销再登录，或者重启 ddm 之后）时，登录界面没有输入框，只剩一张静止的壁纸，有时会报 `Failed to get auth info from display manager`。锁屏后解锁不受影响。

查下来是 ddm 0.4.0 自己卡死了：ddm 为新会话 fork 出会话领头进程，领头进程打开 PAM 会话后再 fork 一个用户身份的子进程去 exec `dde-session`。出问题时这个子进程**还没 exec 就卡在 futex 上**（`/proc/<pid>/exe` 仍是 `/usr/bin/ddm`），领头进程停在 `wait4`，ddm 主线程在同步读管道等会话 ID（内核栈停在 `pipe_read`），整个 daemon 不再处理 greeter 的消息，登录界面也就出不来了。fork 之后具体卡在哪把锁，到现在还没抓到。

根因没找到之前，先上一个看门狗 `hisi-ddm-watchdog.service`：每 5 秒扫一次，找“非 root、exe 仍是 `/usr/bin/ddm`、父进程也是 ddm、存活超过 15 秒”的进程，先把内核栈和 gdb 用户态栈写进 `/var/log/hisi-ddm-watchdog.log`，再 SIGKILL。正常情况下这种子进程几毫秒内就会 exec，不会误杀。杀掉之后领头进程会继续拉起会话，堆积的 greeter 消息也会被处理掉。判断 ddm 有没有卡住：`sudo cat /proc/$(pgrep -xo ddm)/stack`，正常停在 `do_sys_poll`，卡住时是 `pipe_read`。

### 10.2 指纹能用于 sudo，解不了锁

指纹是汇顶 HC460，走 SPI + TEE，不经过 fprintd/libfprint。deepin 25 上不可用的直接原因是守护进程没起来：从 UOS 复制厂商库和 `hwfp_init` 服务，补一个 `/usr/bin/fingerprint_daemon` 符号链接，设备就能识别了。人脸这一层本身是好的，摄像头和人脸服务都在工作，只是还没录入。

但锁屏和登录界面依然只能输密码：我在 treeland 0.11（`libtreeland.so`、`liblockscreen.so`）和 ddm 0.4 里都没找到 `Authenticate1` 或任何生物认证的引用，ddm 只在 greeter 把密码发过来之后才调用 `pam_authenticate`。所以指纹和人脸目前只能用在 sudo、su 和 polkit 弹窗上，单靠改配置解决不了锁屏。另外 UOS 和 deepin 共用 TEE 里的指纹存储，在一边重新录入会影响另一边。

### 10.3 兼做构建机：box64 和只读的 UOS 用户态

这台机器在 UOS 下还兼做 Android/Linux 包的构建机，换到 deepin 后有两个坑：

- Android SDK/NDK 里的 aapt2、clang/ld.lld、cmake 和 Flutter 的 gen_snapshot 全是 x86_64 程序，UOS 上靠 box64 透明执行。把 box64 和它的 binfmt 注册原样搬过来就行；
- Linux 包要保证产物在老系统上能跑（GLIBC ≤ 2.28），而 deepin 25 的 glibc 新得多。解决办法是把 UOS 的根分区 Roota 只读挂载，用 `bwrap` 进入 UOS 用户态构建，源码和工具链目录按需 bind 进去。宿主是 deepin，编出来的包等于在 UOS 上编的。

## 十一、给其他机器装 v25 的检查清单

1. **内核优先**：通用 ISO 能不能装？先看 UFS/显示是否在 initramfs 里；厂商模块 ABI 对不上就不要硬塞。
2. **保留原系统**：默认启动项维持原厂，新系统用 `grub-reboot` 一次性进入；/rootb 分区和 `/data` 先备份。
3. **exec 卡死** → 查 LSM / 验证钩子（elfverify 只是其中一种）。
4. **Wi‑Fi/D-Bus 卡死** → 分开查 `p2p-dev`、nl80211 dump、DNS；LD_PRELOAD 垫片只给对应守护进程。
5. **控制中心无响应** → 先查 display/session 服务是否齐全；X11 路径坑多时直接切发行版原生 Wayland。
6. **Wayland 黑屏** → 查 render node 是否真是 GPU、`dmabuf` fd 权限、实时调度是否被内核拒绝。
7. **CPU 高/鼠标卡** → `grep llvmpipe /proc/<pid>/maps` 或看线程名；软件渲染和 GPU 渲染的差距是数量级的。
8. **厂商 EGL 库** → 合成器用 drop-in 单独加载；客户端别靠全局 `LD_LIBRARY_PATH`/`LD_PRELOAD`（带 capability 的程序会被 glibc 清掉，子进程还会继承），改走 glvnd 的 vendor JSON，只认领自己能处理的显示，其余交回 Mesa。客户端 `eglInitialize` 失败时用 `WAYLAND_DEBUG=1` 看它是不是在找私有协议。
9. **动态壁纸** → 分开查下载（DNS/apt）和协议（`AlreadyUsed` / 壁纸工厂日志），别把“点了没反应”当成一个 bug；桌面上定格是 treeland 的设计，锁屏才连续播放；别手动杀壁纸工厂；只能软解时，把视频重编码到屏幕分辨率比什么优化都管用。
10. **补丁防覆盖** → 本地补丁一多就写自检：黄金副本 + 包版本基线 + apt 钩子；有意修改后记得重抓基线。
11. **验证**：每项修改都保留回退文件；`systemctl daemon-reload` 后再重启对应服务；有用户会话时别乱用 root 跑合成器二进制。

## 十二、最终状态

- deepin 25 用户态在 Rootb，沿用 UOS 内核；
- ddm + treeland 正常进桌面，壁纸、任务栏、控制中心可用；
- treeland 用 Mali-G78 GPU 渲染，CPU 从约 500% 降到个位数；dde-shell、控制中心、mpv、壁纸进程等 Wayland 客户端经 glvnd 垫片也走 libmali，Xwayland 程序仍是软件渲染；
- 动态壁纸锁屏循环播放、桌面定格；重编码成 1440p 后，播放时约 40% CPU（仍是软件解码）；
- 注销再登录时 ddm 卡死由看门狗兜底；指纹可用于 sudo 和 polkit，锁屏只能输密码；
- 所有补丁由 `hisi-patch-check` 盯着，每次 apt 装完包都会自检；
- 默认启动仍是 UOS，一次性 `grub-reboot` 进 deepin，安全兜底。

这套方案不优雅，但把“厂商内核 + 新用户态”的边界控制住了：**只动 DE 和用户态，不动内核与驱动**。对其他同样被私有驱动锁死的 ARM 机器，这是最现实的升级路径之一。

文中提到的垫片源码、drop-in、自检脚本和看门狗都在 [kegechen/kirin9006c-deepin25](https://github.com/kegechen/kirin9006c-deepin25)，仓库的 `docs/` 里有更细的 GPU 架构说明和已知问题列表。
