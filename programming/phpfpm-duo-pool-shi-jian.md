---
description: '2024-09-20'
---

# Php-fpm 多pool实践

### 背景

项目中用到大量外部请求，容易导致fpm慢查询fpm对于每个请求，都会创建一个独立的进程，当进程数达max\_children，nginx等待期间若无可用fpm进程处理请求，就会返回502错误。由于外部请求常常不可控，本篇不讨论熔断机制，只谈分离。引发了慢查询的外部服务，如若不加限制，有可能导致所有可用fpm进程被用完，基于此。对于调用了外部服务的接口，期望由进程池1消费，其余接口由于进程池2消费。这样即便外部服务慢查询占用完了所有进程，也不会阻塞常规请求

### 实现

1. Php-fpm 创建两个pool, 并设定不同的端口(9000,9001)

```nginx
[global]
; Ensure worker stdout and stderr are sent to the main error log.
error_log = /var/log/php-fpm/laravel.log
;log_level = alert
; https://github.com/docker-library/php/pull/725#issuecomment-443540114
log_limit = 8192

[www]
; if we send this to /proc/self/fd/1, it never appears
;access.log = /proc/self/fd/2
;slowlog = /var/log/php-fpm/slow-log
clear_env = no

; Ensure worker stdout and stderr are sent to the main error log.
catch_workers_output = yes
decorate_workers_output = yes

pm = dynamic
pm.max_children = 100
pm.start_servers = 4
pm.min_spare_servers = 4
pm.max_spare_servers = 6
listen = 9000

[www2]
; 会阻塞的请求
; if we send this to /proc/self/fd/1, it never appears
;access.log = /proc/self/fd/2
;slowlog = /var/log/php-fpm/slow-log
clear_env = no
user = www
; Ensure worker stdout and stderr are sent to the main error log.
catch_workers_output = yes
decorate_workers_output = yes

pm = dynamic
pm.max_children = 100
pm.start_servers = 4
pm.min_spare_servers = 4
pm.max_spare_servers = 6
listen = 9001 ;开放不同的端口
```

2. nginx转发对应请求到不同的pool, 通过端口号区分

```nginx
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    root /var/www/backend/public;
    index index.php;
    server_name _;

    # 会阻塞的外部请求
    location /api/healthcheck/sleep {
        try_files $uri $uri/ /index.php2$is_args$args;
        client_max_body_size 100M;
    }

    # 常规请求
    location / {
        set $userId '';
        access_by_lua '
        local jwt = require("nginx-jwt")
        jwt.getJWTUserId()
        ';
        try_files $uri $uri/ /index.php$is_args$args;
        client_max_body_size 100M;
    }

    location ~ \.php2$ {
        internal;
        try_files $uri /index.php =404;
        fastcgi_pass backend:9001; #进程池2
        fastcgi_index index.php;
        fastcgi_buffers 4 128k;
        fastcgi_buffer_size 128k;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
        client_max_body_size 100M;
    }

    location ~ \.php$ {
        internal;
        try_files $uri /index.php =404;
        fastcgi_pass backend:9000; #进程池1
        fastcgi_index index.php;
        fastcgi_buffers 4 128k;
        fastcgi_buffer_size 128k;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
        client_max_body_size 100M;
    }

    location /api/theme {
        add_header Cache-Control no-store;
        add_header Pragma no-cache;
        try_files /../storage/app/theme/$http_x_host.json /../storage/app/theme/$http_host.json /../storage/app/theme/default.json;
    }

}
```

### 验证

直接把会堵塞的接口干满，上面定义了最大进程数100，这里直接100并发

```bash
 $ab -n 1000 -c 100 "http://localhost:8888/api/healthcheck/sleep?seconds=3"
```

在被干满的情况下，继续访问这个接口只会处于等待，若直接访问常规接口，由于是独立的进程池，应该快速响应。

```bash
 $ab -n 5 -c 1 "http://localhost:8888/"
```

![](<../.gitbook/assets/image (16).png>)\
