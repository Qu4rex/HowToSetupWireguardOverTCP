# Ubuntu 24.04
# Настройка WireGuard 

**1. Установка и настройка образа WireGuard server в Docker Compose**
 - Создадим файл конфигурации docker-compose.yml

```
services:
  wireguard:
    image: lscr.io/linuxserver/wireguard:latest
    container_name: wireguard
    cap_add:
      - NET_ADMIN
      - SYS_MODULE #optional
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Etc/UTC
      - SERVERURL=192.168.168.17 #optional for local, use 'auto' for world
      - SERVERPORT=51820 #optional
      - PEERS=1 #optional
      - PEERDNS=auto #optional
      - INTERNAL_SUBNET=10.13.13.0 #optional
      - ALLOWEDIPS=0.0.0.0/0 #optional
      - PERSISTENTKEEPALIVE_PEERS= #optional
      - LOG_CONFS=true #optional
    volumes:
      - ./wireguard:/config
      - /lib/modules:/lib/modules #optional
    ports:
      - 51820:51820/udp
    sysctls:
      - net.ipv4.conf.all.src_valid_mark=1
    restart: unless-stopped
```
 - Запустим наш контейнер  ``` docker compose up -d ```
   
   Супер, наш сервер работает!
   
**2. Установка и настройка WireGuard client**
- Выполним установку wireguard ``` sudo apt install wireguard ```
- Копируем файл конфигурации из сервера для peer1 (пользователь 1) из ./wireguard/peer1/peer1.conf на клиент /etc/wireguard/wg0.conf

Получили:
```
[Interface]
Address = 10.13.13.2
PrivateKey = WNY5gxbBRI2+WWxn19WiFQaFKihDipgeF7aPtlwECGA=
ListenPort = 51820
DNS = 10.13.13.1

[Peer]
PublicKey = 5l0hIO/Ij1Qpuy0o5qz6IUopORQXyuwb6Q9pAeluEhQ=
PresharedKey = qPBo9D07tprO0E+K+rCTy58tRNReYhkRokT5R7lyRUc=
Endpoint = 192.168.168.17:51820
AllowedIPs = 0.0.0.0/0
```
 - Выполняем подключение к VPN серверу ``` wg-quick up wg0 ```
   > Отключиться ``` wg-quick down wg0 ```

 - Проверим доступ к интернету ``` ping 8.8.8.8 ```

![image](https://github.com/user-attachments/assets/1b03df5c-a299-4830-a9db-8b29f6f3755f)

 - На сервере мы увидим

![image](https://github.com/user-attachments/assets/f78b233b-4ba0-4623-84e7-b776d8ab0a84)

Отлично! VPN работает.

**3. Настройка udptunnel для заворачивания TCP пакетов в UDP**

 - Клонируем репозиторий на обоих устройствах в удобную нам папку (сервер, клиент) ``` git clone https://github.com/rfc1036/udptunnel.git ```
 - Выполним команду ``` make ``` в папке, куда клонировали репозиторий
 - Запустим udptunnel в режиме сервера на сервере с докером ``` ./udptunnel -s 192.168.168.17:9000 127.0.0.1:51820 -v ```
   
   192.168.168.17:9000 - открывает порт, на который будут прилетать TCP пакеты
   
   127.0.0.1:51820 -локальный адрес wireguard server и порт, куда будут уходить уже UDP пакеты

 - Запустим udptunnel в режиме клиента на клиенте ``` ./udptunnel 127.0.0.1:51821 192.168.168.17:9000 -v ```
   
   127.0.0.1:51821 - откроем  порт по протоколy UDP
   
   192.168.168.17:9000 - адрес на который будут отправляться TCP пакеты

 - Перенастроим конфиг Wireguard для клиента /etc/wireguard/wg0.conf
  ```
  Endpoint = 127.0.0.1:51821
  ```
  - Выполняем подключение к VPN серверу ``` wg-quick up wg0 ```
    
   Отлично! VPN работает, теперь мы можем принмать TCP пакеты.
