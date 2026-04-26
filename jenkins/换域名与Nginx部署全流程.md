# 换域名与 Nginx 部署全流程

## 1. 背景与目标

当前项目由两套系统组成：

- 诊断系统
  - 前端代码目录：`/root/lixiang/new-diagnostic-system/sales-diagnosis-system/frontend`
  - 后端代码目录：`/root/lixiang/new-diagnostic-system/sales-diagnosis-system/backend`
  - 后端服务端口：`127.0.0.1:5002`
- 推荐系统
  - 代码目录：`/root/caiqiyue_files/recommend-system/recommend_system_project`
  - 服务端口：`127.0.0.1:8000`

目标是把两套系统统一挂到同一个域名：

- 域名：`http://insight.lab.strong365.com`

最终访问规则：

- `/` -> 诊断系统前端页面
- `/api/` -> 诊断系统后端 `127.0.0.1:5002`
- `/admin/`、`/sso/`、`/core/`、`/recommend/`、`/speech/`、`/portrait/`、`/neo4j/` 等 -> 推荐系统 `127.0.0.1:8000`
- `/recommend-static/` -> 推荐系统静态资源
- `/recommend-media/` -> 推荐系统媒体文件


## 2. 为什么必须加 Nginx

外部访问 `http://insight.lab.strong365.com` 时，浏览器默认访问的是 `80` 端口。

而项目本身实际监听的是：

- 诊断后端：`127.0.0.1:5002`
- 推荐系统：`127.0.0.1:8000`

所以必须有一层入口服务监听 `80`，再把请求按路径分流到不同服务。这里采用的是 `Nginx`。


## 3. 最终目录规划

建议使用下面这套最终目录，避免 Nginx 去读 `/root/...` 目录导致权限问题：

- 诊断前端静态目录：`/var/www/insight/diagnosis`
- 推荐系统静态目录：`/var/www/insight/recommend-static`
- 推荐系统媒体目录：`/var/www/insight/recommend-media`

准备目录：

```bash
mkdir -p /var/www/insight/diagnosis
mkdir -p /var/www/insight/recommend-static
mkdir -p /var/www/insight/recommend-media
```


## 4. 本地代码需要调整的配置

### 4.1 诊断系统前端

文件：`new-diagnostic-system/sales-diagnosis-system/frontend/.env`

应包含：

```env
DANGEROUSLY_DISABLE_HOST_CHECK=true
PORT=3000
REACT_APP_RECOMMEND_SYSTEM_URL=http://insight.lab.strong365.com
```

说明：

- 线上虽然不通过 `3000` 对外访问，但构建时仍会读取这个环境变量
- 推荐系统入口统一走域名，不再写死 `:8000`

### 4.2 诊断系统后端

文件：`new-diagnostic-system/sales-diagnosis-system/backend/.env`

关键项：

```env
PORT=5002
RECOMMEND_SYSTEM_BASE_URL=http://127.0.0.1:8000
```

说明：

- 诊断后端在服务器内部调用推荐系统时，直接走本机回环地址

### 4.3 推荐系统

文件：`recommend-system/app.env`

关键项：

```env
DJANGO_ALLOWED_HOSTS=127.0.0.1,localhost,121.196.229.207,insight.lab.strong365.com
DIAGNOSIS_SYSTEM_URL=http://insight.lab.strong365.com
DJANGO_STATIC_URL=/recommend-static/
DJANGO_MEDIA_URL=/recommend-media/
DJANGO_STATIC_ROOT=/var/www/insight/recommend-static
DJANGO_MEDIA_ROOT=/var/www/insight/recommend-media
```

说明：

- `DJANGO_STATIC_ROOT` 和 `DJANGO_MEDIA_ROOT` 一定要提交到仓库
- 否则 Jenkins 下次同步代码时，会把服务器上手工改过的 `app.env` 覆盖掉


## 5. 服务器安装 Nginx

### 5.1 安装

```bash
yum install -y nginx
systemctl enable nginx
```

### 5.2 Nginx 配置

建议将配置文件落到：

```bash
/etc/nginx/conf.d/insight.lab.strong365.com.conf
```

内容如下：

```nginx
upstream diagnosis_backend {
    server 127.0.0.1:5002;
}

upstream recommend_backend {
    server 127.0.0.1:8000;
}

server {
    listen 80;
    server_name insight.lab.strong365.com 121.196.229.207;

    access_log /var/log/nginx/insight.lab.strong365.com.access.log;
    error_log /var/log/nginx/insight.lab.strong365.com.error.log;

    client_max_body_size 50M;

    location ^~ /recommend-static/ {
        alias /var/www/insight/recommend-static/;
        expires 7d;
        add_header Cache-Control "public";
    }

    location ^~ /recommend-media/ {
        alias /var/www/insight/recommend-media/;
        expires 1d;
        add_header Cache-Control "public";
    }

    location ^~ /api/ {
        proxy_pass http://diagnosis_backend;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_connect_timeout 300s;
        proxy_send_timeout 300s;
        proxy_read_timeout 300s;
        proxy_buffering off;
    }

    location ~ ^/(admin/|sso/|core/|course/|excellent/|speech/|resource/|recommend/|portrait/|neo4j/) {
        proxy_pass http://recommend_backend;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_redirect off;
    }

    location / {
        root /var/www/insight/diagnosis;
        index index.html;
        try_files $uri $uri/ /index.html;
    }
}
```

### 5.3 检查并启动

```bash
nginx -t
systemctl start nginx
systemctl status nginx --no-pager
ss -lntp | grep ':80 '
```


## 6. 系统服务建议

### 6.1 诊断系统后端

推荐保持为 systemd 服务，监听 `127.0.0.1:5002`。

关键点：

- `DIAGNOSIS_BACKEND_ENV_FILE` 指向服务器上的 `backend/.env`
- `RECOMMEND_SYSTEM_BASE_URL=http://127.0.0.1:8000`

### 6.2 推荐系统

推荐使用 `gunicorn` + systemd，监听 `127.0.0.1:8000`。

关键点：

- `RECOMMEND_SYSTEM_ENV_FILE=/root/caiqiyue_files/recommend-system/app.env`

### 6.3 开机自启

```bash
systemctl enable sales-diagnosis-backend
systemctl enable recommend_system
systemctl enable nginx
```


## 7. Jenkins 自动发布流程

整体流程：

1. 本地改代码
2. 推送到 Codeup
3. Jenkins 拉取代码
4. 同步代码到服务器
5. 诊断系统执行前端构建
6. 推荐系统执行 `collectstatic`
7. 重启后端服务
8. Nginx 按既定规则继续对外服务


## 8. 诊断系统 Jenkins 关键点

诊断系统前端不再走 `3000` 对外提供服务，而是：

- 在服务器上执行 `npm run build`
- 再把 `build/` 同步到 Nginx 使用的目录 `/var/www/insight/diagnosis/`

如果只执行 `npm run build`，但没有把 `build/` 复制到 `/var/www/insight/diagnosis/`，会出现：

- Jenkins 显示成功
- 但线上首页仍然是旧版本

所以 Jenkins 远程部署脚本必须包含：

```bash
mkdir -p /var/www/insight/diagnosis
rsync -av --delete build/ /var/www/insight/diagnosis/
```


## 9. 推荐系统 Jenkins 关键点

推荐系统每次发布时，必须执行：

```bash
export RECOMMEND_SYSTEM_ENV_FILE=/root/caiqiyue_files/recommend-system/app.env
python manage.py collectstatic --noinput
systemctl restart recommend_system
```

原因：

- `collectstatic` 会把静态资源写到 `DJANGO_STATIC_ROOT`
- Nginx 读取的是 `/var/www/insight/recommend-static/`
- 如果 `app.env` 没有提交 `DJANGO_STATIC_ROOT`，下一次发布会覆盖掉服务器手工修改，导致静态资源路径不一致


## 10. 每次发布后是否还要手动弄 Nginx

正常情况下，不需要。

只要下面这些条件不变，Nginx 配一次就够：

- 域名不变
- 分流规则不变
- 诊断前端发布目录不变
- 推荐系统静态和媒体目录不变

也就是说，日常发版只要：

- Jenkins 正常 `build`
- Jenkins 正常 `collectstatic`
- 服务正常重启

就不需要手工再改 Nginx。

只有在下面场景才需要手动改 Nginx：

- 域名变化
- 从 HTTP 切 HTTPS
- 路径前缀变化
- 静态资源目录变化
- 增加新的反向代理规则


## 11. 验证步骤

### 11.1 在服务器内验证

```bash
curl -I http://127.0.0.1:5002/api/health
curl -I http://127.0.0.1:8000/core/admin/login/
curl -I http://127.0.0.1
curl -I -H 'Host: insight.lab.strong365.com' http://127.0.0.1/api/health
```

预期：

- `5002` 返回 `200`
- `8000` 返回 `302` 是正常的，表示推荐系统活着
- `127.0.0.1:80` 返回 `200`
- 带 Host 头访问 `/api/health` 返回 `200`

### 11.2 在外部验证

```bash
curl --noproxy '*' -I http://insight.lab.strong365.com
curl --noproxy '*' -I http://insight.lab.strong365.com/api/health
curl --noproxy '*' -I http://121.196.229.207
```

### 11.3 浏览器验证

先访问：

- `http://insight.lab.strong365.com/`
- `http://insight.lab.strong365.com/api/health`

注意：

- 当前只配了 `HTTP`
- 如果直接访问 `https://insight.lab.strong365.com`，会因为 `443` 未配置而失败


## 12. 常见问题

### 12.1 外部访问返回 503

先分两层判断：

- 如果服务器内部 `curl http://127.0.0.1` 正常，但外部不通
  - 查阿里云安全组
  - 查 Linux 防火墙
  - 查是否有代理/网络设备拦截
- 如果服务器内部 `curl http://127.0.0.1` 也不通
  - 查 Nginx 是否启动
  - 查是否监听 `80`

### 12.2 浏览器里偶尔 503，但 `curl --noproxy '*'` 正常

这通常不是服务器问题，而是本机代理问题。

典型现象：

```bash
curl -I http://insight.lab.strong365.com
```

返回 `503`，并带：

```text
Proxy-Connection: close
```

但：

```bash
curl --noproxy '*' -I http://insight.lab.strong365.com
```

返回 `200`

说明：

- 服务器正常
- 是本地代理/VPN/浏览器代理插件在拦截

解决：

- 关闭代理
- 或把 `insight.lab.strong365.com` 配成 `DIRECT`

### 12.3 线上页面不更新

重点检查两件事：

1. 诊断系统 Jenkins 是否把 `frontend/build/` 同步到了 `/var/www/insight/diagnosis/`
2. 推荐系统 `app.env` 是否已经提交了：

```env
DJANGO_STATIC_ROOT=/var/www/insight/recommend-static
DJANGO_MEDIA_ROOT=/var/www/insight/recommend-media
```


## 13. 最终结论

只要下面 4 件事同时满足，后续每次本地推代码并触发 Jenkins，通常就不需要再手工干预：

1. Nginx 已安装并保持启用
2. 诊断前端 `build` 会同步到 `/var/www/insight/diagnosis/`
3. 推荐系统 `collectstatic` 会写到 `/var/www/insight/recommend-static/`
4. 服务和 Nginx 已设置开机自启

做到这一步后，日常发版就会变成：

- 推代码
- Jenkins 自动发布
- 线上自动更新

不需要每次再手工修改 Nginx。
