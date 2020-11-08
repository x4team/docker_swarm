***Конфигурация приложения Syncthing***

**1.0)** Создаем 2 приложения Syncthing(через docker-compose) с разными портами доступа и синхронизируем их дефолтные папки между собой, выбирая синхронизацию с первого приложения на второй без двойной синхронизации. 

**1.1)** Устанавливаем логины и пароли на эти приложения
**1.2)** Оставляем первое(главное) приложение и меняем файл докер-композ:

  Указываем в нем внешний том, который будет создан позже через swarm:

  **(ВНИМАНИЕ: По умолчанию в environment: PUID и PGID равные 1000)
               * если вы не хотите работать из под рута ( 0 ) в syncthing, то 
                 можно выполнить 1 раз команду для контейнера syncthing ниже
               ```command: chown -R abc:abc /config/Sync && chmod -R 775 /config/Sync```**
**ФАЙЛ КОНФИГУРАЦИИ:**
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
      - ./config:/config 
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
2) При первом старте в директории syncthing_main не должно быть папки ./config/Sync и также ./config/index-... (ОЧИСТИТЬ)

3) Разбираемся теперь со swarm директорией syncthing_child (бывший второй синксинг 
- см пункт 1.0)
3.1) Также очищаем папку (см пункт 2) 
Меняем весь конфиг на этот:  
(не забываем что наши конфиги берутся из temp - как доп защита от перезаписи в корне)

version: "3.8"
services:
  syncthing_child:
    image: linuxserver/syncthing
    environment:
      - PUID=0
      - PGID=0
      - TZ=Europe/London
      - PASSWORD=lalala
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


4) НА ТОМЕ ОБЯЗАТЕЛЬНО ДОЛЖЕН ПРИСУТСТВОВАТЬ ПУСТОЙ ФАЙЛ .stfolder
Если главный Syncthing ругается - то зайти через docker exec -it syncthing_main sh
И создать пустой файл touch /config/Sync/.stfolder



5)И ПОТОМ ЧТОБЫ ЗАВЕРШИТЬ СТЭК, НАМ НАДО СНАЧАЛО ЗАВЕРШИТЬ MAINSYNCTHING обычный композ,
а уже потом сам docker stack , так как MAINSYNCTHING использовал наш том из swarm

