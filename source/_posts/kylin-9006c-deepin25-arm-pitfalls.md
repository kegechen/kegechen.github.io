---
title: 在 ARM 笔记本上给 deepin 25 换心不换骨：麒麟 9006C 驱动兼容踩坑全记录
date: 2026-11-25 10:30:00
permalink: 2026/11/25/kylin-9006c-deepin25-arm-pitfalls/
tags:
  - deepin 25
  - ARM64
  - 麒麟 9006C
  - treeland
  - Mali
  - Wayland
  - NetworkManager
  - Linux 内核
categories:
  - 技术笔记
---

> 一台华为擎云 L540（麒麟 9006C / ARM64）出厂是 UOS。想装 deepin 25，ISO 启动后却看不到硬盘、点不亮屏幕。最后我们把 **deepin 25 的用户态装进 Rootb，沿用 UOS 的厂商内核和 initrd**，一路打通到 ddm + treeland 桌面；随后又修掉了 NetworkManager 100% CPU、控制中心卡死、黑屏、500% CPU 鼠标卡顿、动态壁纸选不上等问题，还顺带调研了厂商 Mali 私有 Wayland 协议。
>
> 这篇不是“一键安装教程”，而是一份排障笔记：每个症状背后是什么、怎么验证、怎么修、哪些做法**不能**照搬。对任何“通用发行版 ISO + 厂商私有驱动 ARM 机器”都有参考价值。

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

UOS 里有闭源 `libmali.so.0.23.0`（Valhall r23p0-01rel0，EGL 1.4 + GLES 3.2，自带 gbm）。复制到 `/usr/local/lib/hisi-mali`，**不要放进全局 ld.so 搜索路径**，只在 treeland 的 drop-in 里设置：

```ini
Environment=LD_LIBRARY_PATH=/usr/local/lib/hisi-mali
Environment="LD_PRELOAD=/usr/local/lib/hisi-mali/libmali-compat.so /usr/local/lib/hisi-mali/libmali.so.0 /usr/local/lib/hisi-treeland/libtl-norender.so"
```

验证数据：

- `gbm_create_device(card0)` 成功，backend `armsoc`；
- EGL vendor `ARM`，GL renderer `Mali-G78`；
- 2160×1440 全屏 8 层混合，约 3.94 ms/帧；
- 切到 GPU 后 treeland CPU 从约 500% 降到 3–6%，鼠标不再卡。

### 6.2 libmali-compat 要补什么

厂商库是按 UOS 用户态编译的，放到 deepin 25 会缺这些：

1. `wl_proxy_ref/unref/get_refcount`：UOS 魔改 libwayland-client 的符号，上游没有；treeland 是服务端，给空实现即可；
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

## 七、动态壁纸：选不上、选上了不动，其实是三件事

控制中心里的动态壁纸先是点下去没反应，修好之后又“选上了但不动”。拆开看是三个独立问题。

### 7.1 在线壁纸下载失败

控制中心只是发起 lastore/apt 安装 `deepin-livewallpaper-*` 包，DNS 坏掉时报的是 `getaddrinfo (16: Device or resource busy)`，用户看到的就是“点了没反应”。修好热点 DNS 后，日志里能看到这些包其实已成功下载并安装；之前没装上时才需要重装。

### 7.2 已安装的壁纸被判定 `AlreadyUsed`

点本地 `default.mp4` 时，treeland 的日志是：

```text
wallpaper failed: ".../default.mp4" - source already configured
```

对照源码（`treeland/src/wallpaper/wallpapermanager.cpp`）：`onVideoChanged()` 先用 `globalValidWallpaper()` 收集当前所有输出/工作区在用壁纸；如果新文件已出现在其中，就回 `AlreadyUsed` 而不是 `sendAdd`。也就是说，这个错误通常是**配置里已经引用了同一路径**，不是 mp4 解码失败。

### 7.3 选上了但“不动”：treeland 0.11 本来就这么设计

选上视频壁纸后桌面是一张静止画面。我一开始以为是播放器坏了，翻源码才发现是有意为之（treeland 0.11.0）：

- 视频由 treeland 拉起的子进程 `treeland-wallpaper-factory` 用 libmpv 解码，再通过私有协议 `treeland_wallpaper_shell_v1` 把画面作为壁纸 surface 交给合成器；合成器用 `play` / `pause` / `slow_down` 事件控制它。
- **锁屏**（`plugins/lockscreen/qml/Greeter.qml`）：锁屏界面显示时 `wallpaper.play = true`，会一直循环播放。
- **桌面**（`core/qml/SourceOutput.qml`）：只有解锁后“显示桌面”那一刻才 `play = true` 并紧接着 `slowDown()`；工厂端用 `OutExpo` 曲线减速，速度低于 0.5 就暂停——按公式大约 0.3 秒后就停了。平时切换工作区或换壁纸时（`wallpaperitem.cpp`），只要没锁屏也会直接 `setPlay(false)`。

所以“桌面上不动”是正常现象：**动态壁纸在锁屏上播放，回到桌面时减速定格**。这也合理——这台机器上 4K H.264 是纯软件解码，播放时壁纸工厂自己就要吃掉约 120% CPU，再加上 llvmpipe 合成，常驻播放会很费电。

### 7.4 千万别手动杀壁纸工厂

排查时我看它线程全在睡，以为是卡死了，就 `kill -9` 掉了工厂进程。treeland 会立刻重新拉起它，新进程也开始解码，但**画面不会重新挂上去**：

- `WallpaperSwitcherItem::handleWallpaperUpdate()` 和 `WallpaperItem::updateSurface()` 只比较壁纸的**路径**；
- 新工厂报上来的 surface 路径和原来一样，于是被当成“没变化”直接忽略，桌面和锁屏还挂着已经销毁的旧 surface；
- 结果是新工厂在没人显示的情况下持续解码，白白吃掉 120% CPU，也没有任何 `pause` 会发给它。

恢复方法：在控制中心先换成任意一张静态壁纸，再切回动态壁纸，让路径变一次就能重新绑定；重启（或重启 treeland）当然也行。教训是：**线程空闲可能只是被暂停了**，先看源码里的播放控制，再决定要不要动进程。

### 7.5 想让系统应用也 GPU 加速：卡在厂商私有协议

合成器已经走 Mali GPU，但 dde-shell、控制中心这些 Wayland **客户端**仍是 Mesa/llvmpipe。控制中心开着个性化页面时实测能吃到 100%~350% CPU，是最大的耗电项。我试了给 Qt 客户端也注入 libmali：

- Qt6 桌面版 EGL 配置要求 `EGL_OPENGL_BIT`，libmali 只有 GLES，得在 `eglChooseConfig` 垫片里换成 `EGL_OPENGL_ES2_BIT`；
- Qt Wayland 的 `wayland-egl` 插件直接链接 `eglGetPlatformDisplay` 等 EGL 1.5 函数，libmali 只有 `EXT` 版本，也要补；
- 但客户端 `eglInitialize` 仍失败（0x3001）。

用 `WAYLAND_DEBUG=1` 看得很清楚：libmali 枚举了一遍 registry，**一个 global 都没 bind** 就放弃了——它在找合成器提供的私有协议 `mali_buffer_sharing`，treeland 没有。只在客户端伪造这个 global 也不行，`wl_registry.bind` 到合成器那里会直接报协议错误。

后来专门调研了一下这个协议，结论比预想乐观：

- **协议是公开逆向过的**。panfork（Mesa 的一个分支）里有 XML 和服务端实现，和这台机器上 r23p0 libmali 导出的接口逐字一致：v5，请求 `create_buffer(new_id, width, height, stride, format, unk1, unk2, fd)`、`auth(magic)`，事件 `alloc_device(busid)`。
- **服务端要做的事很少**：客户端自己分配好 buffer，通过 `create_buffer` 把 dma-buf fd 传过来，合成器只需像 `wl_drm` 那样把它包成 `wlr_buffer` 导入；`auth` 转给 `drmAuthMagic`；bind 时发一次 `alloc_device`。合成器侧不需要任何 Mali ioctl。
- **可以不改 treeland 源码**：`libtreeland.so` 动态调用 `wlr_drm_create` / `wlr_allocator_autocreate`，可以像前面的 `libtl-norender` 一样用 LD_PRELOAD 挂钩，在里面拿 DRM master fd、额外注册这个 global，估计 300 行左右 C。
- **主要风险在客户端**：libmali 用到的 `wl_proxy_ref/unref` 现在是空实现，放到客户端里要改成真实引用计数；Qt/DTK 的桌面 GL 着色器能否在 GLES 上跑也要实测；libmali 初始化失败时 Qt 不会自动回退，必须先探测到 global 再给客户端注入。
- **捷径走不通**：新版 Rockchip libmali 有 `MALI_WAYLAND_DMABUF_PROTOCOL` 开关可以改走标准 linux-dmabuf，但这台机器的 r23p0 里没有。

**结论**：短期内 libmali 只给合成器用，**不要全局给所有用户进程注入**。想让客户端也上 GPU，最现实的路是“合成器侧补 `mali_buffer_sharing` + 只给探测通过的少数客户端注入”，值得花一天做个验证，但要做好随 treeland/wlroots 升级重新适配的准备。

## 八、给其他机器装 v25 的检查清单

1. **内核优先**：通用 ISO 能不能装？先看 UFS/显示是否在 initramfs 里；厂商模块 ABI 对不上就不要硬塞。
2. **保留原系统**：默认启动项维持原厂，新系统用 `grub-reboot` 一次性进入；/rootb 分区和 `/data` 先备份。
3. **exec 卡死** → 查 LSM / 验证钩子（elfverify 只是其中一种）。
4. **Wi‑Fi/D-Bus 卡死** → 分开查 `p2p-dev`、nl80211 dump、DNS；LD_PRELOAD 垫片只给对应守护进程。
5. **控制中心无响应** → 先查 display/session 服务是否齐全；X11 路径坑多时直接切发行版原生 Wayland。
6. **Wayland 黑屏** → 查 render node 是否真是 GPU、`dmabuf` fd 权限、实时调度是否被内核拒绝。
7. **CPU 高/鼠标卡** → `grep llvmpipe /proc/<pid>/maps` 或看线程名；软件渲染和 GPU 渲染的差距是数量级的。
8. **厂商 EGL 库** → 只给合成器单独用；不要全局 `LD_LIBRARY_PATH`，小心私有 Wayland 协议和子进程继承。
9. **动态壁纸** → 分开查下载（DNS/apt）和协议（`AlreadyUsed` / 壁纸工厂日志），别把“点了没反应”当成一个 bug；桌面上定格是 treeland 的设计，锁屏才连续播放；别手动杀壁纸工厂。
10. **验证**：每项修改都保留回退文件；`systemctl daemon-reload` 后再重启对应服务；有用户会话时别乱用 root 跑合成器二进制。

## 九、最终状态

- deepin 25 用户态在 Rootb，沿用 UOS 内核；
- ddm + treeland 正常进桌面，壁纸、任务栏可用；
- treeland 使用 Mali-G78 GPU 渲染，CPU 从约 500% 降到个位数；
- 动态壁纸能下载、能选中，锁屏循环播放、桌面定格（软件解码，播放时约 120% CPU）；
- dde-shell / 控制中心客户端仍是软件渲染；`mali_buffer_sharing` 已摸清，是否动手留待验证；
- 默认启动仍是 UOS，一次性 `grub-reboot` 进 deepin，安全兜底。

这套方案不优雅，但把“厂商内核 + 新用户态”的边界控制住了：**只动 DE 和用户态，不动内核与驱动**。对其他同样被私有驱动锁死的 ARM 机器，这是最现实的升级路径之一。
