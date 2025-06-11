На просторах интернета нашел всего пару русскоязычных статей по установке и настройке сервера OCSERV которые были не слишком актуальные, поэтому решил написать свою здесь :) 

Для данной инструкции использовал виртуальный сервер на Ubuntu 22.04, команды выполняем от имени **root** или через **sudo**.

### 1. Установка зависимостей

Так как пакет ocserv который мы можем установить с помощью apt достаточно старой версии, утилиту нужно собрать самому.

Сначала обновляем пакеты и устанавливаем необходимые зависимости для сборки OCSERV:
```
sudo apt update
sudo apt upgrade -y
sudo apt install -y \
    git build-essential autoconf automake libtool \
    pkg-config libgnutls28-dev libev-dev libpam0g-dev \
    liblz4-dev libseccomp-dev libreadline-dev libnl-route-3-dev \
    libkrb5-dev libradcli-dev libcurl4-gnutls-dev libhttp-parser-dev \
    libc-ares-dev libgeoip-dev libprotobuf-c-dev libtalloc-dev \
    libssl-dev libpcl1-dev libopts25-dev libnl-nf-3-dev \
    gperf iptables protobuf-c-compiler
```

### 2. Клонируем репозиторий
```
git clone https://gitlab.com/openconnect/ocserv.git
cd ocserv
```

### 3. Сборка и установка

Запускаем автосборочный скрипт и собираем проект:
```
autoreconf -fvi
./configure --prefix=/usr --sysconfdir=/etc
make
sudo make install
```

### 5. Проверяем версию
Убеждаемся, что ocserv установился:
```
ocserv --version
```

### 6. Включаем маршрутизацию пакетов

Добавляем в файл **/etc/sysctl.conf** строчку:
```
net.ipv4.ip_forward = 1
```
Проверяем:
```
sysctl -p
```

### 7. Настраиваем timesyncd

Добавить в файл **/etc/systemd/timesyncd.conf** строчку с NTP серверами:
```
NTP=0.pool.ntp.org 1.pool.ntp.org 2.pool.ntp.org 3.pool.ntp.org
```
Перезапускаем службу:
```
systemctl restart systemd-timesyncd
```
Проверяем статус:
```
systemctl status systemd-timesyncd
```
Состояние синхронизации проверяем командой:
```
timedatectl
```

### 8. Настройка маршрутизации

> Для каждого случая данные параметры настраиваются индивидуально. Я использовал Policy-Based Routing

Создаём новую таблицу маршрутизации (`/etc/iproute2/rt_tables` ):
```
echo "200 vpntable" | sudo tee -a /etc/iproute2/rt_tables
```

Добавляем правила:

> Корректируйте параметры под себя, а именно подсеть, интерфейс и шлюз

```
ip rule add from 192.168.1.0/24 lookup vpntable
ip route add default via 1.1.1.1 dev eth0 table vpntable  # 1.1.1.1 — шлюз

```

Разрешаем форвардинг:
```
sudo iptables -A FORWARD -i vpns+ -o eth0 -j ACCEPT
```

Для сохранения правил устанавливаем пакет:
```
sudo apt install iptables-persistent
sudo netfilter-persistent save
```

На роутере нужно создать маршрут, в моем случае на Mikrotik это делается такой командой:

`192.168.1.0/24` — подсеть VPN
`10.10.10.20` — IP адрес сервера OCSERV

```
/ip route add dst-address=192.168.1.0/24 gateway=10.10.10.20 scope=30 target-scope=10 disabled=no distance=1 check-gateway=unicast
```

### 9. Создание сертификата от Let's Encrypt

Установка **certbot**:
```
sudo apt install certbot
```
Получаем сертификат:
```
certbot certonly -d vpn.mydomen.ru --standalone
```
Добавляем в настройки **etc/letsencrypt/renewal/vpn.mydomen.ru.conf** автоматический перезапуск OCSERV при renewal сертификата:
```
post_hook = systemctl restart ocserv
```
Создадим файл с параметрами Диффи-Хеллмана:
```
openssl dhparam -out /etc/ocserv/dh.pem 3072
```

### 10. Настройка OCSERV

Есть два варианта настройки.

Первый скопировать пример конфига:
```
sudo cp doc/sample.config /etc/ocserv/ocserv.conf
sudo nano /etc/ocserv/ocserv.conf
```
И настроить по своим хотелкам, либо второй вариант, оставим только нужные нам строки:
Создаем конфиг файл:
```
sudo nano /etc/ocserv/ocserv.con
```

Заполняем:
```
server-cert = /etc/letsencrypt/live/vpn.mydomain.ru/fullchain.pem #Путь к сертификату
server-key = /etc/letsencrypt/live/vpn.mydomain.ru/privkey.pem #Путь к сертификату
dh-params = /etc/ocserv/dh.pem

tcp-port = 443 #Порт подключения

run-as-user = nobody #Запуск от имени nobody
run-as-group = daemon #Запуск от имени группы daemon

auth = "plain[passwd=/etc/ocserv/ocserv.passwd]" #Путь к файлу с пользователями
default-domain = vpn.mydomain.ru #Домен
ipv4-network = 192.168.1.0/24 #Подсеть VPN клиентов
dns = 10.10.10.11 #DNS сервер
route = 10.10.10.0/255.255.255.0 #Маршруты разрешенные для VPN клиентов
split-dns = mydomain.ru #Внутренний домен вашей сети

socket-file = /var/run/ocserv-socket
max-clients = 16
max-same-clients = 2
isolate-workers = false
rate-limit-ms = 100
server-stats-reset-time = 604800
keepalive = 32400
dpd = 90
mobile-dpd = 1800
switch-to-tcp-timeout = 25
try-mtu-discovery = false
cert-user-oid = 0.9.2342.19200300.100.1.1
tls-priorities = "NORMAL:%SERVER_PRECEDENCE:%COMPAT:-VERS-SSL3.0:-VERS-TLS1.0:-VERS-TLS1.1"
auth-timeout = 240
min-reauth-time = 300
max-ban-score = 80
ban-reset-time = 1200
cookie-timeout = 300
deny-roaming = false
rekey-time = 172800
rekey-method = ssl
use-occtl = true
pid-file = /var/run/ocserv.pid
log-level = 2
device = vpns
predictable-ips = true
ping-leases = false
cisco-client-compat = true
dtls-legacy = true
cisco-svc-client-compat = false
client-bypass-protocol = false
```

Параметры `default-domain`, `ipv4-network`, `dns`, `route`, `server-cert`, `server-key`, `split-dns` вам нужно подбить под свои данные.

### 11. Создание пользователей OpenConnect
Команда для создания пользователя:
```
ocpasswd -c /etc/ocserv/ocserv.passwd username
```
Команда для блокировки:
```
ocpasswd -c /etc/ocserv/ocserv.passwd -l username
```
Команда для разблокировки:
```
ocpasswd -c /etc/ocserv/ocserv.passwd -u username
```
Команда для удаления пользователя:
```
ocpasswd -c /etc/ocserv/ocserv.passwd -d username
```

### 12. Создаем systemctl для автоматического запуска
```
sudo nano /etc/systemd/system/ocserv.service
```
Пример содержимого:
```
[Unit]
Description=OpenConnect SSL VPN server
After=network.target

[Service]
ExecStart=/usr/sbin/ocserv --foreground --config /etc/ocserv/ocserv.conf
Restart=on-failure

[Install]
WantedBy=multi-user.target
```
Включаем службу:
```
sudo systemctl enable --now ocserv
```

Проверяем статус службы:
```
systemctl status ocserv
```

После полной установки рекомендую перезапустить сервер и повторно проверить статус службы:
```
reboot
systemctl status ocserv
```

На этом все, пробуем подключиться, например через Cisco AnyConnect :)
