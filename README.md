# Hometask #5. SystemD

 В работе задействован ряд скриптов,которые исполняются при старте виртуалки из Вагрант файла.

* `watchlog.sh` (берет вспомогательные файлы из `wachlog_data`)
* `fcgi.sh` (берет вспомогательные файлы из `fcgi_data`)
* `httpd.sh` (берет вспомогательные файлы из `httpd_data`)

Рассмотрим более подробно поэтапно что каждый из них делает.

## 1. Написание сервиса мониторинга

_Написать service, который будет раз в 30 секунд мониторить лог на предмет наличия ключевого слова (файл лога и ключевое слово должны задаваться в /etc/sysconfig);_

Для начала создадим конфигурационный файл в целевой директории
`/etc/sysconfig` .
```
[root@lvm vagrant]# cd /etc/systemd/system/

[root@lvm vagrant]# vi /etc/sysconfig/watchlog

[root@lvm vagrant]# cat /etc/sysconfig/watchlog
# Configuration file for my watchlog service
# Place it to /etc/sysconfig

# File and word in that file that we will be monitored
WORD="ALERT"
LOG=/var/log/watchlog.log
```

Теперь напишем скрипт:
```
[root@lvm vagrant]# vi /opt/watchlog.sh

[root@lvm vagrant]# cat /opt/watchlog.sh
#!/bin/bash

WORD=$1
LOG=$2
DATE=`date`

if grep $WORD $LOG &> /dev/null
then
    logger "$DATE: The word was found!"
else
    exit 0
fi
```

Дадим права на исполнение
```
[root@lvm vagrant]# chmod +x /opt/watchlog.sh
```

Запишем выдуманные тестовые данные в файл лога `/var/log/watchlog.log`
```
[root@lvm vagrant]# echo "test ALARM test test" > /var/log/watchlog.log
```

Создадим юнит для сервиса (в `/etc/systemd/system/`):
```
[root@lvm vagrant]# vi watchlog.service
[root@lvm vagrant]# cat watchlog.service
[Unit]
Description=My watchlog service

[Service]
Type=oneshot
EnvironmentFile=/etc/sysconfig/watchlog
ExecStart=/opt/watchlog.sh $WORD $LOG
```

Создадим юнит файл для таймера:
```
[root@lvm vagrant]# vi watchlog.timer
[root@lvm vagrant]# cat watchlog.timer
[Unit]
Description=Run watchlog script every 30 second

[Timer]
# Run every 30 second
OnUnitActiveSec=30
Unit=watchlog.service

[Install]
WantedBy=multi-user.target
```

стартанем таймер:
```
[root@lvm vagrant]# systemctl start watchlog.timer

[root@lvm vagrant]# systemctl status watchlog.timer
● watchlog.timer - Run watchlog script every 30 second
   Loaded: loaded (/etc/systemd/system/watchlog.timer; enabled; vendor preset: disabled)
   Active: active (elapsed) since Sat 2020-09-12 12:51:50 UTC; 25min ago

Sep 12 12:51:50 lvm systemd[1]: Started Run watchlog script every 30 second.

```

Проверим что пишется в логах, обнаружим там наше сообщение о находке ключевого слова:
```
Sep 12 15:21:09 lvm systemd[1]: Started Run watchlog script every 30 second.
[root@lvm vagrant]# tail -f /var/log/messages
Sep 12 15:23:26 localhost systemd: Started Session 6 of user vagrant.
Sep 12 15:23:26 localhost systemd-logind: New session 6 of user vagrant.
Sep 12 15:23:35 localhost systemd: Starting My watchlog service...
Sep 12 15:23:35 localhost root: Sat Sep 12 15:23:35 UTC 2020: The word was found!
Sep 12 15:23:36 localhost systemd: Started My watchlog service.
Sep 12 15:23:36 localhost su: FAILED SU (to root) vagrant on pts/0
Sep 12 15:23:41 localhost su: (to root) vagrant on pts/0
Sep 12 15:24:05 localhost systemd: Starting My watchlog service...
Sep 12 15:24:05 localhost root: Sat Sep 12 15:24:05 UTC 2020: The word was found!
Sep 12 15:24:06 localhost systemd: Started My watchlog service.
Sep 12 15:24:35 localhost systemd: Starting My watchlog service...
Sep 12 15:24:36 localhost root: Sat Sep 12 15:24:35 UTC 2020: The word was found!

```


## 2. Переписать init скрипт

_Из репозитория epel установить spawn-fcgi и переписать init-скрипт на unit-файл (имя service должно называться так же: spawn-fcgi);._

Первым делом установим пакет `spawn-fcgi` со всеми необходимыми зависимостями:
```
[root@lvm vagrant]# yum install epel-release -y && yum install spawn-fcgi php php-cli mod_fcgid httpd -y
```

Активируем некоторые опции, сняв комменты:
```
[root@lvm vagrant]# vi /etc/sysconfig/spawn-fcgi
[root@lvm vagrant]# cat /etc/sysconfig/spawn-fcgi
# You must set some working options before the "spawn-fcgi" service will work.
# If SOCKET points to a file, then this file is cleaned up by the init script.
#
# See spawn-fcgi(1) for all possible options.
#
# Example :
SOCKET=/var/run/php-fcgi.sock
OPTIONS="-u apache -g apache -s $SOCKET -S -M 0600 -C 32 -F 1 -P /var/run/spawn-fcgi.pid -- /usr/bin/php-cgi"
```

Создадим Unit файл:
```
[root@lvm vagrant]# vi /etc/systemd/system/spawn-fcgi.service
[root@lvm vagrant]# cat /etc/systemd/system/spawn-fcgi.service
[Unit]
Description=Spawn-fcgi startup service by Otus
After=network.target

[Service]
Type=simple
PIDFile=/var/run/spawn-fcgi.pid
EnvironmentFile=/etc/sysconfig/spawn-fcgi
ExecStart=/usr/bin/spawn-fcgi -n $OPTIONS
KillMode=process

[Install]
WantedBy=multi-user.target
```

Теперь проверим:
```
[root@lvm vagrant]# systemctl start spawn-fcgi

[root@lvm vagrant]# systemctl status spawn-fcgi
● spawn-fcgi.service - Spawn-fcgi startup service by Otus
   Loaded: loaded (/etc/systemd/system/spawn-fcgi.service; enabled; vendor preset: disabled)
   Active: active (running) since Sat 2020-09-12 15:22:55 UTC; 8min ago
 Main PID: 3627 (php-cgi)
   CGroup: /system.slice/spawn-fcgi.service
           ├─3627 /usr/bin/php-cgi
           ├─3628 /usr/bin/php-cgi
           ├─3629 /usr/bin/php-cgi
           ├─3630 /usr/bin/php-cgi
           ├─3631 /usr/bin/php-cgi
           ├─3632 /usr/bin/php-cgi
           ├─3633 /usr/bin/php-cgi
           ├─3634 /usr/bin/php-cgi
           ├─3635 /usr/bin/php-cgi
           ├─3636 /usr/bin/php-cgi
           ├─3637 /usr/bin/php-cgi
           ├─3638 /usr/bin/php-cgi
           ├─3639 /usr/bin/php-cgi
           ├─3640 /usr/bin/php-cgi
           ├─3641 /usr/bin/php-cgi
           ├─3642 /usr/bin/php-cgi
           ├─3643 /usr/bin/php-cgi
           ├─3644 /usr/bin/php-cgi
           ├─3645 /usr/bin/php-cgi
           ├─3646 /usr/bin/php-cgi
           ├─3647 /usr/bin/php-cgi
           ├─3648 /usr/bin/php-cgi
           ├─3649 /usr/bin/php-cgi
           ├─3650 /usr/bin/php-cgi
           ├─3651 /usr/bin/php-cgi
           ├─3652 /usr/bin/php-cgi
           ├─3653 /usr/bin/php-cgi
           ├─3654 /usr/bin/php-cgi
           ├─3655 /usr/bin/php-cgi
           ├─3656 /usr/bin/php-cgi
           ├─3657 /usr/bin/php-cgi
           ├─3658 /usr/bin/php-cgi
           └─3659 /usr/bin/php-cgi

Sep 12 15:22:55 lvm systemd[1]: Started Spawn-fcgi startup service by Otus.

```


## 3. Переписать  unit-file `apache httpd`

_Дополнить unit-файл httpd (он же apache) возможностью запустить несколько инстансов сервера с разными конфигурационными файлами;_  

Скопируем юнит файл и сделаем из него шаблон:
```
[root@lvm vagrant]# cp /usr/lib/systemd/system/httpd.service /etc/systemd/system/httpd@.service

[root@lvm vagrant]# vi /etc/systemd/system/httpd.service
[root@lvm vagrant]# cat /etc/systemd/system/httpd.service
[Unit]
Description=The Apache HTTP Server
After=network.target remote-fs.target nss-lookup.target
Documentation=man:httpd(8)
Documentation=man:apachectl(8)

[Service]
Type=notify
EnvironmentFile=/etc/sysconfig/httpd-%I
ExecStart=/usr/sbin/httpd $OPTIONS -DFOREGROUND
ExecReload=/usr/sbin/httpd $OPTIONS -k graceful
ExecStop=/bin/kill -WINCH ${MAINPID}
KillSignal=SIGCONT
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

Создадим теперь конфиг файлы для двух инстансов веб сервера:
```
[root@lvm vagrant]# vi /etc/sysconfig/httpd-first
[root@lvm vagrant]# cat /etc/sysconfig/httpd-first
OPTIONS=-f conf/first.conf

[root@lvm vagrant]# vi /etc/sysconfig/httpd-second
[root@lvm vagrant]# cat /etc/sysconfig/httpd-second
OPTIONS=-f conf/second.conf

[root@lvm vagrant]# cp /etc/httpd/conf/httpd.conf /etc/httpd/conf/first.conf
[root@lvm vagrant]# cp /etc/httpd/conf/httpd.conf /etc/httpd/conf/second.conf
```

Внесем изменения во второй конфигурационный файл, установив другие значения.:
```
[root@lvm vagrant]# vi /etc/httpd/conf/second.conf
[root@lvm vagrant]# cat /etc/httpd/conf/second.conf | grep -Ev ^#
...
Listen 8080
PidFile /var/run/httpd-second.pid
...
```

Проверим:
```
[root@lvm vagrant]# systemctl start httpd@first
[root@lvm vagrant]# systemctl status httpd@first
● httpd@first.service - The Apache HTTP Server
   Loaded: loaded (/etc/systemd/system/httpd@.service; disabled; vendor preset: disabled)
   Active: active (running) since Sat 2020-09-12 15:23:06 UTC; 2h 49min ago
     Docs: man:httpd(8)
           man:apachectl(8)
 Main PID: 3792 (httpd)
   Status: "Total requests: 0; Current requests/sec: 0; Current traffic:   0 B/sec"
   CGroup: /system.slice/system-httpd.slice/httpd@first.service
           ├─3792 /usr/sbin/httpd -f conf/first.conf -DFOREGROUND
           ├─3793 /usr/sbin/httpd -f conf/first.conf -DFOREGROUND
           ├─3794 /usr/sbin/httpd -f conf/first.conf -DFOREGROUND
           ├─3795 /usr/sbin/httpd -f conf/first.conf -DFOREGROUND
           ├─3796 /usr/sbin/httpd -f conf/first.conf -DFOREGROUND
           ├─3797 /usr/sbin/httpd -f conf/first.conf -DFOREGROUND
           └─3798 /usr/sbin/httpd -f conf/first.conf -DFOREGROUND

Sep 12 15:23:04 lvm systemd[1]: Starting The Apache HTTP Server...
Sep 12 15:23:05 lvm httpd[3792]: AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using 127.0.0.1. Set the 'ServerNam...his message
Sep 12 15:23:06 lvm systemd[1]: Started The Apache HTTP Server.


[root@lvm vagrant]# systemctl start httpd@second
[root@lvm vagrant]# systemctl status httpd@second
● httpd@second.service - The Apache HTTP Server
   Loaded: loaded (/etc/systemd/system/httpd@.service; disabled; vendor preset: disabled)
   Active: active (running) since Sat 2020-09-12 15:23:08 UTC; 2h 52min ago
     Docs: man:httpd(8)
           man:apachectl(8)
 Main PID: 3802 (httpd)
   Status: "Total requests: 0; Current requests/sec: 0; Current traffic:   0 B/sec"
   CGroup: /system.slice/system-httpd.slice/httpd@second.service
           ├─3802 /usr/sbin/httpd -f conf/second.conf -DFOREGROUND
           ├─3803 /usr/sbin/httpd -f conf/second.conf -DFOREGROUND
           ├─3804 /usr/sbin/httpd -f conf/second.conf -DFOREGROUND
           ├─3805 /usr/sbin/httpd -f conf/second.conf -DFOREGROUND
           ├─3806 /usr/sbin/httpd -f conf/second.conf -DFOREGROUND
           ├─3807 /usr/sbin/httpd -f conf/second.conf -DFOREGROUND
           └─3808 /usr/sbin/httpd -f conf/second.conf -DFOREGROUND

Sep 12 15:23:07 lvm systemd[1]: Starting The Apache HTTP Server...
Sep 12 15:23:08 lvm httpd[3802]: AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using 127.0.0.1. Set the 'ServerNam...his message
Sep 12 15:23:08 lvm systemd[1]: Started The Apache HTTP Server.
Hint: Some lines were ellipsized, use -l to show in full.



[root@lvm vagrant]# ss -tnulp | grep httpd
tcp    LISTEN     0      128    [::]:8080               [::]:*                   users:(("httpd",pid=3808,fd=4),("httpd",pid=3807,fd=4),("httpd",pid=3806,fd=4),("httpd",pid=3805,fd=4),("httpd",pid=3804,fd=4),("httpd",pid=3803,fd=4),("httpd",pid=3802,fd=4))
tcp    LISTEN     0      128    [::]:80                 [::]:*                   users:(("httpd",pid=3798,fd=4),("httpd",pid=3797,fd=4),("httpd",pid=3796,fd=4),("httpd",pid=3795,fd=4),("httpd",pid=3794,fd=4),("httpd",pid=3793,fd=4),("httpd",pid=3792,fd=4))
[root@lvm vagrant]#

```
