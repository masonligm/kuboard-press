# Kuboard v4 本地部署排错记录

本文档记录用 Docker Compose 在 **Windows + Docker Desktop** 上部署 Kuboard v4(镜像 `swr.cn-east-2.myhuaweicloud.com/kuboard/kuboard:v4`,版本 v4.1.0.3)、并接入本地 Docker Desktop Kubernetes 集群过程中遇到的全部问题与解决办法。

最终可用的编排文件见同目录 [`docker-compose.yaml`](./docker-compose.yaml)。

---

## 目录

1. [数据库连接：`localhost` 连不上](#1-数据库连接localhost-连不上)
2. [核心坑：Kuboard v4 不支持标准 PostgreSQL](#2-核心坑kuboard-v4-不支持标准-postgresql)
3. [改用 MySQL 8 后成功启动](#3-改用-mysql-8-后成功启动)
4. [网页登录：用户名/密码错误](#4-网页登录用户名密码错误)
5. [用 DBeaver 等本地工具连接 MySQL](#5-用-dbeaver-等本地工具连接-mysql)
6. [接入 Docker Desktop 的 Kubernetes 集群](#6-接入-docker-desktop-的-kubernetes-集群)
7. [速查表](#速查表)

---

## 1. 数据库连接：`localhost` 连不上

**现象**：最初的 `docker run` 里写 `DB_URL=jdbc:postgresql://localhost:5432/...`，容器启动后连不上数据库。

**原因**：容器使用默认网络时，容器内的 `localhost` 指向**容器自己**，而不是宿主机或数据库容器。

**解决**：
- 同一个 compose 内的服务之间，用**服务名**互通（如 `mysql`、`postgres`），不要用 `localhost`。
- 若数据库在宿主机上，用 `host.docker.internal`（并按需加 `--add-host=host.docker.internal:host-gateway`）。

---

## 2. 核心坑：Kuboard v4 不支持标准 PostgreSQL

**现象**：用 `DB_DRIVER=org.postgresql.Driver` 连 PostgreSQL 12/15，Kuboard 启动即崩溃重启，反复报：

```
org.postgresql.util.PSQLException: Protocol error.  Session setup failed.
    at org.postgresql.core.v3.ConnectionFactoryImpl.doAuthentication(...)
  (来自 cn.kuboard.config.DatabaseIdConfig.sqlMode → DdlService)
```

**排查关键**：
- 同网络下用 `psql` 客户端 + **相同凭据** 能成功连上同一个 PostgreSQL（`select version()` 正常）。
- 但 Kuboard 始终连不上，且换 PostgreSQL 12 / 15 都报一模一样的错。
- → 说明问题不在网络/密码/schema，而在 **Kuboard 自带的 JDBC 驱动与协议**。

**根本原因**：检查镜像内 `kuboard-server.jar` 的 `BOOT-INF/lib/`，只捆绑了三个数据库驱动：

```
mysql-connector-j-8.4.0.jar
mariadb-java-client-3.5.7.jar
opengauss-jdbc-6.0.3.jar      ← 配置里的 org.postgresql.Driver 实际加载的是它
```

`org.postgresql.Driver` 实际加载的是 **openGauss 的 JDBC 驱动**（pgjdbc 的分支，沿用了 `org.postgresql` 包名）。它的认证握手是 openGauss 协议，连真正的 PostgreSQL 必然在认证阶段失败。**Kuboard v4 官方根本不支持标准 PostgreSQL。**

**官方支持的数据库**（来自 <https://kuboard.cn/v4/install/>）：

| 数据库 | 版本 | DB_DRIVER |
|---|---|---|
| MySQL | >= 5.7 | `com.mysql.cj.jdbc.Driver` |
| MariaDB | — | `org.mariadb.jdbc.Driver` |
| OpenGauss | >= 3.0 | `org.postgresql.Driver` |

> 你最初抄到的 `org.postgresql.Driver` 配置，其实是文档里给 **OpenGauss** 用的，不是给标准 PostgreSQL 的。

---

## 3. 改用 MySQL 8 后成功启动

把数据库换成官方支持的 **MySQL 8.4**：

```yaml
DB_DRIVER: com.mysql.cj.jdbc.Driver
DB_URL: "jdbc:mysql://mysql:3306/kuboard?serverTimezone=Asia/Shanghai&useUnicode=true&characterEncoding=utf8"
DB_USERNAME: kuboard
DB_PASSWORD: Kuboard123
```

启动日志正常：

```
数据库名称: MySQL, 版本: 8.4.9
执行数据库安装脚本: 1-create-tables.sql
执行数据库安装脚本: 2-init-data.sql      ← 初始化数据(含 admin 账号)
...
Tomcat started on port 80 (http)
Started KuboardServer in 30.28 seconds
```

**关于一条无害的 WARN**：

```
WARN  执行 SQL 文件失败 .../v4.1.0.2/kb_cluster_sync_data.sql：
      ALTER TABLE kb_cluster_sync_data DROP INDEX idx_kb_cluster_sync_data_events
```

这是升级脚本想删除一个索引，但**全新安装**的库里该索引本就不存在，删除失败 → 只记 WARN 并继续，**不影响功能**。

访问地址：**http://localhost:8098**（或 `http://宿主机IP:8098`）。

---

## 4. 网页登录：用户名/密码错误

**现象**：登录页报「服务异常！用户名或密码错误」。

**原因**：把**数据库用户名 `kuboard`** 当成了网页登录账号。两者完全无关。

**解决**：Kuboard 网页后台默认管理员账号：

| 字段 | 值 |
|---|---|
| 用户名 | **`admin`** |
| 密码 | **`Kuboard123`**（大写 K，区分大小写） |

> ⚠️ 巧合：官方默认**登录密码**与本 compose 里设的**数据库密码**都恰好是 `Kuboard123`，但它们是独立的两个东西。
>
> 密码策略：连续错 **5 次锁定 60 秒**（`kb_u_user` 表的 `password_try_count` / `password_lock_*` 字段）。

---

## 5. 用 DBeaver 等本地工具连接 MySQL

容器默认不对外暴露端口，需在 compose 的 `mysql` 服务加端口映射（用 `13306` 避免与宿主机本地 MySQL 3306 冲突）：

```yaml
ports:
  - "13306:3306"
```

**连接参数**：

| 参数 | 值 |
|---|---|
| Host | 本机 `127.0.0.1`；其它机器用宿主机局域网 IP |
| Port | `13306` |
| Database | `kuboard` |
| Username | `kuboard`（推荐）或 `root` |
| Password | `Kuboard123` |

**MySQL 8.4 常见坑**（`caching_sha2_password`）：明文 TCP 连接时 DBeaver 常报 `Public Key Retrieval is not allowed`。在连接设置里：

- Driver properties：`allowPublicKeyRetrieval` = `true`
- SSL：不勾启用（或 `useSSL=false`）

等价 JDBC URL：`jdbc:mysql://127.0.0.1:13306/kuboard?allowPublicKeyRetrieval=true&useSSL=false`

**给其它机器访问**还需在宿主机放行入站端口（管理员 PowerShell）：

```powershell
New-NetFirewallRule -DisplayName "Kuboard MySQL 13306" -Direction Inbound -Protocol TCP -LocalPort 13306 -Action Allow
```

**账号权限区别**：
- `kuboard@%`：权限限于 `kuboard` 库，日常够用，推荐。
- `root@%`：超级权限，谨慎使用。
- （`root@localhost` 经端口映射连不进来，因为来源 IP 不是 localhost。）

---

## 6. 接入 Docker Desktop 的 Kubernetes 集群

Kuboard「添加集群」要求粘贴 kubeadm 生成的 kubeconfig。**Docker Desktop 的 K8s 正是 kubeadm 引导的**，其 kubeconfig 在 `C:\Users\<用户>\.kube\config`（context `docker-desktop`），符合要求，但**不能原样粘贴**。

### 两个必须处理的问题

**① server 地址要换成容器可达的地址**

原文件是 `server: https://127.0.0.1:6443`，在 Kuboard 容器内 `127.0.0.1` 指向容器自己，连不上。

经测试，Kuboard 容器内 `kubernetes.docker.internal` 能解析（→ `192.168.65.3`）且 `6443` 可达。把 server 改为：

```yaml
server: https://kubernetes.docker.internal:6443
```

**② 必须保留 `certificate-authority-data`，且主机名要匹配证书 SAN**

- Kuboard 导入逻辑**强制要求** `certificate-authority-data`；删掉它（改用 `insecure-skip-tls-verify`）会报 `certificate-authority-data is invalid`。
- Docker Desktop API Server 证书的 SAN 为：

  ```
  DNS: docker-for-desktop, kubernetes, kubernetes.default,
       kubernetes.default.svc, kubernetes.default.svc.cluster.local,
       kubernetes.docker.internal, localhost, vm.docker.internal
  IP : 10.96.0.1, 0.0.0.0, 127.0.0.1, 192.168.65.3
  ```

- `host.docker.internal` **不在** SAN 里，但 `kubernetes.docker.internal` 在 → 所以用它既能解析、又能通过 TLS 校验，**无需** `insecure-skip-tls-verify`，也无需改 compose。

### 最终可导入的 kubeconfig（相对原文件只改了 server 一行）

```yaml
apiVersion: v1
kind: Config
current-context: docker-desktop
clusters:
- cluster:
    certificate-authority-data: <保留原文件的值不变>
    server: https://kubernetes.docker.internal:6443   # 唯一改动：127.0.0.1 → kubernetes.docker.internal
  name: docker-desktop
contexts:
- context:
    cluster: docker-desktop
    user: docker-desktop
  name: docker-desktop
users:
- name: docker-desktop
  user:
    client-certificate-data: <保留原文件的值不变>
    client-key-data: <保留原文件的值不变>
```

> ⚠️ 安全提醒：`client-key-data` 是集群 admin 私钥（最高权限），**切勿公开分享**。

### 诊断命令备忘

```bash
# 在 kuboard-v4-deploy 目录下，验证容器到 API Server 的可达性
docker compose exec -T kuboard sh -c "getent hosts kubernetes.docker.internal"
docker compose exec -T kuboard sh -c "timeout 5 bash -c '</dev/tcp/kubernetes.docker.internal/6443' && echo REACHABLE"

# 查看 API Server 证书 SAN（决定 server 用哪个主机名/IP）
docker run --rm mysql:8.4 sh -c 'echo | openssl s_client -connect kubernetes.docker.internal:6443 2>/dev/null | openssl x509 -noout -text' | grep -A2 "Subject Alternative Name"
```

---

## 速查表

| 项目 | 值 |
|---|---|
| Kuboard 访问地址 | http://localhost:8098 |
| Kuboard 登录账号 | `admin` / `Kuboard123` |
| 数据库 | MySQL 8.4（**不能用标准 PostgreSQL**） |
| DB 连接（容器内） | host=`mysql` port=`3306` |
| DB 连接（本地工具） | host=`127.0.0.1` port=`13306`，user `kuboard`/`Kuboard123` |
| DB 额外参数 | `allowPublicKeyRetrieval=true&useSSL=false` |
| 集群 kubeconfig server | `https://kubernetes.docker.internal:6443`（保留 CA） |

### 常用运维命令（在 `kuboard-v4-deploy` 目录下）

```bash
docker compose up -d            # 启动
docker compose ps               # 状态
docker compose logs -f kuboard  # 看 Kuboard 日志
docker compose down             # 停止（保留数据）
docker compose down -v          # 停止并删除数据卷（清空数据库，慎用）
```
