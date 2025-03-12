# openVPN-server_on_AlpineLinux

Работаем от root

Версия сервера - openVPN 2.6.12

Версия библиотеки - OpenSSL 3.3.3

Настрока производится для VPN сети - 10.8.0.1/24

Порт/протокол обмена - UDP:2020

Сервер имеет глобальный адрес - 

Сервер имеет локальный адрес - 192.168.4.181/24

### 1. Впишем новые репозитории для обновления пакетов
```
vi /etc/apk/repositories
	I
	http://dl-cdn.alpinelinux.org/alpine/v3.21/main
	http://dl-cdn.alpinelinux.org/alpine/v3.21/community
	Esc
	:wq
 ```

## 2. Обновим систему
```
apk update
apk upgrade
```

## 2.5. Впишем доп.папку для комитов (эта папка по-умолчанию не включена в список для коммита)
```
lbu include /etc/init.d/
```

## 3. Установим веб-админку. Будет сгенерирован самоподписанный сертификат для шифрования соединения. Работает только по протоколу httpS (без перенарпавления)
```
setup-acf
```

## 4. Установим пакеты для удобства работы
```
apk add mc
apk add nanp
apk add htop
```

## 5. Установим пакеты для работы сервера openVPN
```
apk add openvpn
apk add openssl
apk add acf-openvpn
apk add acf-openssl
apk add iptables
```

## 6. Добавим сервер openVPN в автозагрузку
```
rc-update add openvpn default
```

modprobe tun
echo "tun" >> /etc/modules-load.d/tun.conf


nano /etc/sysctl.conf
	net.ipv4.ip_forward=1
sysctl -p

## X. Создаем правила для iptables и сохраняем их
```
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -A FORWARD -i eth0 -o tun0 -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -A FORWARD -s 10.8.0.0/24 -o eth0 -j ACCEPT
iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o eth0 -j MASQUERADE
/etc/init.d/iptables save
```

## X. Добавим iptables в автозагрузку
```
rc-update add iptables default
rc-update add iptables sysinit
```


ifconfig

...

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

...

openvpn --genkey secret /etc/openvpn/openvpn_certs/ta.key
	tls-auth /etc/openvpn/openvpn_certs/ta.key 0

## X. Блокируем доступ к SSH не из мети VPN (это необязательно и может привести к потере доступа) 
```
nano /etc/ssh/sshd_config
	ListenAddress 10.8.0.1
service sshd restart
```

#
# 2048 bit OpenVPN static key
#
-----BEGIN OpenVPN Static key V1-----
1eb675ed4442450bd1fe719995b40335
бла-бла-бла
299493b34477defb31e8cba42c067c24
-----END OpenVPN Static key V1-----

## X. Для разделения сгенерированного общего файла (.pfx) на отдельные файлы ключей (.pem) используем команды в Линукс (необходимо знать пароль)
```
openssl pkcs12 -in file.pfx -cacerts -nokeys -out cacert.pem
openssl pkcs12 -in file.pfx -nocerts -nodes -out key.pem
openssl pkcs12 -in file.pfx -nokeys -clcerts -out cert.pem
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

## X. Для сохрания коммита в постоянную память Alpine используем команду
```
lbu commit
```
