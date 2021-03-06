# Cài đặt Nextcloud 20 (Php72 Nginx MariaDB) trên Centos 7

# Chuẩn bị
Cập nhật
```
yum update -y
```
Cài EPEL repo
```
yum install epel-release -y
```
Trong bài lab này thay vì cấu hình SELinux và Firewalld thì mình sẽ disable.
```
systemctl disable --now firewalld
setenforce 0
sed -i s/SELINUX=enforcing/SELINUX=disabled/g /etc/selinux/config
reboot
```

# Cài đặt MariaDB
```
yum install mariadb-server -y
systemctl enable --now mariadb.service
mysql_secure_installation
```

# Cài đặt Nginx
```
yum install nginx -y
systemctl enable --now nginx.service
```

# Cài đặt PHP-fpm
```
yum install http://rpms.remirepo.net/enterprise/remi-release-7.rpm
yum install yum-utils
yum-config-manager --enable remi-php72
yum update
yum install php php-mysqlnd php-fpm php-opcache php-gd php-xml php-mbstring php-common php-json php-curl php-zip php-bz2 php-intl php-proccess php-bcmath php-gmp php-imagick
```
Cấu hình php-fpm, mặc định là apache.
```
vi /etc/php-fpm.d/www.conf
...
user = nginx
group = nginx
```
```
systemctl enable --now php-fpm.service
```

Chỉnh sửa file /etc/my.cnf 
```
[server]
skip_name_resolve = 1
innodb_buffer_pool_size = 128M
innodb_buffer_pool_instances = 1
innodb_flush_log_at_trx_commit = 2
innodb_log_buffer_size = 32M
innodb_max_dirty_pages_pct = 90
query_cache_type = 1
query_cache_limit = 2M
query_cache_min_res_unit = 2k
query_cache_size = 64M
tmp_table_size= 64M
max_heap_table_size= 64M
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 1

[client]
default-character-set = utf8mb4

[mysqld]
character_set_server = utf8mb4
collation_server = utf8mb4_general_ci
transaction_isolation = READ-COMMITTED
binlog_format = ROW
innodb_large_prefix=on
innodb_file_format=barracuda
innodb_file_per_table=1
innodb_buffer_pool_size=1G
innodb_io_capacity=4000
innodb_large_prefix=true

datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock
# Disabling symbolic-links is recommended to prevent assorted security risks
symbolic-links=0
# Settings user and group are ignored when systemd is used.
# If you need to run mysqld under a different user or group,
# customize your systemd unit file for mariadb according to the
# instructions in http://fedoraproject.org/wiki/Systemd

[mysqld_safe]
log-error=/var/log/mariadb/mariadb.log
pid-file=/var/run/mariadb/mariadb.pid

#
# include all files from the config directory
#
!includedir /etc/my.cnf.d
```

Tải source code nextcloud
```
yum install wget unzip -y
cd /tmp/
wget https://download.nextcloud.com/server/releases/nextcloud-20.0.0.zip
```
```
unzip nextcloud-20.0.0.zip -d /usr/share/nginx/
```
```
chown -R nginx:nginx /usr/share/nginx/nextcloud
chgrp -R nginx /var/lib/php/{opcache,session,wsdlcache}
```

Tạo Database cho Nextcloud, user là nextcloud, password là nextcloud
```
mysql -u root
CREATE DATABASE nextcloud DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
CREATE USER nextcloud@localhost IDENTIFIED BY 'nextcloud';
GRANT ALL PRIVILEGES ON nextcloud.* TO nextcloud@localhost;
FLUSH PRIVILEGES;
EXIT;
```

Tạo file nextcloud.conf. Thay domain name và đường dẫn ssl sao cho phù hợp. 
```
vi /etc/nginx/conf.d/nextcloud.conf
upstream php-handler {
    server 127.0.0.1:9000;
    #server unix:/var/run/php/php7.4-fpm.sock;
}

server {
    listen 80;
    listen [::]:80;
    server_name nextcloud.com;

    # Enforce HTTPS
    return 301 https://$server_name$request_uri;
}

server {
    listen 443      ssl http2;
    listen [::]:443 ssl http2;
    server_name nextcloud.com;

    # Use Mozilla's guidelines for SSL/TLS settings
    # https://mozilla.github.io/server-side-tls/ssl-config-generator/
    ssl_certificate     /etc/nginx/cert/nextcloud.crt;
    ssl_certificate_key /etc/nginx/cert/nextcloud.key;

    # HSTS settings
    # WARNING: Only add the preload option once you read about
    # the consequences in https://hstspreload.org/. This option
    # will add the domain to a hardcoded list that is shipped
    # in all major browsers and getting removed from this list
    # could take several months.
    add_header Strict-Transport-Security "max-age=15768000; includeSubDomains; preload;" always;

    # set max upload size
    client_max_body_size 512M;
    fastcgi_buffers 64 4K;

    # Enable gzip but do not remove ETag headers
    gzip on;
    gzip_vary on;
    gzip_comp_level 4;
    gzip_min_length 256;
    gzip_proxied expired no-cache no-store private no_last_modified no_etag auth;
    gzip_types application/atom+xml application/javascript application/json application/ld+json application/manifest+json application/rss+xml application/vnd.geo+json application/vnd.ms-fontobject application/x-font-ttf application/x-web-app-manifest+json application/xhtml+xml application/xml font/opentype image/bmp image/svg+xml image/x-icon text/cache-manifest text/css text/plain text/vcard text/vnd.rim.location.xloc text/vtt text/x-component text/x-cross-domain-policy;

    # Pagespeed is not supported by Nextcloud, so if your server is built
    # with the `ngx_pagespeed` module, uncomment this line to disable it.
    #pagespeed off;

    # HTTP response headers borrowed from Nextcloud `.htaccess`
    add_header Referrer-Policy                      "no-referrer"   always;
    add_header X-Content-Type-Options               "nosniff"       always;
    add_header X-Download-Options                   "noopen"        always;
    add_header X-Frame-Options                      "SAMEORIGIN"    always;
    add_header X-Permitted-Cross-Domain-Policies    "none"          always;
    add_header X-Robots-Tag                         "none"          always;
    add_header X-XSS-Protection                     "1; mode=block" always;

    # Remove X-Powered-By, which is an information leak
    fastcgi_hide_header X-Powered-By;

    # Path to the root of your installation
    root /usr/share/nginx/nextcloud;

    # Specify how to handle directories -- specifying `/index.php$request_uri`
    # here as the fallback means that Nginx always exhibits the desired behaviour
    # when a client requests a path that corresponds to a directory that exists
    # on the server. In particular, if that directory contains an index.php file,
    # that file is correctly served; if it doesn't, then the request is passed to
    # the front-end controller. This consistent behaviour means that we don't need
    # to specify custom rules for certain paths (e.g. images and other assets,
    # `/updater`, `/ocm-provider`, `/ocs-provider`), and thus
    # `try_files $uri $uri/ /index.php$request_uri`
    # always provides the desired behaviour.
    index index.php index.html /index.php$request_uri;

    # Default Cache-Control policy
    expires 1m;

    # Rule borrowed from `.htaccess` to handle Microsoft DAV clients
    location = / {
        if ( $http_user_agent ~ ^DavClnt ) {
            return 302 /remote.php/webdav/$is_args$args;
        }
    }

    location = /robots.txt {
        allow all;
        log_not_found off;
        access_log off;
    }

    # Make a regex exception for `/.well-known` so that clients can still
    # access it despite the existence of the regex rule
    # `location ~ /(\.|autotest|...)` which would otherwise handle requests
    # for `/.well-known`.
    location ^~ /.well-known {
        # The following 6 rules are borrowed from `.htaccess`

        rewrite ^/\.well-known/host-meta\.json  /public.php?service=host-meta-json  last;
        rewrite ^/\.well-known/host-meta        /public.php?service=host-meta       last;
        rewrite ^/\.well-known/webfinger        /public.php?service=webfinger       last;
        rewrite ^/\.well-known/nodeinfo         /public.php?service=nodeinfo        last;

        location = /.well-known/carddav     { return 301 /remote.php/dav/; }
        location = /.well-known/caldav      { return 301 /remote.php/dav/; }

        try_files $uri $uri/ =404;
    }

    # Rules borrowed from `.htaccess` to hide certain paths from clients
    location ~ ^/(?:build|tests|config|lib|3rdparty|templates|data)(?:$|/)  { return 404; }
    location ~ ^/(?:\.|autotest|occ|issue|indie|db_|console)              { return 404; }

    # Ensure this block, which passes PHP files to the PHP process, is above the blocks
    # which handle static assets (as seen below). If this block is not declared first,
    # then Nginx will encounter an infinite rewriting loop when it prepends `/index.php`
    # to the URI, resulting in a HTTP 500 error response.
    location ~ \.php(?:$|/) {
        fastcgi_split_path_info ^(.+?\.php)(/.*)$;
        set $path_info $fastcgi_path_info;

        try_files $fastcgi_script_name =404;

        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $path_info;
        fastcgi_param HTTPS on;

        fastcgi_param modHeadersAvailable true;         # Avoid sending the security headers twice
        fastcgi_param front_controller_active true;     # Enable pretty urls
        fastcgi_pass php-handler;

        fastcgi_intercept_errors on;
        fastcgi_request_buffering off;
    }

    location ~ \.(?:css|js|svg|gif)$ {
        try_files $uri /index.php$request_uri;
        expires 6M;         # Cache-Control policy borrowed from `.htaccess`
        access_log off;     # Optional: Don't log access to assets
    }

    location ~ \.woff2?$ {
        try_files $uri /index.php$request_uri;
        expires 7d;         # Cache-Control policy borrowed from `.htaccess`
        access_log off;     # Optional: Don't log access to assets
    }

    location / {
        try_files $uri $uri/ /index.php$request_uri;
    }
}
```

source:https://docs.nextcloud.com/server/20/admin_manual/installation/nginx.html

Nếu không có SSL thì có thể tự generate local ssl.
```
mkdir -p /etc/nginx/cert/
openssl req -new -x509 -days 365 -nodes -out /etc/nginx/cert/nextcloud.crt -keyout /etc/nginx/cert/nextcloud.key
chmod 700 /etc/nginx/cert
chmod 600 /etc/nginx/cert/*
```

```
systemctl restart nginx.service
```

Tạo thư mục chứa data người dùng.
```
mkdir /data
chown nginx. /data/
```
Sau khi hoàn tất thì có thể truy cập vào nextcloud server bằng browser để tiến hành cài đặt.

----------
Error when assembling chunks, status code 504
Tăng thời gian timeout để Nextcloud join file sau khi upload xong.
Sửa file /etc/nginx/nginx.conf
Insert các dòng sau trong tab http

```
vi /etc/nginx/nginx.conf
...
fastcgi_connect_timeout 60;
fastcgi_send_timeout 1800;
fastcgi_read_timeout 1800;
```

This allows up to 30 mins of time to pass. It probably doesn't need to be quite that high, but it all depends on how fast your server disk I/O is.

It'd be great if a future version of Nextcloud had a daemon to handle this kind of task out-of-proc (like Seafile does), since larger files are becoming pretty normal.
