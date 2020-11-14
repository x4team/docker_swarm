**Before we start configuring Swarm for the Syncthing application, let's define what tasks should be accomplished with this.**

_I think 90% of the time we want to solve the problem of fast synchronization of heavy data (or big data) on mounted volumes. The Syncthing app will handle this.
Before setting up, we need to understand the outline of our synchronization logic.
So in short:_
1) We prepare 2 configurations of the Syncthing application via docker-compose (we synchronize the default directories with each other), the configurations of which we will use further for synchronization.
That is, we will have one main Syncthing, and the second child, which will be used for Swarm.

2) Our synchronization will be one-way from the main node to the workers, since for important reasons the data must be reference so that we can rely on this.
And we also cannot configure two-way synchronization, since workers may not always work the same way (network problems, worker failures, and so on).
And we also need to understand that Syncthing is configured between two devices, but not between itself.

3) Next, using the configuration of the Syncthing child application, we deploy the stack and services through the configuration, in which we can immediately determine the volume that will participate in synchronization. For example, if it is written as html, and we named the stack "syncthing", then the volume name will be "syncthing_html".
And in addition, we put the: z key for this volume so that it can be used by two applications at once on the master node (the main Syncthing via docker-compose and the child Syncthing via Swarm).

4) We launch our main Syncthing via docker-compose up -d, and PRELIMINARY specify our volume "syncthing_html", which is located right there in our master node, in the configuration.

5) Thus, we got the synchronization of our heavy data in the volume on the master node and then it spreads quickly to our workers.
And as soon as we add a new worker, the child Syncthing is deployed on it, and all data from the master node is quickly transferred to it._


***Syncthing application configuration***

**1.0)** We create 2 Syncthing applications (via docker-compose) with different access ports and synchronize their default folders with each other, choosing synchronization from the first application to the second (without double synchronization). 

**1.1)** We set logins and passwords for these applications

**1.2)** When everything has synchronized, we terminate them, and in the first (main) application, we change the docker-compose config file:

**ATTENTION:**
By default, in the docker-compose file, the environment: PUID and PGID should be 1000)
* if you do not want to work from under the root (PUID and PGID equal to 0) in syncthing, then
you can execute the command for the syncthing container below 1 time
```command: chown -R abc:abc /config/Sync && chmod -R 775 /config/Sync```
               
**THE CONFIGURATION FILE:**
* We indicate in it the external volume, which will be created later through Swarm. Now its name is syncthing_html:


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
      - ./config:/config #External monitoring of the .config directory to the host
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
**2)** Now we DO NOT start the first Syncthing, but at the first start in its syncthing_main directory there should be no ./config/Sync folder and also ./config/index -... (CLEAR)

**3)** We are now dealing with the Swarm service (the syncthing_child directory).

It is the second Syncthing (see point 1.0), but is launched first as a Swarm service

*We also clear its directory (see point 2)

**3.1)** Change the whole config for syncthing_child to this one:

*Do not forget that our configurations are taken from the temp subdirectory - as additional protection,
further, when Swarm creates them in its environment, they are no longer needed in the directory.

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

**3.2)** THE html VOLUME MUST BE PRESENT AN EMPTY FILE .stfolder

*(in ```docker volume ls``` it will show up as syncthing_html) 

*If the main Syncthing swears in the web panel, then go to him through ```docker exec -it syncthing_main sh```
and create an empty file ```touch /config/Sync/.stfolder```

**3.3)** RUN SWARM STACK:
```cd syncthing_child```
```sudo docker stack deploy -c docker-compose.yml syncthing```

*If it is necessary to COMPLETE the stack, we first complete syncthing_main (see paragraph 3.4 below - this is a normal docker compose),
and only then the docker stack itself, since syncthing_main used our volume from swarm.

**3.4)** LET'S START our first main syncthing_main (see section 1.2):

```sudo docker-compose up -d```
