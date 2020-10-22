Install Nextcloud 20

setenforce 0
sed -i 's/^SELINUX=.*/SELINUX=permissive/g' /etc/selinux/config

yum install epel-release

yum -y install nginx

systemctl enable --now nginx

#Install PHP7.2
yum install http://rpms.remirepo.net/enterprise/remi-release-7.rpm
yum install yum-utils
yum-config-manager --enable remi-php72

yum update
yum install php72
yum install php72-php-fpm php72-php-gd php72-php-json php72-php-mbstring php72-php-mysqlnd php72-php-xml php72-php-xmlrpc php72-php-opcache


php --version

php72 --modules


systemctl enable --now php72-php-fpm.service

vi /etc/opt/remi/php72/php-fpm.d/www.conf

Set user and group to nginx:
user = nginx
group = nginx

systemctl restart php72-php-fpm.service


Edit/add as follows in server section:

    ## enable php support ##
    location ~ \.php$ {
        root /usr/share/nginx/html;
        fastcgi_pass   127.0.0.1:9000;
        fastcgi_index  index.php;
        include        fastcgi_params;
        fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
    }
    
    
systemctl restart nginx

#Install Mariadb server
yum -y install mariadb mariadb-server

systemctl enable --now mariadb

mysql_secure_installation


create database nextcloud_db;
create user nextclouduser@localhost identified by 'nextclouduser@';
grant all privileges on nextcloud_db.* to nextclouduser@localhost identified by 'nextclouduser@';
flush privileges;


mkdir -p /etc/nginx/cert/

And generate a new SSL certificate file with the the openssl command below.
openssl req -new -x509 -days 365 -nodes -out /etc/nginx/cert/nextcloud.crt -keyout /etc/nginx/cert/nextcloud.key

Finally, change the permission of all certificate files to 600 with chmod.

chmod 700 /etc/nginx/cert
chmod 600 /etc/nginx/cert/*

#Download and install Nextcloud

yum -y install wget unzip

cd /tmp
wget https://download.nextcloud.com/server/releases/nextcloud-20.0.0.zip
unzip nextcloud-20.0.0.zip
mv nextcloud/ /usr/share/nginx/html/


cd /usr/share/nginx/html/
mkdir -p nextcloud/data/

chown nginx:nginx -R nextcloud/

cd /etc/nginx/conf.d/
vim nextcloud.conf


```
upstream php-handler {
    server 127.0.0.1:9000;
    #server unix:/var/run/php5-fpm.sock;
}
 
server {
    listen 80;
    server_name cloud.nextcloud.co;
    # enforce https
    return 301 https://$server_name$request_uri;
}
 
server {
    listen 443 ssl;
    server_name cloud.nextcloud.co;
 
    ssl_certificate /etc/nginx/cert/nextcloud.crt;
    ssl_certificate_key /etc/nginx/cert/nextcloud.key;
 
    # Add headers to serve security related headers
    # Before enabling Strict-Transport-Security headers please read into this
    # topic first.
    add_header Strict-Transport-Security "max-age=15768000;
    includeSubDomains; preload;";
    add_header X-Content-Type-Options nosniff;
    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-XSS-Protection "1; mode=block";
    add_header X-Robots-Tag none;
    add_header X-Download-Options noopen;
    add_header X-Permitted-Cross-Domain-Policies none;
 
    # Path to the root of your installation
    root /usr/share/nginx/html/nextcloud/;
 
    location = /robots.txt {
        allow all;
        log_not_found off;
        access_log off;
    }
 
    # The following 2 rules are only needed for the user_webfinger app.
    # Uncomment it if you're planning to use this app.
    #rewrite ^/.well-known/host-meta /public.php?service=host-meta last;
    #rewrite ^/.well-known/host-meta.json /public.php?service=host-meta-json
    # last;
 
    location = /.well-known/carddav {
      return 301 $scheme://$host/remote.php/dav;
    }
    location = /.well-known/caldav {
      return 301 $scheme://$host/remote.php/dav;
    }
 
    # set max upload size
    client_max_body_size 512M;
    fastcgi_buffers 64 4K;
 
    # Disable gzip to avoid the removal of the ETag header
    gzip off;
 
    # Uncomment if your server is build with the ngx_pagespeed module
    # This module is currently not supported.
    #pagespeed off;
 
    error_page 403 /core/templates/403.php;
    error_page 404 /core/templates/404.php;
 
    location / {
        rewrite ^ /index.php$uri;
    }
 
    location ~ ^/(?:build|tests|config|lib|3rdparty|templates|data)/ {
        deny all;
    }
    location ~ ^/(?:\.|autotest|occ|issue|indie|db_|console) {
        deny all;
    }
 
    location ~ ^/(?:index|remote|public|cron|core/ajax/update|status|ocs/v[12]|updater/.+|ocs-provider/.+|core/templates/40[34])\.php(?:$|/) {
        include fastcgi_params;
        fastcgi_split_path_info ^(.+\.php)(/.*)$;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
        fastcgi_param HTTPS on;
        #Avoid sending the security headers twice
        fastcgi_param modHeadersAvailable true;
        fastcgi_param front_controller_active true;
        fastcgi_pass php-handler;
        fastcgi_intercept_errors on;
        fastcgi_request_buffering off;
    }
 
    location ~ ^/(?:updater|ocs-provider)(?:$|/) {
        try_files $uri/ =404;
        index index.php;
    }
 
    # Adding the cache control header for js and css files
    # Make sure it is BELOW the PHP block
    location ~* \.(?:css|js)$ {
        try_files $uri /index.php$uri$is_args$args;
        add_header Cache-Control "public, max-age=7200";
        # Add headers to serve security related headers (It is intended to
        # have those duplicated to the ones above)
        # Before enabling Strict-Transport-Security headers please read into
        # this topic first.
        add_header Strict-Transport-Security "max-age=15768000;
        includeSubDomains; preload;";
        add_header X-Content-Type-Options nosniff;
        add_header X-Frame-Options "SAMEORIGIN";
        add_header X-XSS-Protection "1; mode=block";
        add_header X-Robots-Tag none;
        add_header X-Download-Options noopen;
        add_header X-Permitted-Cross-Domain-Policies none;
        # Optional: Don't log access to assets
        access_log off;
    }
 
    location ~* \.(?:svg|gif|png|html|ttf|woff|ico|jpg|jpeg)$ {
        try_files $uri /index.php$uri$is_args$args;
        # Optional: Don't log access to other assets
        access_log off;
    }
}
```

nginx -t
systemctl restart nginx
