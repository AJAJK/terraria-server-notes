# Terraria 1.4.4.9 Docker 部署复盘报告（从“进不去”到稳定运行）

日期：2026-03-14 09:21:06 UTC  
环境：阿里云 ECS（Linux，CentOS/RHEL 系），Docker Compose  
客户端版本：Terraria 1.4.4.9（要求服务端必须匹配）

## 1. 背景与目标

- 目标：搭建一个 **Terraria 原版（vanilla）** 服务器，要求：
  - 客户端保持 **1.4.4.9**
  - 服务端也必须是 **1.4.4.9**
  - 运行方式优先用 Docker（方便管理、可迁移、可重启）
  - 世界/配置要持久化

## 2. 原理概览（为什么会“进不去”）

Terraria 联机常见的“进不去”可以分成两大类：

### 2.1 版本不匹配（100% 进不去）

- 客户端与服务端版本不一致时，会在握手阶段失败。
- 表现可能是“卡连接”“连接失败”“直接断开”等。
- 关键原则：**服务端版本必须与客户端一致**（此案例要求 1.4.4.9）。

### 2.2 网络与协议问题（会出现“偶尔能进、偶尔进不去”）

即使版本一致，也可能卡住：

- **TCP 端口通** ≠ 一定能玩：
  - Terraria 在连接与游戏过程中对 UDP 很敏感，UDP 抖动/丢包会导致：
    - 卡在“连接中”
    - 连接后立刻掉线
    - 服务器认为玩家还在线（幽灵会话）
- “折扣 is already on this server” 的典型原因：
  - 上一次连接异常断开（网络抖动/客户端崩溃/半连接），服务端仍保留会话状态；
  - 新连接会被拒绝或踢出。

### 2.3 Docker 端口映射链路的“路径差异”（本次最关键坑）

本次出现了一个非常典型但不容易想到的问题：

- 容器显示端口已映射、宿主机 `ss` 也显示 `LISTEN`：
  - `0.0.0.0:27015->7777/tcp`、`0.0.0.0:27015->7777/udp`
- 但客户端 `Test-NetConnection` 仍然失败；
- 使用 `tcpdump` 抓包发现：
  - 客户端 SYN 已到达 ECS 内网 IP（例如 `172.17.11.176:27015`）
  - 宿主机却立即回复 RST（Connection refused）
- 这说明：**实际到达主机的流量没有走到 Docker 端口转发表**（或被某种路径/接口差异绕过），导致主机网络栈直接拒绝连接。

最终通过 `network_mode: host` 绕过端口映射链路，问题消失。

## 3. 遇到的问题与定位过程

### 3.1 最初问题：使用镜像 `latest`，服务端变成 1.4.5.6

- 使用 `ghcr.io/beardedio/terraria:latest` 启动后，服务端显示 1.4.5.6
- 客户端为 1.4.4.9，必然无法进入
- 结论：仅更换镜像源/加速无法解决版本不匹配，因为镜像内容本身就是新版本。

### 3.2 获取并验证原版 1.4.4.9 服务端

- 获取到：`terraria-server-1449.zip`（Linux 版本文件夹里包含 `TerrariaServer.bin.x86_64`）
- 通过运行验证版本：
  - `./TerrariaServer.bin.x86_64 -help` 输出 `Terraria Server v1.4.4.9`

### 3.3 构建本地固定版本 Docker 镜像

- 不依赖镜像仓库提供 tag
- 直接把 1.4.4.9 服务端文件 COPY 到镜像里
- entrypoint 脚本按环境变量生成 `serverconfig.txt`
- 世界文件挂载为 volume 持久化

### 3.4 启动后仍偶发“卡连接”“幽灵在线”

- 日志可见 `is connecting...`，但客户端可能卡住
- 偶发能进入一次，然后再次连接提示 “already on this server”
- 结论：UDP 抖动/异常断开导致会话残留
- 临时处理：重启容器清理会话；更好的做法：提供 kick/save（tty 或 rcon）

### 3.5 改端口到 27015 后 TCP 不通：定位到端口映射路径问题

- 宿主机 `ss` 看见 `docker-proxy` `LISTEN :27015`
- Windows `Test-NetConnection` 仍失败
- `tcpdump` 看到 SYN 到达 ECS 内网 IP，但主机回 RST
- 结论：流量没进入 Docker 转发链路/路径存在差异

解决：使用 `network_mode: host` 运行容器。

## 4. 最终成功流程（可复用）

- 验证版本：运行 `TerrariaServer.bin.x86_64 -help`
- 构建本地镜像：COPY 服务器文件 + 生成 serverconfig
- Compose：`network_mode: host`，`PORT=27015`
- 安全组：放行 TCP/UDP 27015
- 验证：
  - `docker logs -f terraria`
  - Windows `Test-NetConnection`
  - `tcpdump` 抓包确认链路

## 5. 经验总结

1. 版本必须一致，`latest` 很容易踩坑  
2. Terraria 对 UDP 很敏感，TCP 通不代表可玩  
3. 日志 + 抓包是排障最快方法  
4. 云环境下 Docker 端口映射可能遇到路径/接口差异，`host` 网络很稳  
5. 运维要有 kick/save 手段，别只能靠重启

## 6. 本次排障对话链接

- https://github.com/copilot/share/006241be-0240-8407-9853-860c84dc0077