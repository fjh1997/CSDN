---
title: Windows 微信 / QQNT 阻止自动更新（配合 RevokeMsgPatcher 版本上限）
abbrlink: 2026072501
url: /posts/2026072501.html
date: 2026-07-25 21:00:00
tags:
  - 微信
  - QQ
  - QQNT
  - Windows
  - hosts
  - 防更新
  - RevokeMsgPatcher
---

## 为什么要阻止升级

[RevokeMsgPatcher](https://github.com/huiyadanli/RevokeMsgPatcher) 通过对客户端二进制做特征 patch，实现 PC 微信 / QQ 的防撤回等能力。特征是跟**具体版本**绑定的，客户端一升，偏移和指令序列往往就失效。

社区 PR [huiyadanli/RevokeMsgPatcher#1193](https://github.com/huiyadanli/RevokeMsgPatcher/pull/1193)（`fix: update QQNT anti-recall patterns`）把 QQNT 防撤回特征更新到了：

| 客户端 | PR / 工具侧可用上限（本文写作时） |
|--------|-----------------------------------|
| **QQNT** | **9.9.20-36330** |
| **微信（Weixin 4.x）** | **4.1.9.30** |

也就是说：

- 你装了更高版本的 QQNT / 微信，这个 PR 里的规则**不一定还能匹配**；
- 腾讯客户端默认会**后台检查 + 热更新/补丁下载**，一重启就可能悄悄升上去；
- 想稳定用防撤回，就得先把版本**钉住**，再跑 patch 工具。

本文记录在 Windows 宿主机上实际踩过的路径：只拦 `dldir*.qq.com` **不够**；QQNT 和微信 4.x 的更新通道、本地缓存位置都不一样。

> 说明：这是**延后升级**，不是永久锁死。服务端若强制最低版本，旧客户端仍可能无法登录。也不要用来路不明的「免更新破解版」。

---

## 历史版本从哪里下

当前版本已经高于 patch 上限时，需要先**降回可用版本**，再立刻做防更新。不要去搜来路不明的「绿色版 / 免更新破解包」；社区有专门归档安装包的仓库（Release 资产，可自行核对哈希）：

| 客户端 | 历史版本仓库 |
|--------|----------------|
| **QQNT** | [PRO-2684/qqnt-version-history](https://github.com/PRO-2684/qqnt-version-history) |
| **微信 Windows** | [cscnk52/wechat-windows-versions](https://github.com/cscnk52/wechat-windows-versions) |

配合本文目标上限时，优先找：

- QQNT：`9.9.20-36330`（或 PR / 工具说明支持的最近一档）
- 微信：`4.1.9.30`（Weixin 4.x）

建议流程：

1. 卸载当前过高版本（或至少退出客户端）  
2. 从上述仓库的 **Releases** 下载对应安装包  
3. 安装完成后**先不要登录**，立刻做本文后面的 hosts / 清缓存 / 锁目录  
4. 再登录，并运行 RevokeMsgPatcher  

> 仓库由第三方维护，安装包是否完整、是否被服务端拒绝登录，以你本机实测为准；下载后尽量核对 Release 里公布的校验信息。

---

## 先确认当前版本

### QQNT

安装目录常见：

```text
C:\Program Files\Tencent\QQNT\
```

看当前热更新配置：

```text
C:\Program Files\Tencent\QQNT\versions\config.json
```

关键字段：

```json
{
  "curVersion": "9.9.20-36330",
  "readyVersion": "",
  "readyType": ""
}
```

- `curVersion`：正在用的版本目录名  
- `readyVersion` 非空：本地**已经下好/解压好**，提示「新版本已就绪」，重启就会切过去  

进程命令行里也能看到：

```text
--app-path="C:\Program Files\Tencent\QQNT\versions\9.9.20-36330\resources\app"
```

### 微信（新版 Weixin 4.x）

新版安装目录是 `Weixin`，不是老的 `WeChat`：

```text
C:\Program Files\Tencent\Weixin\
C:\Program Files\Tencent\Weixin\4.1.9.30\
```

用户数据与更新缓存：

```text
%AppData%\Tencent\xwechat\
%AppData%\Tencent\xwechat\update\
```

更新组件：

```text
%AppData%\Tencent\xwechat\xplugin\plugins\WeixinUpdate\<id>\extracted\WeixinUpdate.exe
```

---

## 总原则：更新分两段

| 阶段 | 含义 | 只改 hosts 够不够 |
|------|------|-------------------|
| 检查更新 | 问服务器有没有新版本 → **弹窗** | 通常**不够** |
| 下载 / 落地 | 把安装包或热更新包写到本地 | **hosts / 权限 / 防火墙** 能挡 |
| 已就绪 | 包已在本地，`readyVersion` 或 `update\patch` 已齐 | **必须删本地缓存**，光拦域名没用 |

所以会出现：

- hosts 已生效，但客户端仍提示「发现新版本 / 新版本已就绪」——正常；
- 点「立即更新」仍能下——说明走了**别的域名**，或**IPv6 / 已建立连接**没被拦；
- 点了就能装——说明包**早就下完了**。

---

## 一、QQNT：真正的热更新域名不是 dldir

### 1. 特征串（从 `major.node` 里能抠到）

```text
https://qqpatch.gtimg.cn
https://qqpatch.gtimg.cn/hotUpdate_new
https://qqpatch.gtimg.cn/dynamic_module/
```

实际下载时 DNS 缓存里还出现过腾讯下载 CDN 变体，例如：

```text
downv6.ydcdn.tcdnos.com
downv6.aly.tcdnos.com
```

（会随线路变化，以本机 DNS 缓存 / 抓包为准。）

### 2. 本地落地位置

热更新包会进：

```text
C:\Program Files\Tencent\QQNT\versions\
  9.9.20-36330\          ← 当前版本目录
  9.9.xx-xxxxx\          ← 已解压的新版本
  9.9.xx-xxxxx.zip       ← 下载中的包（有时还会出现奇怪的 .zip.zip）
  config.json
```

若 `config.json` 里已有：

```json
"readyVersion": "9.9.32-51246",
"readyType": "HOT_UPDATE"
```

则界面就是「新版本已就绪」——**不必再下载**。

### 3. hosts（管理员编辑）

路径：`C:\Windows\System32\drivers\etc\hosts`

建议同时写 **IPv4 + IPv6**（只写 `127.0.0.1` 时，IPv6 仍可能通）：

```text
# QQNT 热更新
127.0.0.1 qqpatch.gtimg.cn
::1 qqpatch.gtimg.cn

# 实测出现过的下载 CDN（可按抓包增减）
127.0.0.1 downv6.ydcdn.tcdnos.com
::1 downv6.ydcdn.tcdnos.com
127.0.0.1 downv6.aly.tcdnos.com
::1 downv6.aly.tcdnos.com
127.0.0.1 down.ydcdn.tcdnos.com
::1 down.ydcdn.tcdnos.com
127.0.0.1 down.aly.tcdnos.com
::1 down.aly.tcdnos.com
127.0.0.1 downv6.tcdnos.com
::1 downv6.tcdnos.com
127.0.0.1 down.tcdnos.com
::1 down.tcdnos.com
```

然后：

```bat
ipconfig /flushdns
```

验证：

```bat
ping qqpatch.gtimg.cn
```

应指向 `127.0.0.1`。

### 4. 清掉「已就绪」的新版本

1. 托盘彻底退出 QQ  
2. 删除高于目标版本的目录和 zip，例如不要的 `9.9.32-*`  
3. 编辑 `versions\config.json`，保证：

```json
"curVersion": "9.9.20-36330",
"readyVersion": "",
"readyInstaller": "",
"readyType": ""
```

4. 再启动 QQ  

可选加固：给 `versions` 目录去掉普通用户的写入权限（过狠可能影响正常运行，慎用）。

### 5. 副作用

- `qqpatch.gtimg.cn` 还承载部分主题 / 动态资源，拦截后个别皮肤资源可能异常。  
- `tcdnos.com` 若被其它腾讯下载共用，也可能受影响。

---

## 二、微信 Weixin 4.x：dldir + 本地 update 目录

### 1. 二进制里的下载入口

`Weixin.dll` 里能看到类似：

```text
https://dldir1.qq.com
https://dldir1v6.qq.com
https://dldir1v6.qq.com/weixin/Universal/Windows/XPlugin/updateConfigUniWin.xml
https://support.weixin.qq.com/update/
```

老办法「只 hosts 几个 dldir」对**下载**有帮助，但：

1. 要补 **IPv6 `::1`**，否则 v6 线路可能绕过；  
2. 若 `%AppData%\Tencent\xwechat\update\` 里已经有完整 patch，点更新照样能装。

### 2. 本地更新缓存长什么样

示例（版本号按你机器上的为准）：

```text
%AppData%\Tencent\xwechat\update\
  download\4.1.11.55\Weixin.dll.zip
  download\4.1.11.55\Weixin.exe.zip
  download\4.1.11.55\patch.zip
  patch\4.1.11.55\Weixin.dll
  patch\4.1.11.55\Weixin.exe
  ...
```

同时进程列表里可能出现：

```text
WeixinUpdate.exe
```

路径类似：

```text
%AppData%\Tencent\xwechat\xplugin\plugins\WeixinUpdate\9013\extracted\WeixinUpdate.exe
```

### 3. hosts

```text
# 微信 / 腾讯通用下载域（IPv4 + IPv6）
127.0.0.1 dldir1.qq.com
::1 dldir1.qq.com
127.0.0.1 dldir1v6.qq.com
::1 dldir1v6.qq.com
127.0.0.1 dldir2.qq.com
::1 dldir2.qq.com
127.0.0.1 dldir3.qq.com
::1 dldir3.qq.com
127.0.0.1 dldir4.qq.com
::1 dldir4.qq.com
127.0.0.1 dldir5.qq.com
::1 dldir5.qq.com
127.0.0.1 dldir6.qq.com
::1 dldir6.qq.com
```

```bat
ipconfig /flushdns
```

### 4. 删缓存 + 锁 update 目录

1. 退出微信（含托盘）  
2. 结束残留的 `WeixinUpdate.exe`  
3. 删除整个：

```text
%AppData%\Tencent\xwechat\update
```

4. 重建空目录后，对 `update`：**拒绝当前用户 / Users 的写入、修改、删除**（Administrators / SYSTEM 保留完全控制）。  

这样即使检查到更新，也**写不进缓存**。

### 5. 防火墙拦 WeixinUpdate.exe（推荐）

`Win + R` → `wf.msc` → **出站规则** → 新建规则 → 程序 → 选中上述 `WeixinUpdate.exe` → **阻止连接**。

插件版本目录（如 `9012` / `9013`）可能并存，每个路径各建一条，或更新后检查是否换了路径。

主程序 `Weixin.exe` **不要**整进程出站全拦，容易直接没法登录/收消息。

### 6. 副作用

- `dldir*.qq.com` 是腾讯通用下载域，QQ 安装包、其它腾讯软件下载也可能受影响。  
- 锁 `update` 后，官方热更新基本失效——对「钉死 4.1.9.30」是预期行为。

---

## 三、推荐操作顺序（想钉在 RevokeMsgPatcher 可用版本）

以目标 **QQNT 9.9.20-36330**、**微信 4.1.9.30** 为例：

1. **先确认当前就是目标版本**；若更高，从历史版本仓库装回：
   - QQNT：[PRO-2684/qqnt-version-history](https://github.com/PRO-2684/qqnt-version-history)
   - 微信：[cscnk52/wechat-windows-versions](https://github.com/cscnk52/wechat-windows-versions)  
2. **立刻**改 hosts（QQNT 的 `qqpatch` + 微信的 `dldir`，都带 `::1`）。  
3. **立刻**清 QQNT `versions` 里多余版本 + 清空 `readyVersion`。  
4. **立刻**清微信 `xwechat\update`，并锁写 + 拦 `WeixinUpdate.exe`。  
5. 再运行 [RevokeMsgPatcher](https://github.com/huiyadanli/RevokeMsgPatcher)（含 PR #1193 那套特征）。  
6. 日常少点「立即更新」；看到「已就绪」优先当本地缓存，先清再谈拦域名。

---

## 四、怎么确认拦截生效

### DNS

```bat
ping qqpatch.gtimg.cn
ping dldir1v6.qq.com
```

应解析到 `127.0.0.1` / 本机。

### QQNT

- `versions` 下不应再长出新的大体积 zip / 新版本目录；  
- `config.json` 的 `readyVersion` 应保持空。  

### 微信

- `xwechat\update` 下不应再出现 `download\` / `patch\` 大文件；  
- 若仍弹更新但装不上、写目录失败，说明权限锁在起作用。  

### 仍能下载时

1. 在下载过程中看 DNS 缓存：

```powershell
Get-DnsClientCache | Where-Object { $_.Entry -match 'qq|weixin|gtimg|tcdnos|dldir' }
```

2. 看进程出站连接（`Get-NetTCPConnection -OwningProcess <pid>`），把新域名补进 hosts。  
3. 已建立的 TCP 连接不会因改 hosts 立刻断，需退出客户端再试。

---

## 五、和 RevokeMsgPatcher 的关系（再强调一次）

防更新的目的不是「永远不用新版本」，而是：

> **把客户端钉在 patch 特征仍然匹配的版本上，避免热更新把防撤回打失效。**

PR #1193 侧当前明确测过 / 面向的上限是：

- QQNT：`9.9.20-36330`（含私聊 C2C 被动撤回路径）  
- 微信：`4.1.9.30`  

版本一涨，需要重新找特征、更新 `patch.json` / 资源，而不是指望旧二进制一直能用。  
上游合并进度、后续版本支持范围以仓库与 PR 讨论为准：  
[https://github.com/huiyadanli/RevokeMsgPatcher/pull/1193](https://github.com/huiyadanli/RevokeMsgPatcher/pull/1193)

---

## 六、恢复方法（简要）

1. 从改 hosts 前的备份恢复 `C:\Windows\System32\drivers\etc\hosts`，或删掉本文相关段落。  
2. `ipconfig /flushdns`  
3. 删除防火墙规则「Block WeixinUpdate outbound*」  
4. 恢复 `xwechat\update` 目录权限（去掉 Deny 写入）  
5. 需要升级时再走官方更新即可  

---

## 小结

| 客户端 | 关键域名 | 本地缓存 | 额外手段 |
|--------|----------|----------|----------|
| QQNT | `qqpatch.gtimg.cn`，以及 `*.tcdnos.com` 下载 CDN | `QQNT\versions\` + `config.json` 的 `readyVersion` | 删已就绪版本目录 |
| 微信 4.x | `dldir*.qq.com`（补 IPv6） | `%AppData%\Tencent\xwechat\update\` | 锁目录 + 拦 `WeixinUpdate.exe` |

**只拦 dldir、不管本地已就绪包，是最常见的「hosts 设了还是在更」原因。**  
为了配合 RevokeMsgPatcher 的版本上限，应把「拦域名 + 清缓存 + 必要时权限/防火墙」当成一套流程，而不是单点技巧。
