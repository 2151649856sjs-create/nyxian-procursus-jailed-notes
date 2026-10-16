# images/

放文档里用到的截图。

## ⚠️ 放进来之前必须先检查

截图里很容易带上**个人信息**，而这个目录是会被公开的。提交前逐张确认下面这些**没有露出来**：

| 要挡住的 | 一般出现在哪 |
|---|---|
| Apple ID 邮箱 | SideStore / AltStore 的证书页，`Requester:` 那一行 |
| 证书序列号 / 证书 ID | 证书页的 `Serial:` / `ID:` 那一行，以及证书名里 |
| 设备 UDID | iLoader 之类的侧载工具界面、配对文件列表 |
| 你的姓名 | 证书页的 `Created By:`、以及 `xxx 的 iPhone` 这种证书标题 |

**最省事的办法不是打码，是重拍**：拍照前用手指或者一张便签把这几行挡住。

已经排查过、**可以直接用**的几张图，见仓库外的 `配图清单.txt`。

## 命名

建议按文档编号，方便对照：

```
01-signing-setup.jpg      对应 docs/01
02-recovery-menu.jpg      对应 docs/02
03-flash-log.jpg          对应 docs/03
...
```

## 在 Markdown 里引用

```markdown
![说明文字](images/01-signing-setup.jpg)
```
