# Nyxian + Procursus Jailed 上手与踩坑记录

在 **未越狱（stock）** 的 iPhone 上跑起 [Nyxian](https://github.com/emexlab/Nyxian)（一个用户态微内核 + 板载 IDE），
然后把 **Procursus Jailed** ROM 刷进去，得到一个带包管理器和命令行的环境。

这份文档记录的是**实测过程、验证过的事实、以及踩过的坑**。

---

## 先说清楚它不是什么

网上关于这套东西的描述很多是错的，包括一些看起来挺正规的站点。以下三条务必先看清楚：

| 它不是 | 为什么 |
|---|---|
| **不是虚拟机** | 官方给自己的定性是 `userspace microkernel`。虚拟机模拟硬件，它**不模拟硬件** —— 它跑的是**真实的 arm64 iOS 二进制**，靠在用户态拦截并转发系统调用骗过 guest。 |
| **不是 IPA 安装器** | 它不会把 App 装到你的真实桌面。实测：内层 Nyxian 主程序里 `.ipa` 出现 **0 次**，它根本不认这个格式。你要装 IPA 得用 TrollStore / SideStore / AltStore，那是另一条线。 |
| **不是越狱替代品** | 它借不到 iOS 内核的任何特权。它是系统眼里的**一个普通 App**。 |

**一句话定位**：它是一台装在 iPhone 里的「开发机 + 命令行环境」，不是一台「装别人 App 的机器」。

---

## 适用范围（重要）

本文所有结论都在下面这套具体环境上验证过。**任何一个变量不同，都可能不通。**

| 项目 | 值 |
|---|---|
| 设备 | iPhone 11 |
| 系统 | iOS 27.0 Beta 1 |
| 系统构建号 | `24A5355q` |
| 是否越狱 | **否** |
| 侧载工具 | iLoader（iloader 2.3.8） |
| Apple ID | 免费账号 |
| Nyxian | `0.11.5 "Scriptura" Beta`，构建号 `20261012.26.RELEASE` |
| ROM | Procursus Jailed `1.0` |

补充几个容易搞混的版本号：

- 官方 GitHub Releases 与官方 AltStore 源里，**最新只到 `0.11.4`（2026-09-10）**。
  本文用的 `0.11.5 / 20261012.26.RELEASE` **不在任何官方渠道里**，来自 Discord 的测试构建。
- Nyxian 的 bootstrap 服务器上那份内层固件（`Nyxian_0.11.5_3.ipa`，2026-09-23 上传）内部版本是
  `20260920.8.RELEASE`，`MinimumOSVersion 18.0`，`DTSDKBuild 24A5408c`。
- **版本漂移是这套东西最大的坑**：引导器脚本（服务端 2026-09-23）和新版 App（本地 2026-10-12）
  的 Recovery 菜单**完全不一样**。见 [docs/02-恢复模式.md](docs/02-恢复模式.md) 里的对照表。

---

## 当前状态

- ✅ Nyxian 侧载成功，能打开
- ✅ 签名证书导入成功（`Set Up Signing` 弹窗消失）
- ✅ Procursus Jailed `1.0` **刷入成功**（`flashed Procursus Jailed 1.0`）
- ✅ ROM 引导成功，容器跑起来
- ✅ 容器里的包管理器可工作（实测装上了 `launchctl`、`uiktools`）
- ⬜ 尚未验证：如何真正进入容器敲命令（终端窗口 / SSH 入口）
- ⬜ 尚未验证：SpringBoard 插件注入可不可行
- ❓ 已知异常：Nyxian 设置里的 `Installed Root Certificate Authorities` 点不开

---

## 目录

| 文档 | 内容 |
|---|---|
| [docs/01-侧载与证书.md](docs/01-侧载与证书.md) | 侧载 Nyxian、导出并导入开发者证书（`.p12`） |
| [docs/02-恢复模式.md](docs/02-恢复模式.md) | Recovery 怎么进（含音量键触发的真实机制和一个大坑） |
| [docs/03-刷ROM.md](docs/03-刷ROM.md) | Flash ROM 全流程、日志逐行解读、卡启动图的处理 |
| [docs/04-它能干什么.md](docs/04-它能干什么.md) | 能力清单、三种"装"的区别、性能与发热 |
| [docs/05-逆向笔记.md](docs/05-逆向笔记.md) | `.nxt2c` 格式、bootstrap 顺序、内部类名（附方法） |
| [docs/99-未解决.md](docs/99-未解决.md) | 未确认清单、已排除方向、已知失效点 |

---

## 外部资源与校验值

**本文档不托管任何第三方文件。** 下面只给地址和校验值，请自己下载、自己核对。

| 文件 | 地址 | 大小 | SHA-256 |
|---|---|---|---|
| rootca 公钥 | `https://nyxian.app/bootstrap/org.emexlabs.rootca.v1.pub.nxt2c` | 119 B | `47332d1ff5e309e411ee0e54c2ece986d5cf0ea4de1cc0d286c1975add45036c` |
| Procursus Jailed ROM | 见下 | 45,739,361 B | `78f7c07f39bb81c4fc1385ca949dce824f1f7795783e061f829e5d295f32ba2b` |
| 内层固件 | `https://nyxian.app/bootstrap/Nyxian_0.11.5_3.ipa` | 112,146,995 B | 未记录 |
| 引导器脚本 | `https://nyxian.app/bootstrap/`（目录可列） | — | — |

`https://nyxian.app/bootstrap/` 这个目录**可以直接列出内容**（返回 418，但正文是 nginx 的目录索引），
里面能看到 `manifest.js` / `JEFI.js` / `LoaderUI.js` / 各版本 SDK zip / 上面这些文件。

**Procursus Jailed ROM 不在 GitHub 上**，只能从这个 bootstrap 目录或 Discord 拿，
所以没有办法 star 跟踪它的更新。

---

## 免责声明

- 本文只是操作记录，**不保证对你的设备有效**。刷 ROM、改 Boot Configuration 都有让 App 起不来的风险。
- 动手前请确认你**没有重要数据**放在这个 App 的容器里。
  注意：Recovery 里的 `Wipe data / factory reset`、`Unflash ROM`，以及 Nyxian 设置里的
  `Restore Virtual Environment` / `Restore Everything`，**都会真的抹掉东西**，别顺手点。
- 请自己遵守所在地法律法规与相关软件许可协议。本文不涉及、也不支持任何绕过付费或侵犯版权的用法。
- 文中所有的版本号都对应上面「适用范围」那张表；换了版本，结论可能就不成立了。

## License

本文档（README 与 `docs/` 下的全部内容）以 **CC BY-SA 4.0** 发布，完整法律文本见 [LICENSE](LICENSE)。

**简单说 —— 你可以：** 自由复制、转载、改编本文档，包括用于商业用途。

**条件是：** **署名**（注明来源并附协议链接）、**相同方式共享**（你改编后的版本也必须用同样的协议发布）。

几句边界说明：

- 本文档**不包含也不托管**任何第三方软件、IPA、ROM 或 SDK 文件。外部资源只给地址与校验值，请自行获取并遵守其各自的许可协议。
- 文中提到的 **Nyxian 本体**以 **AGPL-3.0** 发布，那是它自己的许可，与本文件采用的 CC BY-SA 4.0 无关。
- 引用原文（例如官方 README 的句子）时，请一并注明原始出处。

文中的技术结论来自对本地文件与公开资源的实测，**欢迎纠错与补充 —— 开 Issue 即可**。
提交时请带上设备型号、系统构建号与 Nyxian 版本，少了版本号结论没法用。
