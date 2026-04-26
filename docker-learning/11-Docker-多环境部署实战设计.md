# Docker 多环境部署实战设计文档

> 本文档记录一个真实的 Docker 生产实践案例：从传统服务器部署迁移到 Docker 容器化部署。

---

## 1. 背景与现状

### 1.1 业务背景

| 项目 | 内容 |
|------|------|
| **业务类型** | AI 应用开发平台 |
| **技术栈** | Django + LangChain |
| **用户访问方式** | Web API 服务 |
| **对外入口** | Nginx 反向代理 |
| **数据存储** | MySQL（业务数据）+ Neo4j（知识图谱） |
| **未来规划** | 可能引入 Redis（缓存/消息队列） |

### 1.2 当前架构

```
用户请求
    ↓
┌─────────────────────────────────────────────────────────┐
│                    Linux 服务器                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐      │
│  │   MySQL    │  │   Neo4j     │  │   Django    │      │
│  │  (直接安装)  │  │  (直接安装)  │  │ (conda 虚拟环境) │
│  └─────────────┘  └─────────────┘  └─────────────┘      │
│                                                  ↓      │
│                                          ┌─────────────┐│
│                                          │   Nginx     ││
│                                          │  (直接安装)  ││
│                                          └─────────────┘│
└─────────────────────────────────────────────────────────┘
    ↑
代码同步：Jenkins 监听 Codeup main 分支 → git pull → 重启系统服务
```

### 1.3 当前痛点

| 痛点 | 说明 |
|------|------|
| **环境混杂** | 只有一套开发环境，dev 和 staging（灰度）没有分离，容易相互干扰 |
| **数据库污染** | MySQL、Neo4j 直接安装在服务器上，版本管理困难，升级影响大 |
| **环境不一致** | 开发环境、测试环境、生产环境配置分散，难以保证一致性 |
| **数据不隔离** | 两套环境共享同一套 MySQL 和 Neo4j，数据互相影响 |
| **服务耦合** | Django 应用依赖 conda 虚拟环境，重启流程复杂 |

---

## 2. 需求分析

### 2.1 核心需求

| 需求 | 描述 |
|------|------|
| **多环境隔离** | 同时运行 dev（开发/测试）和 staging（灰度/内部体验）两套环境 |
| **数据隔离** | 两套环境拥有独立的 MySQL 和 Neo4j 实例，数据完全隔离 |
| **数据持久化** | 容器重启或重建后数据不丢失 |
| **平滑迁移** | 现有 MySQL 和 Neo4j 数据能够迁移到 Docker 容器中 |
| **自动化部署** | Jenkins 自动化构建镜像、推送到私有仓库、部署到服务器 |
| **配置分离** | 不同环境使用不同的配置文件（.env） |

### 2.2 环境规划

| 环境 | 用途 | 入口路径 | 代码来源 |
|------|------|----------|----------|
| **dev** | 开发和测试 | `/dev/` | `dev` 分支（手动触发） |
| **staging** | 灰度/内部体验 | `/staging/` | `main` 分支（手动触发） |

### 2.3 非功能性需求

| 项目 | 要求 |
|------|------|
| **数据安全** | 容器删除后数据不丢失，必须配置 volume 持久化 |
| **迁移安全** | 首次迁移不影响现有数据，支持回滚 |
| **维护简单** | 单一代码仓库，通过不同配置文件区分环境 |
| **扩展性** | 未来可以方便地添加更多环境（如 prod） |

---

## 3. 设计方案

### 3.1 架构设计

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          Jenkins 服务器                                  │
│                                                                         │
│   监听 Codeup 分支 → 构建 Docker 镜像 → 推送到 Harbor → SSH 部署          │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ 镜像推送 (docker push)
                                    ↓
┌─────────────────────────────────────────────────────────────────────────┐
│                          部署 Linux 服务器                               │
│                                                                         │
│   ┌────────────────────────────────────────────────────────────────┐   │
│   │                    宿主机 Nginx (80/443)                         │   │
│   │                          │                                      │   │
│   │          ┌───────────────┴───────────────┐                     │   │
│   │          ↓                               ↓                      │   │
│   │     /dev/* 路径                    /staging/* 路径             │   │
│   └────────────────────────────────────────────────────────────────┘   │
│                │                                       │               │
│                ↓                                       ↓               │
│   ┌─────────────────────────┐       ┌─────────────────────────┐       │
│   │   docker network: dev  │       │ docker network: staging │       │
│   │                         │       │                         │       │
│   │   Django Container      │       │   Django Container      │       │
│   │   MySQL Container      │       │   MySQL Container      │       │
│   │   Neo4j Container      │       │   Neo4j Container      │       │
│   │                         │       │                         │       │
│   │   volumes:             │       │   volumes:             │       │
│   │   - mysql_dev_data     │       │   - mysql_staging_data │       │
│   │   - neo4j_dev_data     │       │   - neo4j_staging_data │       │
│   └─────────────────────────┘       └─────────────────────────┘       │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.2 技术选型理由

| 组件 | 选型 | 理由 |
|------|------|------|
| **应用镜像** | `python:3.11-slim` | 轻量级官方镜像，有 LTS 版本 |
| **包管理工具** | `uv` | 用户已在本地使用 uv，团队统一工具链 |
| **数据库 MySQL** | `mysql:8.0` | 与现有服务器版本匹配，数据可直接迁移 |
| **图数据库 Neo4j** | `neo4j:5` | 与现有服务器版本匹配，数据可直接迁移 |
| **容器编排** | `docker-compose` | 适合单服务器多容器管理，配置简单 |
| **配置管理** | `.env` 文件 + `python-dotenv` | 团队已有此习惯，无需额外学习成本 |
| **反向代理** | 宿主机 Nginx | 保持现有架构，一个证书搞定 HTTPS |
| **镜像仓库** | 私有 Harbor | 企业级镜像管理，支持权限控制 |
| **CI/CD** | Jenkins | 团队已有 Jenkins，只需扩展 pipeline |

### 3.3 目录结构设计

```
/opt/app/                          # 项目根目录
├── Dockerfile                     # 统一的应用镜像构建文件
├── docker-compose.dev.yml         # 开发/测试环境编排文件
├── docker-compose.staging.yml     # 灰度环境编排文件
├── .env.dev                       # 开发/测试环境变量
├── .env.staging                   # 灰度环境变量
├── nginx/                         # Nginx 配置目录
│   ├── nginx.conf                 # 公共 Nginx 配置
│   └── conf.d/                    # 子配置文件目录
│       ├── dev.upstream.conf     # dev 环境 upstream
│       └── staging.upstream.conf # staging 环境 upstream
├── scripts/                       # 辅助脚本
│   ├── init_mysql.sh             # MySQL 初始化脚本
│   └── init_neo4j.sh             # Neo4j 初始化脚本
└── project/                       # Django 项目代码
    ├── manage.py
    ├── requirements.txt
    ├── app/                       # Django 主应用
    ├── configs/                   # 项目配置
    └── ...
```

**为什么这样设计：**

- **单一 Dockerfile**：两套环境使用同一个镜像，只通过配置区分，减少维护成本
- **分离的 docker-compose 文件**：两套环境独立管理，互不干扰
- **分离的 .env 文件**：配置与环境对应，便于管理敏感信息
- **Nginx 配置分离**：入口规则清晰，便于维护

**初始化脚本模板**

`init_mysql.sh` 和 `init_neo4j.sh` 用于首次启动时初始化数据（如导入 SQL 文件、设置初始图谱数据）。模板如下：

**scripts/init_mysql.sh**
```bash
#!/bin/bash
set -e
echo "Starting MySQL initialization..."
until mysql -u root -p"${MYSQL_ROOT_PASSWORD}" -e "SELECT 1"; do
    echo "Waiting for MySQL to be ready..."
    sleep 2
done
if [ -f /docker-entrypoint-initdb.d/initial_data.sql ]; then
    echo "Importing initial data..."
    mysql -u root -p"${MYSQL_ROOT_PASSWORD}" ${MYSQL_DATABASE} < /docker-entrypoint-initdb.d/initial_data.sql
fi
echo "MySQL initialization completed"
```

**scripts/init_neo4j.sh**
```bash
#!/bin/bash
set -e
echo "Starting Neo4j initialization..."
until cypher-shell -u ${NEO4J_USER} -p ${NEO4J_PASSWORD} "RETURN 1" &>/dev/null; do
    echo "Waiting for Neo4j to be ready..."
    sleep 5
done
if [ -f /docker-entrypoint-initdb.d/initial_graph.cypher ]; then
    echo "Importing knowledge graph data..."
    cypher-shell -u ${NEO4J_USER} -p ${NEO4J_PASSWORD} < /docker-entrypoint-initdb.d/initial_graph.cypher
fi
echo "Neo4j initialization completed"
```

> **注意**：这些是模板，`initial_data.sql` 和 `initial_graph.cypher` 需要替换为实际的初始化数据文件。如果不需要初始化数据，可删除对应的挂载行。

### 3.4 网络设计

```
┌─────────────────────────────────────────────────────────────────┐
│                        宿主机网络                                 │
│                          (物理网络)                               │
└─────────────────────────────────────────────────────────────────┘
                │                                   │
                │                                   │
   ┌────────────┴────────────┐     ┌────────────────┴────────────┐
   │    docker network: dev  │     │  docker network: staging  │
   │                         │     │                            │
   │   172.18.0.0/16         │     │   172.19.0.0/16            │
   │                         │     │                            │
   │   ┌─────────────────┐  │     │   ┌─────────────────┐      │
   │   │  Django :8000   │  │     │   │  Django :8000   │      │
   │   └─────────────────┘  │     │   └─────────────────┘      │
   │   ┌─────────────────┐  │     │   ┌─────────────────┐      │
   │   │  MySQL :3306    │  │     │   │  MySQL :3306    │      │
   │   └─────────────────┘  │     │   └─────────────────┘      │
   │   ┌─────────────────┐  │     │   ┌─────────────────┐      │
   │   │  Neo4j :7474   │  │     │   │  Neo4j :7474   │      │
   │   │  Neo4j :7687   │  │     │   │  Neo4j :7687   │      │
   │   └─────────────────┘  │     │   └─────────────────┘      │
   └─────────────────────────┘     └────────────────────────────┘
```

**为什么用独立的 docker network：**
- dev 和 staging 的 Django 需要连接各自的数据库
- 网络隔离确保服务只能访问本环境的数据库
- 防止误操作导致跨环境数据污染

---

## 4. 详细配置

### 4.1 Dockerfile

```dockerfile
# Dockerfile
FROM python:3.11-slim

# 设置工作目录
WORKDIR /app

# 安装系统依赖（MySQL client 用于数据库迁移）
RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc \
    default-libmysqlclient-dev \
    pkg-config \
    && rm -rf /var/lib/apt/lists/*

# 安装 uv（轻量级 Python 包管理器）
RUN pip install uv

# 复制依赖文件
COPY requirements.txt .

# 使用 uv 安装依赖（比 pip 更快）
RUN uv pip install --system -r requirements.txt

# 复制项目代码
COPY . .

# 创建非 root 用户（安全加固）
RUN useradd -m -u 1000 appuser && chown -R appuser:appuser /app
USER appuser

# 暴露 Django 端口
EXPOSE 8000

# 启动命令（使用 gunicorn，适合生产环境）
# 本地开发时可在 docker-compose.dev.yml 中用 runserver 覆盖
CMD ["gunicorn", "--bind", "0.0.0.0:8000", "--workers", "4", "--threads", "2", "config.wsgi:application"]
```

> **重要**：`requirements.txt` 中需要包含 `gunicorn`。如果使用 gunicorn 的 Greenlet 支持（提升并发性能），还需要 `psycopg2-binary`（如果用 PostgreSQL）或 `mysqlclient`（如果用 MySQL）。本地开发时可用 `requirements-dev.txt` 单独管理调试用依赖。

| 设计点 | 理由 |
|--------|------|
| `python:3.11-slim` | 相比完整镜像体积小约 5 倍，slim 够用 |
| `apt-get install` | 安装 MySQL client 用于数据库迁移操作（如 `mysql` 命令） |
| `uv pip install` | 用户团队已在使用 uv，构建速度比 pip 快 10 倍以上 |
| `USER appuser` | 以非 root 用户运行容器，提升安全性 |
| `gunicorn` | 生产级 ASGI/WSGI 服务器，支持多 worker |
| `本地开发覆盖` | dev 环境可在 docker-compose 中用 `runserver` 覆盖，方便调试 |

---

**补充说明：开发环境启动命令覆盖**

在 `docker-compose.dev.yml` 的 django 服务中添加 `command` 覆盖：

```yaml
django:
  # ...
  command: python manage.py runserver 0.0.0.0:8000  # 开发模式，覆盖 gunicorn
```

这样：
- **staging/prod**：使用镜像默认的 gunicorn（生产级）
- **dev**：使用 docker-compose 覆盖为 runserver（方便调试）

### 4.2 docker-compose.dev.yml

```yaml
# docker-compose.dev.yml
services:
  django:
    image: harbor.example.com/project/django-app:dev-${IMAGE_TAG}
    container_name: django_dev
    restart: unless-stopped
    ports:
      - "8001:8000"                    # 映射到宿主机 8001（调试用）
    env_file:
      - .env.dev
    volumes:
      - ./project:/app                  # 挂载代码目录（开发时启用）
    depends_on:
      mysql:
        condition: service_healthy
      neo4j:
        condition: service_healthy
    networks:
      - dev
    command: python manage.py runserver 0.0.0.0:8000  # 开发模式，覆盖默认 gunicorn

  mysql:
    image: mysql:8.0
    container_name: mysql_dev
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
    volumes:
      - mysql_dev_data:/var/lib/mysql   # 持久化数据
      - mysql_dev_logs:/var/log/mysql
      - ./scripts/init_mysql.sh:/docker-entrypoint-initdb.d/init.sh
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - dev

  neo4j:
    image: neo4j:5
    container_name: neo4j_dev
    restart: unless-stopped
    environment:
      NEO4J_AUTH: ${NEO4J_USER}/${NEO4J_PASSWORD}
      NEO4J_dbms_memory_heap_initial__size: 2g
      NEO4J_dbms_memory_heap_max__size: 4g
    volumes:
      - neo4j_dev_data:/data            # 持久化数据
      - neo4j_dev_logs:/logs
      - neo4j_dev_conf:/conf
    healthcheck:
      test: ["CMD-SHELL", "cypher-shell -u ${NEO4J_USER} -p ${NEO4J_PASSWORD} 'RETURN 1' || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - dev

networks:
  dev:
    driver: bridge

volumes:
  mysql_dev_data:
    name: mysql_dev_data
  mysql_dev_logs:
    name: mysql_dev_logs
  neo4j_dev_data:
    name: neo4j_dev_data
  neo4j_dev_logs:
    name: neo4j_dev_logs
  neo4j_dev_conf:
    name: neo4j_dev_conf
```

**关键设计点解释：**

| 设计点 | 理由 |
|--------|------|
| `image: ${IMAGE_TAG}` | Jenkins 构建时注入标签，实现版本管理 |
| `restart: unless-stopped` | 服务器重启后自动启动，意外崩溃也重启 |
| `depends_on + condition` | 确保数据库就绪后再启动 Django，避免连接失败 |
| `healthcheck` | 健康检查，Jenkins 可通过 `docker ps` 确认服务状态 |
| `volumes 命名` | 显式命名 volume，删除容器后数据不丢失 |
| `./scripts/init_mysql.sh` | 挂载到 entrypoint 目录，首次启动自动执行 |

### 4.3 docker-compose.staging.yml

```yaml
# docker-compose.staging.yml
# 与 dev 版本类似，主要区别：
# 1. 镜像标签使用 staging
# 2. 端口映射到 8002
# 3. volume 名称使用 staging 前缀
# 4. network 名称为 staging

services:
  django:
    image: harbor.example.com/project/django-app:staging-${IMAGE_TAG}
    container_name: django_staging
    restart: unless-stopped
    ports:
      - "8002:8000"
    env_file:
      - .env.staging
    depends_on:
      mysql:
        condition: service_healthy
      neo4j:
        condition: service_healthy
    networks:
      - staging

  mysql:
    image: mysql:8.0
    container_name: mysql_staging
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
    volumes:
      - mysql_staging_data:/var/lib/mysql
      - mysql_staging_logs:/var/log/mysql
      - ./scripts/init_mysql.sh:/docker-entrypoint-initdb.d/init.sh
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - staging

  neo4j:
    image: neo4j:5
    container_name: neo4j_staging
    restart: unless-stopped
    environment:
      NEO4J_AUTH: ${NEO4J_USER}/${NEO4J_PASSWORD}
      NEO4J_dbms_memory_heap_initial__size: 2g
      NEO4J_dbms_memory_heap_max__size: 4g
    volumes:
      - neo4j_staging_data:/data
      - neo4j_staging_logs:/logs
      - neo4j_staging_conf:/conf
    healthcheck:
      test: ["CMD-SHELL", "cypher-shell -u ${NEO4J_USER} -p ${NEO4J_PASSWORD} 'RETURN 1' || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - staging

networks:
  staging:
    driver: bridge

volumes:
  mysql_staging_data:
    name: mysql_staging_data
  mysql_staging_logs:
    name: mysql_staging_logs
  neo4j_staging_data:
    name: neo4j_staging_data
  neo4j_staging_logs:
    name: neo4j_staging_logs
  neo4j_staging_conf:
    name: neo4j_staging_conf
```

### 4.4 环境变量文件

#### .env.dev

```bash
# Django 配置
DJANGO_SECRET_KEY=your-secret-key-here
DJANGO_DEBUG=True
DJANGO_ALLOWED_HOSTS=localhost,127.0.0.1

# 数据库配置（dev 环境）
MYSQL_ROOT_PASSWORD=dev_root_password
MYSQL_DATABASE=app_dev
MYSQL_USER=app_dev_user
MYSQL_PASSWORD=dev_user_password

# Neo4j 配置（dev 环境）
NEO4J_USER=neo4j
NEO4J_PASSWORD=dev_neo4j_password

# 连接地址（容器间通过服务名访问）
MYSQL_HOST=mysql
MYSQL_PORT=3306
NEO4J_HOST=neo4j
NEO4J_PORT=7474
NEO4J_BOLT_PORT=7687
```

#### .env.staging

```bash
# Django 配置
DJANGO_SECRET_KEY=your-secret-key-here-staging
DJANGO_DEBUG=False
DJANGO_ALLOWED_HOSTS=staging.example.com

# 数据库配置（staging 环境）
MYSQL_ROOT_PASSWORD=staging_root_password
MYSQL_DATABASE=app_staging
MYSQL_USER=app_staging_user
MYSQL_PASSWORD=staging_user_password

# Neo4j 配置（staging 环境）
NEO4J_USER=neo4j
NEO4J_PASSWORD=staging_neo4j_password

# 连接地址
MYSQL_HOST=mysql
MYSQL_PORT=3306
NEO4J_HOST=neo4j
NEO4J_PORT=7474
NEO4J_BOLT_PORT=7687
```

**为什么要这样设计：**

| 设计点 | 理由 |
|--------|------|
| `MYSQL_HOST=mysql` | 容器间通过 docker network DNS 解析服务名 |
| `DJANGO_DEBUG=True/False` | dev 开启调试模式，staging 关闭 |
| `DJANGO_ALLOWED_HOSTS` | 根据环境设置允许的域名 |
| `staging 密码不同` | 两套环境密码隔离，泄漏一套不影响另一套 |

### 4.5 宿主机 Nginx 配置

#### nginx.conf（主配置）

```nginx
# /opt/app/nginx/nginx.conf
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections 1024;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';

    access_log /var/log/nginx/access.log main;

    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;

    # 导入子配置
    include /opt/app/nginx/conf.d/*.conf;
}
```

#### conf.d/dev.upstream.conf

```nginx
# /opt/app/nginx/conf.d/dev.upstream.conf

# dev 环境 upstream
upstream django_dev {
    server 127.0.0.1:8001;  # docker-compose.dev.yml 映射的端口
}

# dev 环境 server 配置
server {
    listen 80;
    server_name localhost;

    # 如果有 HTTPS
    # listen 443 ssl http2;
    # ssl_certificate /path/to/cert.pem;
    # ssl_certificate_key /path/to/key.pem;

    client_max_body_size 100M;

    location /dev/ {
        proxy_pass http://django_dev/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # WebSocket 支持（如果需要）
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }

    location /dev/static/ {
        proxy_pass http://django_dev/static/;
        expires 30d;
        add_header Cache-Control "public, immutable";
    }
}
```

#### conf.d/staging.upstream.conf

```nginx
# /opt/app/nginx/conf.d/staging.upstream.conf

# staging 环境 upstream
upstream django_staging {
    server 127.0.0.1:8002;  # docker-compose.staging.yml 映射的端口
}

# staging 环境 server 配置
server {
    listen 80;
    server_name localhost;

    client_max_body_size 100M;

    location /staging/ {
        proxy_pass http://django_staging/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }

    location /staging/static/ {
        proxy_pass http://django_staging/static/;
        expires 30d;
        add_header Cache-Control "public, immutable";
    }
}
```

**为什么这样设计：**

| 设计点 | 理由 |
|--------|------|
| `/dev/` 和 `/staging/` 路径 | 通过 URL 路径区分环境，共用 80/443 端口 |
| `proxy_set_header` | 传递真实客户端 IP 和协议给 Django |
| `WebSocket 支持` | LangChain 可能需要 WebSocket 进行实时交互 |
| `静态文件缓存` | Django 静态资源通过 Nginx 直接服务，减轻容器压力 |

---

## 5. Jenkins Pipeline 设计

### 5.1 Jenkinsfile

```groovy
// Jenkinsfile
pipeline {
    agent any

    environment {
        HARBOR_URL = 'harbor.example.com'
        PROJECT = 'project'
        APP_NAME = 'django-app'
        DEPLOY_SERVER = 'deploy@example.com'
        DEPLOY_USER = 'deploy'
    }

    parameters {
        choice(
            name: 'ENVIRONMENT',
            choices: ['dev', 'staging'],
            description: '选择要部署的环境'
        )
    }

    stages {
        stage('Checkout') {
            steps {
                echo "Checkout code from ${env.GIT_BRANCH}"
                checkout scm
            }
        }

        stage('Build Image') {
            steps {
                echo "Building Docker image for ${params.ENVIRONMENT}"
                sh '''
                    IMAGE_TAG=$(date +%Y%m%d-%H%M%S)-${GIT_COMMIT:0:8}
                    echo "IMAGE_TAG=${IMAGE_TAG}" > build.env
                    docker build \
                        --build-arg ENV=${params.ENVIRONMENT} \
                        -t ${HARBOR_URL}/${PROJECT}/${APP_NAME}:${params.ENVIRONMENT}-${IMAGE_TAG} \
                        .
                '''
            }
        }

        stage('Push to Harbor') {
            steps {
                echo "Pushing image to Harbor"
                sh '''
                    source build.env
                    docker push ${HARBOR_URL}/${PROJECT}/${APP_NAME}:${params.ENVIRONMENT}-${IMAGE_TAG}
                    # 可选：打 latest 标签
                    docker tag ${HARBOR_URL}/${PROJECT}/${APP_NAME}:${params.ENVIRONMENT}-${IMAGE_TAG} \
                        ${HARBOR_URL}/${PROJECT}/${APP_NAME}:${params.ENVIRONMENT}-latest
                    docker push ${HARBOR_URL}/${PROJECT}/${APP_NAME}:${params.ENVIRONMENT}-latest
                '''
            }
        }

        stage('Deploy to Server') {
            steps {
                echo "Deploying to ${params.ENVIRONMENT} environment"
                sh """
                    # 读取本地 build.env 中的 IMAGE_TAG
                    source build.env
                    export REMOTE_IMAGE_TAG=\${IMAGE_TAG}

                    ssh \${DEPLOY_USER}@\${DEPLOY_SERVER} << 'EOF'
                        set -e

                        cd /opt/app

                        # 拉取指定版本的镜像
                        docker-compose -f docker-compose.\${params.ENVIRONMENT}.yml pull

                        # 重启容器（使用传入的 IMAGE_TAG）
                        IMAGE_TAG=\${REMOTE_IMAGE_TAG} docker-compose -f docker-compose.\${params.ENVIRONMENT}.yml up -d

                        # 等待服务就绪
                        sleep 10

                        # 健康检查
                        docker-compose -f docker-compose.\${params.ENVIRONMENT}.yml ps

                        echo "Deployment completed successfully"
                    EOF
                """
            }
        }

        stage('Cleanup') {
            steps {
                echo "Cleaning up old images"
                sh '''
                    # 删除本地构建的镜像（节省空间）
                    docker system prune -f
                '''
            }
        }
    }

    post {
        success {
            echo "Pipeline completed successfully!"
            emailext(
                subject: "部署成功: ${params.ENVIRONMENT}",
                body: "镜像标签: ${IMAGE_TAG}",
                to: 'team@example.com'
            )
        }
        failure {
            echo "Pipeline failed!"
            emailext(
                subject: "部署失败: ${params.ENVIRONMENT}",
                body: "请查看 Jenkins 日志",
                to: 'team@example.com'
            )
        }
    }
}
```

**Pipeline 设计要点：**

| 设计点 | 理由 |
|--------|------|
| `parameters ENVIRONMENT` | 让开发者手动选择部署到哪个环境，防止误操作 |
| `IMAGE_TAG 时间戳+hash` | 唯一标签，支持回滚到任意版本 |
| `source build.env` | 在不同 stage 间传递构建参数 |
| `docker-compose pull` | 先拉取新镜像，再重启，确保零 downtime |
| `docker-compose up -d` | 后台运行，不阻塞 Jenkins |
| `post success/failure` | 自动发送邮件通知 |
| `docker system prune` | 清理构建缓存，节省磁盘空间 |

### 5.2 Jenkins 触发配置

| 环境 | Git 分支 | 触发方式 |
|------|----------|----------|
| **dev** | `dev` | 开发者手动在 Jenkins 页面选择 `ENVIRONMENT=dev` 触发 |
| **staging** | `main` | 开发者手动在 Jenkins 页面选择 `ENVIRONMENT=staging` 触发 |

**为什么不自动触发：**
- 防止代码提交就自动部署，可能影响正在使用的环境
- 生产级操作需要人工确认
- 未来可根据需要改为 web hook 自动触发

---

## 6. 数据迁移方案

### 6.1 迁移原则

| 原则 | 说明 |
|------|------|
| **先备份** | 迁移前必须完整备份现有数据 |
| **先验证** | 新环境验证通过后再废弃旧环境 |
| **可回滚** | 迁移失败可以回滚到原始状态 |

### 6.2 MySQL 数据迁移

#### 步骤 1：备份现有数据

```bash
# 在服务器上执行
mysqldump -u root -p --all-databases --single-transaction \
    --routines --triggers --events \
    > /backup/mysql_all_backup_$(date +%Y%m%d).sql

# 记录当前 MySQL 配置
mysql -u root -p -e "SHOW VARIABLES;" > /backup/mysql_variables.txt
```

#### 步骤 2：确认数据目录位置

```bash
# 查看当前 MySQL 数据目录
mysql -u root -p -e "SHOW VARIABLES LIKE 'datadir';"
# 通常是 /var/lib/mysql 或 /data/mysql
```

#### 步骤 3：创建 docker-compose 配置（首次迁移用）

```yaml
# docker-compose.migration.yml（临时使用）
services:
  mysql:
    image: mysql:8.0
    container_name: mysql_migration
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
    volumes:
      - /var/lib/mysql:/var/lib/mysql   # 直接挂载现有数据目录
    ports:
      - "3307:3306"                       # 临时端口，避免冲突
    networks:
      - migration

networks:
  migration:
    driver: bridge
```

#### 步骤 4：启动容器并验证

```bash
# 启动临时容器
docker-compose -f docker-compose.migration.yml up -d

# 验证数据完整性
docker exec mysql_migration mysql -u root -p -e "SHOW DATABASES;"
docker exec mysql_migration mysql -u root -p ${MYSQL_DATABASE} -e "SHOW TABLES;"
```

#### 步骤 5：停止服务器原 MySQL 服务

```bash
# 停止 MySQL 服务（确保没有新写入）
sudo systemctl stop mysql
# 或
sudo systemctl stop mysqld
```

#### 步骤 6：使用正式 docker-compose 启动

```bash
# 使用正式的 dev/staging 配置启动
docker-compose -f docker-compose.dev.yml up -d

# 验证数据
docker exec mysql_dev mysql -u root -p -e "SHOW DATABASES;"
```

### 6.3 Neo4j 数据迁移

#### 步骤 1：备份现有数据

```bash
# 停止 Neo4j 服务
sudo systemctl stop neo4j

# 备份数据目录
sudo tar -czvf /backup/neo4j_backup_$(date +%Y%m%d).tar.gz /var/lib/neo4j
# 或 /opt/neo4j/data（取决于安装位置）

# 备份配置文件
sudo tar -czvf /backup/neo4j_conf_backup_$(date +%Y%m%d).tar.gz /etc/neo4j
```

#### 步骤 2：确认数据目录位置

```bash
# 查看 neo4j.conf 中的配置
cat /etc/neo4j/neo4j.conf | grep dbms.directories
# 通常是 /var/lib/neo4j/data 或 /data/neo4j
```

#### 步骤 3：启动 Docker 容器

```yaml
# docker-compose.neo4j.migration.yml
services:
  neo4j:
    image: neo4j:5
    container_name: neo4j_migration
    environment:
      NEO4J_AUTH: ${NEO4J_USER}/${NEO4J_PASSWORD}
    volumes:
      - /var/lib/neo4j:/data    # 直接挂载现有数据目录
      - /var/lib/neo4j/logs:/logs
      - /etc/neo4j:/conf        # 挂载配置（可选）
    ports:
      - "7475:7474"
      - "7688:7687"
    networks:
      - migration

networks:
  migration:
    driver: bridge
```

```bash
docker-compose -f docker-compose.neo4j.migration.yml up -d

# 等待启动
sleep 20

# 验证
curl -s http://localhost:7475/
```

#### 步骤 4：使用正式配置启动

```bash
# 停止迁移容器
docker-compose -f docker-compose.neo4j.migration.yml down

# 启动正式 dev 环境
docker-compose -f docker-compose.dev.yml up -d
```

### 6.4 回滚方案

如果迁移失败，启用回滚：

```bash
# 停止 Docker 容器
docker-compose -f docker-compose.dev.yml down

# 重新启动服务器原服务
sudo systemctl start mysql
sudo systemctl start neo4j

# 数据完整保留在原目录
```

---

## 7. 部署流程

### 7.1 完整部署流程

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          开发者本地                                       │
│                                                                         │
│   1. 修改代码                                                            │
│   2. git add . → git commit -m "feat: ..."                             │
│   3. git push origin dev                                                │
└─────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────┐
│                          Jenkins 服务器                                  │
│                                                                         │
│   4. 收到 Codeup 通知（dev 分支有更新）                                   │
│   5. 拉取 dev 分支代码                                                   │
│   6. docker build 构建镜像                                               │
│   7. docker push 推送到 Harbor                                          │
│   8. SSH 到部署服务器                                                   │
│   9. docker-compose pull 拉取新镜像                                      │
│  10. docker-compose up -d 重启容器                                       │
│  11. 健康检查确认服务正常                                                │
└─────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────┐
│                          部署 Linux 服务器                               │
│                                                                         │
│  宿主机 Nginx → /dev/ → docker network dev → Django Container          │
│                              → MySQL Container                         │
│                              → Neo4j Container                         │
│                                                                         │
│  volumes:                                                              │
│    mysql_dev_data (持久化)                                              │
│    neo4j_dev_data (持久化)                                              │
└─────────────────────────────────────────────────────────────────────────┘
```

### 7.2 日常运维命令

```bash
# 查看所有容器状态
docker-compose -f docker-compose.dev.yml ps
docker-compose -f docker-compose.staging.yml ps

# 查看日志
docker-compose -f docker-compose.dev.yml logs -f django
docker-compose -f docker-compose.staging.yml logs -f django

# 重启服务
docker-compose -f docker-compose.dev.yml restart django

# 进入容器调试
docker exec -it django_dev /bin/bash

# 查看资源使用
docker stats

# 拉取最新镜像并重启
docker-compose -f docker-compose.dev.yml pull && docker-compose -f docker-compose.dev.yml up -d

# 完全重建（慎用，会保留 volume）
docker-compose -f docker-compose.dev.yml down
docker-compose -f docker-compose.dev.yml up -d

# 完全重建 + 删除数据（数据会丢失，慎用）
docker-compose -f docker-compose.dev.yml down -v
docker-compose -f docker-compose.dev.yml up -d
```

### 7.3 镜像版本回滚

```bash
# 1. SSH 到部署服务器
ssh deploy@example.com

# 2. 查看可用的镜像标签
docker images harbor.example.com/project/django-app

# 3. 指定旧版本镜像重启
IMAGE_TAG=20260425-143022-a1b2c3d
docker-compose -f docker-compose.dev.yml pull
docker-compose -f docker-compose.dev.yml up -d
```

---

## 8. 安全注意事项

### 8.1 敏感信息管理

| 项目 | 建议 |
|------|------|
| `.env 文件` | 不提交到 Git，添加到 `.gitignore` |
| `Harbor 密码` | 在 Jenkins 中配置 credentials，不要明文写在 pipeline |
| `服务器 SSH` | 使用 SSH key 认证，不要用密码 |
| `数据库密码` | 使用强密码，不要使用默认密码 |

### 8.2 生产环境加固

| 项目 | 建议 |
|------|------|
| `DJANGO_DEBUG` | 生产环境必须设为 `False` |
| `DJANGO_ALLOWED_HOSTS` | 只允许正式的域名访问 |
| `容器特权` | 不要使用 `--privileged` 选项 |
| `容器 root` | 不需要以 root 运行，可使用 `USER` 指定普通用户 |
| `网络隔离` | 生产环境考虑使用 `network_mode: none` 限制出站流量 |

---

## 9. 未来扩展

### 9.1 添加 Redis

```yaml
# 在 docker-compose.dev.yml 中添加
services:
  redis:
    image: redis:7-alpine
    container_name: redis_dev
    restart: unless-stopped
    ports:
      - "6379:6379"
    volumes:
      - redis_dev_data:/data
    networks:
      - dev
    command: redis-server --appendonly yes

volumes:
  redis_dev_data:
    name: redis_dev_data
```

### 9.2 添加生产环境

```yaml
# docker-compose.prod.yml
services:
  django:
    image: harbor.example.com/project/django-app:prod-${IMAGE_TAG}
    restart: always
    env_file:
      - .env.prod
    # 生产环境去掉端口映射，通过 Nginx 访问
    expose:
      - "8000"
    # 添加更多副本
    deploy:
      replicas: 2
```

### 9.3 使用 Kubernetes 扩展

当单服务器不够用时，可以：
1. 将 `docker-compose` 迁移到 `docker-compose` 格式的 K8s 资源
2. 使用 `Kompose` 工具转换
3. 或手动编写 Deployment/Service 资源清单

---

## 10. 常见问题

### Q1: 容器内无法连接 MySQL？

```bash
# 检查网络是否连通
docker exec django_dev ping mysql

# 检查 MySQL 是否就绪
docker exec django_dev nc -zv mysql 3306

# 检查环境变量是否正确
docker exec django_dev env | grep MYSQL
```

### Q2: Neo4j 启动失败？

```bash
# 检查数据目录权限
ls -la /var/lib/neo4j

# 如果权限不足，修改权限
sudo chown -R 7474:7474 /var/lib/neo4j

# 查看详细日志
docker logs neo4j_dev
```

### Q3: Django 静态文件 404？

```bash
# 在容器内收集静态文件
docker exec django_dev python manage.py collectstatic --noinput

# 或者在 docker-compose 中添加
command: python manage.py runserver 0.0.0.0:8000 && python manage.py collectstatic --noinput
```

### Q4: 如何查看容器资源使用？

```bash
docker stats
docker stats --no-stream  # 非实时
```

### Q5: 镜像构建失败？

```bash
# 本地先构建测试
docker build -t test-build .

# 查看构建日志
docker build --progress=plain .
```

---

## 附录

### A. 快速检查清单

```
部署前检查：
□ 服务器已安装 Docker 和 docker-compose
□ Harbor 仓库已配置并可访问
□ Jenkins 服务器可 SSH 到部署服务器
□ .env 文件已正确配置
□ 现有数据已备份
□ Nginx 配置已更新

部署后检查：
□ docker-compose ps 确认所有容器运行中
□ 健康检查通过（docker-compose logs 无报错）
□ curl 测试入口可访问
□ 数据库连接正常
□ Neo4j 可访问
```

### B. 关键文件路径汇总

| 文件 | 路径 |
|------|------|
| Docker 项目根目录 | `/opt/app/` |
| Django 项目代码 | `/opt/app/project/` |
| Nginx 主配置 | `/opt/app/nginx/nginx.conf` |
| Nginx 子配置 | `/opt/app/nginx/conf.d/` |
| Docker 编排（dev） | `/opt/app/docker-compose.dev.yml` |
| Docker 编排（staging） | `/opt/app/docker-compose.staging.yml` |
| 环境变量（dev） | `/opt/app/.env.dev` |
| 环境变量（staging） | `/opt/app/.env.staging` |
| MySQL 数据目录 | `/var/lib/mysql` |
| Neo4j 数据目录 | `/var/lib/neo4j` |
