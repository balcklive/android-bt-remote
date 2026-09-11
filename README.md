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
| **`0004-http-remote.patch`** | **HTTP 远程输入微服务**（接口文档见 [`docs/API.md`](docs/API.md)） |

---

# HTTP 远程输入微服务

`0004-http-remote.patch` 在应用内嵌了一个 HTTP 服务，让局域网内任意脚本/程序通过发 HTTP 请求来
**模拟键盘按键**，手机端无需任何操作。

```bash
curl -X POST -H 'Content-Type: application/json' \
     -d '{"key":"ENTER"}' http://192.168.1.100:8080/key/tap
```

**📖 完整接口文档见 [`docs/API.md`](docs/API.md)** —— 端点、键名/修饰键表、状态码、调用时序、
并发语义与全部陷阱都在那里。本节只列速查。

| 端点 | 作用 |
|---|---|
| `GET /status` | 查询连接状态（永远 `200`） |
| `POST /key/tap` | 点按 / 长按（`hold`，钳制 `[25, 10000]` ms） |
| `POST /key/down` | 按下并保持（`timeout` 看门狗强制生效） |
| `POST /key/up` | 释放，幂等 |
| `POST /key/release_all` | 兜底释放——客户端状态错乱时的自救手段 |

几条最容易踩的约束：

- **一次只能按住一个普通键**。HID 键盘报文固定 2 字节 `[修饰键位图, 单个键码]`，键码槽只有 1 个，
  所以「按住 `WASD` 同时按空格」这类操作做不了
- 修饰键可任意叠加，支持 `Ctrl+C`、`Ctrl+Shift+S`；也支持**只按修饰键**（如只按 Win）
- **端口只在进入设备选择界面后才监听**，划掉应用即停止。客户端要区分「端口连不上」和
  「`409 not_connected`」——前者是应用没跑，后者是应用在跑但没配对主机
- 主机必须处于**英文/直接输入模式**，否则输入法会组词，甚至吞掉 `Ctrl+A` 这类组合键
- `/status` 不碰内部的互斥锁，**永远秒回**，适合做健康探测

## ⚠️ 安全

**默认无鉴权且监听 `0.0.0.0`——同一局域网内任何设备都能调用**，可指挥手机在已配对主机上打字，
包括打开终端执行命令。共享/访客 WiFi 下请勿开启。详见 [`docs/API.md`](docs/API.md) §9。

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
