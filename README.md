# Домашнее задание № 12 по теме: "SELinux - когда все запрещено". К курсу Administrator Linux. Professional

## Задание

### Запустить nginx на нестандартном порту 3-мя разными способами

- переключатели setsebool;
- добавление нестандартного порта в имеющийся тип;
- формирование и установка модуля SELinux.

### Обеспечить работоспособность приложения при включенном selinux
- развернуть приложенный стенд https://github.com/mbfx/otus-linux-adm/tree/master/selinux_dns_problems;
- выяснить причину неработоспособности механизма обновления зоны (см. README);
- предложить решение (или решения) для данной проблемы;
- выбрать одно из решений для реализации, предварительно обосновав выбор;
- реализовать выбранное решение и продемонстрировать его работоспособность.

## Выполнение

### Запустить nginx на нестандартном порту 3-мя разными способами (./nginx)

- Проверить включен ли firewall:

```
#systemctl status firewalld

● firewalld.service - firewalld - dynamic firewall daemon
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; disabled; vendor preset: enabled)
   Active: inactive (dead)
     Docs: man:firewalld(1)
```

- Проверка валидности конфигурационного файла nginx

```
# nginx -t

nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

- Проверить режим SELinux

```
# getenforce

Enforcing
```

- Разрешим в SELinux работу nginx на порту TCP 4881 c помощью переключателей setsebool

```
# cat /var/log/audit/audit.log | grep 4881 | audit2why

type=AVC msg=audit(1706695749.338:851): avc:  denied  { name_bind } for  pid=2947 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0

        Was caused by:
        The boolean nis_enabled was set incorrectly.
        Description:
        Allow nis to enabled

        Allow access by executing:
        # setsebool -P nis_enabled 1
```

```
[root@selinux vagrant]# setsebool -P nis_enabled on
[root@selinux vagrant]# systemctl restart nginx
[root@selinux vagrant]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Wed 2024-01-31 10:50:42 UTC; 7s ago
  Process: 3234 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 3232 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 3231 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 3236 (nginx)
   CGroup: /system.slice/nginx.service
           ├─3236 nginx: master process /usr/sbin/nginx
           └─3238 nginx: worker process

Jan 31 10:50:42 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Jan 31 10:50:42 selinux nginx[3232]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Jan 31 10:50:42 selinux nginx[3232]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Jan 31 10:50:42 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
```

Проверить и вернуть как было

```
[root@selinux vagrant]# getsebool -a | grep nis_enabled
nis_enabled --> on
[root@selinux vagrant]# setsebool -P nis_enabled off
[root@selinux vagrant]# getsebool -a | grep nis_enabled
nis_enabled --> off
```

Убедиться, что nginx снова не запускается

```
[root@selinux vagrant]# systemctl restart nginx
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
[root@selinux vagrant]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since Wed 2024-01-31 10:54:20 UTC; 9s ago
  Process: 3234 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 3260 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=1/FAILURE)
  Process: 3259 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 3236 (code=exited, status=0/SUCCESS)

Jan 31 10:54:20 selinux systemd[1]: Stopped The nginx HTTP and reverse proxy server.
Jan 31 10:54:20 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Jan 31 10:54:20 selinux nginx[3260]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Jan 31 10:54:20 selinux nginx[3260]: nginx: [emerg] bind() to 0.0.0.0:4881 failed (13: Permission denied)
Jan 31 10:54:20 selinux nginx[3260]: nginx: configuration file /etc/nginx/nginx.conf test failed
Jan 31 10:54:20 selinux systemd[1]: nginx.service: control process exited, code=exited status=1
Jan 31 10:54:20 selinux systemd[1]: Failed to start The nginx HTTP and reverse proxy server.
Jan 31 10:54:20 selinux systemd[1]: Unit nginx.service entered failed state.
Jan 31 10:54:20 selinux systemd[1]: nginx.service failed.
```

- Разрешить в SELinux работу nginx на порту TCP 4881 c помощью добавления нестандартного порта в имеющийся тип:

Поиск имеющегося типа, для http трафика и добавление порта в него:

```
[root@selinux vagrant]# semanage port -l | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989
[root@selinux vagrant]# semanage port -a -t http_port_t -p tcp 4881
[root@selinux vagrant]# semanage port -l | grep  http_port_t
http_port_t                    tcp      4881, 80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
```

Проверка работоспособности nginx после внесения порта в разрешённые

```
[root@selinux vagrant]# systemctl restart nginx
[root@selinux vagrant]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Wed 2024-01-31 11:01:24 UTC; 2s ago
  Process: 3300 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 3298 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 3297 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 3302 (nginx)
   CGroup: /system.slice/nginx.service
           ├─3302 nginx: master process /usr/sbin/nginx
           └─3304 nginx: worker process

Jan 31 11:01:24 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Jan 31 11:01:24 selinux nginx[3298]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Jan 31 11:01:24 selinux nginx[3298]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Jan 31 11:01:24 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
```

Удалить внесенные изменения и проверить перезапуском nginx

```
[root@selinux vagrant]# semanage port -d -t http_port_t -p tcp 4881
[root@selinux vagrant]# semanage port -l | grep  http_port_t
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
[root@selinux vagrant]# systemctl restart nginx
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
[root@selinux vagrant]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since Wed 2024-01-31 11:04:14 UTC; 6s ago
  Process: 3300 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 3323 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=1/FAILURE)
  Process: 3322 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 3302 (code=exited, status=0/SUCCESS)

Jan 31 11:04:14 selinux systemd[1]: Stopped The nginx HTTP and reverse proxy server.
Jan 31 11:04:14 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Jan 31 11:04:14 selinux nginx[3323]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Jan 31 11:04:14 selinux nginx[3323]: nginx: [emerg] bind() to 0.0.0.0:4881 failed (13: Permission denied)
Jan 31 11:04:14 selinux nginx[3323]: nginx: configuration file /etc/nginx/nginx.conf test failed
Jan 31 11:04:14 selinux systemd[1]: nginx.service: control process exited, code=exited status=1
Jan 31 11:04:14 selinux systemd[1]: Failed to start The nginx HTTP and reverse proxy server.
Jan 31 11:04:14 selinux systemd[1]: Unit nginx.service entered failed state.
Jan 31 11:04:14 selinux systemd[1]: nginx.service failed.
```

- Разрешить в SELinux работу nginx на порту TCP 4881 c помощью формирования и установки модуля SELinux

Сформировать и применить модуль при помощи audit2allow

```
[root@selinux vagrant]# grep nginx /var/log/audit/audit.log | audit2allow -M nginx
******************** IMPORTANT ***********************
To make this policy package active, execute:

semodule -i nginx.pp

[root@selinux vagrant]# semodule -i nginx.pp
```

Проверка nginx

```
[root@selinux vagrant]# systemctl restart nginx
[root@selinux vagrant]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Wed 2024-01-31 11:13:38 UTC; 3s ago
  Process: 3353 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 3351 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 3350 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 3355 (nginx)
   CGroup: /system.slice/nginx.service
           ├─3355 nginx: master process /usr/sbin/nginx
           └─3357 nginx: worker process

Jan 31 11:13:38 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Jan 31 11:13:38 selinux nginx[3351]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Jan 31 11:13:38 selinux nginx[3351]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Jan 31 11:13:38 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
```

*После добавления модуля nginx запустился без ошибок. При использовании модуля изменения сохранятся после перезагрузки.*

Пример листинга и удаления модулей

```
[root@selinux vagrant]# semodule -l
[root@selinux vagrant]# semodule -r nginx
libsemanage.semanage_direct_remove_key: Removing last nginx module (no other nginx module exists at another priority).
```

### Обеспечение работоспособности приложения при включенном SELinux

- Развернуть 2 виртуальные машины (./stage2)

```
vagrant up

vagrant status
Current machine states:

ns01                      running (virtualbox)
client                    running (virtualbox)

This environment represents multiple VMs. The VMs are all listed
above with their current state. For more information about a specific
VM, run `vagrant status NAME`.
```

- На клиенте попытаться внести изменения в зону:

```
[vagrant@client ~]$ nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
update failed: SERVFAIL
> quit
```

- Убедимся, что на клиенте ошибок нет:

```
[vagrant@client ~]$ sudo -s
[root@client vagrant]# cat /var/log/audit/audit.log | audit2why
```

- Проверка на сервере

```
[vagrant@ns01 ~]$ sudo -s
[root@ns01 vagrant]# cat /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1706706000.896:1973): avc:  denied  { create } for  pid=5239 comm="isc-worker0000" name="named.ddns.lab.view1.jnl" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0

        Was caused by:
                Missing type enforcement (TE) allow rule.

                You can use audit2allow to generate a loadable module to allow this access.

```

Ошибка в контексте безопасности. Вместо типа named_t используется тип etc_t.

- Проверка атрибутов файлов из директории /etc/named

```
[root@ns01 vagrant]# ls -laZ /etc/named
drw-rwx---. root named system_u:object_r:etc_t:s0       .
drwxr-xr-x. root root  system_u:object_r:etc_t:s0       ..
drw-rwx---. root named unconfined_u:object_r:etc_t:s0   dynamic
-rw-rw----. root named system_u:object_r:etc_t:s0       named.50.168.192.rev
-rw-rw----. root named system_u:object_r:etc_t:s0       named.dns.lab
-rw-rw----. root named system_u:object_r:etc_t:s0       named.dns.lab.view1
-rw-rw----. root named system_u:object_r:etc_t:s0       named.newdns.lab
```

- Изменить и проверить контекст в атрибутах файлов

```
[root@ns01 vagrant]# chcon -R -t named_zone_t /etc/named
[root@ns01 vagrant]# ls -laZ /etc/named
drw-rwx---. root named system_u:object_r:named_zone_t:s0 .
drwxr-xr-x. root root  system_u:object_r:etc_t:s0       ..
drw-rwx---. root named unconfined_u:object_r:named_zone_t:s0 dynamic
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.50.168.192.rev
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab.view1
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.newdns.lab
```

- Попробуем снова внести изменения с клиента

```
[vagrant@client ~]$ nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
> quit

[vagrant@client ~]$ dig www.ddns.lab

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.15 <<>> www.ddns.lab
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 37178
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;www.ddns.lab.                  IN      A

;; ANSWER SECTION:
www.ddns.lab.           60      IN      A       192.168.50.15

;; AUTHORITY SECTION:
ddns.lab.               3600    IN      NS      ns01.dns.lab.

;; ADDITIONAL SECTION:
ns01.dns.lab.           3600    IN      A       192.168.50.10

;; Query time: 0 msec
;; SERVER: 192.168.50.10#53(192.168.50.10)
;; WHEN: Wed Jan 31 13:30:47 UTC 2024
;; MSG SIZE  rcvd: 96
```

Изменения применились.

Внести изменения в стенд. Добавить 2 задачи для хоста ns01 (./provisioning/playbook.yml):

```
  - name: Change SELinux context /etc/named
    community.general.sefcontext:
      target: '/etc/named(/.*)?'
      setype: named_zone_t
      state: present

  - name: Apply new SELinux file context to filesystem
    ansible.builtin.command: restorecon -r /etc/named
```

С внесёнными изменения стенд поднимается рабочим
