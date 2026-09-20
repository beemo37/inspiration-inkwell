# macOS 本地网络权限导致内网访问异常 —— 完整排查记录

> 本文所有域名、IP 地址、网段与配置标识符均已替换为示例值，不影响结论与操作步骤。
> 网页版（带排版）：<https://beemo37.github.io/inspiration-inkwell/>


> 排查日期：2026-09-20
> 机器：Apple M1 Pro，macOS（Darwin 27.0.0），FileVault 已开启
> 结论：**已根治**（方案 B）

---

## 一、摘要

| | |
|---|---|
| **现象** | 连上 VPN 后，公司内网部分地址打不开，且"时好时不好"；同一根网线在 Windows 上正常 |
| **根因** | macOS 的「本地网络」权限表 `/Library/Preferences/com.apple.networkextension.plist` 中，`com.google.Chrome` 名下堆积了 68 条指向已删除临时目录的僵尸记录，导致系统设置里的开关**点了不生效** |
| **为什么只有 Chrome 中招** | 其他 app 只有 1 条干净记录，开关能改；Chrome 有 77 条，改到不存在的那些就整批回滚 |
| **为什么"时好时不好"** | 不是随机 —— 取决于目标地址在不在**直连网段**内（见 [§5.4](#54-为什么时好时不好)） |
| **临时绕过** | `chrome-lan` 脚本 / `Chrome 内网.app`（用"终端身份"启动 Chrome） |
| **根治** | 关 SIP → 摘掉坏掉的 networkprivacy 配置 → 系统自动重建干净的 → 开回 SIP |

---

## 二、问题背景

### 2.1 使用场景

公司内网开发，需要访问：

| 地址 | 用途 | 位置 |
|---|---|---|
| `ci.example.com` → `10.20.11.34` | Jenkins | 直连网段内 |
| `10.20.11.185:1521` | Oracle 开发库 | 直连网段内 |
| `10.20.9.1` | 内网服务 | 直连网段内 |
| `10.20.65.x` | maven / test-dify | 直连网段外 |

### 2.2 网络环境

| 接口 | 地址 | 网关 |
|---|---|---|
| `en8`（网线） | `10.20.6.182/20` | `10.20.0.49` |
| `en0`（WiFi） | `10.99.6.61/21` | `10.99.0.253` |

**关键**：`en8` 的掩码是 `/20`，覆盖 `10.20.0.0 – 10.20.15.255`。
这整个范围对 macOS 来说都是「**同一个网段**」，也就是「**本地网络**」。

---

## 三、现象（按发现顺序）

1. **部分内网地址打不开，跟没挂 VPN 一样**，并且"时好时不好使"
2. **Safari 能打开 `http://ci.example.com/login`，Chrome 报 `ERR_ADDRESS_UNREACHABLE`**
   —— 同一个地址、同一时刻，两个浏览器结果不同
3. **同一根网线插在公司的 Windows 机器上完全正常**
4. **Oracle `10.20.11.185:1521` 一直连不上**
5. 在系统设置里给 Navicat 授予本地网络权限 → **Navicat 立刻好了**
6. 用同样方式给 Chrome 授权 → **没有任何反应**，依然打不开
7. 「本地网络」列表里出现 **N 个 Chrome 条目**（豆包也有两个）
8. 拔网线改用 WiFi 后，`ci.example.com` 和 `10.20.9.1` **反而能访问了**
   —— 但 Navicat 连数据库反而连不上了
9. 全程**从未弹出过**"XXX 想要访问本地网络"的授权框

---

## 四、排查过程

### 4.1 先被证伪的（避免重复劳动）

| 假设 | 验证方法 | 结果 |
|---|---|---|
| 系统代理 / 代理环境变量 | 查网络设置与 `env` | ❌ 无代理 |
| 第三方防火墙 | 查已安装的安全软件、内核扩展 | ❌ 无 |
| macOS 自带防火墙 (ALF) | `socketfilterfw --getglobalstate` | ❌ 未启用 |
| VPN 路由表劫持 | `netstat -rn`、抓包 | ❌ 路由正常 |
| DNS 轮询到不同 IP | `dig` 多次 | ❌ DNS 一致 |
| Chrome 代码签名损坏 | `codesign -vvv` | ❌ 签名有效 |
| Chrome 版本冲突 | 查 `Current` 软链指向 | ❌ 曾经误判，已更正 |
| 重启能解决 | 重启 | ❌ 无效 |
| 权限记在 TCC.db | 查 `~/Library/Application Support/com.apple.TCC/TCC.db` | ❌ **根本没有本地网络这一项** |
| 是 macOS 版本问题 | 同版本其他机器 | ❌ **其他同系统的机器没这问题** |

> 第 9 条是**转折点**。通常的隐私权限（摄像头、麦克风、屏幕录制）都记在 TCC.db，
> 但「本地网络」不在这里 —— 它属于网络扩展框架。

### 4.2 找到真正的权限存储位置

```
/Library/Preferences/com.apple.networkextension.plist
```

这是一个 **NSKeyedArchiver 归档的二进制 plist**，结构为：

```
{
  "$version": ...,
  "$archiver": "NSKeyedArchiver",
  "$top": { "<UUID>" → <配置对象>, ... },
  "$objects": [ ... 扁平化的对象池 ... ]
}
```

`$top` 是一张 UUID → 配置 的表。解开后有 3 条配置：

| UUID | 是什么 | 处理 |
|---|---|---|
| `5E6F7A8B-2222-4222-8222-222222222222` | **CorpVPN（L2TP）** | ⚠️ **必须保留** |
| `9C0D1E2F-3333-4333-8333-333333333333` | `application-firewall` | 保留 |
| `1A2B3C4D-1111-4111-8111-111111111111` | `com.apple.preferences.networkprivacy-3A4B5C6D-...` | 🔴 **坏的就是它** |

另有 `SCPreferencesSignature2` —— 内容签名，改动后必须一并清除，否则系统判定"被篡改"。

### 4.3 解开权限表

`networkprivacy` 配置 → `PathController` → `Rules`，是一个**per-app 的规则数组**。

每条规则的字段：

| 字段 | 含义 |
|---|---|
| `SigningIdentifier` | app 的签名标识，如 `com.google.Chrome` |
| `Path` | 具体路径，`$null` 表示不限定 |
| **`DenyMulticast`** | **`False` = 已授权本地网络；`True` = 拒绝** |
| `MulticastPreferenceSet` | `True` = **用户/系统明确设置过**；`False` = 默认值 |
| `AllowEmptyDesignatedRequirement` | 允许空签名要求 |
| `NoDivertDNS` / `DenyCellularFallback` | 其他策略位 |

> **注意字段名：`DenyMulticast`** —— 拒绝**组播**。
> 这个权限的本质是管**广播/组播**的（app 在局域网里广播找人），
> 这解释了后面很多东西。

### 4.4 发现兜底规则（解释了一切）

```
拒绝  PathRuleDefaultNonSystemIdentifier
      DenyMulticast=True    MulticastPreferenceSet=False    Path=$null
```

**这是一条 Catch-all 规则**：所有**没有**自己条目的非系统 app，直接落到它上面，**静默拒绝** —— 不弹框、不提示、连一条记录都不产生。

这台机器的策略因此可以总结成：

| app 类型 | 结果 |
|---|---|
| 系统自带（Safari 等） | 不受管 → ✅ 通 |
| 非系统 app，有"放行"记录 | ✅ 通 |
| 非系统 app，有"拒绝"记录 | ❌ 拦 |
| 非系统 app，**没有记录** | ❌ 落到兜底规则 → **静默拦** |

这一条同时解释了：

- **为什么 Safari 行、Chrome 不行** —— Safari 是系统 app，不受管
- **为什么从来没弹过框** —— 新 app 走兜底规则，系统不问直接拒
- **为什么自己打包的 app 也不行** —— 走了同一个兜底

### 4.5 找到 Chrome 的僵尸记录

解出修复前的记录数：

| | 规则总数 | Chrome 相关 | 僵尸记录 |
|---|---|---|---|
| **修复前** | 181 | 77 | **68** |
| **修复后** | 111 | 11 | **0** |

那 68 条僵尸记录的 `Path` 长这样：

```
$TMPDIR/X/xxxxxxxx/code_sign_clone.xxxxxxxx/Google Chrome.app/...
```

这是 **Chrome 自动更新时留下的**：更新程序把 app 复制到临时目录、签名、再替换回去。
临时目录随后被清理，但 macOS 当时给它建的权限记录**留了下来**。

于是就有了一开始的疑问答案 —— **「为什么有 N 个 Chrome」**：里面只有一个是真的，其余全是这种已经不存在的路径。

**这也解释了"为什么开关点了没用"**：你在系统设置里打开 Chrome 的开关时，系统需要把全部 77 条规则**原子地**改一遍；改到那 68 条不存在的路径时失败，**整批回滚**。表面看开关动了，实际一条没改。

> 对比验证：Navicat 只有 1 条干净记录，所以开关一开就生效。
> 这是"为什么 Navicat 行、Chrome 不行"的完整答案。

### 4.6 为什么"终端里启动 Chrome"就可以

macOS 的这类权限按**责任进程（responsible process）**判定，而不是按二进制文件本身：

| 启动方式 | 责任进程 | 结果 |
|---|---|---|
| 访达 / Dock 双击 | `com.google.Chrome` | ❌ 拦 |
| 终端里 `nohup` 拉起 | `com.apple.Terminal` | ✅ 通（终端是放行的） |

**注意**：Chrome 已经开着的时候，再执行一次它的二进制，只是把 URL **递给已经在跑的老进程**开个新窗口，身份不会变。所以必须先 `Cmd+Q` 完全退出。

### 4.7 时好时不好的真相 —— 地址分段

这是最后一个谜题。用 `netstat` / `traceroute` 对比后：

| 目标 | 是否在 `10.20.0.0/20` 内 | 走法 | 受权限管吗 | 结果 |
|---|---|---|---|---|
| `10.20.11.34` (ci) | ✅ 在 | **直连** | **受管** | 网线下 ❌ 时好时坏 |
| `10.20.11.185` (Oracle) | ✅ 在 | **直连** | **受管** | 网线下 ❌ |
| `10.20.9.1` | ✅ 在 | **直连** | **受管** | 网线下 ❌ |
| `10.20.65.x` (maven) | ❌ 不在 | 走网关 | 不受管 | ✅ 一直正常 |

**在 WiFi 下**（`10.99.6.61/21`），`10.20.x` 全都**不再是邻居**，一律走网关 `10.99.0.253` → **不受管**

这就完美解释了：

- **为什么某些内网地址一直好用** —— 它们本来就走网关，压根不在闸门管辖范围
- **为什么换 WiFi 后 ci 反而能访问了** —— 走网关了，绕开了权限闸门
- **为什么换 WiFi 后连不上数据库** —— 那个 WiFi 网段本身到不了 `10.20.11.185:1521`，是**真·网络不通**，跟权限无关
- **为什么"上午好使下午不好使"** —— 在不同网段的目标之间来回切，看起来像随机

> 深层原因：`172.16.0.0/12` 是 RFC1918 私有地址，公网不可路由。
> 公司 WiFi 到 `10.20.x` 的路径走的是公司内部路由，能到哪一段取决于那个网段的策略。

---

## 五、根因

### 5.1 一句话

**macOS 的「本地网络」权限表里，Chrome 名下有 68 条指向已删除目录的僵尸记录，导致授权开关失效；而公司内网的一批关键地址恰好落在"直连网段"里，正受这个权限管辖。**

### 5.2 这个权限到底管什么

这是最容易误解的地方：

| | 受这个权限管吗 |
|---|---|
| 访问公网（GitHub、扩展市场） | ❌ **不受管** |
| `localhost` / 回环 / Docker | ❌ **不受管** |
| 走网关出去的目标 | ❌ **不受管** |
| **同网段直连的机器** | ✅ **受管** |
| **广播 / 组播发现** | ✅ **受管**（字段名就叫 `DenyMulticast`） |

所以它不是"能不能上网"的开关，而是"**能不能够到隔壁机器**"的开关。

### 5.3 "本地网络" 具体包括什么

| 场景 | 范围 |
|---|---|
| 在家（WiFi） | 手机、平板、另一台电脑、打印机、NAS、电视/音箱、路由器后台（`192.168.x.x`） |
| 在公司（网线） | `10.20.0.0 – 10.20.15.255` —— 所有开发机、ci、Oracle、内网系统 |
| 在公司（WiFi） | `10.99.x.x` 那一段 |

### 5.4 为什么"时好时不好"

见 [§4.7](#47-时好时不好的真相--地址分段)。**它从来不是随机的**，取决于目标地址在不在直连网段内。

---

## 六、解决方案

### 6.1 方案 A：绕过（临时）

**思路**：既然 `com.apple.Terminal` 是放行的，那就**让终端去启动 Chrome** —— 责任进程变成终端，绕开闸门。

**优点**：不改任何系统配置，不需要关 SIP，随时可用
**缺点**：每次都要先完全退出 Chrome；是个 workaround，不是修复

三个层次的工具（见 [附录 A](#附录-a工具源码)）：

| 工具 | 位置 | 用法 |
|---|---|---|
| `chrome-lan` | `~/.local/bin/chrome-lan` | 终端里敲 `chrome-lan` |
| `Chrome 内网.command` | 桌面 | 双击 |
| `Chrome 内网.app` | 桌面 | 双击（Dock 里能放正经图标） |
| `lan-run` | `~/.local/bin/lan-run` | 通用版，任何 app：`lan-run Navicat` |

> **`Chrome 内网.app` 的关键设计**：app 自己**不**拉起 Chrome —— 实测那样会被静默拦掉
> （adhoc 签名的 app 在系统眼里"来路不明"，不弹框，直接拒）。
> 它只做一件事：`open -a Terminal` 把活儿交给终端，自己当门面。

### 6.2 方案 B：根治（推荐，已执行）

**思路**：把坏掉的那条 `networkprivacy` 配置整个摘掉，让 macOS 重新生成一份干净的。

**前提**：文件受 SIP 保护（`/Library/Preferences` 带 `sunlnk` 标志 + 根卷 sealed），
**连 root 都写不进去**，必须先关 SIP。

**步骤**（详细版见桌面上的 `B-操作步骤.txt`）：

```
第 1 段  恢复模式 → csrutil disable → 重启
         (M1 芯片：长按电源键进恢复模式，不是 Cmd+R)
         (FileVault 开着：要先选磁盘、输密码)

第 2 段  正常系统 → sudo bash ~/corp-vpn-fix-lan-permission.sh
         看到 "=> 写入成功" → 立刻 sudo reboot

第 3 段  验证：Chrome 能开 ci.example.com；CorpVPN 还在；
         重新给 Navicat 授权

第 4 段  恢复模式 → csrutil enable → 重启
```

**脚本关键设计**（`corp-vpn-fix-lan-permission.sh`）：

1. **只摘坏的那一条**，保留 CorpVPN 和防火墙配置 —— 不能整个文件删掉
2. **同目录临时文件 + 原子替换**（`mv -f`），不是 `cat >` 直接覆盖
   —— 避免写到一半中断把原文件弄残
3. 写入前后各校验一次，任何一步失败都明确报告"**原文件未动**"
4. 自动清掉 `SCPreferencesSignature2`，让系统重新签名
5. 自带 SIP 检查，没关 SIP 直接告诉你，不白忙

### 6.3 执行结果

| | 修复前 | 修复后 |
|---|---|---|
| 文件大小 | 58,788 字节 | 33,690 字节 |
| 规则总数 | 181 | 111 |
| Chrome 相关 | 77 | **11** |
| `code_sign_clone` 僵尸 | **68** | **0** |
| networkprivacy UUID | `1A2B3C4D-...`（坏） | `7E8F9A0B-...`（系统重建） |
| CorpVPN 配置 | ✅ | ✅ **保留** |
| 防火墙配置 | ✅ | ✅ **保留** |

**结果**：Chrome 直接（不经终端）就能打开 `ci.example.com` ✅

### 6.4 副作用（重要）

重建配置会**清掉之前手动授过的权限**。修复后需要重新授权：

- `com.navicat.NavicatPremiumLite`（+ `.Launcher`）
- `org.jkiss.dbeaver.core.product`（如果还用 DBeaver）

授权方式：**系统设置 → 隐私与安全性 → 本地网络 → 打开开关**
（这次开关是真的能生效了，因为僵尸记录已经清掉了）

> 修复后仍是"放行"的只有 4 条：`com.google.Chrome`、`com.navicat.NavicatPremiumLite`、
> `com.microsoft.VSCode`、`node`。其余 107 条都是系统预填的默认拒绝。

---

## 七、以后再弹"是否允许访问本地网络"怎么判断

### 7.1 判断口诀

> **「这个 app 会不会去够另一台机器？」**

- 会（连数据库、开内网网页、投屏、打印、给旁边的设备传文件）→ **允许**
- 不会（纯工具、纯联网服务）→ **拒绝**

### 7.2 该允许

| 类型 | 例子 |
|---|---|
| 浏览器 | Chrome / Safari / Edge —— 要开内网系统、路由器后台 |
| 数据库客户端 | Navicat / DBeaver / Redis Desktop |
| 开发工具 | 终端 / VSCode / IntelliJ / SSH 工具（**内置终端里跑的也算**） |
| 投屏 / 远程控制 | 向日葵 / ToDesk / UU远程 |
| 打印 / 扫描 | 打印机驱动 |
| NAS / 网盘客户端 | 群晖、极空间 |
| 虚拟机 / Docker | 桥接网络时算本地 |

### 7.3 该拒绝

AI 助手（豆包、ChatGPT 桌面版）、输入法、影音播放、下载/解压、截图/录屏、纯云同步网盘。

### 7.4 看情况

- **微信 / 企业微信 / 飞书** —— 同一 WiFi 下传文件、会议投屏**会走局域网加速**
- **游戏** —— 局域网联机才需要
- **远程控制软件** —— 见下

### 7.5 远程控制软件的特例（UU远程 / 向日葵 / ToDesk）

**「支持公网」和「需要本地权限」不矛盾**，一个 app 可以两条路都走：

| 场景 | 走哪条路 | 受管吗 |
|---|---|---|
| 在家 → 公司机器 | 公网中转 | ❌ |
| 在公司，两端**不同网段** | 公网中转 | ❌ |
| 在公司，两端**同网段** | **局域网直连**（快得多） | ✅ |

它的**失败方式比较温和**：局域网直连被拦 → 退回公网中转 → **还能连上，只是慢、画质差**。
所以不会报错，只会觉得"今天怎么这么卡"。

### 7.6 最重要的原则：拿不准就允许

| | 代价 |
|---|---|
| **允许错了** | 这个 app 能扫描你局域网里的设备（对方是你在用的商业软件，实际风险很小） |
| **拒绝错了** | **静默失败** —— 不弹框、不报错、没有任何提示，就是本记录里查了一整天的那种 |

**这个不对称性是关键。** 还要意识到：它是**隐私控制，不是防火墙** ——
拦的是"乱扫内网"，拦不住"把数据传到公网"。

---

## 八、值得记住的几条

1. **macOS 的「本地网络」权限不在 TCC.db**，在 `/Library/Preferences/com.apple.networkextension.plist`
2. **它的字段名是 `DenyMulticast`** —— 本质是管广播/组播的
3. **它按"责任进程"判定**，不是按二进制文件 —— 谁启动的，就算谁的
4. **它只管同网段 + 广播**，不管公网、不管回环、不管走网关的
5. **`/20` 掩码会让一大片地址都变成"隔壁机器"** —— 公司的 `10.20.0.0/20` 就是这样
6. **同一个地址，对你是"内网系统"，对 macOS 是"隔壁的机器"** —— 同一件事的两个名字
7. **"点开关没反应"通常意味着那个 app 名下有坏记录**，整批更新失败回滚
8. **诊断这类问题先看地址在不在直连网段**，能省掉一大半时间

---

## 附录 A：工具源码

### A.1 `~/.local/bin/chrome-lan`

```sh
#!/bin/sh
# chrome-lan —— 用"终端身份"启动 Chrome, 绕过本地网络权限。
#
# 背景 (2026-09-20 查出来的):
#   macOS 的本地网络权限不是记在 TCC.db 里, 而是记在
#     /Library/Preferences/com.apple.networkextension.plist
#   的 networkprivacy 配置 -> PathController -> 路径规则里。
#   每条规则有个 DenyMulticast 字段, False = 已授权本地网络。
#
#   这个权限是按"责任进程 (responsible process)"判定的:
#     - 从访达/Dock 启动     -> 算 com.google.Chrome -> 拦, 172.x 内网连不上
#     - 由终端 fork 出来     -> 算 com.apple.Terminal -> 放行
#   公网不受影响, 因为这道闸门只管本地/私有网段。这就解释了
#   "外网都好、只有内网时好时不好"。
#
# 前提 —— Chrome 必须完全退出:
#   已经有一个 Chrome 在跑的时候, 再执行一次二进制, 只是把 URL 递给那个
#   老进程开个新窗口; 老进程的身份还是"访达身份", 照样不通。
#   所以本脚本检测到 Chrome 在跑就直接拒绝, 不会假装成功。
#
# 撤销: 删掉本文件即可。它不写任何系统配置。

set -u

CHROME="/Applications/Google Chrome.app/Contents/MacOS/Google Chrome"

if [ ! -x "$CHROME" ]; then
  echo "chrome-lan: 找不到 Chrome —— $CHROME" >&2
  exit 1
fi

if pgrep -x "Google Chrome" >/dev/null 2>&1; then
  cat >&2 <<'EOF'
chrome-lan: Chrome 正在运行, 不能直接启动。

  先完全退出 Chrome (Cmd+Q), 再执行本命令。

  原因: 正在跑的那个进程是"访达身份"起的, 本地网络权限没批下来。
        再启动一次只会往它里面塞个新窗口, 身份不会变, 内网照样打不开。
EOF
  exit 1
fi

# nohup: 关掉终端窗口后 Chrome 别跟着死。
nohup "$CHROME" >/dev/null 2>&1 &
pid=$!
disown 2>/dev/null || true

echo "chrome-lan: 已用终端身份启动 Chrome (pid $pid)"
echo "chrome-lan: 验证 —— 打开 http://ci.example.com/ , 应该出 Jenkins 登录页。"
```

### A.2 `~/.local/bin/lan-run`（通用版）

```sh
#!/bin/sh
# lan-run —— 用"终端身份"启动任意 app, 绕过本地网络权限。
#
# 用法:
#   lan-run /Applications/Navicat\ Premium\ Lite.app
#   lan-run "/Applications/DBeaver.app"
#   lan-run Navicat          (自动去 /Applications 和 ~/Applications 找)
#
# 前提: 目标 app 必须完全退出 (Cmd+Q)。
# 撤销: 删掉本文件即可, 不写任何系统配置。

set -u

usage() {
  cat >&2 <<'EOF'
用法: lan-run <app 路径 或 可执行文件路径>

例子:
  lan-run "/Applications/Navicat Premium Lite.app"
  lan-run "/Applications/DBeaver.app"
  lan-run Navicat
EOF
  exit 2
}

[ $# -ge 1 ] || usage
target="$*"

# 1. 解析出真正的可执行文件
if [ -d "$target" ] && [ "${target%.app}" != "$target" ]; then
  exe_name=$(defaults read "$target/Contents/Info.plist" CFBundleExecutable 2>/dev/null)
  if [ -z "$exe_name" ]; then
    echo "lan-run: 读不出 $target 的 CFBundleExecutable" >&2
    exit 1
  fi
  exe="$target/Contents/MacOS/$exe_name"
elif [ -x "$target" ]; then
  exe="$target"
else
  found=""
  for dir in /Applications "$HOME/Applications"; do
    for cand in "$dir/$target.app" "$dir/$target"*.app; do
      [ -d "$cand" ] && { found="$cand"; break; }
    done
    [ -n "$found" ] && break
  done
  if [ -z "$found" ]; then
    echo "lan-run: 找不到 $target" >&2
    exit 1
  fi
  echo "lan-run: 匹配到 $found"
  exe_name=$(defaults read "$found/Contents/Info.plist" CFBundleExecutable 2>/dev/null)
  exe="$found/Contents/MacOS/$exe_name"
fi

[ -x "$exe" ] || { echo "lan-run: 不是可执行文件 —— $exe" >&2; exit 1; }

# 2. 已经在跑的话, 拒绝, 不假装成功
proc_name=$(basename "$exe")
if pgrep -x "$proc_name" >/dev/null 2>&1; then
  cat >&2 <<EOF
lan-run: $proc_name 正在运行, 不能直接启动。

  先完全退出它 (Cmd+Q), 再执行本命令。

  原因: 正在跑的那个是"访达身份"起的, 本地网络权限没批下来。
        再启动一次只会把它拉到前台, 身份不会变, 内网照样连不上。
EOF
  exit 1
fi

# 3. 用终端身份启动
nohup "$exe" >/dev/null 2>&1 &
pid=$!
disown 2>/dev/null || true

echo "lan-run: 已用终端身份启动 $proc_name (pid $pid)"
echo "lan-run: 这个终端窗口可以关, 但别退出「终端」这个 app 本身。"
```

### A.3 `Chrome 内网.app` 的结构

```
Chrome 内网.app/
├── Contents/
│   ├── Info.plist
│   │     CFBundleIdentifier = com.example.chromelan
│   │     CFBundleExecutable = ChromeLan
│   │     CFBundleIconFile   = AppIcon
│   ├── PkgInfo                       APPL????
│   ├── MacOS/
│   │   └── ChromeLan                 ← 门面：只负责 open -a Terminal
│   └── Resources/
│       ├── AppIcon.icns              ← 从 Chrome 拷的图标
│       └── launch.command            ← 真正的逻辑
```

**`Contents/MacOS/ChromeLan`**（门面）：

```bash
#!/bin/bash
# 本程序自己不碰网络、不启动 Chrome。它只做一件事:
# 让"终端"去执行 Contents/Resources/launch.command。
#
# 为什么绕这一下:
#   由本 app 直接拉起 Chrome 是不行的 —— adhoc 签名的 app 在系统眼里
#   属于"来路不明", 不弹框问, 直接默默按 com.google.Chrome 拦掉
#   (2026-09-20 实测: 不弹框、权限表里连一条记录都不产生)。
#   交给终端去做, 走的就是已验证可行的那条路。

set -u

HERE="$(cd "$(dirname "$0")" && pwd)"
CMD="$HERE/../Resources/launch.command"

if [ ! -f "$CMD" ]; then
  osascript -e 'display notification "app 包不完整, 找不到 launch.command" with title "Chrome 内网"' >/dev/null 2>&1
  exit 1
fi

open -a Terminal "$CMD"
exit 0
```

**`Contents/Resources/launch.command`**（真正的逻辑）：

```bash
#!/bin/bash
# 真正的干活脚本。由 Chrome 内网.app 交给"终端"执行。

set -u

CHROME="/Applications/Google Chrome.app/Contents/MacOS/Google Chrome"

pause() { printf '\n按回车关闭本窗口... '; read -r _; }

if [ ! -x "$CHROME" ]; then
  echo "找不到 Chrome —— $CHROME" >&2
  pause; exit 1
fi

if pgrep -x "Google Chrome" >/dev/null 2>&1; then
  cat <<'EOF'
Chrome 正在运行。

  正在跑的那个是"访达身份"起的, 内网打不开。
  要用内网, 得先把它完全退出, 再用终端身份重新启动。

  好消息: Chrome 重开后会自己恢复上次的标签页。

EOF
  printf '要现在退出并重启 Chrome 吗? [y/N] '
  read -r ans
  case "$ans" in
    y|Y)
      echo "正在退出 Chrome..."
      osascript -e 'quit app "Google Chrome"' >/dev/null 2>&1
      n=0
      while pgrep -x "Google Chrome" >/dev/null 2>&1; do
        n=$((n + 1))
        if [ "$n" -ge 40 ]; then
          echo
          echo "Chrome 十几秒了还没退干净 —— 可能有页面在拦着(比如没保存的表单)。" >&2
          echo "你手动 Cmd+Q 一下, 再点一次 Chrome 内网。" >&2
          pause; exit 1
        fi
        sleep 0.25
      done
      echo "已退出。"
      ;;
    *)
      echo "好的, 没动 Chrome。"
      pause; exit 0
      ;;
  esac
fi

nohup "$CHROME" >/dev/null 2>&1 &
pid=$!
disown 2>/dev/null || true

cat <<EOF

已用终端身份启动 Chrome (pid $pid)
验证: 打开 http://ci.example.com/ , 应该出 Jenkins 登录页。

--------------------------------------------------------
这个终端窗口可以直接关, Chrome 不会跟着退。

以后每次想用内网, 点一下 Chrome 内网.app 就行。
--------------------------------------------------------
EOF
pause
```

> **方案 B 生效后，这套工具就不需要了。** 它们不写任何系统配置，随时可删。

---

## 附录 B：技术细节与命令速查

### B.1 权限表结构

```
/Library/Preferences/com.apple.networkextension.plist
└── $top
    ├── Version / Generation / Index
    ├── 5E6F7A8B-2222-4222-8222-222222222222   ← CorpVPN (L2TP)
    ├── 9C0D1E2F-3333-4333-8333-333333333333   ← application-firewall
    ├── 1A2B3C4D-...  (修复前) / 7E8F9A0B-...  (修复后)  ← networkprivacy
    └── SCPreferencesSignature2                 ← 内容签名
```

`networkprivacy` 配置内部：

```
Name: com.apple.preferences.networkprivacy-3A4B5C6D-4444-4444-8444-444444444444
PathController
├── Enabled: True
├── Rules: [ 每条对应一个 app ]
│   ├── SigningIdentifier
│   ├── Path
│   ├── DenyMulticast           ← False = 已授权
│   ├── MulticastPreferenceSet  ← True = 用户明确设置过
│   └── ...
├── PayloadAppRules
├── IgnoreFallback / IgnoreRouteRules
└── cellularFallbackFlags
```

### B.2 常用命令

**查看某条配置是否还在：**

```bash
python3 -c "import plistlib;t=plistlib.load(open('/Library/Preferences/com.apple.networkextension.plist','rb'))['\$top'];print('坏配置还在' if '1A2B3C4D-1111-4111-8111-111111111111' in t else '已清除');print('CorpVPN', '在' if '5E6F7A8B-2222-4222-8222-222222222222' in t else '不见了!')"
```

**看某条配置的规则（可复用）：**

```python
import plistlib
d = plistlib.load(open('/Library/Preferences/com.apple.networkextension.plist','rb'))
objs = d['$objects']

def unarch(v, depth=0):
    if depth > 90: return None
    if hasattr(v, 'data'):
        try: return unarch(objs[v.data], depth+1)
        except Exception: return None
    if isinstance(v, dict):
        if 'NS.keys' in v and 'NS.objects' in v:
            ks, vs = unarch(v['NS.keys'], depth+1), unarch(v['NS.objects'], depth+1)
            if isinstance(ks, list) and isinstance(vs, list):
                return {str(k): x for k, x in zip(ks, vs)}
            return {}
        if 'NS.objects' in v: return unarch(v['NS.objects'], depth+1)
        return {k: unarch(x, depth+1) for k, x in v.items() if not k.startswith('$')}
    if isinstance(v, list): return [unarch(x, depth+1) for x in v]
    return v

seen = set()
for o in objs:
    if not isinstance(o, dict): continue
    r = unarch(o)
    if not isinstance(r, dict): continue
    n = r.get('Name')
    if not isinstance(n, str) or n in seen or 'networkprivacy' not in n: continue
    seen.add(n)
    for x in r['PathController']['Rules']:
        if not isinstance(x, dict): continue
        print(f"{'拒绝' if x.get('DenyMulticast') else '放行'}  "
              f"{x.get('SigningIdentifier')}  "
              f"设置过={x.get('MulticastPreferenceSet')}")
```

**判断一个地址在不在直连网段：**

```bash
ifconfig | grep 'inet '
# 拿本机地址和掩码，算一下目标在不在同一个子网里
```

**看谁在够本地设备：**

```bash
LC_ALL=C lsof -nP -iTCP -sTCP:ESTABLISHED | awk 'NR>1{print $1"  "$9}' \
  | grep -E '172\.17\.|192\.168\.' | sort -u
```

**SIP：**

```bash
csrutil status          # 查看
csrutil disable         # 关闭（恢复模式里执行）
csrutil enable          # 开启（恢复模式里执行）
```

### B.3 备份文件

| 文件 | 说明 |
|---|---|
| `~/corp-vpn-netpolicy-backup-beforefix-*.plist` | **脚本自动创建，还原用这个** |
| `~/corp-vpn-netpolicy-backup-20260920-131421.plist` | 手工备份（给 Navicat 授权前的状态） |
| `~/corp-vpn-netpolicy-backup-manual-20260920-131859.plist` | 手工备份（同上） |

**还原命令**（需要 SIP 处于关闭状态）：

```bash
sudo cp ~/corp-vpn-netpolicy-backup-beforefix-<时间戳>.plist \
        /Library/Preferences/com.apple.networkextension.plist
sudo reboot
```

### B.4 相关文件清单

| 路径 | 作用 |
|---|---|
| `~/corp-vpn-fix-lan-permission.sh` | 修复脚本（方案 B） |
| `~/Desktop/B-操作步骤.txt` | 方案 B 的详细操作手册 |
| `~/Desktop/B-steps.txt` | 同上，英文文件名（恢复模式里能敲） |
| `~/.local/bin/chrome-lan` | 终端身份启动 Chrome |
| `~/.local/bin/lan-run` | 终端身份启动任意 app |
| `~/Desktop/Chrome 内网.app` | 门面 app |
| `~/Desktop/Chrome 内网.command` | 同上，脚本版 |

---

## 附录 A：方案 B 的详细操作步骤

> 这一段是当时照着一步步敲的操作手册。**所有步骤原样保留**，只把公司内网域名、IP、脚本名换成了示例值。
>
> 恢复模式里看不到这份文档 —— 那是个独立的系统，没有浏览器。事先把这几步抄下来、拍照存手机，
> 或用另一台设备打开。另外注意：**恢复模式的终端默认是美式键盘，中文文件名敲不出来**，
> 涉及文件名一律用英文。

### 开始之前，先知道三件事

1. 操作机器是 Apple M1 Pro。
   进恢复模式是**长按电源键**，不是 Intel 机器那种 Cmd+R。
   别去找 Cmd+R，这机器上没用。

2. 机器开了 FileVault 磁盘加密。
   进恢复模式后要先"选磁盘、输密码"才能到工具菜单，
   这一步是正常的，不是出错。

3. 中途要关掉 SIP。
   SIP 是 macOS 的"系统文件保护"，平时拦着不许改系统文件。
   要修的那个文件被它拦着，所以必须先关。
   关掉的这段时间系统少一层保护 —— 所以做完**必须**开回来（第 4 段）。
   忘了开回来电脑照常能用，只是保护少一层，随时补开都行。

### 全景（先看一眼，心里有数）

```
第 1 段  进恢复模式，关 SIP              → 重启
第 2 段  正常系统里跑修复脚本            → 重启
第 3 段  验证修好了没有
第 4 段  进恢复模式，开回 SIP            → 重启
```

任何一段卡住，都可以原样停下 —— 第 2 段会自动留备份，用备份能还原，命令在第 3 段末尾。

---

### 第 1 段：关掉 SIP

1. 保存好手头的东西，然后关机。
   苹果菜单 → 关机

2. 按住电源键不松手，一直按着。
   屏幕会先黑，然后出现"正在载入启动选项"再松手。
   （大概按 10 秒左右。松早了会正常开机，那就关掉重来。）

3. 屏幕上会列出磁盘。选中【选项】那个齿轮图标，点【继续】。

4. 它会问你要密码 —— 输平时开机的那个密码。
   （这一步是 FileVault 在解密磁盘，正常。）

5. 进到一个深色背景、有"实用工具"窗口的界面。这就是恢复模式。

6. 顶部菜单栏点【实用工具】→【终端】。
   会弹出一个终端窗口。

7. 在这个终端里，敲这一行，回车：

   ```
   csrutil disable
   ```

   看到 `Successfully disabled System Integrity Protection` 就成了。

8. 敲这一行重启：

   ```
   reboot
   ```

---

### 第 2 段：跑修复脚本

正常开机、登录进来之后。

1. 打开【终端】app（启动台里搜"终端"，或者聚焦搜索 `Cmd+空格` 打"终端"）。

2. 先确认 SIP 真的关了。敲：

   ```
   csrutil status
   ```

   要看到：

   ```
   System Integrity Protection status: disabled.
   ```

   如果显示 `enabled`，说明第 1 段没成，回第 1 段重做。

3. 跑修复脚本。敲这一行，回车，然后输开机密码：

   ```
   sudo bash ~/corp-vpn-fix-lan-permission.sh
   ```

4. 盯着输出看。会依次打印：

   ```
   ===== 0. SIP 状态 =====              -> SIP 已关, 继续
   ===== 1. 备份 =====                  -> 打印出备份文件路径
   ===== 2. 试写一下, 确认真能写 =====   -> /Library/Preferences 可写
   ===== 3. 改到临时文件并自检 =====     -> 自检通过
   ===== 4. 写入原文件 =====             -> 改前/改后 的 ls -l 两行
   ===== 5. 从真实文件重新读一遍验证 ===== -> => 写入成功
   ```

   最后一行必须是 `=> 写入成功`

   脚本是"先在同目录写好新文件、校验通过、再原子替换"的，
   中途断了原文件也不会坏。任何一步不对它会自己停下并告诉你，
   并且会明确说"原文件未动"。

5. 看到 `=> 写入成功` 之后，**立刻**重启，别拖：

   ```
   sudo reboot
   ```

   脚本末尾也会把这句话打出来。

   > 别等的原因：系统内存里还存着旧的权限状态，拖久了可能被写回去。

   **把备份那个路径记下来。** 脚本第 1 段会打印一行备份路径，长这样：

   ```
   ~/corp-vpn-netpolicy-backup-beforefix-20260101-000000.plist
   ```

   第 3 段万一要还原，就用这个。不用抄，恢复模式里能重新看到 ——
   脚本自己也在结尾又打了一遍。

---

### 第 3 段：验证

重启回来之后，逐条对一遍。

**1）权限表里那条坏配置没了**

终端里敲：

```
python3 -c "import plistlib;t=plistlib.load(open('/Library/Preferences/com.apple.networkextension.plist','rb'))['\$top'];print('坏配置还在' if '1A2B3C4D-1111-4111-8111-111111111111' in t else '已清除');print('CorpVPN', '在' if '5E6F7A8B-2222-4222-8222-222222222222' in t else '不见了!')"
```

期望：

```
已清除
CorpVPN 在
```

> 如果 CorpVPN 显示"不见了!" —— 别慌，用备份还原，见本节末尾。

**2）VPN 配置还在**

系统设置 → 网络 → 右边列表里应该有 VPN / CorpVPN。
顺便点一下能不能连上。

**3）开 Chrome 访问内网**

直接在 Dock 点 Chrome（这次不用点别的启动器了），打开：

```
http://ci.example.com/
```

应该出 Jenkins 登录页。

如果弹出"Google Chrome 想要访问本地网络" —— 点【允许】。这是好事，
说明系统重新认识 Chrome 了。

如果没弹、还是打不开：
去 系统设置 → 隐私与安全性 → 本地网络，找 Google Chrome，打开开关。
（之前这个开关对 Chrome 是失效的，原因就是那条坏配置；
修完之后应该能正常开上了。）

**4）顺手确认 Navicat / 数据库**

看看平时连的 Oracle（`10.20.11.185:1521`）通不通。

> **注意**：如果在 WiFi 上，这个库本来就连不上 —— 那是网络本身到不了，
> 跟这个权限问题无关，插网线再试。

每一条都过了，就可以去第 4 段把 SIP 开回来。

**万一要还原**

（还原同样需要 SIP 处于关闭状态，所以要在第 4 段之前做）

```
sudo cp ~/corp-vpn-netpolicy-backup-beforefix-<那串时间>.plist \
        /Library/Preferences/com.apple.networkextension.plist
sudo reboot
```

路径里的时间戳照抄脚本第 1 段打印出来的那个。

> 桌面上另有两个更早的备份（13:14 / 13:18 的），也能用，
> 但它们是"给 Navicat 开权限之前"的状态，拷回去会丢掉那个授权。
> 优先用脚本自己新做的那个。

---

### 第 4 段：把 SIP 开回来

和第 1 段一模一样的操作，只是第 7 步的命令换成 `enable`。

1. 关机。

2. 按住电源键不松，看到"正在载入启动选项"再松手。

3. 选【选项】→【继续】→ 输开机密码。

4. 顶部菜单 实用工具 → 终端。

5. 敲：

   ```
   csrutil enable
   ```

   看到 `Successfully enabled System Integrity Protection` 就成了。

6. 敲：

   ```
   reboot
   ```

7. 起来之后开终端确认一下：

   ```
   csrutil status
   ```

   要看到：

   ```
   System Integrity Protection status: enabled.
   ```

到这一步，全部结束。

---

### 如果哪一步卡住了

| 情况 | 怎么办 |
|---|---|
| **开机进不去系统了** | 备份还原命令见第 3 段末尾。再不行还有恢复模式里的"从时间机器备份恢复" |
| **`csrutil disable` 报错** | 确认是在【恢复模式】的终端里敲的，不是在正常系统里。正常系统里敲一定是报错的，这是设计如此 |
| **脚本说"SIP 还开着"或"目录还是写不了"** | 第 1 段没做干净，回第 1 段重做 |
| **脚本说"临时文件校验不过"或"替换失败"** | 它自己会说"原文件未动"，也就是说没造成任何损坏。把完整输出留着，回来分析 |
| **修完 Chrome 还是打不开内网** | 先看第 3 段第 3 条的排查。另外确认一下连的是网线还是 WiFi，以及那台地址到底在不在 `10.20.0.0-10.20.15.255` 这个范围内 |
| **不想继续了** | 任何时候都可以停。唯一要记得的是：如果停在第 1 段和第 4 段之间，SIP 是关着的。回到电脑前把第 4 段做完就行。暂时不开回来也没有实际危害 |

---

### 这套东西到底是怎么回事

- 机器上有个"本地网络"权限开关。它管的是"这个 app 能不能访问和你在同一个网段里的机器"。
  公司内网的 ci、Oracle 库，在插网线的时候正好就在同一个网段里，所以被这个开关管着。

- 这个开关的记录不在通常的隐私数据库里，而在
  `/Library/Preferences/com.apple.networkextension.plist`。

- 那个文件里的记录坏掉了：Chrome 名下挂着 **77 条**规则，
  其中 **68 条**指向早已被删掉的临时目录（Chrome 自动更新留下的垃圾）。
  所以在系统设置里打开 Chrome 的开关时，系统要同时改这 77 条，
  改到不存在的那些就失败，整批回滚 —— 开关看着动了，其实没生效。

- 脚本做的事只有一件：把这条坏掉的 `networkprivacy` 配置整个摘掉，
  让 macOS 下次重新生成一份干净的。
  CorpVPN 的配置和防火墙配置都原样保留，不碰。

- 之所以要关 SIP：这个文件在 `/Library/Preferences` 下面，
  那个目录被系统打了保护标志，连 root 都写不进去。

- 为什么不直接删掉整个文件重来：因为里面还存着 CorpVPN 配置。删了就没了。
  所以是精准摘掉坏的那一条，不是整个文件。
