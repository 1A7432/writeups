# 必应 SEO 投毒仿冒软件下载站群技术分析：改配置的 RdViewer 远控与 Nuitka 侧加载 Shellcode 加载器

| 项 | 内容 |
|---|---|
| 作者 | [1A7432](https://github.com/1A7432) |
| 版本 | v1.0，2026-11-22 |
| 数据采集时间 | 2026-11-10 至 2026-11-11（UTC+8） |
| 分发级别 | TLP:CLEAR |
| 分析方式 | 静态分析、页面抓包、注册信息与第三方记录查询；**样本未在任何系统上运行** |

---

## Executive Summary (English)

A Bing-focused SEO-poisoning operation impersonated DeepSeek, WPS Office, Soda Music (汽水音乐) and at least ten other Chinese software brands. As of 2026-11-10, look-alike `.com.cn` domains held top-10 organic positions on cn.bing.com for queries such as "deepseek网页版" (rank 1); none appeared on the first page of Baidu, 360 or Sogou. Entry pages carry an obfuscated `head.js` that redirects only visitors whose `document.referrer` is a search engine and whose User-Agent is not a crawler, so direct visits by reviewers see a benign page.

Two distinct payload families were recovered from the download sites:

- **Package A (DeepSeek lure):** a PECompact-packed dropper that installs a commercial remote-administration client (RdViewer 6.9.8.0). The operator-set server is stored in a 130-byte `host.dat` encrypted with CryptoAPI AES-128-CBC under the hard-coded password `123456`; decryption yields `66.212.58[.]50:7788` (2026-11-04 build) and `143.92.56[.]3:7788` (2026-11-10 build). The vendor relay `node.rdviewer[.]com` found in strings is only a built-in default. The dropper and the embedded client were signed with the same individual Certum code-signing certificate, issued 2026-11-02 and revoked 2026-11-08.
- **Package B (WPS / Soda Music lures):** a Nuitka onefile Python installer that drops the genuine vendor installer plus a WeGame service binary (`MultiTip.exe`) side-loading an unsigned `t4g2n9.dll`. The DLL gates on ≥4 CPUs and >4 GB RAM, AES-256-CBC-decrypts `MultiTip.log` with a key built from 12 immediate `mov` instructions, and executes the resulting x86 shellcode, whose reversed UTF-16 config (`|p1:186.244.226[.]17|o1:9809|…|fz:wps|bz:2026.11.10|`) matches the format of the "Silver Fox" (银狐) family described in CNCERT's May 2026 advisory.

Infrastructure is tied together by shared 51.la analytics IDs, a common CNAME pivot domain (`*.ikunzf[.]online`) and shared WHOIS registrant emails (174 domains / 15 emails). All conclusions are from static analysis; no sample was executed and no C2 traffic was observed. Reproduction steps and decryption code are in Appendix B.

---

## 1 关键结论

| # | 结论 | 证据类型 | 置信度 |
|---|---|---|---|
| K1 | 一批仿冒 DeepSeek、WPS、汽水音乐等软件的站点在 cn.bing.com 多个关键词下排进自然结果前 10，部分为第 1 位；百度、360、搜狗首页未出现 | 页面抓包 | 高 |
| K2 | 入口页通过 `head.js` 做条件跳转：仅当来源为搜索引擎且 UA 不是爬虫时跳往下载页 | 页面抓包 + 脚本静态分析 | 高 |
| K3 | DeepSeek 线的安装包是 RdViewer 商用远控的释放器，**配置中**的服务器为 `66.212.58[.]50:7788`、`143.92.56[.]3:7788` | 静态分析（离线解密） | 高 |
| K4 | WPS、汽水音乐线的安装包是 Nuitka 打包器 + 白文件侧加载 + 加密 shellcode，**配置中**的服务器为 `186.244.226[.]17:9809` | 静态分析（离线解密） | 高 |
| K5 | 两条线共用同一套建站与引流基础设施（统计 ID、跳板域名、注册邮箱） | 页面抓包 + 注册信息 | 高 |
| K6 | B 线配置格式及整体打法与 CNCERT 2026-05 通报的银狐木马传播活动一致 | 对照公开通报 | 中 |
| K7 | 两条线的后端是否为同一操作方 | — | **无证据，不作判断** |

“配置中的服务器”指从安装包解出并解密的配置值，不代表观测到了连接。

---

## 2 范围、方法与局限

**取得的材料**：4 个安装包（DeepSeek 线 11-04 与 11-10 两版、WPS、汽水音乐），入口页 / 下载页 / 脚本原文与响应头，搜索结果页快照，DNS、RDAP/whois、被动 DNS、urlscan、RIPEstat 记录。

**方法**：

- 页面侧：用后台浏览器抓包工具 [tabcap](https://github.com/1A7432/tabcap) 记录从搜索结果点击到下载页的完整请求链，另以无 Referer 的直接访问作对照。
- 二进制侧：DeepSeek 线外层壳在 Unicorn x86 模拟器中执行样本自带的解压例程（Win32 API 以桩函数替代，无系统调用、无网络），其余均为纯字节层面的解析、解压与解密；反汇编使用 IDA。
- 每条结论标注证据类型：页面抓包 / 静态分析 / 注册信息查询 / 第三方记录。

**局限**：

- 无动态分析、无沙箱、无网络流量。文中不出现“回连”“上线”等需要运行时证据的表述。
- 初次接触的入口页与关键词未留存记录；第 3 节的跳转链是 2026-11-10 晚间的复现。
- 下文所有在线状态均截至 2026-11-10/11，之后未持续跟踪。
- 域名注册人、证书主体等个人信息不在本文公开，已随原始记录提交执法机关。

---

## 3 投递：SEO 投毒与条件跳转

### 3.1 搜索排名（2026-11-10 12:07–12:11Z，自然结果名次）

| 搜索引擎 | 关键词 | 命中的本案域名（名次） |
|---|---|---|
| 必应 | deepseek网页版 | `deepseek-no[.]com[.]cn`（1）、`www.deekseep-ai[.]com[.]cn`（3）、`platform-deepseek[.]com[.]cn`（5）、`www.deepseek-ot[.]com[.]cn`（6）、`ai.deepseek-pf[.]com[.]cn`（7）、`ai.deepseek-oo[.]com[.]cn`（8） |
| 必应 | deepseek官网 | `cn.deepseek-ag[.]com[.]cn`（1）及其余 5 个 |
| 必应 | deepseek下载 / 官网下载 / 电脑版下载 | `cn.deepseek-ag[.]com[.]cn`（2 / 2 / 7）等 |
| 必应 | wps官网 | `gc.wpscp[.]com[.]cn`（2）、`run.wpser[.]com[.]cn`（9） |
| 必应 | 汽水音乐电脑版 | `pc.qishuic[.]com[.]cn`（2）、`sso.`（4）、`ssl.`（5）、`cdn.`（7）、`qishuic[.]com[.]cn`（8） |
| 必应 | deepseek（单词） | 前 9 条无本案域名 |
| 百度 / 360 / 搜狗 | deepseek网页版 / 官网 / 下载 | 首页均无本案域名 |

投放明显集中在必应，且瞄准的是带“网页版”“官网”“下载”的长尾词。

### 3.2 跳转逻辑

入口页本身是一个内容完整、没有任何下载链接的仿冒介绍页，唯一的恶意部分是外部脚本 `https://1.cdn-<品牌>/head.js`（DeepSeek 版 24,053 字节，SHA-256 `8929b429…0b75b`）。脚本做了字符串数组混淆，但保留了函数名：`isFromSearchEngine`、`isSpider`、`getRandomSubdomain`、`getRandomNoFour`、`getCurrentMainDomain`。

末尾的执行顺序还原如下：

```js
if (await window[/* 未还原，字符串表含 "__geoCheck" 片段 */]) return;
if (!isFromSearchEngine()) return;   // referrer 含 google./baidu./bing./sogou./360./yandex./yahoo./duckduckgo/sm.
if (isSpider()) return;              // UA 含 googlebot/bingbot/baiduspide/360spider/…/crawler/spider/bot
window.location.href = 'https://' + getRandomSubdomain();   // → <随机数>web.cdn-<品牌>.<tld>
```

`getRandomNoFour` 生成的随机数跳过数字 4。下载页主域名为泛解析，任意 `<n>web.` 子域都返回同一页面，封禁单个子域无效。

### 3.3 实测跳转链（页面抓包）

| # | 操作 | 结果 |
|---|---|---|
| J1 | cn.bing.com 搜“deepseek网页版”，点第 1 位 `deepseek-no[.]com[.]cn` | 加载 `1.cdn-deepseek[.]com/head.js`，由该脚本发起导航到 `683web.cdn-deepseek[.]com`，弹窗出现 |
| J2 | 无 Referer 直接打开同一入口页 | 停留在介绍页，无下载链接 |
| J3 | 直接打开 `2199web.cdn-deepseek[.]com` | 直接返回下载页并弹窗 |
| J4 | 必应搜“wps官网”，点第 2 位 `gc.wpscp[.]com[.]cn` | 跳到 `2122web.cdn-wps[.]cn` |
| J5 | 必应搜“汽水音乐电脑版”，点第 2 位 `pc.qishuic[.]com[.]cn` | 跳到 `216web.cdn-qishui[.]com[.]cn` |

J1 与 J2 的差异解释了这类站点“举报后审核方无法复现”的现象：审核员直接访问时，看到的是无害页面。

### 3.4 下载页

DeepSeek 下载页（9,932 字节，多个 `<n>web` 子域返回同一 SHA-256 `11776dd8…0f30a1`）冒用 DeepSeek 标识与公司全称。页面上的“开始对话”“电脑版下载”按钮和外部脚本 `download-zh[.]com[.]cn/min.js`（1,910 字节）插入的弹窗——“当前浏览器版本过低 / 请安装插件后使用在线搜索功能”——都指向同一下载地址。弹窗用 localStorage 键 `plugin-install-notice-shown` 保证每个浏览器只弹一次。

| 品牌 | 下载地址（2026-11-10） | 托管 |
|---|---|---|
| DeepSeek | `hxxps://aass1112222.tos-cn-beijing.volces[.]com/Dssiookle_setup.zip` | 火山引擎 TOS |
| WPS | `hxxps://pub-6e9b1d58f82c4d80b88eb72512b2d5bc.r2[.]dev/11.10WPSawrwfdaw.zip` | Cloudflare R2 |
| 汽水音乐 | `hxxps://pub-649d96c8616b4fc48e22b458f993d054.r2[.]dev/11.10Qsyyasunfidanwif.zip` | Cloudflare R2 |

三个地址返回的 ETag 均等于本文所分析压缩包的 MD5，可确认样本即线上分发的文件。DeepSeek 包对象的 `Last-Modified` 为 2026-11-10 10:02:46Z，说明操作方在持续替换载荷。

---

## 4 基础设施

### 4.1 归簇依据

一个域名同时满足以下至少两项，归入本簇：

1. 页面引用 `https://1.cdn-{deepseek.com | wps.cn | qishui.com.cn}/head.js`；
2. 页面嵌入表 4.2 中的 51.la 统计 ID；
3. DNS CNAME 到 `*.ikunzf[.]online`，或 A 记录为 `154.219.100[.]185` / `156.254.20[.]74`。

### 4.2 51.la 统计 ID

| ID | 用途 |
|---|---|
| `3RAaczHhjbGkSIGi` | DeepSeek 入口页（65 个主机名 / 41 个注册域名共用） |
| `3RB9al0Af3OM99l4` | `*.cdn-deepseek[.]com` 下载页 |
| `3R7ROrVmRUjaW6nI` | WPS 入口页 |
| `3RZP4CkrYckegNZm` | `*.cdn-wps[.]cn` 下载页 |
| `3R9zsv6lD2WwzjJi` | 汽水音乐入口页 |
| `3R0wE2dK04YuioTO` | `*.cdn-qishui[.]com[.]cn` 下载页 |

每个品牌的入口页和下载页各配一个 ID，等于分别统计“搜索进站量”和“落地下载量”，这是典型的转化漏斗监测。

### 4.3 CNAME 跳板

```
<任意>.cdn-deepseek[.]com  → hebwf2rp.ikunzf[.]online ┐
<任意>.cdn-wps[.]cn        → p6twfxyh.ikunzf[.]online ├→ 3zufgc72.ikunzf[.]online → 154.219.100[.]185
<任意>.cdn-qishui[.]com[.]cn → vmqjpxw4.ikunzf[.]online ┤
download-zh[.]com[.]cn     → pfvuth8s.ikunzf[.]online ┘
```

`ikunzf[.]online` 注册于 2025-12-01（Spaceship），使用阿里云 DNS，另有至少 10 个随机子域服务于入口页。后端 IP 被封时，只需改一条记录即可让前端全部域名恢复。2026-11-11 复查时，`3zufgc72` 的 A 记录已从 `.185` 改为 `154.219.100[.]82`，与这一设计相符。

### 4.4 规模

| 维度 | 数值 | 来源 |
|---|---|---|
| DeepSeek 名称的已核验主机名 | 79 个，对应 47 个注册域名 | 被动 DNS、urlscan、必应结果，逐个抓取 |
| 已取得 whois 的同簇域名（全部品牌） | 174 个，登记 19 个姓名、15 个邮箱；同一邮箱跨多个仿冒品牌出现 | CNNIC 等 whois |
| 结构相同的其他仿冒名称 | 鲁大师、快连、搜狗输入法、钉钉、豆包、千问、元宝、QuickQ、Clash Verge、HelloWorld、SafeW | 页面抓包 |
| 同 IP 上的疑似关联站 | `156.254.20[.]74` 上另有 60 余个“电子税务局”类主机名（未抓取、未认定） | 被动 DNS、urlscan |

入口域名注册时间集中在 2026-09-17 至 2026-11-10，下载包按“日期 + 品牌缩写 + 随机字母”命名（如 `11.10LUdsahnuiafs.zip`），呈现按日批量上新的运营节奏。

### 4.5 网络归属

| IP | 角色 | 登记 / 宣告 |
|---|---|---|
| `154.219.100[.]185` | 入口页、下载页、脚本、跳板的解析地址 | AFRINIC，Vapeline Technology(HK)；AS137899 / AS401701 |
| `156.254.20[.]74` | 部分 DeepSeek 入口页 | HK；AS153706 Stack Network |
| `66.212.58[.]50` | A 线配置服务器（11-04 版） | ARIN，Xhostserver, LLC；AS153706 |
| `143.92.56[.]3` | A 线配置服务器（11-10 版） | APNIC，CTG Server（HK）；AS152194 |
| `186.244.226[.]17` | B 线配置服务器 | ARIN，Uscloud Inc；AS154376 Cloudvalley |

---

## 5 载荷 A：DeepSeek 线（RdViewer 释放器）

### 5.1 结构

```
Dssiookle_setup.zip
└── 50.exe（11-04）/ 3.exe（11-10）    PE32, PECompact2, 要求管理员权限
    └── 资源 FILE
        ├── CFGINI    100 B   → C:\ProgramData\host\cfg.ini
        ├── HOSTDAT   130 B   → C:\ProgramData\host\host.dat
        ├── DISKMGR   14.6 MB → C:\ProgramData\host\diskmgr.exe（3.exe 为 diskhelper.exe）
        │                       = rdclient.exe, RdViewer 企业版客户端 6.9.8.0
        └── RDSERVICE 2.6 MB  = rdservice.exe, RdViewer 服务程序 6.6.8.0
```

外层版本资源分别伪装为 “Disk Manager Service Installer”（50.exe）和 “System Cache Installer”（3.exe）。两版的 `DISKMGR`、`RDSERVICE` 完全相同，变化的只有配置和外壳。

### 5.2 脱壳

PECompact2 的解压例程连同其依赖的十余个 Win32 API（以桩函数替代）在 Unicorn 中执行，得到 17,407,132 字节的节数据（SHA-256 `edd256fb…cb139ba`），结果可重复。资源表中 `FILE` 类型条目的数据即位于这段输出中。

### 5.3 配置加密

`rdclient.exe` 中相关函数（源文件路径字符串 `D:\work\RdViewer\RdViewer\RdClient\uCrypto.pas`）：

| VA | 作用 |
|---|---|
| `0xAD62F8` | `CryptCreateHash(CALG_SHA_256)` → `CryptHashData(UTF-16LE 口令)` → `CryptDeriveKey(CALG_AES_128)` → `KP_MODE = CBC` |
| `0xAD6640` | Base64 解码 → `CryptDecrypt` |
| `0xAD63E8` | 加密：明文前补 8 个空格，UTF-16LE，`CryptEncrypt` |
| `0xD68CF8` | 读取 `cfg.ini [Main] Self` 与 `host.dat [Main] host / name / mark`，调用时传入的口令为字面量 **`123456`**；将 `host` 按 `:` 拆分，写入全局变量 `0xDCEAD4`（初值指向 `auth.rdviewer[.]com`）和 `0xDCEAE8`（初值 8888） |

等价算法：`key = SHA256(UTF16LE("123456"))[:16]`，AES-128-CBC，IV 全零，PKCS#7 去填充（代码见附录 B）。

### 5.4 解密结果

| 样本 | host | mark | Self | User |
|---|---|---|---|---|
| 50.exe（11-04） | `66.212.58[.]50:7788` | `35284*****` | `diskmgr.exe` | `admin` |
| 3.exe（11-10） | `143.92.56[.]3:7788` | `35284*****` | `diskhelper.exe` | `admin` |

**关键判断**：`rdclient.exe` 字符串中的 `node.rdviewer[.]com:7709`、`auth.rdviewer[.]com` 等是**出厂默认值**，运行时会被 `host.dat` 的值覆盖。仅凭字符串做判断，会误以为样本只连接厂商官方中转、没有攻击者自建服务器。两版换了服务器，`mark` 值保持不变，看起来跟随操作方而不跟随版本，但其具体语义尚无证据（完整值已提交执法机关）。

客户端内置功能字符串包括远程文件管理、命令终端、屏幕 H264 编码、进程列表、开机自启、托盘隐藏、`/INSTALL /SILENT` 静默安装等，与完整远控能力相符。

### 5.5 代码签名

| 文件 | 证书 | 状态 |
|---|---|---|
| `50.exe`（外层） | Certum Code Signing 2021 CA 签发的**个人**证书，序列号 `23FF0D00EE80E58892220C95AE8EA33B` | 2026-11-02 生效；**2026-11-08 06:13Z 吊销**，原因 Privilege Withdrawn |
| `rdclient.exe`（内嵌） | 同上，PE 摘要匹配 | 同上 |
| `rdservice.exe`（内嵌） | 软件厂商公司证书（WoTrus），2025-01-30 已过期 | — |
| `3.exe`（11-10 版外层） | 无签名 | — |

内嵌客户端由操作方的个人证书重新签名，而服务程序保留厂商原签名。证书签发 2 天后用于签名样本，6 天后被吊销；吊销后的新版本直接放弃签名。`rdclient.exe` 与厂商发布的同版本文件是否逐字节一致，尚未比对。

---

## 6 载荷 B：WPS / 汽水音乐线（Nuitka + 侧加载 + Shellcode）

### 6.1 结构

```
11.10WPSawrwfdaw.zip / 11.10Qsyyasunfidanwif.zip
└── <同名>.exe   x64, 无签名, 版本资源 CreativeSoft Labs / SpeedOptimizer / Program.exe
    └── RCDATA 27: "KAY" + zstd  (Nuitka onefile)
        ├── Program.exe + python39.dll 等
        └── Program.exe 常量区 __main__.CHUNKS  (16–32 万个 Base64 片段)
            └── 拼接 → b64decode → zlib → inner.zip
                ├── WPS_Setup_24657.exe / SodaMusic-v3.1.0-official-win32_x64.exe   ← 真实官方安装包
                ├── MultiTip.exe     WeGame 服务组件 (PDB: …\wegame_client\…\wegameservice.pdb)
                ├── t4g2n9.dll       无签名，导出表与 MultiTip.exe 的 10 个导入一一对应
                └── MultiTip.log     2,432 B, 高熵
```

主模块常量按出现顺序为 `elevate → CHUNKS → zlib.decompress → base64.b64decode → … → zipfile.ZipFile → extractall → SUB_TARGET → subprocess.Popen → time.sleep → MAIN_TARGET → webbrowser.open`，即“提权 → 解包 → 启动加载链 → 再启动真安装包”。受害者确实能装上想要的软件，这降低了察觉概率。

`MultiTip.exe` 的安全目录指向超出文件末尾的位置，文件中残留 DigiCert 证书链 URL 字符串，签名结构不完整。

### 6.2 `t4g2n9.dll`

| VA | 作用 |
|---|---|
| `0x10002540` | 环境检查：处理器数 > 3 且物理内存 > 4096 MB，否则退出（规避低配沙箱）；`SHGetFolderPathA(CSIDL 35，失败则 26)` + 单字节异或还原的 `client\MultiTip.log`；读文件 → 解密 → `VirtualProtect(…, 0x20 PAGE_EXECUTE_READ)` → 从 `kernel32.dll` 动态取得 `CreateThread`，以解密后的内存为入口执行 |
| `0x100023A0` | BCrypt：`AES` / `ChainingModeCBC` / 32 字节对称密钥 / `BCryptDecrypt` |
| `0x10003690` | 用 12 条 `mov dword ptr [addr], imm32` 在运行时写入密钥（`0x1001F418`）与 IV（`0x1001F438`）；两处静态内容均为 `0xFF`，静态扫描常量无法直接得到密钥 |

导入表另含 `CreateServiceA`、`CreateProcessAsUserA`、`CopyFileA`，其调用关系尚未分析。

- 密钥：`547256b719c07efa4d60dd9b890029aaad26f25fdbbfc1c088d19a551341bc17`
- IV：`6d29efe0e68250ac5a348af37a0e79bf`

### 6.3 Shellcode 与配置

解密结果为 x86 机器码（开头 `55 8B EC 81 EC`，标准栈帧），含 ASCII 字符串 `codemark`、`186.244.226.17`，以及一段倒序存放的 UTF-16 配置。还原后（WPS 包）：

```
|p1:186.244.226[.]17|o1:9809|t1:1|p2:186.244.226[.]17|o2:9809|t2:1|p3:127.0.0.1|o3:80|t3:1|dd:1|cl:1|fz:wps|bb:1.0|bz:2026.11.10|jp:1|bh:0|ll:0|dl:0|sh:0|kl:0|bd:0|
```

`p/o/t` 三组为主机 / 端口 / 类型，`fz` 为分组（WPS 包为 `wps`，汽水音乐包为空），`bz` 为备注日期，`bb` 为版本。两个包的 `MultiTip.exe` 与 `t4g2n9.dll` 哈希相同，仅 `MultiTip.log` 不同，即同一加载器按投放渠道换配置。

---

## 7 关联与归因评估

**同一套引流设施（高置信）**：A、B 两线的入口页结构相同，脚本函数名相同，共用 `ikunzf[.]online` 跳板与同一后端 IP，注册邮箱跨品牌重叠。

**与银狐活动的关系（中置信）**：对照 CNCERT 2026-05-22 发布的银狐木马钓鱼网站风险提示（[IT之家转述](https://www.ithome.com/0/954/007.htm)）：

| 维度 | CNCERT 通报 | 本案 |
|---|---|---|
| 引流 | 批量仿冒站 + 搜索引擎 | 必应 SEO |
| 访问控制 | Referer 检查 | Referer + UA 检查，直接访问时显示仿冒介绍页 |
| 流量监测 | 51.la | 51.la，入口页 / 下载页各一个 ID |
| 仿冒对象 | WPS、Chrome、通讯 / 代理类为主 | 新增 DeepSeek、豆包、千问、元宝等 AI 产品 |
| 落地方式 | Shellcode 注入执行远控 | B 线：侧加载 + 加密 shellcode，配置格式为 `pN/oN/tN` 型 |
| IOC 重合 | — | 未发现 |

**不能下的结论**：A 线载荷（改配置的商用远控）与 B 线载荷（加载器 + shellcode）技术路线完全不同。现有证据只能说明“引流环节为同一方或同一服务”，不能说明后端为同一操作方。

---

## 8 处置情况（截至 2026-11-11）

本案已于 2026-11-10 向公安机关、CNCERT、12321、必应、相关云服务商、域名注册商及被仿冒厂商报告。可公开核实的结果：

| 对象 | 结果 | 时间 |
|---|---|---|
| 火山引擎 TOS 存储桶（DeepSeek 下载地址） | 工单答复“经核实举报内容属实，已对涉及存储桶实行封禁”；之后 GET 返回 403 | 2026-11-10 22:30 |
| `cdn-deepseek[.]com` | RDAP 状态变为 `client hold`，下载页与 `head.js` 不再解析 | 2026-11-11 01:29Z |
| 入口页 `deepseek-no[.]com[.]cn` | 仍在线，但引用的 `head.js` 已失效，跳转链断开 | 2026-11-11 复查 |
| WPS、汽水音乐及其他品牌 | 未复查 | — |

---

## 9 检测与缓解建议

**终端侧**

- 文件：`C:\ProgramData\host\{diskmgr.exe, diskhelper.exe, host.dat, cfg.ini}`；`%USERPROFILE%\Documents\SpeedOptimizer\`（由 `Program.exe` 常量 `expanduser("~")`、`Documents`、`APP_NAME` 推得，未运行验证）；`<CSIDL_COMMON_APPDATA 或 CSIDL_APPDATA>\client\MultiTip.log`。
- 行为：WeGame 安装目录以外出现 PDB 为 `wegameservice.pdb`、签名结构不完整的 `MultiTip.exe`，且同目录有未签名 DLL；`rdclient.exe` / `rdservice.exe` 不在资产清单中却出现在 `C:\ProgramData\host\`。
- 证书：吊销列表中序列号 `23FF0D00EE80E58892220C95AE8EA33B` 签名的任何文件。
- 狩猎：Nuitka onefile（`RCDATA` 以 `KAY` 开头）且版本资源为 `SpeedOptimizer` 的未签名 x64 程序。

**网络侧**

- 阻断附录 A 中的 IP 与域名通配；DNS 日志中 CNAME 指向 `*.ikunzf[.]online` 的解析视为高危。
- 未部署 RdViewer 的环境，可将 `*.rdviewer[.]com` 及任何访问 7788 / 9809 端口的外联列入告警。

**审核与平台方**

- 复核此类举报时，应携带搜索引擎 Referer（如 `https://cn.bing.com/`）并使用普通浏览器 UA 访问；直接访问看不到恶意行为。

**用户**

- 软件只从官网域名下载。搜索结果第 1 位、未标“广告”的自然结果同样可能是仿冒站。

---

## 10 未解决问题

1. 样本运行时行为与网络流量（需要动态分析）。
2. `head.js` 第一步 `await window[…]`（疑似地理位置检查）的完整语义。
3. `t4g2n9.dll` 中服务创建、文件复制、以用户身份启动进程等函数的调用关系；shellcode 连接服务器后的逻辑。
4. `rdclient.exe` 与厂商同版本发布文件是否逐字节一致。
5. 鲁大师、快连、搜狗输入法三个 R2 压缩包尚未下载分析；钉钉、豆包等品牌的下载地址未取得。
6. 部分入口域名的 whois 注册时间晚于 urlscan / 被动 DNS 的首次记录，原因未查明。

欢迎持有本批样本动态分析结果的同行交流对账。

---

## 附录 A：IOC

**配置中的服务器地址**（离线解密所得，未观测到连接）

```
66.212.58[.]50:7788       A 线 2026-11-04
143.92.56[.]3:7788        A 线 2026-11-10
186.244.226[.]17:9809     B 线（WPS、汽水音乐）
```

**基础设施 IP**

```
154.219.100[.]185   154.219.100[.]82   156.254.20[.]74
历史解析：154.219.113[.]52  103.204.13[.]101  45.156.87[.]201  176.65.132[.]146
```

**域名**

```
*.cdn-deepseek[.]com      *.cdn-wps[.]cn             *.cdn-qishui[.]com[.]cn
*.cdn-ludashi[.]com[.]cn  *.cdn-kuailianvpn[.]com[.]cn *.cdn-sogou[.]cn
*.cdn-dingding[.]cn       *.cdn-doubao[.]cn          *.cdn-qianwen[.]cn
*.cdn-yuanbao[.]com[.]cn  *.cdn-quickq[.]com         *.cdn-clashverge[.]com
*.cdn-helloworld[.]cn     *.cdn-safew[.]com
download-zh[.]com[.]cn    *.ikunzf[.]online

入口域名（部分）：
deepseek-no[.]com[.]cn  deepseek-ag[.]com[.]cn  deekseep-ai[.]com[.]cn  deepseeks-ai[.]com[.]cn
deepseek-pf[.]com[.]cn  deepseek-ot[.]com[.]cn  deepseek-ts[.]com[.]cn  deepseek-oo[.]com[.]cn
platform-deepseek[.]com[.]cn  despseek[.]com[.]cn  deepseek-kt[.]com[.]cn
wpscp[.]com[.]cn  wpser[.]com[.]cn  wpsdo[.]com[.]cn  wps-l[.]com[.]cn  wps-o[.]com[.]cn  wps-r[.]com[.]cn
qishuic[.]com[.]cn  qishuiduoyin[.]com[.]cn
ludashix[.]com[.]cn  ludashiz[.]com[.]cn  doubaoql[.]com[.]cn  doubaol[.]com[.]cn  doubaop[.]com[.]cn
dingdinger[.]com[.]cn  dingdingc[.]com[.]cn  qianwen-aii[.]com[.]cn  aia-qianwen[.]com[.]cn
```

**下载地址**

```
hxxps://aass1112222.tos-cn-beijing.volces[.]com/Dssiookle_setup.zip
hxxps://pub-6e9b1d58f82c4d80b88eb72512b2d5bc.r2[.]dev/11.10WPSawrwfdaw.zip
hxxps://pub-649d96c8616b4fc48e22b458f993d054.r2[.]dev/11.10Qsyyasunfidanwif.zip
hxxps://pub-1ecd02c1eb2a4a29a73cc88a613bc4d8.r2[.]dev/11.10LUdsahnuiafs.zip      （未分析）
hxxps://pub-d3fc556cb10f4612b2981704c31ca696.r2[.]dev/11.10KLinauabwfww.zip      （未分析）
hxxps://pub-e1457ff2174c492db55509b4aa74b5a3.r2[.]dev/10.11SGsrfainascoi.zip     （未分析）
```

**51.la 统计 ID**

```
3RAaczHhjbGkSIGi 3RB9al0Af3OM99l4 3R7ROrVmRUjaW6nI 3RZP4CkrYckegNZm 3R9zsv6lD2WwzjJi 3R0wE2dK04YuioTO
3RZtaIc5gsh8OSVw 3RZ23mgJew9UmvOB 3RAWpCqSnn0EXEJJ 3RZ23640uQkOcRM5 3RbF7uaOULIaRHlp 3R6LDNwgTvh4xgqE
3RAXIgJ8cIt0CSxu 3R9Vz1lYpOskkV3A 3R9wfaOEo0c6Cp1U 3RCHBuWzoUJu0o5Y 3R7TLZ54sm8g0rjQ 3RSHdaDWAsAsmeAz
3R3d68pWnmwcF4NQ 3R7ZgxK5hbTuNtiy
```

**文件 SHA-256**

| 文件 | SHA-256 |
|---|---|
| Dssiookle_setup.zip（11-04） | `a242121f119ccdcb8c66dea928ecd0f45c94f8a0d438ecd83042e8d9b14e3ec9` |
| Dssiookle_setup.zip（11-10） | `ff04f8000de34deb5734c78d2d506f0da092ba7f23233f0d215e13df024eb23b` |
| 50.exe | `5f3abb5fc7f72e2ab7b43d9257b6388b2cb65f80ab2e890b3ffe3da5e80f4f89` |
| 3.exe | `0ec4191ab40ab770b4d130a7dd36730970facc825b4124dae1b2c6d6120d65db` |
| rdclient.exe（DISKMGR） | `4cc1bd35fc1ef912ad154135d17c1db5d302bc32c7b4e36709edc8980e2cac86` |
| rdservice.exe（RDSERVICE） | `7ed2a077a1ce34fcb182e5135ae75ad7bd96207c3204c9f343095f967bfa109f` |
| host.dat（50 / 3） | `986fccc4a2e13adc89b00744a57c1f8bf4d4a2b2508c798b7a3e86ef7161e9f7` / `7ad5a99dbec1372e4c774b187c333757ad492b9c74831cee6c4de4491a4fb7ad` |
| 11.10WPSawrwfdaw.zip | `7657dc5a670fd852b838b47e76bfcbbab5daa22cc0d90b3325d38c38e4a5d9a7` |
| 11.10WPSawrwfdaw.exe | `a3a4ba07496b526d35c2f3946bd361912bca8701104c1a4bd275c53720398887` |
| 11.10Qsyyasunfidanwif.zip | `79d7dfc9dd126979f5a6df78091c15a642169c60bb0d00a5dbbcf5fd27cc5afd` |
| 11.10Qsyyasunfidanwif.exe | `771144c189211053711ec13910eddcff772610fb36d6100ef642fcdd417c6a46` |
| Program.exe（WPS / 汽水） | `a3352e44c93c21f74b845aac8c040baeaf81d941492ef8cbb4b36f804ad0bd77` / `ea9c5bf277b2c41ab760b4942d9cbf3e1f33dbabe82dca327c6383a075b18511` |
| t4g2n9.dll | `1bc2465dcbef5e2f0c4e3d7529096436fead622dea913a61c981534fdca0a2f2` |
| MultiTip.exe | `3a5256a18f4d52e4ede7d6055f150b1cfc0c9c9653fbdcb0f6a7b9e405c43025` |
| MultiTip.log（WPS / 汽水） | `621170074b3478c2e4dd3a8f375335118b3734acc10d83dd4b3a59601da4beba` / `3938c1cd6fbe1f07bd7d71dc0c81d877f8300011d2e684018b38ff6f5640c8cc` |
| 解密后 shellcode（WPS / 汽水） | `b89404e18e0bfda196710e9946fcf67d3811e675e4d349f299280b9050113d00` / `69ff7f44a38b897cf8550ce47e84a8461c1344556ef4dfe5bc90d27252ee4b6e` |
| head.js（DeepSeek） | `8929b4298fae5f6d3dbb96a0c84432fad79eccdbf4409211378ad3521ae0b75b` |
| min.js（弹窗） | `fe2f9307253dad89c99e631223828099b02a7bef58b0fa8569161e0b5617b6e7` |

**证书**：Certum Code Signing 2021 CA，序列号 `23FF0D00EE80E58892220C95AE8EA33B`（已吊销）。

**样本共享**：以下样本已于 2026-11-22 提交 MalwareBazaar（新账号提交需管理员审核，链接在审核通过后生效）：

- `50.exe` — <https://bazaar.abuse.ch/sample/5f3abb5fc7f72e2ab7b43d9257b6388b2cb65f80ab2e890b3ffe3da5e80f4f89/>
- `diskmgr.exe`（rdclient.exe） — <https://bazaar.abuse.ch/sample/4cc1bd35fc1ef912ad154135d17c1db5d302bc32c7b4e36709edc8980e2cac86/>
- `t4g2n9.dll` — <https://bazaar.abuse.ch/sample/1bc2465dcbef5e2f0c4e3d7529096436fead622dea913a61c981534fdca0a2f2/>

---

## 附录 B：复现

全部步骤离线进行，不运行样本。

**B.1 A 线配置解密**（`host.dat` / `cfg.ini` 中的 Base64 值）

```python
import base64, hashlib
from cryptography.hazmat.primitives.ciphers import Cipher, algorithms, modes

def rd_decrypt(b64: str, password: str = "123456") -> str:
    key = hashlib.sha256(password.encode("utf-16le")).digest()[:16]   # CryptDeriveKey(CALG_AES_128)
    d = Cipher(algorithms.AES(key), modes.CBC(b"\0" * 16)).decryptor()
    pt = d.update(base64.b64decode(b64)) + d.finalize()
    return pt[:-pt[-1]].decode("utf-16le").lstrip(" ")

print(rd_decrypt("lxW394sBNaxHt/BG9Gg520Z74Q8YrUt7Ch8Th3NZsHjAF6B7R9JAA2JCddyBm9f5"))  # 66.212.58.50:7788
print(rd_decrypt("S36wTF/5PIGXgegIULdD0fGmVmsp4BK/nkmg8liEkDlGQ0QwvEoLrFGTAe6Z9gb9"))  # 143.92.56.3:7788
```

**B.2 B 线 `MultiTip.log` 解密**

```python
import hashlib, sys
from cryptography.hazmat.primitives.ciphers import Cipher, algorithms, modes

KEY = bytes.fromhex("547256b719c07efa4d60dd9b890029aaad26f25fdbbfc1c088d19a551341bc17")
IV  = bytes.fromhex("6d29efe0e68250ac5a348af37a0e79bf")
ct = open(sys.argv[1], "rb").read()
d = Cipher(algorithms.AES(KEY), modes.CBC(IV)).decryptor()
pt = d.update(ct) + d.finalize(); pt = pt[:-pt[-1]]
print(len(pt), hashlib.sha256(pt).hexdigest())
# WPS 包：2424 b89404e1…13d00
```

以上两段代码已在本文所列样本上重新运行，输出与正文一致。

**B.3 层级提取要点**

| 步骤 | 输入 | 方法 | 输出 |
|---|---|---|---|
| A3 | 50.exe | Unicorn 执行 PECompact2 自带解压例程，API 桩替代 | 节数据 17,407,132 B |
| A4 | 节数据 + PE 资源表 | 按 `FILE` 资源 RVA 定位切片 | CFGINI / HOSTDAT / DISKMGR / RDSERVICE |
| B2-1 | 外层 x64 exe | 取 `RCDATA 27`，跳过 `KAY` 头，zstd 解压，按“UTF-16 文件名 + 8 字节长度 + 数据”切分 | Program.exe 等 9 个文件 |
| B2-2 | Program.exe | 取 `RCDATA 3`（Nuitka 常量区），定位 `__main__` 的 `CHUNKS`，拼接 → Base64 → zlib | inner.zip |
| B2-3 | t4g2n9.dll | 定位向连续地址写 12 个 imm32 的指令序列（8 个为密钥，4 个为 IV），以 PKCS#7 填充合法性校验 | 密钥 / IV |

---

## 附录 C：时间线

| 时间 | 事件 | 类型 |
|---|---|---|
| 2025-12-01 06:23Z | `ikunzf[.]online` 注册 | 注册信息 |
| 2026-06-01 05:00Z | 内嵌 `rdclient.exe` PE 时间戳 | 静态分析 |
| 2026-09-09 | `download-zh[.]com[.]cn` 注册 | 注册信息 |
| 2026-09-17 起 | DeepSeek 名称入口域名陆续注册（持续至 11-10） | 注册信息 |
| 2026-11-02 10:42Z | 个人代码签名证书生效 | 静态分析 |
| 2026-11-02 22:47Z | `t4g2n9.dll` PE 时间戳 | 静态分析 |
| 2026-11-04 07:31Z | `50.exe` PE 时间戳（签名版） | 静态分析 |
| 2026-11-05 08:41Z | `cdn-deepseek[.]com` 注册；08:57Z `head.js` 上线 | 注册信息 / 页面抓包 |
| 2026-11-08 06:13Z | 证书吊销 | 静态分析（CRL） |
| 2026-11-10 08:02–08:42Z | WPS、汽水音乐包构建并上传 R2 | 静态分析 / 页面抓包 |
| 2026-11-10 08:30Z | `3.exe` PE 时间戳（无签名版） | 静态分析 |
| 2026-11-10 10:02Z | TOS 上 DeepSeek 包被替换为 `3.exe` 版本 | 页面抓包 |
| 2026-11-10 12:07Z | 搜索排名与跳转链取证 | 页面抓包 |
| 2026-11-10 22:30 +08 | TOS 存储桶被封禁 | 服务商答复 |
| 2026-11-11 01:29Z | `cdn-deepseek[.]com` 进入 client hold | 注册信息 |
