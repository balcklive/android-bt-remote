# CLAUDE.md — docs/

## 目录职责

本目录存放本 fork 的补充文档，**纯文档，不参与构建**。`build.sh`、CI 与 Gradle 工程都不读取这里，
在此新增文件不会影响 APK 产物。

本目录只描述**本 fork 自己引入的东西**（即 `patches/` 中的改动）。上游 `src/` 自身的文档不在此处。

## 文件清单

| 文件 | 说明 |
|---|---|
| `API.md` | HTTP 远程输入微服务（补丁 `0004-http-remote.patch`、`0005-Turn-off-Nagle-on-HTTP-client-sockets.patch`）的完整接口说明：端点、键名/修饰键表、状态码、调用时序、并发语义、TCP 连接复用（keep-alive）与延迟、限制与安全 |
| `CLAUDE.md` | 本文件 |

## 调用链

```
README.md「# HTTP 远程输入微服务」（摘要 + 端点速查表）
    └── 链接到 docs/API.md（细节的唯一来源）

docs/API.md
    └── 描述 patches/0004-http-remote.patch 与 0005-Turn-off-Nagle-on-HTTP-client-sockets.patch
        的运行时行为（后者只在 accept 后设 TCP_NODELAY，改变的是延迟，不是接口）
            └── 权威实现——改代码时以这三个文件为准：
                src/app/src/main/java/com/atharok/btremote/
                    ├── presentation/http/HttpRemoteServer.kt
                    │       HTTP 层：路由、状态码、body 上限、并发上限、Content-Type 判定
                    ├── domain/usecases/HttpKeyboardUseCase.kt
                    │       并发与看门狗：Mutex、held 互锁、holdToken 失效机制
                    └── domain/entities/remoteInput/keyboard/KeyboardKeyNames.kt
                            键名表与修饰键名表（含自带 Shift 的字符）
```

## 关键规则

- **细节只写一处**。`README.md` 只保留摘要与链接，篇幅细节一律放 `docs/API.md`。
  同一份接口在两处描述，是本仓库已经栽过的坑：README 曾声称「所有 `/key/*` 未连接都返回
  `409 not_connected`」「必须带 `Content-Type: application/json`」，两处均与代码不符，
  于 `e13d2f1` 修正。
- **改 HTTP 服务必须同步改 `docs/API.md`**。它描述的是**行为**——状态码、钳制范围、并发与排队
  语义——这类信息在 diff 里读不出来，改代码时最容易漏。改动以下任一项都要回来更新：
  端点或字段、键名/修饰键名表、状态码与错误码、各项钳制范围、服务生命周期、并发语义、
  连接与延迟特性（keep-alive、socket 选项）。
- **写文档前先读代码，不要照抄旧文档或上游**。文中每个数值（`8080`、`25`、`10000`、`1000`、
  `30000`、`8`、`4096`）都对应 `common/utils/Constants.kt` 里的具名常量，引用时以那里为准。
- 文档中每处「无法做到」都要说清**为什么**（例如「一次只能按一个普通键」源于 2 字节 HID 报文），
  否则下一个人会当成实现偷懒而去「修」它。
