**Перед тем как начать конфигурировать Swarm для приложения Syncthing, давайте определимся какие задачи должны быть решены с помощью этого.**

_Я думаю в 90% случаев мы хотим решить проблему быстрой синхронизации тяжелых данных (или больших данных) на подключенных томах. С этим приложение Syncthing справится.
Перед настройкой мы должны понять схему нашей логики синхронизации. 
Итак вкратце:_

1) Мы подготавливаем 2 конфигурации приложения Syncthing через docker-compose(синхронизируем дефолтные директории между собой), конфигурации которых мы будем использовать дальше для синхронизации. 
То есть у нас будет один главный Syncthing, а второй дочерний, который будет использоваться для Swarm._

2) Наша синхронизация будет односторонней с главной ноды на воркеры, так как по важным причинам данные должны быть эталонные, чтобы мы могли опираться на это. 
И так же мы не можем настраивать двустороннюю синхронизацию, так как воркеры могут работать не всегда одинаково (неполадки сети, сбои воркеров и так далее). 
И также мы должны понимать, что Syncthing настраивается между двумя устройствами, но никак не между самим собой._

3) Далее используя конфигурацию дочернего приложения Syncthing мы разворачиваем stack и сервисы через конфигурацию, в которой мы сразу можем определить тот том, который будет участвовать в синхронизации. К примеру, если он прописан как html, а stack мы назвали "syncthing", то название тома будет "syncthing_html". 
И дополнительно мы ставим ключ  :z для этого тома, чтобы он мог использоваться двумя приложениями сразу на master node(главным Syncthing через docker-compose и дочерним Syncthing через Swarm)._

4) Мы запускаем наш главный Syncthing через docker-compose up -d, и ПРЕДВАРИТЕЛЬНО указываем ему в конфигурации наш том "syncthing_html", который расположен тут же нашей мастер ноде._

5) Тем самым мы получили синхронизацию наших тяжелых данных в томе на мастер ноде и уже далее они распространяются быстро на наши воркеры. 
И как только мы добавляем новый воркер, то на нем разворачивается дочерний Syncthing, и все данные с мастер ноды быстро переносятся на него._


***Конфигурация приложения Syncthing***

**1.0)** Создаем 2 приложения Syncthing(через docker-compose) с разными портами доступа и синхронизируем их дефолтные папки между собой, выбирая синхронизацию с первого приложения на второй (без двойной синхронизации). 

**1.1)** Устанавливаем логины и пароли на эти приложения

**1.2)** Когда все просинхронизировалось, мы завершаем их, и в первом(главном) приложении мы меняем файл конфигурации докер-композ:

**ВНИМАНИЕ:** 
По умолчанию в файле docker-compose в строке environment: PUID и PGID должны быть равными 1000)
*если вы не хотите работать из под рута (PUID и PGID равные 0 ) в syncthing, то 
можно выполнить 1 раз команду для контейнера syncthing ниже
```command: chown -R abc:abc /config/Sync && chmod -R 775 /config/Sync```
               
**ФАЙЛ КОНФИГУРАЦИИ:**
*Указываем в нем внешний том, который будет создан позже через swarm. Сейчас его имя стоит как syncthing_html:


```
version: "3.8"
services:
  syncthing_main:
    image: linuxserver/syncthing
    environment:
      - PUID=0
      - PGID=0
      - TZ=Europe/London
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
    volumes:
      - syncthing_html:/config/Sync:z
      - ./config:/config #Внешнее монитрование директории .config на хост
    ports:
      - 8384:8384
      - 22000:22000
      - 21027:21027/udp
    restart: unless-stopped
    networks:
      - network

networks:
  network:
    driver: overlay

volumes:
  syncthing_html:
    external: true
```
**2)** Сейчас мы НЕ запускаем первый Syncthing, но при первом старте в его директории syncthing_main не должно быть папки ./config/Sync и также ./config/index-... (ОЧИСТИТЬ)

**3)** Разбираемся теперь со Swarm сервисом (директорией syncthing_child).

Он является вторым синксингом - см пункт 1.0),но запускается первым как сервис Swarm

*Также очищаем его директорию (смотри пункт 2) 

**3.1)** Меняем весь конфиг для syncthing_child на этот:  

*Не забываем что наши конфигурации берутся из поддиректории temp - как доп защита,
далее уже когда Swarm создает их у себя в окружении, то они больше не нужны в директории.

```
version: "3.8"
services:
  syncthing_child:
    image: linuxserver/syncthing
    environment:
      - PUID=0
      - PGID=0
      - TZ=Europe/London
    volumes:
      - html:/config/Sync:z
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
      - source: cert.pem
        target: /config/cert.pem
      - source: csrftokens.txt
        target: /config/csrftokens.txt
      - source: https-cert.pem
        target: /config/https-cert.pem
      - source: https-key.pem
        target: /config/https-key.pem
      - source: key.pem
        target: /config/key.pem
      - source: config_child.xml
        target: /config/config.xml
      - source: config_child_0.xml
        target: /config/config.xml.v0
    ports:
      - 8385:8384
      - 22001:22000
      - 21028:21027/udp
    restart: unless-stopped
    networks:
      - app-network

  webserver:
    image: nginx:alpine
    restart: unless-stopped
    tty: true
    ports:
      - "80:80"
      - "443:443"
    networks:
      - app-network
    volumes:
      - html:/usr/share/nginx/html:z

configs:
  config_child.xml:
    file: ./config/config.xml
  config_child_0.xml:
    file: ./config/config.xml.v0
  cert.pem:
    file: ./config/temp/cert.pem
  csrftokens.txt:
    file: ./config/temp/csrftokens.txt
  https-cert.pem:
    file: ./config/temp/https-cert.pem
  https-key.pem:
    file: ./config/temp/https-key.pem
  key.pem:
    file: ./config/temp/key.pem
networks:
  app-network:
    driver: overlay

volumes:
  html:
```

**3.2)** НА ТОМЕ html ОБЯЗАТЕЛЬНО ДОЛЖЕН ПРИСУТСТВОВАТЬ ПУСТОЙ ФАЙЛ .stfolder

*(в ```docker volume ls``` он будет отображаться как syncthing_html) 

*Если главный Syncthing ругается в веб-панели, то зайти на него через ```docker exec -it syncthing_main sh```
и создать пустой файл touch /config/Sync/.stfolder

**3.3)** ЗАПУСКАЕМ SWARM:

```sudo docker stack deploy -c docker-compose.yml syncthing```

*Если будет необходимо ЗАВЕРШИТЬ стэк, сначала мы завершаем syncthing_main (смотри пункт ниже 3.4 - это обычный докер-композ),
а уже потом сам docker stack , так как syncthing_main использовал наш том из swarm.

**3.4)** ЗАПУСКАЕМ наш первый главный syncthing_main (смотри пункт 1.2):

```sudo docker-compose up -d```
