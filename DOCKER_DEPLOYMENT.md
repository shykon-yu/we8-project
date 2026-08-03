# Soccer 与对战平台 Docker 部署结构

## 1. 目标

- Laravel (`soccer_php`) 是唯一账号来源，保存账号、密码、启用状态和队员资料。
- Go (`platform/backend`) 不保存密码，只保存 Soccer 用户 ID 与平台业务数据。
- MySQL 和 Redis 各运行一个容器，通过数据库、账号和 key 空间隔离应用。
- 本地先使用 `docker-compose.local.yml` 跑通，线上沿用相同边界并更换密钥、域名和持久化策略。

## 2. 本地容器结构

```text
浏览器 / Tauri 客户端
  |
  +-- http://localhost:8852 -> soccer-web (Vue 静态文件)
  +-- http://localhost:8088 -> soccer-nginx -> soccer-app (Laravel PHP-FPM)
  +-- http://localhost:1421 -> platform-web (Vue 静态文件)
  +-- http://localhost:8082 -> platform-api (Go HTTP/WebSocket)

Docker 内部网络 pes8
  |
  +-- mysql
  |    +-- soccer 数据库
  |    +-- platform 数据库
  +-- redis
  +-- soccer-migrate (一次性迁移容器)
  +-- soccer-app
  +-- soccer-nginx
  +-- platform-api
  +-- soccer-web
  +-- platform-web
```

### 容器职责

| 服务 | 职责 | 本地端口 |
| --- | --- | --- |
| `mysql` | 同一 MySQL 实例中的两个独立数据库 | `3308` |
| `redis` | Laravel 缓存/JWT 黑名单与 Go 临时状态 | `6381` |
| `soccer-migrate` | 执行 Laravel migration，并初始化系统菜单、权限和管理员 | 无 |
| `soccer-app` | Laravel PHP-FPM | 仅内部 `9000` |
| `soccer-nginx` | Laravel HTTP 入口 | `8088` |
| `platform-api` | Go 平台 API、房间和 VPN 业务 | `8082` |
| `soccer-web` | 构建并托管 `soccer_v3` | `8852` |
| `platform-web` | 构建并托管 platform 前端 | `1421` |

## 3. 数据库边界

一个 MySQL 容器中创建两个数据库：

| 数据库 | 访问账号 | 所属应用 |
| --- | --- | --- |
| `soccer` | `soccer` | Laravel |
| `platform` | `platform` | Go |

Laravel 的数据库结构不从旧库导入 SQL，而是执行当前代码中的全部 migration。随后只初始化系统菜单、权限、角色和管理员，不复制旧数据，也不生成联盟、战队、比赛或队员数据。

平台的 `platform_users` 是身份映射，不是第二套账号表：

```text
platform_users
  id                    平台内部主键
  soccer_user_id        Laravel users.id，唯一
  username_snapshot     最近一次登录时的用户名快照
  nickname_snapshot     最近一次登录时的昵称快照
  status                平台自身准入状态
  last_login_at         最近登录时间
```

`room_ip_leases.user_id` 关联 `platform_users.id`。禁止建立跨数据库外键，也禁止 Go 直接读取 Laravel 的密码字段。

## 4. 登录流程

```text
1. 用户在 platform 前端输入 Soccer 账号和密码。
2. platform 前端请求 Go 的 POST /api/v1/auth/login。
3. Go 通过 Docker 内网请求：
   http://soccer-nginx/api/v1/auth/login
4. Laravel 校验账号、密码、删除状态和启用状态。
5. Laravel 成功后返回用户资料；Go 不保存 Laravel JWT 和用户密码。
6. Go 按 soccer_user_id 新增或更新 platform_users 快照。
7. Go 签发平台自己的 JWT，后续平台请求只使用该 JWT。
```

关键操作后续可以增加一次 Laravel 用户状态复核，但普通房间接口不需要每次调用 Laravel。

## 5. 本地启动

在 `/Users/shykon/project` 执行：

```bash
docker compose -f docker-compose.local.yml up -d --build
docker compose -f docker-compose.local.yml ps
docker compose -f docker-compose.local.yml logs -f soccer-migrate soccer-app platform-api
```

首次启动会自动：

1. 创建全新的 `pes8-local_pes8_mysql_data` 数据卷。
2. 创建空的 `soccer` 和 `platform` 数据库。
3. 初始化 platform 表和默认房间。
4. 执行 Laravel migrations，生成完整且无业务数据的 Soccer 表结构。
5. 执行 `SystemSeeder`，创建系统菜单、权限、角色和初始管理员。

查看数据库表：

```bash
docker compose -f docker-compose.local.yml exec mysql \
  mysql -uroot -proot-local-password -e "SHOW TABLES FROM soccer; SHOW TABLES FROM platform;"
```

初始管理员账号：

```text
用户名：admin
密码：admin123456
```

该密码只用于本地首次登录。部署到线上后必须立即修改，并在生产配置中改为通过环境变量提供初始凭据。

## 6. 停止与重建空数据库

只停止容器并保留数据库：

```bash
docker compose -f docker-compose.local.yml down
```

删除本地统一环境的数据库和 Redis 数据，重新得到空库：

```bash
docker compose -f docker-compose.local.yml down -v
docker compose -f docker-compose.local.yml up -d --build
```

`down -v` 会永久删除这套 Compose 的数据卷，只能用于确认不要保留数据的本地环境。

## 7. 线上调整

线上不要原样使用本地密码和端口：

- MySQL、Redis、PHP-FPM 不映射公网端口。
- 使用 Caddy 或 Nginx 作为唯一公网入口并开启 HTTPS。
- 密码、`APP_KEY`、Laravel `JWT_SECRET` 和 Go `JWT_SECRET` 使用独立随机值。
- Laravel 与 Go 使用不同数据库账号，账号只能访问自己的数据库。
- 前端 API 地址改成正式 HTTPS 域名。
- Laravel PHP 镜像不挂载源码，Go 使用 Dockerfile 的 `production` target。
- MySQL volume 定时备份到服务器目录或对象存储。
- SoftEther 建议独立运行，仅向 platform API 开放内网管理接口。
- phpMyAdmin、Mailpit、开发热更新和调试端口不进入生产 Compose。

推荐域名：

```text
soccer.example.com          Soccer 前端
soccer.example.com/api      Laravel API
platform.example.com        对战平台前端
platform.example.com/api    Go API
platform.example.com/ws     Go WebSocket
```
