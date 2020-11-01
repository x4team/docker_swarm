Создаем КЛАСТЕР NextCloud на Docker Swarm (Debian Linux)
==============================

![Кластер на Docker Swarm](https://miro.medium.com/max/653/1*A7hv9CX6Su-5SBR3WB0SDA.png "Докер Swarm")

### 0. Подготовка системы Debian и установка Docker

**0.1) Заходим по ssh на сервер и выолняем вход под рутом**

```ssh user@your_ip```

```su -l```

**0.2)  Установим нужный софт включая ```sudo``` и ```ufw```:**

```
apt-get update -y && apt-get upgrade -y && \
apt install wget git nano vim htop apt-transport-https \
ca-certificates curl gnupg2 gnupg-agent software-properties-common ufw sudo -y
``` 

**0.3)  Добавим нашего пользователя в группу ```sudo```(заменить на своего):**
```
cp /etc/sudoers /etc/sudoers.orginal && \
chmod  0440  /etc/sudoers && \
service sshd restart
```
```
usermod -aG sudo nameuser
```
**0.4) Выйдем и перелогинимся по ssh в систему под нашим пользователем:**
```
exit
```
```
exit
```
```
ssh user@your_ip
```

**0.5) Настроем локали для Perl:**
```
export LANGUAGE=en_US.UTF-8 && \
export LANG=en_US.UTF-8 && \
export LC_ALL=en_US.UTF-8 && \
sudo localedef -i en_US -f UTF-8 en_US.UTF-8
```
**0.6) Добавим ключ репозитория Docker и сам репозиторий:**

```
sudo curl -fsSL https://download.docker.com/linux/debian/gpg | sudo apt-key add - 
```

```
sudo add-apt-repository \
"deb [arch=amd64] https://download.docker.com/linux/debian \
$(lsb_release -cs) stable"
```
**0.7) Устанавливаем Docker и Docker-Compose:**
```
sudo apt-get update && \
sudo apt-get install docker-ce docker-ce-cli containerd.io -y && \
sudo systemctl start docker && \
sudo systemctl enable docker && \
sudo /lib/systemd/systemd-sysv-install enable docker && \
sudo curl -L "https://github.com/docker/compose/releases/download/1.27.4/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose && \
sudo chmod +x /usr/local/bin/docker-compose && \
sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
```
**0.8) Добавим пользователя в группу docker и откроем порты ufw:**
```
sudo usermod -aG docker user && \
sudo ufw allow 2376/tcp && sudo ufw allow 7946/udp && \
sudo ufw allow 7946/tcp && sudo ufw allow 80/tcp && \
sudo ufw allow 2377/tcp && sudo ufw allow 4789/udp
```
**0.9) Перезагрузим ```ssh```,```ufw```,```docker```:**

```
sudo service sshd restart && sudo ufw allow 22/tcp && \
sudo systemctl restart docker
```
```
sudo ufw enable && sudo ufw reload
```

**0.10) Выйдем и перезайдем по ```ssh```:**

```
exit
```

```
ssh user@your_ip
```

**0.11) Расширим swap до 4GB:**
```
sudo swapoff --all && \
sudo fallocate -l 4G /swapfile && \
sudo mkswap /swapfile && \
sudo swapon /swapfile && \
free -h
```

### 1. Приступаем к создания кластера через ```swarm```
**1.1) Узнаем наш IP и нициализируем кластер:**

```
ip a
```
```
docker swarm init --advertise-addr CHANGE_TO_YOUR_IP
```
* В итоге система нам выдаст такой токен для "воркера":
```
docker swarm join --token SWMTKN-1-1pyzhz98g7egxfrpwm8583ocdepp5zskw4tyq2k1cv5srv63zt-1i5anzoy4i5lx8f8v9xt7eo47 YOUR_IP_ADDRESS:2377
```
Этой командой мы создаем воркера на другой нашей машине(ноде).

* Чтобы заново просмотреть токен для "воркера" или для "менеджера", вводим:
```
docker swarm join-token worker
```
```
docker swarm join-token manager
```
 **Внимание! \
 Менеджеров мы должны добавить нечетное количество,
 либо 1, \
 либо 3 и более. \
 При двух менеджерах, в случае потери одного, - второй не сможет \
 переизбрать себя лидером. И также менеджеры потребляют большое \
 количество ресурсов. \
 По воркерам также действует правило нечетности.**
* Посмотреть help по команде:
```
docker swarm init --help
```
* Проверить список доступных нод после добавления воркеров/менеджеров:
```
docker node ls
```

**2.1) Теперь в домашней директории создадим папку ```nextcloud```:**
```
mkdir ~/nextcloud && cd ~/nextcloud
```
**2.2) Создадим конфиг nginx через vim/nano:**
```
vim nginx.conf
```
* **И вставим туда этот код:**

```
worker_processes auto;

error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;


events {
    worker_connections  1024;
}


http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    keepalive_timeout  65;

    #gzip  on;

    upstream php-handler {
        server app:9000;
    }

    server {
        listen 80;

        # Add headers to serve security related headers
        # Before enabling Strict-Transport-Security headers please read into this
        # topic first.
        #add_header Strict-Transport-Security "max-age=15768000; includeSubDomains; preload;" always;
        #
        # WARNING: Only add the preload option once you read about
        # the consequences in https://hstspreload.org/. This option
        # will add the domain to a hardcoded list that is shipped
        # in all major browsers and getting removed from this list
        # could take several months.
        add_header Referrer-Policy "no-referrer" always;
        add_header X-Content-Type-Options "nosniff" always;
        add_header X-Download-Options "noopen" always;
        add_header X-Frame-Options "SAMEORIGIN" always;
        add_header X-Permitted-Cross-Domain-Policies "none" always;
        add_header X-Robots-Tag "none" always;
        add_header X-XSS-Protection "1; mode=block" always;

        # Remove X-Powered-By, which is an information leak
        fastcgi_hide_header X-Powered-By;

        # Path to the root of your installation
        root /var/www/html;

        location = /robots.txt {
            allow all;
            log_not_found off;
            access_log off;
        }

        # The following 2 rules are only needed for the user_webfinger app.
        # Uncomment it if you're planning to use this app.
        #rewrite ^/.well-known/host-meta /public.php?service=host-meta last;
        #rewrite ^/.well-known/host-meta.json /public.php?service=host-meta-json last;

        # The following rule is only needed for the Social app.
        # Uncomment it if you're planning to use this app.
        #rewrite ^/.well-known/webfinger /public.php?service=webfinger last;

        location = /.well-known/carddav {
            return 301 $scheme://$host:$server_port/remote.php/dav;
        }

        location = /.well-known/caldav {
            return 301 $scheme://$host:$server_port/remote.php/dav;
        }

        # set max upload size
        client_max_body_size 10G;
        fastcgi_buffers 64 4K;

        # Enable gzip but do not remove ETag headers
        gzip on;
        gzip_vary on;
        gzip_comp_level 4;
        gzip_min_length 256;
        gzip_proxied expired no-cache no-store private no_last_modified no_etag auth;
        gzip_types application/atom+xml application/javascript application/json application/ld+json application/manifest+json application/rss+xml application/vnd.geo+json application/vnd.ms-fontobject application/x-font-ttf application/x-web-app-manifest+json application/xhtml+xml application/xml font/opentype image/bmp image/svg+xml image/x-icon text/cache-manifest text/css text/plain text/vcard text/vnd.rim.location.xloc text/vtt text/x-component text/x-cross-domain-policy;

        # Uncomment if your server is build with the ngx_pagespeed module
        # This module is currently not supported.
        #pagespeed off;

        location / {
            rewrite ^ /index.php;
        }

        location ~ ^\/(?:build|tests|config|lib|3rdparty|templates|data)\/ {
            deny all;
        }
        location ~ ^\/(?:\.|autotest|occ|issue|indie|db_|console) {
            deny all;
        }

        location ~ ^\/(?:index|remote|public|cron|core\/ajax\/update|status|ocs\/v[12]|updater\/.+|oc[ms]-provider\/.+)\.php(?:$|\/) {
            fastcgi_split_path_info ^(.+?\.php)(\/.*|)$;
            set $path_info $fastcgi_path_info;
            try_files $fastcgi_script_name =404;
            include fastcgi_params;
            fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
            fastcgi_param PATH_INFO $path_info;
            # fastcgi_param HTTPS on;

            # Avoid sending the security headers twice
            fastcgi_param modHeadersAvailable true;

            # Enable pretty urls
            fastcgi_param front_controller_active true;
            fastcgi_pass php-handler;
            fastcgi_intercept_errors on;
            fastcgi_request_buffering off;
        }

        location ~ ^\/(?:updater|oc[ms]-provider)(?:$|\/) {
            try_files $uri/ =404;
            index index.php;
        }

        # Adding the cache control header for js, css and map files
        # Make sure it is BELOW the PHP block
        location ~ \.(?:css|js|woff2?|svg|gif|map)$ {
            try_files $uri /index.php$request_uri;
            add_header Cache-Control "public, max-age=15778463";
            # Add headers to serve security related headers (It is intended to
            # have those duplicated to the ones above)
            # Before enabling Strict-Transport-Security headers please read into
            # this topic first.
            #add_header Strict-Transport-Security "max-age=15768000; includeSubDomains; preload;" always;
            #
            # WARNING: Only add the preload option once you read about
            # the consequences in https://hstspreload.org/. This option
            # will add the domain to a hardcoded list that is shipped
            # in all major browsers and getting removed from this list
            # could take several months.
            add_header Referrer-Policy "no-referrer" always;
            add_header X-Content-Type-Options "nosniff" always;
            add_header X-Download-Options "noopen" always;
            add_header X-Frame-Options "SAMEORIGIN" always;
            add_header X-Permitted-Cross-Domain-Policies "none" always;
            add_header X-Robots-Tag "none" always;
            add_header X-XSS-Protection "1; mode=block" always;

            # Optional: Don't log access to assets
            access_log off;
        }

        location ~ \.(?:png|html|ttf|ico|jpg|jpeg|bcmap|mp4|webm)$ {
            try_files $uri /index.php$request_uri;
            # Optional: Don't log access to other assets
            access_log off;
        }
    }
}
```
**2.2) Создадим конфигурацию ```docker-compose```:**
```
vim docker-compose.yml
```
* **Добавим кода. Не забудь заменить ```NEXTCLOUD_TRUSTED_DOMAINS``` на свое) \
 и другие данные (пароли,пользователи):**
 * **ОБЪЯСНИТЬ КОНФИГ**

```
version: '3.8'

services:
  db:
    image: mariadb
    command: --transaction-isolation=READ-COMMITTED --binlog-format=ROW
    restart: always
    volumes:
      - db:/var/lib/mysql
    deploy:
      replicas: 1
      resources:
        limits:
          memory: 240m
          cpus: "0.33"
        reservations:
          memory: 50m
          cpus: "0.1"
      restart_policy:
        condition: on-failure
    environment:
      - MYSQL_ROOT_PASSWORD=Llalalal
      - MYSQL_PASSWORD=ASdmlkasdm
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud
    networks: 
      - nextcloud

  app:
    image: nextcloud:fpm
    links:
      - db
    volumes:
      - nextcloud:/var/www/html
    depends_on:
      - db
    deploy:
      replicas: 1
      resources:
        limits:
          memory: 340m
          cpus: "0.5"
        reservations:
          memory: 50m
          cpus: "0.1"
      restart_policy:
        condition: on-failure
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/London
      - NEXTCLOUD_ADMIN_USER=User
      - NEXTCLOUD_ADMIN_PASSWORD=passNEXTCLOUD
      - NEXTCLOUD_TRUSTED_DOMAINS=10.20.0.27
      - MYSQL_HOST=db
      - MYSQL_USER=nextcloud
      - MYSQL_PASSWORD=ASdmlkasdm
      - MYSQL_DATABASE=nextcloud
    restart: always
    networks: 
      - nextcloud
  web:
    image: nginx
    ports:
      - 80:80
    links:
      - app
    volumes:
      - nextcloud:/var/www/html:ro
    depends_on:
      - app
    deploy:
      replicas: 1
      resources:
        limits:
          memory: 240m
          cpus: "0.33"
        reservations:
          memory: 50m
          cpus: "0.1"
      restart_policy:
        condition: on-failure
    configs:
      - source: nginx.conf
        target: /etc/nginx/nginx.conf
    restart: always
    networks: 
      - nextcloud 
configs:
  nginx.conf:
    file: ./nginx.conf
networks:
  nextcloud:
volumes:
  nextcloud:
  db:
```
* **Также в конфигурации можно поменять значения ```memory```, ```cpu``` \
для каждого сервиса**


**2.3)  Создаем stack нашего приложения!**
```
docker stack deploy -c docker-compose.yml nextcloud
```
* ```nextcloud``` - это названия нашего приложения(сервисов)
* Для создания стэка может потребовать какое-то время \
(обычно несколько минут)
* **Встроенный Load Balancer распределяет каждый запрос к \
серверу в режиме реального времени на разные контейнеры, \
ТЕ, которые сейчас БОЛЕЕ доступны/НЕ нагружены**

**2.4)  Проверяем статус наших сервисов (на каждой ноде):**
```
docker service ls
```
*  **Показатели реплик должны быть 1/1 на каждом сервисе**

**2.5) Тестируем наш стэк:**
```
http://mainworker_ip_address
http://worker2_ip_address
http://worker3_ip_address
```
### 3. Полезные команды ```swarm``` и ```docker```:
**3.1) Тестируем отказоустойчивость стэка. Оставновливаем ноду:**
```
sudo systemctl stop docker
```
* Если мы остановили лидера, то просмотрев ```docker node ls```, \
у нас должны поменяться роли наших нод.

**3.2) Тестируем отказоустойчивость сервисов:**
```
docker service ls
```
```
docker rm -r 913hjk2489
```
```
docker service ls
```
* Сервис должен пересоздаться автоматически.

**3.3) Если возникли ошибки стэка (ПРО БЭКАП ЧИТАЙТЕ НИЖЕ):**
* Обновить рой при ошибках:
```
docker swarm init --force-new-cluster
```
* Если происходят "чудеса" с одним из воркеров:
```
docker swarm leave --force 
```
И далее перезайти заново по токену
*Либо нужно полностью обновить ноду:
```
sudo systemctl docker stop && \
sudo rm -rf /var/lib/docker/swarm/* && \
sudo docker volume prune && \
sudo docker system prune && \
sudo rm -rf /var/lib/docker/overlay2/* && \
sudo systemctl docker start && \
sudo reboot
```
И заново законнектить ноду по токену воркера или менеджера

**3.4) ВАЖНО! БЭКАП SWARM:**

```
sudo systemctl docker stop && \
sudo tar cvzf ~/swarm_$(date +%F).tar.gz /var/lib/docker/swarm && \
sudo systemctl docker start
```
* Либо бэкапим полностью /var/lib/docker

**3.5) Создать сервис вручную без docker-compose:**

```docker service create --replicas 3 --name my-service --network my-network my-image```
* ОБНОВИТЬ количество реплик сервиса на 1 ноде:
```
docker service update --replicas 4 my-service
```

**3.6) Создать конфиг вручную (все про конфиги):**
```
docker config create conf_nginx nginx.conf
```
* Просмотреть все конфиги:
```
docker config ls
```
* Просмотреть содержимое конфига:
```
docker config inspect conf_nginx --pretty
```
После создания объекта - мы можем удалить конфиг файл (или оставим по желанию)
* Обновляем сервис, подключаем конфиг:
```
docker service update --config-add conf_nginx nginx
```
* Что бы подключить конфиг в определённый каталог — используем src и target:
```
docker service update --config-add src=conf_nginx,target="/etc/nginx/conf.d/dc-swarm.local.conf" nginx
```
* Отключаем конфиг от сервиса:
```
docker service update --config-rm conf_nginx nginx
```
* Удаляем старый конфиг из Swarm:
```
docker config rm vhost_nginx conf_nginx
```
**3.7) Создать сервисы вручную:**
```
docker service create --name db \
	--env MYSQL_DATABASE=nextcloud  --env MYSQL_USER=nextcloud \
	--env MYSQL_PASSWORD=NEXTCLOUDPASSUSER  --env MYSQL_ROOT_PASSWORD=LALALALAf \
	--network nextcloud --mode global mariadb
```
* ```--mode global``` - делаем по 1 копии контейнера на каждую ноду глобально
***
```
docker service create --name nextcloud -p 80:80 \
	--env MYSQL_DATABASE=nextcloud --env MYSQL_USER=nextcloud \
	--env MYSQL_PASSWORD=NEXTCLOUDPASSUSER  --env MYSQL_HOST=db \
	--env NEXTCLOUD_ADMIN_USER=User --env NEXTCLOUD_ADMIN_PASSWORD=passNEXTCLOUD \
	--env NEXTCLOUD_TRUSTED_DOMAINS=10.20.0.27 --network nextcloud --mode global nextcloud:latest
```
**3.8) Создать сеть вручную**
```
docker network create --driver overlay nextcloud
```

**3.9) Управление конфигурацией вместо YML можно использовать \
  Fabricio (с помощью python)**

**3.10) КОНФИГ ДЛЯ NGINX С секретами и конфигами:** \
https://devopscon.io/blog/continuous-deployment-docker-swarm/
```
version: "3.4"
  
services:
  proxy:
    image: "${NGINX_IMAGE:-nginx:alpine}"
    networks:
      - app
    ports:
      - "8080:80"
      - "8443:443"
    deploy:
      placement:
        constraints:
          - node.role==worker
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
      restart_policy:
        condition: any
    configs:
      - source: https.conf
        target: /etc/nginx/conf.d/https.conf
    secrets:
      - site.key
      - site.crt
  whoami:
    image: emilevauge/whoami:latest
    networks:
      - app
    deploy:
      placement:
        constraints:
          - node.role==worker
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
      restart_policy:
        condition: on-failure
  
networks:
  app:
    driver: overlay
  
configs:
  https.conf:
    file: ./https-backend.conf
  
secrets:
  site.key:
    file: ./cert/site.key
  site.crt:
    file: ./cert/site.crt
```



**3.11)ПОЛЕЗНЫЕ КОМАНДЫ ДОКЕР:**
*Создать самостоятельный том можно следующей командой:
```
docker volume create —-name my_volume
docker volume ls
```
*Исследовать конкретный том можно так:
  ```
  docker volume inspect my_volume
  ```
*Удалить все тома, которые не используются:
```
  docker volume prune
  ```
*Очистка системы докера если есть серьезные ошибки (удаляет все!):
```
  docker system prune
  ```
* Обменяться данными(томами) двум приложениям:
```
docker run -v /data/myfolder --name container1 image-name-1
docker run --volumes-from container1 image-name-2
```
* Дополнительные полезные команды, которыми можно пользоваться при работе с томами Docker:
```
	docker volume create
	docker volume ls
	docker volume inspect
	docker volume rm
	docker volume prune
	```
	```docker-compose down -v``` *Удаление контейнеров вместе с волюмами
*Список часто используемых параметров для --mount, применимых \
в команде вида docker run --mount my_options my_image:
```
	type=volume
	source=volume_name
	destination=/path/in/container
	readonly
```
