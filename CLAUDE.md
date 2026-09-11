# CLAUDE.md — android-bt-remote

## 目录职责

本仓库是上游 [Atharok/BtRemote](https://gitlab.com/Atharok/BtRemote) 的**构建外壳（fork wrapper）**，
自身**不含应用源码**。它的全部职责是：把 `patches/` 里的补丁打到 `src/` 子模块上、覆盖资源、改写包名与版本号，
然后交给上游的 Gradle 工程编译出 APK/AAB。

因此**任何代码改动都不会直接写在本仓库里**，而是写成 `patches/NNNN-*.patch`（见下方「开发流程」）。

## 文件清单

| 路径 | 说明 |
|---|---|
| `build.sh` | 核心构建脚本：`git am` 应用全部补丁 → 复制 `res/` 覆盖层 → `sed` 改写包名/版本号/链接。**不调用 gradle** |
| `generate.sh` | 从 `src` 的提交历史重新生成 `patches/*.patch` |
| `patches/*.patch` | 本 fork 相对上游的**全部**改动。这是 fork 唯一的产物，丢失即等于丢失全部工作 |
| `src/` | git submodule，指向上游 BtRemote。**不要在里面直接留下未提交的改动** |
| `res/` | 资源覆盖层，按路径合并进 `src/app/src/main/res/`（同名文件覆盖） |
| `fastlane/` | F-Droid 上架元数据（描述、截图） |
| `.github/workflows/apk.yml` | CI：打补丁 → Gradle 构建 → 发 GitHub Release |
| `README.md` | 面向用户的说明。HTTP API 一节只留摘要，细节链接到 `docs/API.md` |
| `docs/API.md` | HTTP 远程输入微服务的**完整接口文档**（对应补丁 `0004`、`0005`）。描述的是**行为**，不在补丁 diff 里 |
| `LICENSE` | GPLv3 |

## 调用链

```
.github/workflows/apk.yml
    └── ./build.sh
            ├── cd src && git am ../patches/*.patch   # 字典序应用，补丁编号即顺序
            ├── cp -R ../res/. app/src/main/res/
            └── sed -i  改 applicationId / versionCode / versionName / app_name / 链接

开发者本地（逆过程）
    cd src → 改代码 → git commit → ../generate.sh → patches/*.patch
```

`build.sh` 里 `sed` 改写的目标文件：`app/build.gradle.kts`（包名、版本号、版本码）、
`app/src/main/res/values*/strings.xml`（`Android TV` 文案、`app_name`）、
`app/src/main/java/com/atharok/btremote/common/utils/Constants.kt`（`SOURCE_CODE_LINK`、`WEB_SITE_LINK`）。

## 关键规则与注意事项

### 开发流程（顺序不能变）

1. `cd src && git am --whitespace=nowarn --keep-non-patch ../patches/*.patch` 把现有补丁打成真实提交
2. 在 `src/` 内改代码，`git commit`
3. **先自查基线**：`git merge-base origin/main HEAD` 必须仍是上游基线提交，
   `git log --oneline <基线>..HEAD` 必须正好是「现有补丁数 + 1」个提交
4. 回到仓库根目录运行 `./generate.sh`
5. 可选：`git checkout -B build-loop <基线>` 后跑 `./build.sh` 验证整个补丁栈能干净应用。
   没装 gradle 时用 worktree 等效验证（不动当前 HEAD，也不需要 Android SDK）：
   `git -C src worktree add --detach /tmp/pv <基线>` → 在 `/tmp/pv` 里
   `git am --whitespace=nowarn --keep-non-patch <仓库>/patches/*.patch` →
   `git rev-parse HEAD^{tree}` 应与 `git -C src rev-parse http-remote^{tree}` **逐字节相同** →
   `git -C src worktree remove /tmp/pv --force`

**顺序恒为：改 → commit → `generate.sh` →（可选）`build.sh`。绝不能先 `build.sh` 再 `generate.sh`**——
`build.sh` 的 `sed -i` 会就地修改子模块工作树，若之后 `git add -A`，这些构建期改写会被污染进补丁。

### 陷阱

- **绝不在仓库根目录执行 `git add -A` / `git add .`**。`src` 是子模块，在子模块里建分支或提交后，
  根仓库的 `git status` 会显示 ` M src`。一旦把这个 gitlink 提交进去，CI 就会 check out 到错误的
  上游提交，`git am` 直接失败。只提交 `patches/` 和文档。
- **`generate.sh` 没有安全检查**：它先 `rm -f patches/*.patch` 再 `format-patch`。若基线取不到，
  会产出 0 个补丁而旧补丁已被删除。因为 `patches/` 受本仓库跟踪，可用
  `git checkout -- patches/` 恢复。所以第 3 步的自查每次都要做。
- **`generate.sh` 复现不出原始补丁的排版**。现有的 `0001`–`0003` 是 1 行上下文（`-U1`），
  而 `git format-patch` 默认 3 行上下文并会合并相邻 hunk。重跑 `generate.sh` 会让这三个文件
  出现纯排版差异（blob 哈希不变，功能等价）。**新补丁只提交自己那一个，不要顺带重写无关补丁。**
- **新增补丁必须零填充命名**（`0004-`、`0005-`…）。`build.sh` 靠 shell 通配符展开，字典序即应用顺序。
  补丁的 diff 上下文依赖它前面所有补丁已应用，因此不能重排、不能改名、不能删中间某个。
- **`build.sh` 不能连跑两次**：第二次 `git am` 会发现补丁已应用而报错。重测前先 reset 到基线。
- **版本号硬编码在 `build.sh` 顶部**（`VERSION_CODE` / `VERSION_NAME`）。git tag 只用于命名
  GitHub Release，**不影响 APK 内的版本号**。要改 APK 版本必须改 `build.sh`。

### 文档与代码的一致性

`docs/API.md` 描述的是**运行时行为**——状态码、钳制范围、并发排队语义、服务生命周期。
这类信息**在补丁 diff 里看不出来**，所以改动 `src` 里的 HTTP 服务后必须回来同步更新它，
否则文档会静默偏离实现。本仓库已经发生过一次：README 曾声称「所有 `/key/*` 未连接都返回
`409 not_connected`」「必须带 `Content-Type: application/json`」，两处均与代码不符。

权威实现是这三个文件（详见 `docs/CLAUDE.md`）：

- `presentation/http/HttpRemoteServer.kt` — 路由、状态码、上限、Content-Type 判定
- `domain/usecases/HttpKeyboardUseCase.kt` — Mutex、`held` 互锁、看门狗
- `domain/entities/remoteInput/keyboard/KeyboardKeyNames.kt` — 键名与修饰键名表

### CI

触发方式：推 `release/**` 分支、推 `v*.*.*` tag、或在 Actions 页手动 `workflow_dispatch`（需填版本号）。

需要两个仓库 Secret，否则 release 签名失败：

- `STORE` — keystore 文件的 base64
- `LOCAL` — `local.properties` 的 base64，内容为 `storeFile` / `storePassword` / `keyAlias` / `keyPassword`

CI 只构建 `default` flavor（`assembleDefaultRelease` + `bundleDefaultRelease`）。

### 子模块内的架构（改代码时相关）

若无 `src/CLAUDE.md`，以下是最关键的几条：

- 所有 HID 写入都收敛到 `data/bluetooth/BluetoothHidCore.sendReport(id, bytes)` 一个函数
- 键盘报文固定 **2 字节**：`[修饰键位图, 单个键码]`。同一时刻只能按一个非修饰键，
  修饰键可在位图里任意叠加。描述符定义在 `common/utils/Constants.kt` 的 `bluetoothHidDescriptor`
- `REMOTE_INPUT_NONE` 被键盘报表（`0x01`）和遥控报表共用，**不要加宽或写入它**
- 依赖注入用 Koin，注册集中在 `common/injections/Modules.kt`
- 键盘映射枚举 `KeyboardKey` 是**位置式命名**（`ROW_3_KEY_00` 是 `a`），
  `KEY_DELETE(0x2A)` 实际是 **Backspace**，不是 Delete
- 该 Gradle 工程**没有启用 `kotlin.serialization` 编译器插件**，`kotlinx-serialization-json`
  只能用作运行时 API（`Json.parseToJsonElement` 等），`@Serializable` 编译不过
