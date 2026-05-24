# Vulhub Redis Unauthorized Access Lab

## 项目介绍

本项目基于 Vulhub 搭建 Redis 未授权访问漏洞复现环境，对 Redis 服务在未设置认证、端口暴露等情况下产生的未授权访问风险进行安全验证与分析。

本项目主要完成：

- 下载并使用 Vulhub 漏洞环境
- 启动 Redis 未授权访问漏洞环境
- 使用 Docker Compose 管理漏洞容器
- 使用 redis-cli 连接 Redis 服务
- 验证无需密码即可访问 Redis
- 执行 PING、INFO、DBSIZE、SET、GET 等安全测试命令
- 分析 Redis 未授权访问漏洞成因
- 总结 Redis 安全加固建议
- 整理 Payload、漏洞报告和实验截图

---

## 技术栈

- Windows 11
- Docker Desktop
- Docker Compose
- Vulhub
- Redis
- redis-cli
- Git / GitHub

---

## 项目环境

| 组件 | 说明 |
|---|---|
| 操作系统 | Windows 11 |
| 容器环境 | Docker Desktop |
| 漏洞环境 | Vulhub |
| 漏洞服务 | Redis |
| 默认端口 | 6379 |
| 测试工具 | redis-cli |
| 测试类型 | 未授权访问 |

---

## 项目结构

```text
vulhub_redis_unauthorized_lab/
│
├── README.md
├── payloads/
│   └── redis_commands.txt
├── reports/
│   └── redis_unauthorized_report.md
└── screenshots/
    ├── vulhub_redis_directory.png
    ├── docker_redis_running.png
    ├── redis_unauthorized_ping.png
    ├── redis_info_success.png
    └── redis_set_get_success.png
```

---

## 一、什么是 Vulhub

Vulhub 是一个基于 Docker 的漏洞复现环境集合，里面包含 Redis、Tomcat、Fastjson、Log4j2、Spring、ThinkPHP 等多种真实漏洞环境。

相比 DVWA，Vulhub 更接近真实漏洞复现。  
DVWA 更适合学习 Web 基础漏洞，Vulhub 更适合学习真实服务漏洞和 CVE 复现。

---

## 二、什么是 Redis 未授权访问

Redis 是一种常见的内存数据库，默认端口通常为：

```text
6379
```

Redis 未授权访问漏洞指的是：

```text
Redis 服务暴露在网络中，并且没有设置认证或访问控制，导致攻击者无需密码即可连接 Redis 服务。
```

如果 Redis 未设置密码、监听外部地址，并且防火墙没有限制访问，就可能被未授权连接。

本实验只进行安全验证，包括：

- 连接 Redis
- 执行 PING
- 查看基础信息
- 写入和读取测试数据

不进行破坏性操作。

---

## 三、下载 Vulhub

建议将 Vulhub 下载到 E 盘。

打开 CMD 或 PowerShell，执行：

```bash
cd /d E:\
git clone --depth 1 https://github.com/vulhub/vulhub.git
```

下载完成后，会生成：

```text
E:\vulhub
```

---

## 四、进入 Redis 漏洞目录

进入 Redis 未授权访问环境目录：

```bash
cd /d E:\vulhub\redis\4-unacc
```

如果目录不存在，可以先查看 Redis 目录：

```bash
dir E:\vulhub\redis
```

根据实际目录名进入对应的 Redis 未授权访问环境。

实验截图：

![Vulhub Redis Directory](screenshots/vulhub_redis_directory.png)

---

## 五、启动漏洞环境

在漏洞目录下执行：

```bash
docker compose up -d
```

启动完成后查看容器状态：

```bash
docker ps
```

如果 Redis 容器正常运行，并且端口映射类似：

```text
0.0.0.0:6379->6379/tcp
```

说明 Redis 漏洞环境启动成功。

实验截图：

![Docker Redis Running](screenshots/docker_redis_running.png)

---

## 六、连接 Redis

### 方法一：本机安装了 redis-cli

如果 Windows 本机已经安装 redis-cli，可以直接执行：

```bash
redis-cli -h 127.0.0.1 -p 6379
```

进入 Redis 后输入：

```bash
PING
```

如果返回：

```text
PONG
```

说明连接成功。

---

### 方法二：进入 Docker 容器执行 redis-cli

如果本机没有 redis-cli，可以使用容器内的 redis-cli。

先查看容器名称：

```bash
docker ps
```

然后执行：

```bash
docker exec -it 容器名 redis-cli
```

例如：

```bash
docker exec -it redis-4-unacc-redis-1 redis-cli
```

进入后输入：

```bash
PING
```

如果返回：

```text
PONG
```

说明 Redis 可以在未认证情况下被访问。

实验截图：

![Redis Unauthorized Ping](screenshots/redis_unauthorized_ping.png)

---

## 七、验证未授权访问

### 1. PING 测试

```bash
PING
```

返回：

```text
PONG
```

说明 Redis 服务可连接。

---

### 2. 查看 Redis 信息

```bash
INFO
```

如果无需输入密码即可查看 Redis 版本、系统信息、连接信息等，说明 Redis 存在未授权访问风险。

实验截图：

![Redis Info Success](screenshots/redis_info_success.png)

---

### 3. 查看数据库 Key 数量

```bash
DBSIZE
```

返回类似：

```text
(integer) 0
```

说明可以直接查看数据库状态。

---

### 4. 写入测试数据

```bash
SET testkey hello_vulhub
```

如果返回：

```text
OK
```

说明当前连接具有写入权限。

---

### 5. 读取测试数据

```bash
GET testkey
```

如果返回：

```text
"hello_vulhub"
```

说明可以读取刚刚写入的数据。

实验截图：

![Redis Set Get Success](screenshots/redis_set_get_success.png)

---

## 八、漏洞成因分析

Redis 未授权访问通常由以下原因造成：

1. Redis 未设置访问密码；
2. Redis 监听地址配置不安全；
3. Redis 服务端口暴露到外部网络；
4. 防火墙或安全组没有限制访问来源；
5. 业务上线时忽略了 Redis 安全配置；
6. 内部服务被错误暴露到公网。

正常情况下，Redis 不应该允许任意客户端直接连接并执行命令。

---

## 九、漏洞危害

Redis 未授权访问可能造成：

- 敏感缓存数据泄露；
- Redis 中业务数据被读取；
- Redis 数据被篡改；
- Redis 数据被删除；
- 服务可用性受到影响；
- 在错误配置情况下可能造成进一步入侵风险。

本实验只验证基础连接和数据读写，不进行破坏性测试。

---

## 十、防御建议

为了防止 Redis 未授权访问，应采取以下措施：

1. 配置 Redis 认证密码；
2. 不将 Redis 暴露到公网；
3. 使用防火墙或安全组限制访问来源；
4. 将 Redis 绑定到本地地址或内网地址；
5. 禁止危险命令或重命名危险命令；
6. 使用最小权限原则部署服务；
7. 定期检查开放端口；
8. 对 Redis 访问日志进行监控；
9. 容器环境中避免不必要的端口映射。

---

## 十一、停止实验环境

实验结束后，在漏洞目录下执行：

```bash
docker compose down -v
```

该命令会停止并清理容器和相关数据卷。

---

## 项目总结

本项目完成了 Vulhub Redis 未授权访问漏洞的基础复现与分析。

通过本实验，掌握了：

- Vulhub 漏洞环境的下载和使用方式；
- Docker Compose 启动真实漏洞环境的方法；
- Redis 默认端口和基本连接方式；
- redis-cli 的基础使用；
- Redis 未授权访问的验证方法；
- PING、INFO、DBSIZE、SET、GET 等基础命令；
- Redis 未授权访问漏洞的成因；
- Redis 服务的安全加固方式。

本实验是从 DVWA 基础漏洞复现过渡到真实漏洞复现的重要一步。

---

## 免责声明

本项目仅用于 Web 安全学习、漏洞复现和安全研究，禁止用于任何非法用途。