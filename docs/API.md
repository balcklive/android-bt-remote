# HTTP 远程输入 API

本 fork 的 `patches/0004-http-remote.patch` 在应用内嵌了一个 HTTP 服务，让局域网内任意脚本/程序
通过发 HTTP 请求来**模拟键盘按键**，手机端无需任何操作。

**基址** `http://<手机IP>:8080`　|　明文 HTTP，无 TLS，无鉴权，监听 `0.0.0.0`

一句话：POST 一个 JSON 到手机，手机通过蓝牙 HID 在**已配对的主机**上敲键盘。

```bash
curl -X POST -H 'Content-Type: application/json' \
     -d '{"key":"ENTER"}' http://192.168.1.100:8080/key/tap
```

> 实现位于 `presentation/http/HttpRemoteServer.kt`（HTTP 层）、
> `domain/usecases/HttpKeyboardUseCase.kt`（并发与看门狗）、
> `domain/entities/remoteInput/keyboard/KeyboardKeyNames.kt`（键名表）。本文件描述的行为
> 以这三个文件为准。

## 0. 前置条件与范围

**前置条件**

1. 手机已安装本应用，且已**与目标主机完成蓝牙配对**
2. 手机与调用方在**同一局域网**（或使用 `adb forward`，见 §7.7）
3. 应用处于运行状态且 HID 前台服务已启动——**端口不是开机就监听的**，见 §7.3

查手机 IP：

```bash
adb shell ip -f inet addr show wlan0 | grep inet
```

**本 API 不提供**（屏幕上的遥控界面仍然支持，只是不经 HTTP 暴露）：

- **鼠标**：无移动、点击、滚轮、陀螺仪
- **媒体 / 音量 / Home / Back 键**：这批走 HID Consumer Page，不在本 API 范围内
- **批量文本**：只能逐键发送；屏幕键盘的「发送整段文字」不通过 HTTP 暴露
- **小键盘数字区、`F13`–`F24`**
- **两个普通键同时按住**：见 §7.1，这是 HID 描述符的硬性约束，不是实现取舍

## 1. 传输约定

| 项 | 规则 |
|---|---|
| 方法 | `/key/*` **只接受 POST**；`/` 和 `/status` **只接受 GET**，其它方法 → `405` |
| Content-Type | 只要带了该头，就必须含 `json`（大小写不敏感）；**完全不发该头则放行**。详见 §7.6 |
| 响应 | 一律 `application/json; charset=utf-8` |
| 请求体 | ≤ 4096 字节，超出 → `413`（并强制关闭连接） |
| 并发 | 同时处理上限 8，超出 → `503 busy` |
| 路径 | 末尾多余的 `/` 会被忽略（`/status/` 等价 `/status`） |

**通用响应格式**

```json
{"ok":true, ...}                                   // 成功
{"ok":false,"error":"unknown_key","message":"..."} // 失败
```

## 2. 端点

### `GET /` — 端点自省

```json
{"ok":true,"apiVersion":1,
 "endpoints":["GET /status","POST /key/tap","POST /key/down","POST /key/up","POST /key/release_all"]}
```

### `GET /status` — 连接状态

**永远 200**，即使未连接（它是状态查询，不是动作）。

```json
{"ok":true,"hidConnected":true,"deviceName":"DESKTOP-XXXX","state":2,"port":8080,"apiVersion":1}
```

| 字段 | 含义 |
|---|---|
| `hidConnected` | 是否已连上蓝牙 HID 主机。为 `false` 时 `/key/tap`、`/key/down` 会 `409` |
| `deviceName` | 主机名，未连接时 `""` |
| `state` | 原始状态：`0` 未连接（`BluetoothHidDevice.STATE_DISCONNECTED`）、`2` 已连接 |
| `port` | 实际监听端口 |
| `apiVersion` | 当前为 `1` |

### `POST /key/tap` — 点按 / 长按

```json
{"key":"A", "modifiers":["CTRL"], "hold":0}
```

| 字段 | 必填 | 说明 |
|---|---|---|
| `key` | 视情况 | 键名，**大小写不敏感**。`modifiers` 非空时可省略（此时只按修饰键） |
| `modifiers` | 否 | 修饰键名数组，默认 `[]`，可任意叠加 |
| `hold` | 否 | 按住毫秒数，默认 `0`，钳制到 **`[25, 10000]`** |

`hold:0` 实际按 **25 ms**（`DELAY_BETWEEN_KEY_PRESSES_IN_MILLIS`）——太短的按压部分主机会丢帧。
按住期间靠**主机**自动重复，应用不重发报文。

```json
{"ok":true,"key":"A","modifiers":1,"hold":25}
```

`modifiers` 返回的是**实际生效的位图**，含字符自带的 Shift：`{"key":"!"}` → `"modifiers":2`。

### `POST /key/down` — 按下并保持

```json
{"key":"W", "modifiers":[], "timeout":30000}
```

`key` / `modifiers` 同 `/key/tap`；`timeout` 为看门狗毫秒数，默认 `30000`，钳制到
**`[1000, 30000]`**。

**看门狗是强制的**：客户端在 `timeout` 内没发 `/key/up`（崩溃、断网），服务**自动释放**该键——
这是防止主机卡键的可靠性保障，不是可选项。

```json
{"ok":true,"key":"W","modifiers":0,"autoReleaseInMillis":30000}
```

### `POST /key/up` — 释放

body 可为 `{}` 或空。**幂等，且不做连接检查**（未连接也返回 200）：

```json
{"ok":true,"released":true}    // 确实释放了一个按住的键
{"ok":true,"released":false}   // 本来就没有键按住，不算错误
```

### `POST /key/release_all` — 兜底释放

不需要 body，永远 200。释放任何按住的键并解除看门狗。

```json
{"ok":true}
```

> 这是**客户端状态错乱时的唯一自救手段**，否则只能重启应用。

## 3. 键名（`key` 字段）

大小写不敏感。**它们是"键名"不是"字符"**——`"A"` 和 `"a"` 都指物理 A 键，都输出小写 `a`。
要大写请显式加修饰键：`{"key":"a","modifiers":["SHIFT"]}`。

**字母数字**：`A`–`Z`、`0`–`9`（另 `ZERO`…`NINE` 别名）

**标点**

| 字符 | 可用名 |
|---|---|
| `` ` `` | `` ` `` `GRAVE` `BACKTICK` `TILDE` |
| `-` `=` | `-` `MINUS` `HYPHEN` / `=` `EQUALS` `EQUAL` |
| `[` `]` | `[` `LBRACKET` `LEFTBRACKET` / `]` `RBRACKET` `RIGHTBRACKET` |
| `;` `'` | `;` `SEMICOLON` / `'` `APOSTROPHE` `QUOTE` `SINGLEQUOTE` |
| `\` | `\` `BACKSLASH` |
| `,` `.` `/` | `,` `COMMA` / `.` `PERIOD` `DOT` / `/` `SLASH` |
| ISO 反斜杠键 | `INTL_BACKSLASH` `OEM_102` |

**需要 Shift 的符号**（自带 Shift，直接发即可，共 21 个）

`` ~ ! @ # $ % ^ & * ( ) _ + { } : " | < > ? ``

映射与屏幕键盘的美式布局一致，可与修饰键叠加：`{"key":"!","modifiers":["CTRL"]}` =
`Ctrl+Shift+1`。注意 `<` `>` 是 `Shift+,` / `Shift+.`，**不是** ISO 反斜杠键。

**编辑与导航**

| 键 | 可用名 |
|---|---|
| Enter / Esc / Tab / Space | `ENTER` `RETURN` / `ESC` `ESCAPE` / `TAB` / `SPACE` `SPACEBAR` `" "` |
| **Backspace** | `BACKSPACE` `BKSP` |
| **Delete（向后删）** | `DELETE` `DEL` `FORWARDDELETE` |
| 方向键 | `LEFT` `LEFTARROW` / `RIGHT` `RIGHTARROW` / `UP` `UPARROW` / `DOWN` `DOWNARROW` |
| 翻页 / 首尾 | `PAGEUP` `PAGE_UP` `PGUP` / `PAGEDOWN` `PAGE_DOWN` `PGDN` / `HOME` / `END` |
| Insert / Menu | `INSERT` `INS` / `MENU` `APP` `APPLICATION` |
| 锁定键 | `CAPSLOCK` `CAPS_LOCK` `CAPS` / `SCROLLLOCK` `SCROLL_LOCK` / `PAUSE` `BREAK` |
| PrtSc / 功能键 | `PRINTSCREEN` `PRINT_SCREEN` `PRTSC` / `F1`–`F12` |

> ⚠️ **`DELETE` ≠ `BACKSPACE`。** HID 里 `0x2A` 是退格、`0x4C` 才是向后删除。
> 要退格用 `BACKSPACE`，要向后删用 `DELETE`。

**不支持**：小键盘数字区、`F13`–`F24`。

## 4. 修饰键名（`modifiers` 数组）

位图，可任意叠加（最多 8 个同按）。

| 键 | 可用名 | 位 |
|---|---|---|
| 左 Ctrl | `CTRL` `CONTROL` `LCTRL` | `0x01` |
| 左 Shift | `SHIFT` `LSHIFT` | `0x02` |
| 左 Alt | `ALT` `LALT` `OPTION` | `0x04` |
| 左 Meta / Win / Cmd | `META` `GUI` `SUPER` `CMD` `COMMAND` `WIN` `WINDOWS` | `0x08` |
| 右 Ctrl | `RCTRL` | `0x10` |
| 右 Shift | `RSHIFT` | `0x20` |
| 右 Alt / AltGr | `RALT` `ALTGR` | `0x40` |
| 右 Meta | `RMETA` | `0x80` |

**修饰键名不能当 `key` 用**，会被拒为 `400 unknown_key`——这是有意防护，防止把位值当键码发出
（那会发出 HID usage `0x01` Keyboard ErrorRollOver）。

## 5. 状态码 / 错误码

| 状态码 | `error` | 含义 |
|---|---|---|
| `200` | — | 成功 |
| `400` | `bad_json` | body 不是合法 JSON **对象**（数组/标量也算） |
| `400` | `unknown_key` | 按键名无法识别 |
| `400` | `unknown_modifier` | `modifiers` 里有无法识别的名字，或不是字符串数组 |
| `400` | `empty_report` | `key` 和 `modifiers` 都没给 |
| `404` | `not_found` | 路径不存在 |
| `405` | `method_not_allowed` | 方法不对 |
| `409` | `not_connected` | 无已连接的 HID 主机。**仅 `/key/tap`、`/key/down` 会返回** |
| `409` | `key_already_down` | 已有键被按住，先 `/key/up` |
| `413` | `body_too_large` | body > 4096 字节 |
| `415` | `unsupported_media_type` | 带了 `Content-Type` 但不含 `json` |
| `500` | `internal_error` | 兜底异常。**出现 500 说明有异常逃逸，请附日志反馈** |
| `502` | `hid_write_failed` | 已连接但 HID 报文发送失败 |
| `503` | `busy` | 并发请求超过 8 个 |

日志：`adb logcat -s BtRemoteHttp:D`

## 6. 完整使用流程

```bash
PHONE=192.168.1.100
CT='Content-Type: application/json'

# 1) 确认服务在跑、且已连上主机
curl -s http://$PHONE:8080/status
# {"ok":true,"hidConnected":true,"deviceName":"DESKTOP-XXXX","state":2,"port":8080,"apiVersion":1}

# 2) 单键
curl -s -X POST -H "$CT" -d '{"key":"ENTER"}'  http://$PHONE:8080/key/tap

# 3) 组合键
curl -s -X POST -H "$CT" -d '{"key":"C","modifiers":["CTRL"]}'             http://$PHONE:8080/key/tap
curl -s -X POST -H "$CT" -d '{"key":"DELETE","modifiers":["CTRL","ALT"]}'  http://$PHONE:8080/key/tap

# 4) 只按修饰键（如只按 Win 呼出开始菜单）
curl -s -X POST -H "$CT" -d '{"modifiers":["META"]}'  http://$PHONE:8080/key/tap

# 5) 长按（触发主机自动重复，如按住 Delete 删一段）
curl -s -X POST -H "$CT" -d '{"key":"DELETE","hold":2000}'  http://$PHONE:8080/key/tap

# 6) 按住 / 释放分离（做方向键连按、游戏移动）
curl -s -X POST -H "$CT" -d '{"key":"UP","timeout":5000}'  http://$PHONE:8080/key/down
curl -s -X POST -H "$CT" -d '{}'                           http://$PHONE:8080/key/up

# 7) 出了事就自救
curl -s -X POST -H "$CT" -d '{}'  http://$PHONE:8080/key/release_all
```

## 7. 行为细节与坑

### 7.1 一次只能按住一个普通键（硬性约束）

HID 键盘报文固定 2 字节 `[修饰键位图, 单个键码]`，键码槽只有 1 个（标准是 6 个）。且报文是
**全量覆盖**：

```
[0x00,0x52] → 主机认为「↑按住」
[0x00,0x04] → 主机认为「↑已松开，A按下」
```

所以 ✅ 修饰键+一个普通键、✅ 多个修饰键叠加、❌ **两个普通键同按**（如按住 `WASD` 再按空格跳）。

### 7.2 请求在锁上排队，不是一律立即失败

`/key/tap` 在**整个 hold 期间**持锁。因此一次 `hold:5000` 的请求期间：

- 另一个 `/key/tap` 会**阻塞等待**，等前一个结束后才执行（不是立刻 409）
- `/key/down` 在此时发起会等锁释放后**成功**；而在 `held` 已置位时发起才会 `409 key_already_down`
- `/status` 不碰锁，**永远秒回**——所以可以用它做健康探测

`/key/down` 生效后锁即释放，靠 `held` 互锁拦截，此时并发请求是**立刻**拿到 409。

### 7.3 服务生命周期取决于应用状态

端口**只在 HID 前台服务运行时**监听，而服务在**进入设备选择界面**时才启动
（`DevicesSelectionScreen` 的 `LaunchedEffect`）：

- 应用没开 / 没进设备选择界面 → 端口不监听（连接被拒绝）
- 停在设备选择界面时切后台 → 服务停止
- **划掉应用** → 服务必定停止
- 已连主机进入遥控界面后 → 可以切后台

客户端必须区分「端口连不上」（应用没跑）和「`409 not_connected`」（应用在跑但没配对主机）。

### 7.4 主机输入法状态会毁掉输出

实测：主机在中文组词模式下，字母会被组词（`bt`→「不同」），**修饰键组合可能被吞**
（`Ctrl+A` 变成普通 `a`）。与插物理键盘行为一致，不是本实现的缺陷。
**做自动化必须让主机处于英文/直接输入模式。**

### 7.5 卡键的三重保护

① `held` 互锁拒绝重复按下 → ② `/key/down` 看门狗超时自动释放 → ③ 蓝牙断开时监听连接状态流
自动 `releaseAll()`；服务 `onDestroy` 也会先关端口再释放键。

### 7.6 Content-Type 的实际判定

代码是 `contentType.contains("json", ignoreCase = true)`，且**没有该 header 时直接放行**：

- `Content-Type: application/json` → 通过
- **完全不发 Content-Type** → **也通过**
- `application/x-www-form-urlencoded` / `text/plain` → `415`

挡掉后两者是安全设计，见 §9。curl 用 `-d` 时默认发 `application/x-www-form-urlencoded`，
所以 curl 必须显式 `-H 'Content-Type: application/json'`。

### 7.7 硬编码上限

| 项 | 值 |
|---|---|
| 端口 | 固定 `8080`（改需重编译 `DEFAULT_HTTP_SERVER_PORT`） |
| 点按 hold | `25`–`10000` ms |
| 看门狗 timeout | `1000`–`30000` ms，默认 `30000` |
| 并发 | `8` |
| 请求体 | `4096` 字节 |

息屏静止后 Doze 可能丢弃进来的连接（症状：亮屏正常、闲置几分钟后超时），
把应用设为「不受电池限制」可缓解。WiFi 省电也可能增加延迟或丢包，对延迟敏感的场景
建议用 USB 转发：

```bash
adb forward tcp:8080 tcp:8080   # 之后访问 http://127.0.0.1:8080
```

## 8. 故障排查

| 现象 | 原因 |
|---|---|
| 连接被拒绝 | 应用没运行，或没进入设备选择界面 |
| `/status` 返回 `hidConnected:false` | 蓝牙未配对/未连接目标主机 |
| `/key/*` 返回 `415` | `Content-Type` 不含 `json`。curl 的 `-d` 默认发表单类型，加 `-H 'Content-Type: application/json'` |
| `/key/up` 返回 `200` 但似乎没释放 | 本来就没有键按住，`"released":false` 不是错误 |
| 返回 `409 key_already_down` | 有键被按住，先 `/key/up` 或 `/key/release_all` |
| 发出按键但主机无反应 | 主机输入法在组词模式；或主机焦点不在目标窗口 |
| 部分按键丢失 | 点按太短。主机侧有丢帧，重试或改用 `/key/down` + 延迟 + `/key/up` |
| 局域网连不上但 `adb forward` 可以 | WiFi 客户端隔离（访客网络/企业网络） |
| 闲置几分钟后超时 | Doze 或 WiFi 省电，见 §7.7 |

## 9. 🔒 安全

**默认无任何鉴权，且监听 `0.0.0.0`——同一局域网内任何设备都能调用。**

后果：任何人都能指挥手机在已配对主机上打字，**包括打开终端执行命令**。
共享/访客 WiFi 下这是即时可利用的暴露面。

内置两道缓解（**不足以替代鉴权**）：

1. 拒绝 `Content-Type` 不含 `json` 的请求。表单和 `text/plain` 属于 CORS 安全列表类型，
   浏览器可以跨域直接发送而无需 preflight；挡掉它们能阻止网页表单类的 CSRF。
   （不带 `Content-Type` 的裸请求会被放行，但那只影响手写请求——浏览器发起的跨域请求必带该头。）
2. **不返回任何 CORS 头**。跨域 JSON 请求会先发 preflight，而我们不响应 CORS 头，
   preflight 失败，恶意网页无法利用。
   **不要在未加鉴权的情况下添加 `Access-Control-Allow-Origin: *`**，那会拆掉这道防线。

加固按成本从低到高：设置项开关（默认关）→ 共享密钥请求头 → 只绑 WiFi 接口 →
绑 `127.0.0.1` 强制走 `adb forward`（最安全）。
