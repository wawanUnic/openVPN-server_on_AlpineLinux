# openVPN-server_on_AlpineLinux

Процессор / память - VIA Eden Processor 1000MHz (32бит) / 0,5 Гб DDR2

Версия AlpineLinux / Версия ядра Linux - 3.21.3 / 6.12.13-0-lts

Версия сервера / библиотеки - openVPN 2.6.12 / OpenSSL 3.3.3

Настрока производится для VPN сети - 10.8.0.1/24

Порт/протокол обмена - UDP:2020

Сервер имеет глобальный / локальный адрес - 78.10.222.186 / 192.168.4.181/24

Работаем от root

### 1. Впишем новые репозитории для обновления пакетов
```
vi /etc/apk/repositories
	I
	http://dl-cdn.alpinelinux.org/alpine/v3.21/main
	http://dl-cdn.alpinelinux.org/alpine/v3.21/community
	Esc
	:wq
 ```

### 2. Обновим систему
```
apk update
apk upgrade
```

### 2.5. Впишем доп.папку для коммитов (эта папка по-умолчанию не включена в список для коммита)
```
lbu include /etc/init.d/
```

### 3. Установим веб-админку. Будет сгенерирован самоподписанный сертификат для шифрования соединения. Работает только по протоколу httpS (без перенарпавления)
```
setup-acf
```

### 4. Установим пакеты для удобства работы
```
apk add mc
apk add nanp
apk add htop
```

### 5. Установим пакеты для работы сервера openVPN
```
apk add openvpn
apk add openssl
apk add acf-openvpn
apk add acf-openssl
apk add iptables
```

### 6. Добавим сервер openVPN в автозагрузку
```
rc-update add openvpn default
```

### 7. Загружаем модуль ядра TUN. Модуль TUN создаёт виртуальный сетевой интерфейс, который может быть использован для туннелирования IP-пакетов
```
modprobe tun
```

### 8. Добавляем строку "tun" в файл /etc/modules-load.d/tun.conf. Файл tun.conf используется для указания модулей, которые должны загружаться автоматически при загрузке системы. Добавление "tun" в этот файл гарантирует, что модуль TUN будет загружаться автоматически при каждом запуске системы
```
echo "tun" >> /etc/modules-load.d/tun.conf
```

### 9. Разрешаем переанаправление пакетов в системе
```
nano /etc/sysctl.conf
	net.ipv4.ip_forward=1
sysctl -p
```

### 10. Создаем правила для iptables и сохраняем их
```
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -A FORWARD -i eth0 -o tun0 -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -A FORWARD -s 10.8.0.0/24 -o eth0 -j ACCEPT
iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o eth0 -j MASQUERADE
/etc/init.d/iptables save
```

### 11. Добавим iptables в автозагрузку
```
rc-update add iptables default
rc-update add iptables sysinit
```

### 12. Создадим ключи для сервера (тип ключа - ssl_server_cert). Утвердим их. Будет создан файл .pfx, закрытый паролем

### 13. Впишем их в настройки сервера. Необходимо указать на файл .pfx и открыть его паролем

### 14. Добавим ключ Диффи-Хофмана вручную (2048 бит). Штатное средство генерирует слабый ключ (1024 бит)
```
openssl dhparam -out /etc/openvpn/openvpn_certs/dh2048.pem 2028
```

### 15. Генерируем свой OpenVPN Static key V1
```
openvpn --genkey secret /etc/openvpn/openvpn_certs/ta.key
```

### 16. Допишем кофигурацию сервера вручную. Сохраним это и запустим сервер. Убедимся в нормальном запуске
```
port 2020
proto udp
dev tun
ca /etc/openvpn/openvpn_certs/AlpVPN-ca.pem
cert /etc/openvpn/openvpn_certs/AlpVPN-cert.pem
key /etc/openvpn/openvpn_certs/AlpVPN-key.pem
dh /etc/openvpn/openvpn_certs/dh1024.pem
tls-auth /etc/openvpn/openvpn_certs/ta.key 0
server 10.8.0.0 255.255.255.0
ifconfig-pool-persist ipp.txt
push "redirect-gateway def1"
push "dhcp-option DNS 8.8.8.8"
keepalive 10 120
remote-cert-tls client
ncp-ciphers AES-256-GCM:CHACHA20-POLY1305
max-clients 20
persist-key
persist-tun
status openvpn-status.log
log-append /var/log/openvpn.log
verb 3
explicit-exit-notify 1
auth-nocache
push "sndbuf 393216"
push "rcvbuf 393216"
```

### 17. Блокируем доступ к SSH не из сети VPN (это необязательно и может привести к потере доступа!) 
```
nano /etc/ssh/sshd_config
	ListenAddress 10.8.0.1
service sshd restart
```

### 18. Создадим ключи для клиента1 (тип ключа - ssl_client_cert). Клиенты могут и сами себе создавать ключи. Утвердим их. Будет создан файл .pfx, закрытый паролем

### 19. Для разделения сгенерированного общего файла (.pfx) на отдельные файлы ключей (.pem) используем команды в Линукс (необходимо знать пароль)
```
openssl pkcs12 -in file.pfx -cacerts -nokeys -out cacert.pem
openssl pkcs12 -in file.pfx -nocerts -nodes -out key.pem
openssl pkcs12 -in file.pfx -nokeys -clcerts -out cert.pem
```

### 20. Пример настройки клиента во внутренней сети
```
client
dev tun
proto udp
remote 192.168.4.181 2020
resolv-retry infinite
persist-key
persist-tun
remote-cert-tls server
cipher CHACHA20-POLY1305
ca cacert.pem
cert cert.pem
key key.pem
tls-auth ta.key 1
verb 3 
mute 20
```

### 21. Пример настройки клиента во внешней сети (Это слабое устройство, использующее режим inline, т.е. данные вписаны прямо в .ovpn)
```
client
dev tun
proto udp
remote 78.10.222.186 2020
resolv-retry infinite
persist-key
persist-tun
remote-cert-tls server
cipher AES-256-GCM
verb 3
mute 20
<ca>
-----BEGIN CERTIFICATE-----
-----END CERTIFICATE-----
</ca>
<cert>
-----BEGIN CERTIFICATE-----
-----END CERTIFICATE-----
</cert>
<key>
-----BEGIN PRIVATE KEY-----
-----END PRIVATE KEY-----
</key>
<tls-auth>
-----BEGIN OpenVPN Static key V1-----
-----END OpenVPN Static key V1-----
</tls-auth>
```

### 22. Для сохрания коммита в постоянную память Alpine используем команду
```
lbu commit
```

### 23. Проблема в параметре на сервере:
```
tls-auth /etc/openvpn/openvpn_certs/ta.key 0
```
Вот этот вот "0" на конце - это направление, которое может быть 0 или 1. Значение 0 указывает, что ключ используется на стороне сервера, а значение 1 — на стороне клиента. Оно помогает различать, с какой стороны (клиентской или серверной) используется ключ для защиты соединения. Этот параметр важен для правильной настройки и работы функции аутентификации TLS, поскольку сервер и клиент должны знать, как правильно интерпретировать ключ. И это работает только тогда, когда мы кладем файлы ключей .pem (4 шт.) рядом с конфигурационным файлом .ovpn. А если мы вписываем данные ключей прямо в конфигурационный файл (вроде, это называется inline), то это направление нельзя использовать. И слабые клиентские устройства могут принимать данные ключей только в таком формате! Тогда параметр на сервере должен быть таким:
```
tls-auth /etc/openvpn/openvpn_certs/ta.key
```
Без задания направления. Т.е. проверка направления не проводится. Для всех других клиентов этой VPN-сети, которые умеют читать файлы ключей .pem, лежащие рядом с конфигурационным файлом .ovpn, нужно тоже отключить проверку направления.
```
tls-auth ta.key 1 - было
tls-auth ta.key - стало
```
