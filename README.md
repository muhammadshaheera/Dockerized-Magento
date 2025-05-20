# Dockerized-Magento

![Docker Magento](https://github.com/user-attachments/assets/93871130-5c5d-4024-9a07-e5364743ec82)

1. Install HAProxy, MariaDB, Docker and Docker Compose and download Magento
`dnf install -y mariadb-server mariadb-client haproxy openssl docker-compose-plugin docker-ce docker-ce-cli containerd.io unzip tar`
Download Magento from: https://github.com/magento/magento2/releases

3. Run `systemctl enable --now mariadb` and `mysql_secure_installation`

4. Create DB:
`CREATE DATABASE magento;`
`CREATE USER 'magento'@'%' IDENTIFIED BY 'magento';`
`GRANT ALL PRIVILEGES ON magento.* TO 'magento'@'%';`
`FLUSH PRIVILEGES;`
`EXIT;`

5. Edit `/etc/haproxy/haproxy.cfg` file:
```
global
    log /dev/log local0
    user haproxy
    group haproxy
    daemon

defaults
    log global
    mode http
    option httplog
    option dontlognull
    timeout connect 5000
    timeout client 50000
    timeout server 50000

frontend http_front
    bind *:80
    default_backend http_back

backend http_back
    balance roundrobin
    server magento1 192.168.249.131:8081 check
    server magento2 192.168.249.131:8082 check
    server magento3 192.168.249.131:8083 check
    server magento4 192.168.249.131:8084 check

listen stats
    bind *:8404
    stats enable
    stats uri /stats
    stats refresh 10s
```

5. Edit `/opt/magento-cluster/docker-compose.yml` file:
```
version: '3.8'

services:
  magento1:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: magento1
    ports: ["8081:80"]
    volumes:
      - "/var/www/html:/var/www/html"
      - "./php.ini:/usr/local/etc/php/conf.d/custom.ini"
    environment:
      - PHP_MEMORY_LIMIT=2G
    networks: ["magento-network"]

  magento2:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: magento2
    ports: ["8082:80"]
    volumes:
      - "/var/www/html:/var/www/html"
      - "./php.ini:/usr/local/etc/php/conf.d/custom.ini"
    environment:
      - PHP_MEMORY_LIMIT=2G
    networks: ["magento-network"]

  magento3:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: magento3
    ports: ["8083:80"]
    volumes:
      - "/var/www/html:/var/www/html"
      - "./php.ini:/usr/local/etc/php/conf.d/custom.ini"
    environment:
      - PHP_MEMORY_LIMIT=2G
    networks: ["magento-network"]

  magento4:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: magento4
    ports: ["8084:80"]
    volumes:
      - "/var/www/html:/var/www/html"
      - "./php.ini:/usr/local/etc/php/conf.d/custom.ini"
    environment:
      - PHP_MEMORY_LIMIT=2G
    networks: ["magento-network"]

  redis:
    image: redis:7.0
    container_name: redis
    command: redis-server --save 60 1 --loglevel warning
    ports: ["6379:6379"]
    networks: ["magento-network"]

  opensearch:
    image: opensearchproject/opensearch:2.5.0
    container_name: opensearch
    environment:
      - discovery.type=single-node
      - bootstrap.memory_lock=true
      - "OPENSEARCH_JAVA_OPTS=-Xms1g -Xmx1g"
      - plugins.security.disabled=true
    ulimits:
      memlock:
        soft: -1
        hard: -1
    ports: ["9200:9200"]
    networks: ["magento-network"]

networks:
  magento-network:
    driver: bridge
```

6. Edit `/opt/magento-cluster/Dockerfile` file:
```
FROM php:8.2-apache

RUN apt-get update && \
    apt-get install -y \
        libzip-dev \
        libicu-dev \
        libxml2-dev \
        libxslt1-dev \
        libonig-dev \
        libfreetype6-dev \
        libjpeg62-turbo-dev \
        libpng-dev \
        libwebp-dev \
        libgd-dev \
        sendmail

RUN docker-php-ext-configure gd --with-freetype --with-jpeg --with-webp && \
    docker-php-ext-install -j$(nproc) \
        zip \
        intl \
        mbstring \
        pdo \
        pdo_mysql \
        soap \
        xsl \
        gd \
        sockets \
        bcmath

RUN pecl install redis && docker-php-ext-enable redis

RUN a2enmod rewrite headers
```

7. Edit `/opt/magento-cluster/php.ini` file:
```
memory_limit = 2G
upload_max_filesize = 64M
post_max_size = 64M
max_execution_time = 1800
max_input_time = 1800
opcache.enable=1
opcache.enable_cli=1
opcache.memory_consumption=256
opcache.interned_strings_buffer=32
opcache.max_accelerated_files=60000
opcache.validate_timestamps=1
opcache.revalidate_freq=60
```

8. Start Services:
`systemctl enable --now haproxy` and `systemctl start haproxy`
`systemctl enable --now docker` and `systemctl start docker`

9. Installation of Magento:
  a. `mkdir -p /var/www/html`
  b. `tar -xzf /tmp/magento2-2.4.6.tar.gz -C /var/www/html --strip-components=1`
  c. `cd /opt/magento-cluster`
  d. `docker-compose build` and `docker compose up -d`
  e. ` docker exec -it magento1 bash -c "
apt-get update && \
apt-get install -y wget && \
wget https://getcomposer.org/installer -O - | php -- --install-dir=/usr/local/bin --filename=composer && \
chmod +x /usr/local/bin/composer
"`
  f. `docker exec -it magento1 bash -c "
cd /var/www/html && \
composer install && \
composer update && \
chmod -R 777 var/ pub/ generated/ app/etc/"`
  g. `docker exec -it magento1 bash -c "cd /var/www/html &&
find var generated vendor pub/static pub/media app/etc -type f -exec chmod g+w {} + &&
find var generated vendor pub/static pub/media app/etc -type d -exec chmod g+ws {} +"`
  h. `docker exec -it magento1 bash -c "cd /var/www/html && \
php bin/magento setup:install \
--base-url=http://192.168.249.131 \
--db-host=192.168.249.131 \
--db-name=magento \
--db-user=magento \
--db-password=magento \
--admin-firstname=Admin \
--admin-lastname=User \
--admin-email=magento@mail.com \
--admin-user=admin \
--admin-password=magento123 \
--language=en_US \
--currency=USD \
--timezone=Asia/Karachi \
--use-rewrites=1 \
--search-engine=opensearch \
--opensearch-host=opensearch \
--opensearch-port=9200 \
--cache-backend=redis \
--cache-backend-redis-server=redis \
--page-cache=redis \
--page-cache-redis-server=redis \
--session-save=redis \
--session-save-redis-host=redis"`
  i. After above installation process, it will display an admin portal access URL such as `/admin_1azxio`
  j. Verify using `192.168.249.131:8081/admin_1azxio` , `192.168.249.131:8082/admin_1azxio` , `192.168.249.131:8083/admin_1azxio` , `192.168.249.131:8084/admin_1azxio`
