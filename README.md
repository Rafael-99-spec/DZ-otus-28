# ДЗ №28 Фильтрация трафика - firewalld, iptables
--------------------------------------------------------------------------------------------
### Домашнее задание
```
Сценарии iptables
Цель: Студент получил навыки работы с centralServer, inetRouter.
1) реализовать knocking port
- centralRouter может попасть на ssh inetrRouter через knock скрипт
пример в материалах
2) добавить inetRouter2, который виден(маршрутизируется (host-only тип сети для виртуалки)) с хоста или форвардится порт через локалхост
3) запустить nginx на centralServer
4) пробросить 80й порт на inetRouter2 8080
5) дефолт в инет оставить через inetRouter

* реализовать проход на 80й порт без маскарадинга
Критерии оценки: 5 - все сделано 
```
### Практическая часть
ОБРАТИТЕ ВНИМАНИЕ! До того как выполнять настройки iptables, строго рекомендую скачать пакет - ```iptables-services```, чтобы не возникало проблем с запуском автозагрузки и сохранения конфигов iptables(```yum install -y iptables-services```).

- 1) Реализовать knocking port - centralRouter может попасть на ssh inetRouter через knock скрипт
-- Использование таких программ как knock 
Пропишем правила в iptables:
```
[root@centralRouter vagrant]# sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
[root@centralRouter vagrant]# sudo iptables -A INPUT -p tcp --dport 22 -j REJECT
[root@centralRouter vagrant]# service iptables save
iptables: Saving firewall rules to /etc/sysconfig/iptables:[  OK  ]
```

Затем привести файлы конфигурации к следующему виду:

```/etc/knockd.conf```:

```
options]
	UseSyslog

[opencloseSSH]
	sequence      = 2222:tcp,3333:tcp,4444:tcp
        seq_timeout   = 14
        tcpflags      = syn
        start_command = /sbin/iptables -I INPUT 1 -s %IP% -p tcp --dport 22 -j ACCEPT
        cmd_timeout   = 30
        stop_command  = /sbin/iptables -D INPUT -s %IP% -p tcp --dport ssh -j ACCEPT
```

```/etc/sysconfig/knockd```:

```
OPTIONS="-i eth1"
```

Также необходимо добавить на 23 строку файла ```/etc/init.d/knock``` директиву ```sleep 30``` и перезагрузить systemd(```systemctl daemon-reload```). 

Мы своего добились, теперь можно проверить настройку knocking port. Необходимо зайти на centralRouter командой ```vagrant ssh centralRouter``` и выполнить:
```
 for x in 2222 3333 4444; do sudo nmap -Pn --host_timeout 100 --max-retries 0 -p $x 192.168.255.1; done
```
Здесь программа nmap перебирает в цикле порты по определенной последовательности (в продакшен среде не рекомендуется использовать такие простые номера портов и их должно быть больше)

В сучае чего на centalRouter можно установить программу knock и далее выполнить команду ```knock 192.168.4.24 2222 3333 4444 -d 500```, где d это delay - 500мс между портами.

- 2) Добавить inetRouter2, который виден(маршрутизируется (host-only тип сети для виртуалки)) с хоста или форвардится порт через локалхост
-- Добавлен в Vagrantfile директивой box.vm.network "forwarded_port", guest: 8080, host: 1234, host_ip: "127.0.0.1", id: "nginx" 

- 3) Запустить nginx на centralServer
-- Добавлено в Vagrantfile директивами - sudo yum install -y epel-release; sudo yum install -y nginx; sudo systemctl enable nginx; sudo systemctl start nginx 

- 4) Пробросить 80й порт на inetRouter2 8080
-- 
-- 
