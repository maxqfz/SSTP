# Установка SSTP VPN на Linux

На базе SSTP сервера https://github.com/sorz/sstp-server

### Подготовка

* Установите python3 версии не ниже 3.4.4 `sudo apt-get install python3`
* Установите pip3 `sudo apt-get install -y python3-pip`
* Установите pppd `sudo apt-get install ppp`
* Установите openssl `sudo apt-get install openssl`

### Генерация SSL-сертификата и ключа

Для того, чтобы сгенерировать SSL-сертификат и ключ, необходимо выполнить следующую команду:
`openssl req -newkey rsa:2048 -nodes -keyout sstp.key -x509 -days 365 -out sstp.crt`

Все параметры можете оставить пустыми, кроме параметра Common Name - здесь необходимо указать внешний IP адрес Вашего VPN-сервера.

После этого следует скопировать сертификат на компьютеры клиентов, которые будут подключаться к VPN-серверу и установить его в раздел "Доверенные корневые центры сертификации".

### Установка и настройка sstpd-server

Установите sstp-server с помощью pip3:
`pip3 install sstp-server`

Также его можно установить напрямую из GitHub создателя:
`pip3 install git+https://github.com/sorz/sstp-server.git`

Создайте конфигурационный файл /etc/ppp/options.sstpd с помощью команды `nano /etc/ppp/options.sstpd` и добавьте в него следующее содержимое:
```
name sstpd
refuse-pap
refuse-chap
refuse-mschap
require-mschap-v2
nologfd
nodefaultroute
ms-dns 8.8.8.8
ms-dns 8.8.4.4
```

Теперь необходимо создать файл с данными для входа. Для этого выполните команду `nano /etc/ppp/chap-secrets` и укажите ваши данные для входа в формате `логин сервер пароль IP-адреса`, например:
```
user sstpd strongpassword *
max sstpd "" 55.66.77.88
```

Создайте конфигурационный файл sstp-server командой `nano sstpd.ini` и добавьте в него следующее содержимое:
```ini
[DEFAULT]
# 1 to 50. Default 20, debug 10, verbose 5
;log_level = 20

# OpenSSL cipher suite. See ciphers(1).
;cipher = EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH

# Path to pppd
;pppd = /usr/bin/pppd

# pppd config file path
;pppd_config = /etc/ppp/options.sstpd

# SSTP port
listen = 0.0.0.0
listen_port = 443

# PEM-format certificate with key.
pem_cert = sstp.crt
pem_key = sstp.key

# Address of server side on ppp.
local = 192.168.10.1

# If RADIUS is used to mangle IP pool, comment it out.
remote = 192.168.10.0/24
```

Осталось лишь запустить наш сервер командой `sudo nohup sstpd -f sstpd.ini &> sstpd.log&`

*VPN сервер запущен! Осталось лишь настроить доступ в интернет.*

### Настройка доступа в интернет через VPN

Для того, чтобы у клиентов заработал интернет, необходимо выполнить несколько пунктов:
* Для начала нужно включить перенаправление трафика, отредактировав файл /etc/sysctl.conf `nano /etc/sysctl.conf`
* Добавьте или раскомментируйте строку `net.ipv4.ip_forward=1`
* После этого необходимо запустить команду `sysctl -p` для применения изменений.

Также следует добавить следующие правила IPTABLES:
```bash
iptables -I INPUT -p tcp --dport 443 -m state --state NEW -j ACCEPT
iptables -I INPUT -p gre -j ACCEPT
iptables -t nat -I POSTROUTING -o eth0 -j MASQUERADE
iptables -I FORWARD -p tcp --tcp-flags SYN,RST SYN -s 192.168.10.0/24 -j TCPMSS  --clamp-mss-to-pmtu
```

*Теперь у Вас есть готовый SSTP VPN сервер, к которому можно подключаться из любой системы Windows.*