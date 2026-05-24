# Redis 未授权访问漏洞复现报告

# 1. 实验简介

本实验基于 Vulhub 搭建 Redis 未授权访问漏洞环境，对 Redis 服务在未设置认证和访问控制情况下产生的未授权访问风险进行复现与分析。

本实验主要目标：

- 学习 Vulhub 漏洞环境的基本使用；
- 使用 Docker Compose 启动 Redis 漏洞环境；
- 查看 Redis 容器运行状态和端口映射情况；
- 使用容器内置 redis-cli 连接 Redis 服务；
- 验证无需密码即可访问 Redis；
- 执行 PING、INFO、DBSIZE、SET、GET 等安全测试命令；
- 分析 Redis 未授权访问漏洞成因；
- 总结 Redis 安全加固建议。

---

# 2. 实验环境

| 组件 | 说明 |
|---|---|
| 操作系统 | Windows 11 |
| 容器环境 | Docker Desktop |
| 漏洞环境 | Vulhub |
| 漏洞服务 | Redis |
| 默认端口 | 6379 |
| 测试工具 | redis-cli |
| Redis 容器名 | 4-unacc-redis-1 |
| 漏洞类型 | 未授权访问 |

---

# 3. Vulhub 简介

Vulhub 是一个基于 Docker 的漏洞复现环境集合，包含多个常见真实漏洞环境。

相比 DVWA，Vulhub 更适合真实漏洞复现。DVWA 更偏向基础 Web 漏洞学习，而 Vulhub 可以用于复现真实服务漏洞、组件漏洞和 CVE 漏洞。

本实验选择 Redis 未授权访问漏洞作为第一个 Vulhub 真实漏洞复现项目。

---

# 4. Redis 未授权访问漏洞原理

Redis 是一种常见的内存数据库，默认端口通常为：

```text
6379
```

Redis 未授权访问漏洞是指：

```text
Redis 服务暴露在网络中，并且没有设置密码或访问控制，导致客户端无需认证即可连接 Redis 并执行命令。
```

正常情况下，Redis 不应该允许任意客户端直接连接。

如果 Redis 存在以下情况，就可能产生未授权访问风险：

- Redis 未设置访问密码；
- Redis 监听地址配置不安全；
- Redis 端口 6379 暴露到外部；
- 防火墙或安全组没有限制访问来源；
- Docker 容器部署时错误映射 Redis 端口。

---

# 5. 下载 Vulhub

在 Windows 11 中打开 CMD，进入 E 盘：

```bash
cd /d E:\
```

下载 Vulhub：

```bash
git clone --depth 1 https://github.com/vulhub/vulhub.git
```

下载完成后会生成目录：

```text
E:\vulhub
```

---

# 6. 进入 Redis 漏洞环境目录

执行：

```bash
cd /d E:\vulhub\redis\4-unacc
```

如果目录不存在，可以执行：

```bash
dir E:\vulhub\redis
```

查看 Redis 目录下的实际漏洞环境名称。

---

# 7. 启动漏洞环境

在 Redis 未授权访问目录下执行：

```bash
docker compose up -d
```

启动完成后查看容器运行状态：

```bash
docker ps
```

如果看到 Redis 容器正在运行，并且端口映射到：

```text
6379
```

说明漏洞环境启动成功。

本实验中 Redis 容器名称为：

```text
4-unacc-redis-1
```

---

# 8. 使用 redis-cli 连接 Redis

## 8.1 本机 redis-cli 连接方式

如果 Windows 本机已经安装 redis-cli，可以执行：

```bash
redis-cli -h 127.0.0.1 -p 6379
```

进入 Redis 命令行后，提示符通常类似：

```text
127.0.0.1:6379>
```

---

## 8.2 容器内 redis-cli 连接方式

如果本机没有安装 redis-cli，可以使用 Redis 容器内置的 redis-cli。

先查看容器名称：

```bash
docker ps
```

本实验中的 Redis 容器名为：

```text
4-unacc-redis-1
```

因此执行：

```bash
docker exec -it 4-unacc-redis-1 redis-cli
```

进入 Redis 命令行后，提示符会变成类似：

```text
127.0.0.1:6379>
```

该方式表示进入 Redis 容器内部，并使用容器内置的 redis-cli 连接 Redis 服务。

---

# 9. 未授权访问验证

## 9.1 PING 测试

进入 Redis 命令行后执行：

```bash
ping
```

返回：

```text
PONG
```

说明 Redis 服务可以正常连接。

如果没有要求输入密码，说明 Redis 可以在未认证情况下被访问，存在未授权访问风险。

---

## 9.2 INFO 信息查看

执行：

```bash
INFO
```

可以查看 Redis 服务器信息、版本信息、系统信息、连接信息等。

如果无需认证即可查看这些信息，说明 Redis 服务存在信息泄露风险。

---

## 9.3 DBSIZE 查看数据库状态

执行：

```bash
DBSIZE
```

返回结果类似：

```text
(integer) 0
```

说明可以直接查看当前 Redis 数据库中 Key 的数量。

---

## 9.4 SET 写入测试

执行：

```bash
SET testkey hello_vulhub
```

返回：

```text
OK
```

说明可以向 Redis 写入测试数据。

该操作证明当前连接不仅可以访问 Redis，还具备写入权限。

---

## 9.5 GET 读取测试

执行：

```bash
GET testkey
```

返回：

```text
"hello_vulhub"
```

说明可以读取 Redis 中的数据。

该操作证明 Redis 中的数据可以被未授权读取。

---

# 10. 实验结果分析

通过以上测试可以确认：

```text
Redis 服务无需认证即可连接；
Redis 支持未授权执行 PING；
Redis 支持未授权执行 INFO；
Redis 支持未授权执行 DBSIZE；
Redis 支持未授权执行 SET；
Redis 支持未授权执行 GET。
```

因此，该环境存在 Redis 未授权访问漏洞。

本实验只使用了安全验证命令，没有执行破坏性操作。

---

# 11. 漏洞成因

Redis 未授权访问通常由以下原因导致：

1. Redis 未设置访问密码；
2. Redis 监听地址配置为外部可访问；
3. Redis 端口 6379 暴露到公网或宿主机外部；
4. 缺少防火墙或安全组访问限制；
5. Docker 部署时错误映射 Redis 端口；
6. 运维人员忽略 Redis 默认安全配置。

本实验环境中，Redis 服务可以在无需认证的情况下被 redis-cli 直接连接，因此能够验证未授权访问风险。

---

# 12. 漏洞危害分析

Redis 未授权访问可能导致：

- Redis 中缓存数据泄露；
- Redis 中业务数据被读取；
- Redis 中数据被恶意修改；
- Redis 数据被删除；
- 服务稳定性受到影响；
- 敏感配置或业务状态泄露；
- 在错误配置情况下可能进一步扩大攻击面。

Redis 未授权访问在真实环境中属于高风险配置问题。

本实验只验证基础连接、信息查看和普通测试数据读写，不进行破坏性测试。

---

# 13. 防御建议

## 13.1 设置 Redis 访问密码

在 Redis 配置中设置认证密码，避免未授权客户端直接连接。

---

## 13.2 限制监听地址

不要将 Redis 监听到所有网卡。

应尽量绑定到：

```text
127.0.0.1
```

或仅允许内网地址访问。

---

## 13.3 限制端口访问

使用防火墙或安全组限制 Redis 端口：

```text
6379
```

只允许可信主机访问。

---

## 13.4 避免暴露到公网

Redis 不应直接暴露到公网环境。

生产环境中应放在内网，并通过应用服务访问。

---

## 13.5 容器部署时避免错误端口映射

在 Docker 部署 Redis 时，应避免不必要地将 6379 端口映射到宿主机外部网络。

如果只是容器内部服务通信，应使用 Docker 内部网络，而不是直接暴露端口。

---

## 13.6 禁用或重命名高风险命令

可以根据业务需求禁用或重命名高风险命令，降低误操作或攻击风险。

---

## 13.7 最小权限原则

Redis 服务应使用低权限账户运行，降低漏洞影响范围。

---

## 13.8 日志监控

对 Redis 访问行为进行日志审计，及时发现异常连接和异常命令。

---

# 14. 停止实验环境

实验结束后，回到 Redis 漏洞目录：

```bash
cd /d E:\vulhub\redis\4-unacc
```

执行：

```bash
docker compose down -v
```

该命令会停止容器并清理数据卷。

---

# 15. 实验总结

本实验完成了 Vulhub Redis 未授权访问漏洞的基础复现与分析。

实验过程中完成了：

- Vulhub 下载；
- Redis 未授权访问环境启动；
- Docker 容器状态查看；
- 使用容器内 redis-cli 连接 Redis；
- PING 命令验证连接；
- INFO 命令查看 Redis 信息；
- DBSIZE 命令查看数据库状态；
- SET / GET 命令进行安全读写测试；
- Redis 未授权访问漏洞成因分析；
- Redis 安全加固方式总结。

通过本实验，理解了 Redis 未授权访问的核心问题：

```text
服务暴露在网络中，但缺少认证和访问控制。
```

该项目作为从 DVWA 基础漏洞复现过渡到 Vulhub 真实漏洞复现的第一个项目，能够帮助理解真实服务漏洞的复现流程和安全加固思路。

---

# 免责声明

本项目仅用于 Web 安全学习、漏洞复现和安全研究，禁止用于任何非法用途。