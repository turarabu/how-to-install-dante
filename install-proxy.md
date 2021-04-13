# Установка Socks v4 и Socks v5 прокси-сервера

## Интро
Данная инструкция предназначена для CentOS 7

## Внешний IP-адресс и интерфейс соединения
Для начала узнаём свой внешний IP адресс.
Внешний IP-адресс не всегда совпадает с IP-адрессом в интерфейсе соединения,
если есть просежуточные узлы, вроде прокси. Узнать можно только через сторонние сервисы.
Например, командой
```
wget -O - -q icanhazip.com
```

Теперь нужно получить интерфейс интернет соединения.
Делается это командой
```
ifconfig
```

В ответе получим примерно такое
```
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 37.18.88.65  netmask 255.255.255.0  broadcast 37.18.88.255
        inet6 fe80::250:56ff:fe02:7718  prefixlen 64  scopeid 0x20<link>
        ether 00:50:56:02:77:18  txqueuelen 1000  (Ethernet)
        RX packets 155066  bytes 23849111 (22.7 MiB)
        RX errors 0  dropped 45  overruns 0  frame 0
        TX packets 18927  bytes 13215095 (12.6 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 32  bytes 2592 (2.5 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 32  bytes 2592 (2.5 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

В данном случае, нужный для нас интерфейс `eth0`.
Если IP-адресс в поле `inet` совпадает с вашим внешним IP-адрессом, значит это нужный вам интерфейс.
В ином случае находим методом отсеивания.
Например, интерфейс `lo` имеет IP-адресс `127.0.0.1`, что указывает на локальный, то есть не внешний

## Установка Dante
Добавляем репозиторий (тут и далее все действия от root, либо подставляйте su перед командой):
```
yum install http://mirror.ghettoforge.org/distributions/gf/gf-release-latest.gf.el7.noarch.rpm
```

Устанавливаем пакет
```
yum --enablerepo=gf-plus install dante-server
```

Создаём каталог для pid-файла
```
mkdir /var/run/sockd
```

Бэкапим стандартный конфиг
```
mv /etc/sockd.conf /etc/sockd.conf.orig
```

## Настройка прокси-сервера
Конфиг для работы на порту 443 (стандартный порт для HTTPS).
Это подойдёт вам, если на этом же сервере у вас не работает какая-нидудь другая программа на этом же порту
(например, web-сервер Apache или Nginx или любой другой).

Открываем файл конфигурации:
```
nano /etc/sockd.conf
```

И пишем такие настройки:
```
user.privileged: root
user.unprivileged: nobody

# вместо eth0 пишем название вашего интерфейса, если у вас другое
# можно указать несколько портов, по умолчанию 1080. Но мы будем использовать 443
internal: et0 port=443

# аналогично, с internal, только без порта
external: eth0

# файлы для записи логов
logoutput: syslog stdout /var/log/sockd.log
errorlog: /var/log/sockd_err.log

# Правила внешнего соединения. По умолчанию none
# Можно использовать username для входа по логин/паролю
# Для этого нужно создать отдельного пользователя в системе
socksmethod: none

# аналогично, как в примере выше, только для внутреннего соединения
clientmethod: none

client pass {
    from: 0.0.0.0/0 to: 0.0.0.0/0
    log: connect disconnect error
}
  
socks pass {
    from: 0.0.0.0/0 to: 0.0.0.0/0
    log: connect disconnect error
}
```

Для сохранения нажимаем Ctrl+O
Для выхода из редактора Ctrl+x


`socks pass` отвечает за внешние соединения.
То есть правила для тех, кто подключается к прокси-серверу
Чтобы ваш сервер не стал общедоступным, рекомендуется органичить диапазон IP-адрессов

Например, чтобы разрешить подключение только по IP-адрессу 37.18.88.80, пишем следующее:
```
socks pass {
    from: 37.18.88.80/32 to: 0.0.0.0/0
    log: connect disconnect error
}
```

свойство `log` указывает какие логи нужно записовать


`client pass` оставляем как есть. Он отвечает за исходящие соединения.
Если хотите органичить диапазон IP-адрессов, к которым можно обращаться,
указываем, как в примере выше

## Настройка фаерволла
Для файервола нужно разрешить порт 443:
```
firewall-cmd --zone=public --add-service=https
```
```
firewall-cmd --zone=public --permanent --add-service=https
```

Если бы порт был другой, то правило выглядело бы иначе:
```
firewall-cmd --permanent --zone=public --add-port=1080/tcp
firewall-cmd --permanent --zone=public --add-port=1080/udp
```

И перезагружаем сам фаерволл
```
firewall-cmd --reload
```

## Запуск сервиса
Разрешаем автозагрузку прокси сервиса
```
systemctl enable sockd.service
```

И запускаем:
```
systemctl start sockd.service
```
