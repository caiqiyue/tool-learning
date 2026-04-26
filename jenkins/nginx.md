```
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

