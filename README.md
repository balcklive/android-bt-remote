# Bluetooth HID Remote for Android

[![Stars](https://img.shields.io/github/stars/jqssun/android-bt-remote?label=stars&logo=GitHub)](https://github.com/jqssun/android-bt-remote)
[![GitHub](https://img.shields.io/github/downloads/jqssun/android-bt-remote/total?label=GitHub&logo=GitHub)](https://github.com/jqssun/android-bt-remote/releases)
[![license](https://img.shields.io/badge/License-GPLv3-blue.svg)](LICENSE)
[![build](https://img.shields.io/github/actions/workflow/status/jqssun/android-bt-remote/apk.yml?label=build)](https://github.com/jqssun/android-bt-remote/actions/workflows/apk.yml)
[![release](https://img.shields.io/github/v/release/jqssun/android-bt-remote)](https://github.com/jqssun/android-bt-remote/releases)

**Remote** turns any Android device into a Bluetooth (Classic) HID keyboard, mouse, and trackpad designed for all platforms that accept generic Bluetooth input.

It is built on top of the original [BT Remote designed for Android TV](https://gitlab.com/Atharok/BtRemote) but instead made to work with macOS, iOS (+ iPadOS), Windows, Android, ChromeOS, Linux (+ SteamOS) as well as all other platforms that support Bluetooth input devices.

Unlike network remote-control tools, this does not require a companion application on the target. The Android device presents itself as a generic Bluetooth Human Interface Device (HID) and sends HID-compliant keyboard reports, media controls, and mouse input directly over Bluetooth.

[<img height="48" alt="Get it on Google Play" src="https://jqssun.github.io/images/badges/google-play-store.svg">](https://play.google.com/store/apps/details?id=io.github.jqssun.btremote)
[<img height="48" alt="Get it on F-Droid" src="https://jqssun.github.io/images/badges/fdroid.svg">](https://f-droid.org/packages/io.github.jqssun.btremote)
[<img height="48" alt="Get it on GitHub" src="https://jqssun.github.io/images/badges/github.svg">](https://github.com/jqssun/android-bt-remote/releases/latest)

<video loop src='https://github.com/user-attachments/assets/9bddf132-a861-4693-b408-2539646b7e8b' alt="demo" width="1200" style="display: block; margin: auto;"></video>

## Compatibility

- HID device (controller): Android 9.0+ with Bluetooth
- HID host (target): 
    - Android 4 and later
    - Android TV, Google TV, and Fire OS
    - ChromeOS
    - iOS 13 (iPadOS 13) and later
    - iOS 4 and later (keyboard only)
    - tvOS 9.2 and later
    - Mac OS X 10.3 and later
    - Linux kernel 2.6 and later
    - SteamOS
    - Windows XP Service Pack 2 and later

---

# 本 fork 的改动

本仓库是上游 [Atharok/BtRemote](https://gitlab.com/Atharok/BtRemote) 的**构建外壳**：自身不含应用源码，
靠 `patches/*.patch` 打到 `src` 子模块（上游仓库）上，再改包名/版本号后编译。

| 补丁 | 内容 |
|---|---|
| `0001-support-pc.patch` | 面向 PC 平台的常量与文案调整 |
| `0002-direct-input.patch` | 屏幕键盘的「直接输入」模式 |
| `0003-read-keys.patch` | 签名配置从环境变量改为读 `local.properties` |
| **`0004-http-remote.patch`** | **HTTP 远程输入微服务（本文档主要描述的内容）** |

---

# HTTP 远程输入微服务

在应用内嵌一个 HTTP 服务，让局域网内任意脚本/程序通过发 HTTP 请求来**模拟键盘按键**，
无需在手机端做任何操作。

```
curl -X POST -H 'Content-Type: application/json' \
     -d '{"key":"ENTER"}' http://192.168.1.100:8080/key/tap
```

## 它能做什么

- **点按**任意按键（`/key/tap`）
- **长按**任意按键，按住时长自定，期间由主机操作系统自动重复（`/key/tap` 带 `hold`）
- 按下与释放**分离控制**（`/key/down` + `/key/up`）
- 任意**修饰键组合**：`Ctrl+C`、`Ctrl+Shift+S`、`Alt+Tab`、`Win+E`，修饰键之间可任意叠加
- **单独按下修饰键**（如只按 Win 呼出开始菜单）
- 需要 Shift 的**符号直接输入**：`!` `@` `#` `+` `{` `:` `"` `<` `?` 等 21 个
- 查询连接状态（`/status`）
- 卡键自救（`/key/release_all`）

## 它不能做什么

- **没有鼠标**。不提供鼠标移动、点击、滚轮、陀螺仪。
- **没有媒体/音量键**。播放/暂停、音量增减、Home/Back 这批走 Consumer Page，不在 HTTP API 范围内
  （屏幕上的遥控界面仍然支持）。
- **没有批量文本接口**。只能逐键发送；屏幕键盘的「发送整段文字」功能不通过 HTTP 暴露。
- **不能同时按住两个普通键**。见下方「限制」第 1 条，这是硬性约束。
- 没有小键盘数字区、F13–F24。

## 前置条件

1. 手机已安装本应用，且已**与目标主机完成蓝牙配对**
2. 手机与调用方在**同一局域网**（或使用 `adb forward`）
3. 应用处于运行状态且 HID 服务已启动——**端口不是开机就监听的**，见下方「服务生命周期」

## 快速上手

### 1. 让服务起来

打开应用 → 授予蓝牙权限 → 打开蓝牙 → **进入设备选择界面**。此时常驻通知出现，8080 端口开始监听。

用手机或电脑在浏览器打开 `http://<手机IP>:8080/status` 确认：

```json
{"ok":true,"hidConnected":true,"deviceName":"DESKTOP-XXXX","state":2,"port":8080,"apiVersion":1}
```

`hidConnected` 为 `true` 表示已连上主机，可以开始发按键。为 `false` 时所有 `/key/*` 会返回
`409 not_connected`。

### 2. 查手机 IP

```bash
adb shell ip -f inet addr show wlan0 | grep inet
```

或直接在应用界面/系统设置里查看。

### 3. 建立端口转发（推荐，可选）

局域网直连不稳定时（WiFi 客户端隔离、访客网络），用 USB 调试转发更可靠：

```bash
adb forward tcp:8080 tcp:8080
# 之后访问 http://127.0.0.1:8080 即可
```

### 4. 发第一个按键

```bash
# 注意 -H：服务强制要求 application/json，否则返回 415
curl -X POST -H 'Content-Type: application/json' \
     -d '{"key":"H"}' http://192.168.1.100:8080/key/tap
```

---

## API 参考

基址 `http://<手机IP>:8080`，所有响应为 `application/json; charset=utf-8`。
**所有 `/key/*` 请求必须带 `Content-Type: application/json`**，且**只接受 POST**。

### `GET /` — 端点清单

```json
{"ok":true,"apiVersion":1,
 "endpoints":["GET /status","POST /key/tap","POST /key/down","POST /key/up","POST /key/release_all"]}
```

### `GET /status` — 连接状态

永远返回 `200`，即使未连接（它是状态查询而非动作）。

```json
{"ok":true,"hidConnected":true,"deviceName":"DESKTOP-XXXX","state":2,"port":8080,"apiVersion":1}
```

| 字段 | 说明 |
|---|---|
| `hidConnected` | 是否已连上蓝牙 HID 主机 |
| `deviceName` | 主机名，未连接时为空字符串 |
| `state` | 原始连接状态：`0` 未连接、`2` 已连接 |
| `port` | 实际监听端口 |

### `POST /key/tap` — 点按 / 长按

```json
{"key":"A", "modifiers":["CTRL"], "hold":0}
```

| 字段 | 必填 | 说明 |
|---|---|---|
| `key` | 视情况 | 按键名（大小写不敏感）。**当 `modifiers` 非空时可省略**，此时只按修饰键 |
| `modifiers` | 否 | 修饰键名数组，默认 `[]`。可任意叠加 |
| `hold` | 否 | 按住毫秒数，默认 `0`。会被钳制到 `[25, 10000]` |

`hold` 为 `0` 时按 `25` 毫秒——这是应用发文字时的既有节奏，太短的按压部分主机会丢。
按住期间由**主机**做自动重复，应用不会重复发送报文。

成功响应：

```json
{"ok":true,"key":"A","modifiers":1,"hold":25}
```

`modifiers` 是**实际生效的修饰键位图**，包含字符自带的 Shift。例如 `{"key":"!"}` 会返回
`"modifiers":2`。

### `POST /key/down` — 按下并保持

```json
{"key":"W", "modifiers":[], "timeout":30000}
```

| 字段 | 必填 | 说明 |
|---|---|---|
| `key` | 视情况 | 同 `/key/tap`，`modifiers` 非空时可省略 |
| `modifiers` | 否 | 同上 |
| `timeout` | 否 | 看门狗毫秒数，默认 `30000`，钳制到 `[1000, 30000]` |

**看门狗**：若客户端在 `timeout` 内没有发 `/key/up`（例如进程崩溃、网络断开），服务会**自动释放**
该键，避免主机上按键永久卡住。这是可靠性保障，不是可选项。

响应：

```json
{"ok":true,"key":"W","modifiers":0,"autoReleaseInMillis":30000}
```

### `POST /key/up` — 释放

body 可为 `{}`。幂等：

```json
{"ok":true,"released":true}    // 确实释放了一个按住的键
{"ok":true,"released":false}   // 本来就没有键按住，不算错误
```

### `POST /key/release_all` — 兜底释放

不需要 body。释放任何被按住的键并解除看门狗。**客户端状态错乱时的唯一自救手段**，
否则只能重启应用。

```json
{"ok":true}
```

---

## 按键名

**大小写不敏感**——`"A"` 和 `"a"` 都指物理上的 A 键，都输出小写 `a`。它们是**键名**不是字符。
要输出大写请显式加 Shift：`{"key":"a","modifiers":["SHIFT"]}`。

### 字母与数字

`A`–`Z`（26 个）、`0`–`9`，另有 `ZERO`…`NINE` 单词别名。

### 标点

| 字符 | 可用名 |
|---|---|
| `` ` `` | `` ` `` `GRAVE` `BACKTICK` `TILDE` |
| `-` | `-` `MINUS` `HYPHEN` |
| `=` | `=` `EQUALS` `EQUAL` |
| `[` `]` | `[` `LBRACKET` `LEFTBRACKET` / `]` `RBRACKET` `RIGHTBRACKET` |
| `;` | `;` `SEMICOLON` |
| `'` | `'` `APOSTROPHE` `QUOTE` `SINGLEQUOTE` |
| `\` | `\` `BACKSLASH` |
| `,` `.` `/` | `,` `COMMA` / `.` `PERIOD` `DOT` / `/` `SLASH` |
| ISO 反斜杠键 | `INTL_BACKSLASH` `OEM_102` |

### 需要 Shift 的符号（自带 Shift，直接发即可）

`~` `!` `@` `#` `$` `%` `^` `&` `*` `(` `)` `_` `+` `{` `}` `:` `"` `|` `<` `>` `?`

这些的映射与屏幕键盘的美式布局完全一致。可以和修饰键叠加，
`{"key":"!","modifiers":["CTRL"]}` = `Ctrl+Shift+1`。

> 注意 `<` `>` 在美式布局上是 `Shift+,` / `Shift+.`，不是 ISO 的那个反斜杠键。

### 编辑与导航

| 键 | 可用名 |
|---|---|
| Enter | `ENTER` `RETURN` |
| Esc | `ESC` `ESCAPE` |
| Backspace | `BACKSPACE` `BKSP` |
| Tab | `TAB` |
| Space | `SPACE` `SPACEBAR` `" "` |
| 方向键 | `LEFT` `LEFTARROW` / `RIGHT` `RIGHTARROW` / `UP` `UPARROW` / `DOWN` `DOWNARROW` |
| Page Up | `PAGEUP` `PAGE_UP` `PGUP` |
| Page Down | `PAGEDOWN` `PAGE_DOWN` `PGDN` |
| Insert | `INSERT` `INS` |
| Home / End | `HOME` / `END` |
| **Delete（向后删除）** | `DELETE` `DEL` `FORWARDDELETE` |
| Caps Lock | `CAPSLOCK` `CAPS_LOCK` `CAPS` |
| Scroll Lock | `SCROLLLOCK` `SCROLL_LOCK` |
| Pause / Break | `PAUSE` `BREAK` |
| Print Screen | `PRINTSCREEN` `PRINT_SCREEN` `PRTSC` |
| Menu / App | `MENU` `APP` `APPLICATION` |
| 功能键 | `F1`–`F12` |

> ⚠️ **`DELETE` 和 `BACKSPACE` 是两个不同的键。** HID 规范里 `0x2A` 是退格、`0x4C` 才是向后删除。
> 想要退格用 `BACKSPACE`，想要向后删除用 `DELETE`。

## 修饰键名

用在 `modifiers` 数组里，可任意组合叠加（位图，最多 8 个同时按下）。

| 键 | 可用名 | 位 |
|---|---|---|
| 左 Ctrl | `CTRL` `CONTROL` `LCTRL` | `0x01` |
| 左 Shift | `SHIFT` `LSHIFT` | `0x02` |
| 左 Alt | `ALT` `LALT` `OPTION` | `0x04` |
| 左 Meta / Win / Cmd | `META` `GUI` `SUPER` `CMD` `COMMAND` `WIN` `WINDOWS` | `0x08` |
| 右 Ctrl | `RCTRL` | `0x10` |
| 右 Shift | `RSHIFT` | `0x20` |
| 右 Alt / AltGr | `RALT` `ALTGR` | `0x40` |
| 右键 Meta | `RMETA` | `0x80` |

**修饰键名不能当 `key` 用**，会被拒为 `400 unknown_key`——这是有意的防护，
防止把位值当键码发出去（那会发出 HID usage `0x01` Keyboard ErrorRollOver）。

```bash
# Ctrl+C
-d '{"key":"C","modifiers":["CTRL"]}'
# Ctrl+Alt+Del
-d '{"key":"DELETE","modifiers":["CTRL","ALT"]}'
# 只按 Win（呼出开始菜单）
-d '{"modifiers":["META"]}'
```

## 状态码与错误码

| 状态码 | 错误码 | 含义 |
|---|---|---|
| `200` | — | 成功 |
| `400` | `bad_json` | body 不是合法 JSON 对象 |
| `400` | `unknown_key` | 按键名无法识别 |
| `400` | `unknown_modifier` | `modifiers` 里有无法识别的名字 |
| `400` | `empty_report` | `key` 和 `modifiers` 都没给，报文是空的 |
| `404` | `not_found` | 路径不存在 |
| `405` | `method_not_allowed` | 方法不对（`/key/*` 只收 POST） |
| `409` | `not_connected` | 没有已连接的蓝牙 HID 主机 |
| `409` | `key_already_down` | 已有键被按住（先 `/key/up`） |
| `413` | `body_too_large` | body 超过 4096 字节 |
| `415` | `unsupported_media_type` | `Content-Type` 不是 JSON |
| `500` | `internal_error` | 兜底异常。**出现 500 说明有异常逃逸，请附日志反馈** |
| `502` | `hid_write_failed` | 已连接但 HID 报文发送失败 |
| `503` | `busy` | 并发请求超过 8 个 |

错误响应格式：

```json
{"ok":false,"error":"unknown_key","message":"Unknown key: FOO"}
```

日志：`adb logcat -s BtRemoteHttp:D`

---

## ⚠️ 限制（重要）

### 1. 同一时刻只能按住一个普通键

这是**硬性约束**，来自 HID 描述符本身，不是实现偷懒。

本应用的键盘报文是**固定 2 字节**：`[修饰键位图, 单个键码]`。标准键盘描述符会在键码字段开
6 个槽位（6KRO），而这里只有 1 个。

这意味着：

- ✅ **修饰键 + 一个普通键**：可以。`Ctrl+C`、`Ctrl+Shift+Esc`、`Alt+Tab`
- ✅ **多个修饰键叠加**：可以，最多 8 个
- ❌ **两个普通键同时按住**：**做不到**。例如「按住方向键/`WASD` 的同时按空格跳跃」

而且 HID 报文是**全量覆盖**而非增量：

```
第 1 条: [0x00, 0x52]  → 主机认为「↑ 按住」
第 2 条: [0x00, 0x04]  → 主机认为「↑ 已松开，A 按下」
```

主机永远只看到最后一条报文描述的状态，所以**不存在**「边按住移动键边攻击」这种表达。

> 改成标准的 8 字节 6KRO 布局在技术上可行，但要同步改造 17 个虚拟键盘布局文件里约 2000 处
> 内联字节字面量，且描述符改错会导致主机直接拒绝配对。当前未做。

### 2. 修饰键不能在按住期间改变

`/key/down` 期间无法追加或改变修饰键——新报文会覆盖住已按下的键。

### 3. 服务生命周期取决于应用状态

**端口只在 HID 前台服务运行时监听**，而该服务在**进入设备选择界面**时才启动：

- 应用未打开 / 未进入设备选择界面 → 端口不监听（连接被拒绝）
- 停在**设备选择界面**时切到后台 → 服务停止，端口关闭
- **划掉应用**（从最近任务移除）→ 服务必定停止，端口关闭
- 已连接主机并进入遥控界面后 → 可以切后台

客户端需要区分「端口连不上」（应用没运行）和「`409 not_connected`」（应用在跑但没配对主机）。

### 4. 必须已连接主机才能操作

`/key/*` 在未连接时一律返回 `409 not_connected`，不会排队等待。

### 5. 主机输入法状态会影响输出

**这一条在实测中非常明显。** 主机的输入法处于中文等组词模式时：

- 字母会被输入法组词（`bt` → 「不同」）
- **修饰键组合可能被输入法吞掉**（实测 `Ctrl+A` 在中文输入法下变成了普通的 `a`，切到英文后立刻正常）

行为与插一把物理键盘完全一致，不是本实现的缺陷。**要用本 API 做自动化，
主机必须处于英文/直接输入模式。**

### 6. 其他约束

| 项 | 值 |
|---|---|
| 端口 | 固定 `8080`，不可配置（改需重新编译 `DEFAULT_HTTP_SERVER_PORT`） |
| 最大按住时长 | `10000` ms |
| 看门狗超时范围 | `1000`–`30000` ms |
| 并发请求上限 | `8`，超出返回 `503` |
| 请求体上限 | `4096` 字节 |
| 最短点按时长 | `25` ms（更短会被钳制） |
| 协议 | 明文 HTTP，无 TLS |

息屏静止后系统可能进入 Doze 丢弃进来的连接，症状是「亮屏正常、闲置几分钟后超时」。
缓解办法是把应用设为「不受电池限制」。WiFi 省电也可能增加延迟或丢包，
对延迟敏感的场景建议用 `adb forward` 走 USB。

## 🔒 安全警告

**默认无任何鉴权，且监听 `0.0.0.0`。同一局域网内任何设备都能调用本 API。**

后果是：任何人都可以指挥你的手机在**已配对的主机上打字**——包括打开终端并执行命令。
在共享/访客 WiFi 环境下这是**即时可利用**的暴露面。

已内置的两道缓解（**不足以替代鉴权**）：

1. 强制 `Content-Type: application/json`。表单和 `text/plain` 属于 CORS 安全列表类型，
   浏览器可以跨域直接发送；挡掉它们能阻止网页表单类的 CSRF。
2. **不返回任何 CORS 头**。跨域 JSON 请求会先发 preflight，而我们不响应 CORS 头，
   preflight 失败，恶意网页无法利用。
   **不要在未加鉴权的情况下添加 `Access-Control-Allow-Origin: *`**，那会拆掉这道防线。

需要更强保障时，按成本从低到高：

- 加一个设置项开关，**默认关闭**
- 加共享密钥请求头
- 只绑 WiFi 接口
- 绑 `127.0.0.1`，强制走 `adb forward`——外部完全连不上，最安全

## 故障排查

| 现象 | 原因 |
|---|---|
| 连接被拒绝 | 应用没运行，或没进入设备选择界面 |
| `/status` 返回 `hidConnected:false` | 蓝牙未配对/未连接目标主机 |
| 所有 `/key/*` 返回 `415` | 忘了 `-H 'Content-Type: application/json'` |
| 返回 `409 key_already_down` | 有键被按住，先 `/key/up` 或 `/key/release_all` |
| 发出按键但主机无反应 | 主机输入法在组词模式；或主机焦点不在目标窗口 |
| 部分按键丢失 | 点按太短。主机侧有丢帧，重试或改用 `/key/down` + 延迟 + `/key/up` |
| 局域网连不上但 `adb forward` 可以 | WiFi 客户端隔离（访客网络/企业网络） |
| 闲置几分钟后超时 | Doze 或 WiFi 省电，见「限制」第 6 条 |

---

# 构建

## 环境要求

- JDK 21
- Android SDK（`compileSdk 36`）
- 用于签名的 keystore（仅 release 需要）

## 本地构建

```bash
git submodule update --init --recursive
./build.sh                 # 打补丁 + 覆盖资源 + 改写包名/版本号
cd src
./gradlew assembleDefaultDebug     # 不需要签名，最快验证
```

`build.sh` 会做三件事：把 `patches/*.patch` 用 `git am` 打到 `src` 子模块上、
复制 `res/` 覆盖层、用 `sed` 改写 `applicationId` / 版本号 / 应用名 / 链接。

**release 构建需要 `src/local.properties`**（补丁 `0003` 把签名配置改成读它）：

```properties
storeFile=../upload.jks
storePassword=<密码>
keyAlias=<别名>
keyPassword=<密码>
```

> ⚠️ `storeFile` 必须是 `../upload.jks`。Gradle 的 `file()` 在 `app/build.gradle.kts` 里
> 相对 **app 模块目录**解析，而 CI 把 keystore 写在 `src/` 下。

```bash
cd src && ./gradlew assembleDefaultRelease
```

务必用 **release** 变体验证——只有它跑 R8 + 资源压缩，
`assembleDebug` 测不出 ProGuard 规则问题（症状是只在 release 崩的 `NoClassDefFoundError`）。

## 补丁工作流

本仓库的代码改动**都写成补丁**，不直接改 `src`。顺序不能变：

```bash
cd src
git am --whitespace=nowarn --keep-non-patch ../patches/*.patch   # 1. 打成真实提交
# 2. 改代码
# 3. git commit
git merge-base origin/main HEAD      # 4. 自查基线，必须仍是上游基线提交
git log --oneline <基线>..HEAD       #    必须正好是「补丁数」个提交

cd .. && ./generate.sh               # 5. 重新生成 patches/*.patch
```

### 陷阱

- **绝不在仓库根目录 `git add -A`**。`src` 是子模块，子模块里建分支/提交后根仓库会显示
  ` M src`。把这个 gitlink 提交进去会导致 CI checkout 到错误的上游提交、`git am` 直接失败。
  只提交 `patches/` 和文档。
- **`generate.sh` 没有安全检查**：它先 `rm -f patches/*.patch` 再 `format-patch`。
  基线取不到时会产出 0 个补丁而旧补丁已删。`patches/` 受本仓库跟踪，可用
  `git checkout -- patches/` 恢复。所以上面第 4 步每次都要做。
- **`generate.sh` 复现不出原始补丁的排版**：现有 `0001`–`0003` 是 1 行上下文（`-U1`），
  而 `git format-patch` 默认 3 行并会合并相邻 hunk。重跑会让这三个文件出现纯排版差异
  （blob 哈希不变，功能等价）。**新补丁只提交自己那一个，不要顺带重写无关补丁。**
- **补丁必须零填充命名**（`0004-`、`0005-`…），`build.sh` 靠字典序应用。
  补丁的 diff 上下文依赖前面的补丁已应用，不能重排、改名或删除中间某个。
- **`build.sh` 不能连跑两次**，第二次 `git am` 会发现补丁已应用而报错。重测前先 reset 到基线。

## CI

`.github/workflows/apk.yml` 在推 `release/**` 分支或 `v*.*.*` tag、或手动 `workflow_dispatch`
时触发，产出签名 APK 与 AAB 并发 GitHub Release。

需要两个仓库 Secret：

- `STORE` — keystore 文件的 base64
- `LOCAL` — `local.properties` 的 base64（见上方内容）

```bash
base64 -w0 upload.jks            # → STORE
base64 -w0 local.properties      # → LOCAL
# Git Bash 若不支持 -w0：openssl base64 -A -in upload.jks
```

**APK 版本号来自 `build.sh` 顶部硬编码的 `VERSION_CODE` / `VERSION_NAME`**，
git tag 只用于命名 GitHub Release，不影响 APK 内的版本号。

`.github/workflows/ci-verify.yml` 是**验证工作流**：它用 `keytool` 现场生成一次性 keystore
（因此不需要任何 Secret），跑 release 构建，校验 R8 没有裁掉 NanoHTTPD，
并把 APK 作为 artifact 上传。仅手动触发或推 `ci-verify` 分支时运行。

> 该工作流签出的 APK 用的是**一次性密钥**，每次运行都不同。安装前必须先卸载任何
> 同包名的既有版本（包括 Play / GitHub Release 版），否则报 `INSTALL_FAILED_UPDATE_INCOMPATIBLE`。

## Credits

- [Atharok](https://gitlab.com/Atharok/BtRemote) for the original BT Remote
