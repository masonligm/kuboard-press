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

**官方支持的数据库**（来自 [https://kuboard.cn/v4/install/](https://kuboard.cn/v4/install/)）：


| 数据库    | 版本   | DB_DRIVER                  |
| --------- | ------ | -------------------------- |
| MySQL     | >= 5.7 | `com.mysql.cj.jdbc.Driver` |
| MariaDB   | —     | `org.mariadb.jdbc.Driver`  |
| OpenGauss | >= 3.0 | `org.postgresql.Driver`    |

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


| 字段   | 值                                     |
| ------ | -------------------------------------- |
| 用户名 | **`admin`**                            |
| 密码   | **`Kuboard123`**（大写 K，区分大小写） |

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


| 参数     | 值                                         |
| -------- | ------------------------------------------ |
| Host     | 本机`127.0.0.1`；其它机器用宿主机局域网 IP |
| Port     | `13306`                                    |
| Database | `kuboard`                                  |
| Username | `kuboard`（推荐）或 `root`                 |
| Password | `Kuboard123`                               |

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

```yaml
apiVersion: v1
kind: Config
current-context: docker-desktop
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURCekNDQWUrZ0F3SUJBZ0lJTFhVVXZJYVgrWE13RFFZSktvWklodmNOQVFFTEJRQXdGVEVUTUJFR0ExVUUKQXhNS2EzVmlaWEp1WlhSbGN6QWdGdzB5TmpBME1EWXdNRE01TkRCYUdBOHlNVEkyTURNeE16QXdORFEwTUZvdwpGVEVUTUJFR0ExVUVBeE1LYTNWaVpYSnVaWFJsY3pDQ0FTSXdEUVlKS29aSWh2Y05BUUVCQlFBRGdnRVBBRENDCkFRb0NnZ0VCQU5ONTRIaDNCdURyTjEvRDduTUhORmVTTUlVdTRkcmd3QUFnV0J4UjcrMW00Ui8zdUVib2YvQ0MKZEYrMVZpQ1ZLbjUwL0xISXVPclZZOTRDVlNYS0UzOUxIWnQ3czZha3h1YlNQTG00N2NoRDVobHVZTFVqSmY2cwpLN3JiZDlaOFNrTDFtSUZTY25nbFhtSnRGaUZsYnVVUys1dHhZVHZ4WHNMQ0xkb3ZiSitzREt3eVBWTjRDOXRYClBEbVVVWWdWTytOZWN1SGFLSVQ5WDFIUlczUDgzYWp0Mm9WUWpPYUJXMTVtT2ROYWc0TjlDL1BOazRsNzZkMXkKWXlEZEZpUkJlem9va3haQ1hDSU1PV0dIUzJRNWZzdEVSdlZ5QjZ0WjlaT3o2ZjFmekQ2OGIzNC9tc2Rva0EzWQp0NWZKZllDdWVxcmwxVk5rK21NSW5vQ2I4cUNZcmlVQ0F3RUFBYU5aTUZjd0RnWURWUjBQQVFIL0JBUURBZ0trCk1BOEdBMVVkRXdFQi93UUZNQU1CQWY4d0hRWURWUjBPQkJZRUZEQXlvbFpFb25PNHhKaHFZQnhBeXNzNEk3TG4KTUJVR0ExVWRFUVFPTUF5Q0NtdDFZbVZ5Ym1WMFpYTXdEUVlKS29aSWh2Y05BUUVMQlFBRGdnRUJBRlo2M0tyMApqYzRNNFFtVEpWcGJJSDd6YWFuV0w5SzRmWFQveGZiR2FGVzljS3g1SlV6bkNNYmo4cUQ4TjVhcVZVOE52bjFKCklZSE42Q2doVUVVYlhtSVRDY2IwUzlRYTVrbDFGaHF5YmtCWE1WQnh4T0MxYjVNR3YzTGphMWVSYU11eXlFSlIKdk5YWkd5ZXA2ZkFuYkdQbEF4VHpKUXJZSitBb0UwZEN4bzBHSEJNc1ZQb2dRSC9rbEs1K1lkWTJkbkZQcTFHTgpaSU9sSE5IVDJnaGdmK2lRRUpUQUV5VWtBa2g3NUNUdVRxTi80Kzllbzl3c24vZENWekZQTE9kMGhpVkxmUjB0CmZWVEFwSDlDUU42elJrci9jd0hZSmVmNE5yd3NlU0pOMlVPdXBtN3lSblBXZEdjcUwrUzY0UGE3bHVjTEh3NkwKREsxSmFrdThSMkxIOURrPQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
    server: https://kubernetes.docker.internal:6443
  name: docker-desktop
contexts:
- context:
    cluster: docker-desktop
    user: docker-desktop
  name: docker-desktop
users:
- name: docker-desktop
  user:
    client-certificate-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURRakNDQWlxZ0F3SUJBZ0lJSTEvWVVSMTEvTHN3RFFZSktvWklodmNOQVFFTEJRQXdGVEVUTUJFR0ExVUUKQXhNS2EzVmlaWEp1WlhSbGN6QWVGdzB5TmpBME1EWXdNRE01TkRCYUZ3MHlOekEwTURZd01ETTVOREJhTURZeApGekFWQmdOVkJBb1REbk41YzNSbGJUcHRZWE4wWlhKek1Sc3dHUVlEVlFRREV4SmtiMk5yWlhJdFptOXlMV1JsCmMydDBiM0F3Z2dFaU1BMEdDU3FHU0liM0RRRUJBUVVBQTRJQkR3QXdnZ0VLQW9JQkFRRFE5UkM3S1paSWhmdGIKQTA2Zno2QkUxTFZJdnZ3VXlHd2R1MlpMTW1pbVczc1lMOHlMcmwwK0FhU1kySTdyZmtsaitYN254QmNkK2NZdApOTnNsRmo4Ky9weCsraVZrRlFubXg1Z3EzUEwwcVlRbEQwdG9Ub3RMU3BCbjdaWHhFMC9YSWJYblVSWFZBcGZzClBqeDg4TTd0L2FHWm1nbXJwbmN3d29BQ3VnM2ZCeFFvNVd6T3Z6ZDdXRWhYSXEwMlZwQXJQWG80ZlJ5aE9nWEgKY1o0cVdvT2dTVGV5anhodUlGQTVNSE05OTFUclZGTTJHRmcxTytIWXhxcHU5LzB5RTRBMjl1ZUlreHFnTE9uVgo2ajQvUk1jL2JrZDZyRmpqdWRDTVRzeFk5VVVHOS92a1kycU5haXdnaEx2VEgvS1RHbDQxempFbGU4UndFbTJYCjNTK0czMzR0QWdNQkFBR2pkVEJ6TUE0R0ExVWREd0VCL3dRRUF3SUZvREFUQmdOVkhTVUVEREFLQmdnckJnRUYKQlFjREFqQU1CZ05WSFJNQkFmOEVBakFBTUI4R0ExVWRJd1FZTUJhQUZEQXlvbFpFb25PNHhKaHFZQnhBeXNzNApJN0xuTUIwR0ExVWRFUVFXTUJTQ0VtUnZZMnRsY2kxbWIzSXRaR1Z6YTNSdmNEQU5CZ2txaGtpRzl3MEJBUXNGCkFBT0NBUUVBR1lmeGh5TjQzSzl0NmJkNGthS2hTTWNUcGEyazZPamN2alA0a0hOam9EV1g5SVJ5VHlFU0dDNGQKd1RlTGp3OWpMMmY5SFY0bE81a1d0N3lTZ2d2YTJlaFY4bnpzODYweUY5Zy9yVEVkYVBYLzJwUXl2K3JNTXJFYQplRlplN0tMdnhQa3dDeDRRUVpBSVlpbG5ydXc0N2V2eElkQ0tyZUh3NEpWVi8wN3ZlMXluVlM1QTNOOHNab3pICldNaSt5dEppWXdVVkR3YTdqMGhQRXRkemJ1NnpxdzhFTFoyRDN3MXZqUlZTWFd6aStRMFhaMXhLamI2UG9kZWUKOGhwUHlWdlk1ZXVGTys3NVhiSHJkTUlxVm1xNllqNzYyb3RvMEs3cUQ3dG9qU1NEcDZ3TURYeVdSR2FwMENVbgo4MUQ3OFFzVXhPTVIrRjE5T3ArNWR3YnZzMHlyY1E9PQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
    client-key-data: LS0tLS1CRUdJTiBSU0EgUFJJVkFURSBLRVktLS0tLQpNSUlFcEFJQkFBS0NBUUVBMFBVUXV5bVdTSVg3V3dOT244K2dSTlMxU0w3OEZNaHNIYnRtU3pKb3BsdDdHQy9NCmk2NWRQZ0drbU5pTzYzNUpZL2wrNThRWEhmbkdMVFRiSlJZL1B2NmNmdm9sWkJVSjVzZVlLdHp5OUttRUpROUwKYUU2TFMwcVFaKzJWOFJOUDF5RzE1MUVWMVFLWDdENDhmUERPN2YyaG1ab0pxNlozTU1LQUFyb04zd2NVS09Wcwp6cjgzZTFoSVZ5S3RObGFRS3oxNk9IMGNvVG9GeDNHZUtscURvRWszc284WWJpQlFPVEJ6UGZkVTYxUlROaGhZCk5UdmgyTWFxYnZmOU1oT0FOdmJuaUpNYW9DenAxZW8rUDBUSFAyNUhlcXhZNDduUWpFN01XUFZGQnZmNzVHTnEKaldvc0lJUzcweC95a3hwZU5jNHhKWHZFY0JKdGw5MHZodDkrTFFJREFRQUJBb0lCQURMcEdvLzNWSEhNRHc5QQpNWVpYWUcyVXc2eDdOOURhZWhrT1lTQitJaWd5RHI0NFd5WkhMOW1kTGR5OW1xaSt4cVBRbEg1a2ROdHRVVDhQCmF6dFFmYzFiM0laRmJSbkMxSHhCY2c0emNoQnhRc3lDdXZVcGVkKzR4WkZFdmkwVDd2Wno5SzFzd0p4NitzZm8KNi9UbStRdmNLV1RTdG50M1hmL255NmVlYkNYRDAxQVJwMnRlOXpKalc3b1E3dlN2cWtkdkJaQTRKMnNzdnBQZgpEb1o4Z0hlSDU5c2tlTlNtVEdPYnZvYmx5bUtzTHg0Ylg2NjdGd2kzWHhPSEVHUTVOdXMwemhSSVl6dWIzSWplCnlmL3lQbVVlKzJKOWdQQWtlVm9pZVZMdWc4WDRLOFpYNFpyODlJZUhuZzdLUHJodktSZDIzdTZyRHpleWJ5cjQKOWV6dk51TUNnWUVBMkp0V1g2WGFLRUdRV3V6ZlRLYWVhMStnUXRSSGppeWNhWTFBT0w1Zm9yTFppbDRyek1zcwptR3BYYWZvNHBtWVQ1SGxjL0xZc3VYUkc0SWk4TTJVQTJkUEZWb0JLUm1MRU95LzJLNmZhTk9PZDRpMFM4Z3lXCjdESWxpbHRtSmgreEw2OGJzSmNqdmRFdUFLdlZBVytKTUpRd3o0Y0JYc0lIcDZLV3VmNDJvYnNDZ1lFQTl2V1cKTFpYUVdDVG4ycnIzODU1Q1IrZTAxUmVIckUyS3EvZ3RQTVZCeEhIeWkzR3NrcDVubTlsRVR0UGhhM1QySkJuWgpZYnMza0VsbHk1Ri95cTJobkMwT1dyeVdWcGU4SVNJU2hVSzRhWkRpUVlMSDhnWldhN2FodkROQk5RbGc4MjU3ClhEQ0xaUXNZQks0cGJ3Z2pKWFh4VWt5ZnZTWkFkKy9tK3BMZ3pUY0NnWUVBalpNQ0ltUVJzZXdnZ1AxL2VlY1IKZGxhN05kTHZyZ0tFZlF6Z28vWHlKakpGczRXWGxUUmF3b2dHK0hLZW9rdm54cFo0YTRoYXRTQkZ6eTR2N0Z1Zwo4YjdUcFpVV2R1akpIM0phc08vMTFFbk5nTzQ3Q3MrbHVWMlJZZHdaYU9PZitPMjM2SFR3M0hrald6YjBjd3JHCm5XVE9mbVhjUkdZSGdNN3BPMG5ueFU4Q2dZRUFuaWdybEdnVWRNNjEyYVBSdGFoTnhHVUVyMCtSYU95RCtaeEgKeEZxRHd2NUNtY0VrQndZQlRwTDNKeENVbGMvaTdyM0xOTWJFVDloaG85dzduaDVTbUlWV1l3L1JyQVVpeTRsWgptUlJncStMSXM3SEF3U1FENXBtZ3ZMbUtjaC9lZ2lmb1F1TW44bjhIVThBQjh3U2dGWmFTQk9Yaml5eGJMelJwCmxXSEpMRU1DZ1lCSHRKZDFKcVpBdlBRbDZxYlVIRGJYV3U0Y1o1Yy9seTJvenBoZi81SDZiWWpDNjM1dTkxc2kKK1d2RWxGVHpocC9GMk5LaGViNG5tcFBxd0dSMkNZVGFFdStVQVorQW1TQmN1WnhtUCswekIzMEtWWEZWMGhRdgpXRjdCUGFzS2d5bU4xTy9FTFpFL0dhZ2ZkODZFcFVkdXBTcjFRRUY4akEwNGRkeExVZHEySVE9PQotLS0tLUVORCBSU0EgUFJJVkFURSBLRVktLS0tLQo=

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


| 项目                   | 值                                                         |
| ---------------------- | ---------------------------------------------------------- |
| Kuboard 访问地址       | http://localhost:8098                                      |
| Kuboard 登录账号       | `admin` / `Kuboard123`                                     |
| 数据库                 | MySQL 8.4（**不能用标准 PostgreSQL**）                     |
| DB 连接（容器内）      | host=`mysql` port=`3306`                                   |
| DB 连接（本地工具）    | host=`127.0.0.1` port=`13306`，user `kuboard`/`Kuboard123` |
| DB 额外参数            | `allowPublicKeyRetrieval=true&useSSL=false`                |
| 集群 kubeconfig server | `https://kubernetes.docker.internal:6443`（保留 CA）       |

### 常用运维命令（在 `kuboard-v4-deploy` 目录下）

```bash
docker compose up -d            # 启动
docker compose ps               # 状态
docker compose logs -f kuboard  # 看 Kuboard 日志
docker compose down             # 停止（保留数据）
docker compose down -v          # 停止并删除数据卷（清空数据库，慎用）
```
