# INSTALL NGINX
* Step 1 – Install and configure Nginx, Nginx exporter
    * Create nginx.repo files
    ```
    vi /etc/yum.repos.d/nginx.repo
    ```
    ```
    [nginx-stable]
    name=nginx stable repo
    baseurl=http://nginx.org/packages/centos/$releasever/$basearch/
    gpgcheck=1
    enabled=1
    gpgkey=https://nginx.org/keys/nginx_signing.key
    module_hotfixes=true

    [nginx-mainline]
    name=nginx mainline repo
    baseurl=http://nginx.org/packages/mainline/centos/$releasever/$basearch/
    gpgcheck=1
    enabled=1
    gpgkey=https://nginx.org/keys/nginx_signing.key
    module_hotfixes=true
    ```

    * Download Nginx signing key
    ```
    curl -O https://nginx.org/keys/nginx_signing.key
    ```

    * Add the repository to CentOS
    ```
    rpm --import ./nginx_signing.key
    ```

    * Update all packages
    ```
    yum update -y
    ```

    * Install Nginx and module geoip
    ```
    yum install nginx-module-geoip-1.22.1-1.el7.ngx.x86_64
    ```

    * Create config file
    ```
    vi /etc/nginx/conf.d/cors.conf
    ```
    ```  
    vi /etc/nginx/conf.d/forwarding_proxy.conf
    ```
    ```
    vi /etc/nginx/conf.d/ssl.conf
    ```

    * Change default.conf files
    ```
    vi /etc/nginx/conf.d/default.conf
    ```

    * Crete default.conf.example files
    ```
    vi /etc/nginx/conf.d/default.conf.example
    ```

    * Update nginx.conf files
    ```
    mv /etc/nginx/nginx.conf /etc/nginx/nginx.conf.bk
    ```
    ```
    vi /etc/nginx/nginx.conf
    ```

    * Generate dhparam.pem
    ```
    openssl dhparam -out /etc/ssl/certs/dhparam.pem 2048
    ```
    * Download Nginx tarball and extract
    ```
    wget http://nginx.org/download/nginx-1.25.1.tar.gz
    ```
    ```
    tar -zxvf nginx-1.25.1.tar.gz -C /opt/
    ```

    * Clone headers-more-nginx-module
    ```
    cd /opt/nginx-1.25.1
    ```
    ```
    git clone https://github.com/openresty/headers-more-nginx-module.git
    ```

    * Install pcre packages
    ```
    yum install zlib-devel pcre-devel
    ```
    
    * Compile headers-more module
    ```
    ./configure --with-compat --prefix=/opt/nginx-1.25.1 --add-dynamic-module=/opt/nginx-1.25.1/headers-more-nginx-module
    ```
    ```
    make
    ```
    ```
    make install
    ```
    
    * Copy headers-more module
    ```
    cp objs/ngx_http_headers_more_filter_module.so /etc/nginx/modules/ngx_http_headers_more_filter_module.so
    ```
    
    * Start the Nginx
    ```
    systemctl enable nginx
    ```
    ```    
    systemctl start nginx
    ```

# CONFIG NGINX

* ADD CORS
    * By default, cross-origin requests will be blocked if you haven't configured any CORS directives in Nginx.
    
    * Add new header to allow CORS
        ```
        vi /etc/nginx/conf.d/cors.conf
        ```
        * Change this line
            ```
            more_set_headers 'Access-Control-Allow-Headers: Origin,Accept,Authorization,Cache-Control,Content-Type,DNT,If-Modified-Since,Keep-Alive,Origin,User-Agent,X-Requested-With,X-CSRF-TOKEN,X-System-Type' always;
            ```

* Block folder request and allow file
    ```
    location /crm {
        location ~* /[^./]+$ {
            allow 103.29.26.0/24;
            allow 103.29.27.0/24;
            allow 192.168.2.0/23;
            allow 113.161.36.174/32;
            allow 171.244.236.86/32;
            allow 171.239.92.78/32;
            allow 27.67.93.236/32;
            deny all;
            client_max_body_size 512M;
            include conf.d/forwarding_proxy.conf;
            proxy_connect_timeout       300;
            proxy_send_timeout          300;
            proxy_read_timeout          300;
            send_timeout                300;
            proxy_pass http://ups_04_cdn.omicrm.com;
        }

        location ~* /[^./]+/$ {
            allow 103.29.26.0/24;
            allow 103.29.27.0/24;
            allow 192.168.2.0/23;
            allow 113.161.36.174/32;
            allow 171.244.236.86/32;
            allow 27.67.93.236/32;
            deny all;
            client_max_body_size 512M;
            include conf.d/forwarding_proxy.conf;
            proxy_connect_timeout       300;
            proxy_send_timeout          300;
            proxy_read_timeout          300;
            send_timeout                300;
            proxy_pass http://ups_04_cdn.omicrm.com;
        }

        client_max_body_size 512M;
        include conf.d/forwarding_proxy.conf;
        proxy_connect_timeout       300;
        proxy_send_timeout          300;
        proxy_read_timeout          300;
        send_timeout                300;
        proxy_pass http://ups_04_cdn.omicrm.com;
    }
    ```

* Add password
    * Install httpd-tools
        ```
        yum install httpd-tools
        ```

    * Generate file password for user vboss-kafka at /etc/nginx/htpasswd/vboss-kafka-hq
        ```
        htpasswd -c /etc/nginx/htpasswd/vboss-kafka-hq vboss-kafka
        ```

    * Update config using password
        ```
        location / {
            client_max_body_size 512M;
                auth_basic     "Administrator’s vboss-kafka ";
                auth_basic_user_file /etc/nginx/htpasswd/vboss-kafka-hq;
                include conf.d/forwarding_proxy.conf;
                proxy_http_version 1.1;
                proxy_set_header Upgrade $http_upgrade;
                proxy_set_header Connection $connection_upgrade;
                proxy_read_timeout 900s;
                proxy_pass http://ups_02_kafka-hq.vbossapp.com;
        }
        ```

# SSL CONFIG

* NGINX
    * Concatenate the files in order certificate.crt - 2.intermediate.cer.crt - 1.RootCA_R1.crt to create the cert.crt file
        ```
        cat certificate.crt 2.intermediate.cer.crt 1.RootCA_R1.crt > cert.crt
        ```

* HAPROXY
    * Concatenate the files in order certificate.crt - 2.intermediate.cer.crt - 1.RootCA_R1.crt - ssl.key to create the cert.crt file
        ```
        cat certificate.crt 2.intermediate.cer.crt 1.RootCA_R1.crt ssl.key > cert.crt
        ```

# PROXY CONNECT MODULE

* Download Nginx tarball and extract
    ```
    wget http://nginx.org/download/nginx-1.25.1.tar.gz
    ```
    ```
    tar -zxvf nginx-1.25.1.tar.gz -C /opt/
    ```

* Clone headers-more-nginx-module
    ```
    cd /opt/nginx-1.25.1
    ```
    ```
    git clone https://github.com/chobits/ngx_http_proxy_connect_module.git
    ```

* Compile headers-more module
    ```
    patch -p1 < ../ngx_http_proxy_connect_module/patch/proxy_connect_rewrite_102101.patch
    ```
    ```
    ./configure --with-compat --add-dynamic-module=/opt/ngx_http_proxy_connect_module
    ```
    ```
    make && make install
    ```

* Copy headers-more module
    ```
    cp objs/ngx_http_proxy_connect_module.so /etc/nginx/modules/ngx_http_proxy_connect_module.so
    ```