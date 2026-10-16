# 03 · 刷入 ROM 并启动

目标：把 **Procursus Jailed** ROM 刷进去，让容器跑起来。

> 验证环境：iPhone 11 / iOS 27.0 Beta 1 (`24A5355q`) / Nyxian `0.11.5 (20261012.26.RELEASE)`
> ROM：Procursus Jailed `1.0`（`org.emexlabs.procursus.jailed`，要求 Nyxian ≥ 0.11.5）

---

## 一、动手之前

- [ ] 证书已导入（[01-侧载与证书.md](01-侧载与证书.md)）
- [ ] 会进 Recovery（[02-恢复模式.md](02-恢复模式.md)）
- [ ] 进 Recovery → `Boot Configuration` → 把 **`Obfuscate Log` 关掉**（这样出问题才有可读日志）
- [ ] `Unflash ROM` 之类的项**知道是干什么的**，别顺手按

**先明白一件事**：刷 ROM 会改变设备状态。刷完之后，你面对的不再是"干净的未修改环境"。
好消息是**退回来的路是有的**（`Unflash ROM`）。

---

## 二、拿到 ROM 包

RAM 包**不在 GitHub 上**，只能从 Nyxian 的 bootstrap 服务器或 Discord 拿。

```
地址：https://nyxian.app/bootstrap/          （目录可直接列出）
文件：procursus-jailed.zip
大小：45,739,361 字节（约 43.6 MB）
SHA-256：78f7c07f39bb81c4fc1385ca949dce824f1f7795783e061f829e5d295f32ba2b
```

> **本文档不托管这个文件**，请自己从上面的地址下载并核对校验值。

包的结构（用 `unzip -l` 或任何 zip 工具都能看）：**155 个条目，全部在一个 `ROM/` 顶层目录下**，
`ROM/manifest.plist` 在最上面。这个结构和引导器里的 `Documents/Boot/ROM` 是对得上的 ——
**实测刷写器能直接吃这个结构，不需要你把 `ROM/` 这层剥掉。**

### 把包传到手机上（Windows 没有 AirDrop）

用微信或 QQ 的**文件传输助手**发给自己 → 手机上点开 → 「存储到文件」→ 存进「文件」App。

> ⚠️ 必须走「**文件**」方式。用"图片/视频"会被压缩，43.6 MB 的包就废了。
> 传完**核对大小**是不是 45,739,361 字节。

---

## 三、Flash ROM

1. 进 Recovery（音量减，见 [02](02-恢复模式.md)）
2. 光标移到 **`Flash ROM`**，两个音量键同时按住确认
3. 会弹出 **iOS 系统文件选择器**（不是它自带的私有浏览器）——
   也就是说 **ROM 包只要放进「文件」App 就行**，不需要先塞进 Nyxian 的容器
4. 找到 `Downloads`（或你存的位置），选中 `procursus-jailed.zip`
5. 等刷写跑完

### 刷写日志逐行解读

成功的日志长这样：

```
refreshed binpack/usr/lib/libroothide.dylib
refreshed binpack/usr/lib/libvroot.dylib
refreshed binpack/usr/lib/libvrootapi.dylib
refreshed binpack/usr/lib/roothideinit.dylib
adding NXT2 entitlements if needed
added NXT2 entitlements to binpack/bin/binpack
added NXT2 entitlements to binpack/bin/launchctl
added NXT2 entitlements to binpack/usr/bin/clear
added NXT2 entitlements to binpack/usr/bin/curl
added NXT2 entitlements to binpack/usr/bin/dropbearmulti
...（每个可执行文件一行）...
refixing sign files permissions
flashed Procursus Jailed 1.0

Flash complete.
Press both volume buttons to reboot when ready
```

几个值得懂的词：

| 日志 | 含义 |
|---|---|
| `libroothide` / `libvroot` / `libvrootapi` / `roothideinit` | 虚拟根文件系统 + root 隐藏。这是"免越狱容器"的骨架，来自 [roothide](https://github.com/roothide/Bootstrap) 那条线 |
| `adding NXT2 entitlements` | **给每个可执行文件注入 Nyxian 自己的权限块（NXT2）**。因为它们在用户态微内核里跑，需要这套自建权限模型认可。所以会一个文件一行地打出来 |
| `refixing sign files permissions` | 修权限 |
| `flashed Procursus Jailed 1.0` | 成功。版本号来自 ROM 的 `manifest.plist` |

**如果报"找不到 manifest / 无效 ROM 包"之类**：那是包结构的问题。
本 ROM 实测**没有**这个问题；但如果你从别处拿到的是"多一层目录"的包，
把它解压后重新打包，让 zip 根目录直接就是 `manifest.plist` 再试。

---

## 四、重启并观察引导

刷完之后按屏幕提示：**同时按住两个音量键**重启（或者用 `Reboot system now`）。

### 会先看到 ROM 的启动图 —— 但别把它当成进度条

ROM 里自带启动图（`manifest.plist` 里声明的）：

```
NXRomBootLogo = { Light: System/Rom/bootl.jpg, Dark: System/Rom/bootd.png }
NXRomBootLogoDoesPulse = true
```

- `bootl.jpg`：浅色版（白底 + 一张深灰色的桌子）
- `bootd.png`：深色版
- `NXRomBootLogoDoesPulse = true` → **这张图会脉冲式地闪动**

**关键认知（这点很容易误解）：这张图是引导器画的，不是 ROM 的 `init` 画的。**
`NXRomBootLogo*` 是引导器读的 manifest 键。所以——

> **图在闪 ≠ ROM 有进展。它只说明引导器的界面线程还活着、还在等 `System/Rom/init` 交接。**

### 首启会很慢，这正常

ROM 里带了 `bootstrap.tar.zst`（25 MB，解出来约 6670 个文件），首次启动要把它们铺进虚拟根。
而这套环境的文件系统要过一层沙盒扩展 API，**逐个创建几千个文件可能要好几分钟**。

**所以：先等 3~5 分钟，别急着杀。**

### 一直卡在启动图怎么办

按这个顺序试：

1. **再等一会儿**（总共到 10 分钟左右）。首启铺 rootfs 是最慢的一步。
2. **重启一次，重走一遍。**
   首次启动失败、第二次成功，是这类"首启要铺一堆文件"的系统的常见情况
   （第一次可能铺到一半断了，第二次接着来）。**实测就是靠这一步过去的。**
3. **还是不行 → 去拿日志。** 杀掉 App → 按住音量减进 Recovery → `Nyxian Files` →
   找 **`kmsg.txt`** / **`kmsg_old.txt`**。
   如果内容是乱码，说明 `Obfuscate Log` 还开着 → 关掉它 → `Reboot system now` 再走一次 → 再回来读。
   （`Obfuscate Log` 是启动时读的，**改了必须重启才生效**，所以这一步可能要跑两轮。）
4. **兜底**：`Unflash ROM` 退回干净状态，从零再来。刷这一步已经证明能成功，退回来不亏。

### 三种"卡在启动图"长得一模一样

这是为什么要死磕日志：

| 可能 | 处理 |
|---|---|
| `init` 还在慢慢铺 rootfs | 等就行，好消息 |
| `init` 崩了，但引导器没有失败检测，只会一直等 | 坏消息，得换构建 |
| `init` 卡在某个 syscall 上（微内核转发在当前系统版本上对不上） | 坏消息，等上游修 |

光看屏幕区分不了这仨。

---

## 五、成功之后

能进去、能操作了，就说明 ROM 引导通了。接下来：

- 首次启动后容器里的包管理器会开始装基础包（实测会看到 `launchctl`、`uiktools` 之类的下载队列）。
  **让它装完，别去按"取消下载"。**
- 这时候**卡和发烫是正常的**：装包 = 解压 + 大量写文件 + 可能还要重签，CPU 密集。
  再叠加"用户态微内核每条 syscall 都要转发"和"没 JIT"，会更明显。**等它装完会好转。**
- 散热：拿掉手机壳、别边充边玩、放通风处。**别反复重启**（每次都要重走一遍冷启动）。

---

**下一步** → [04-它能干什么.md](04-它能干什么.md)：这套环境的能力边界到底在哪。
