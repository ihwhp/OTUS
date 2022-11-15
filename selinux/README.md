## 1 Часть
    
После запуска vm через vagrant up nginx не стартует:
```
[vagrant@selinux ~]$ systemctl status nginx.service 
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since Thu 2022-10-27 11:08:25 UTC; 2min 8s ago
  Process: 2838 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=1/FAILURE)
  Process: 2837 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)

```
Проверим SELinux:
```
[root@selinux ~]# getenforce 
Enforcing
```
Проверим audit.log на наличие событий по порту 4881:
```
[root@selinux ~]# grep 4881 /var/log/audit/audit.log 
type=AVC msg=audit(1666868905.614:814): avc:  denied  { name_bind } for  pid=2838 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
```
Установим необходимые утилиты:
```
[root@selinux ~]# yum install policycoreutils-python setroubleshoot
```
Проанализируем audit.log:
```
[root@selinux ~]# grep 1666868905.614:814 /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1666868905.614:814): avc:  denied  { name_bind } for  pid=2838 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0

	Was caused by:
	The boolean nis_enabled was set incorrectly. 
	Description:
	Allow nis to enabled

	Allow access by executing:
	# setsebool -P nis_enabled 1
```
Выполним рекомендуемое действие и проверим nginx:
```[root@selinux ~]# setsebool -P nis_enabled on
[root@selinux ~]# systemctl restart nginx
[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Thu 2022-10-27 11:20:49 UTC; 6s ago
  Process: 3326 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 3323 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 3322 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 3328 (nginx)
   CGroup: /system.slice/nginx.service
           ├─3328 nginx: master process /usr/sbin/nginx
           └─3330 nginx: worker process

Oct 27 11:20:49 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Oct 27 11:20:49 selinux nginx[3323]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Oct 27 11:20:49 selinux nginx[3323]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Oct 27 11:20:49 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
[root@selinux ~]# ss -tlnp | grep nginx
LISTEN     0      128          *:80                       *:*                   users:(("nginx",pid=3330,fd=6),("nginx",pid=3328,fd=6))
LISTEN     0      128       [::]:4881                  [::]:*                   users:(("nginx",pid=3330,fd=7),("nginx",pid=3328,fd=7))
```
    
Вернем запрет _setsebool_ _-P_ _nis_enabled_ _off_

Разрешим работу nginx на нестандартном порту:
```
[root@selinux ~]# semanage port -l | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989
```
```
[root@selinux ~]# semanage port -a -t http_port_t -p tcp 4881
```
```
[root@selinux ~]# semanage port -l | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      4881, 80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989
```
Удалим нестандартный порт и перезапустим nginx:
```
[root@selinux ~]# semanage port -d -t http_port_t -p tcp 4881
[root@selinux ~]# systemctl restart nginx
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
```
Воспользуемся утилитой audit2allow для того, что бы сделать модуль:
```
[root@selinux ~]# grep nginx /var/log/audit/audit.log | audit2allow -M nginx
******************** IMPORTANT ***********************
To make this policy package active, execute:

semodule -i nginx.pp

[root@selinux ~]# semodule -i nginx.pp 
[root@selinux ~]# systemctl restart nginx
[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Thu 2022-10-27 11:38:00 UTC; 6s ago
  Process: 3446 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 3444 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 3443 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 3448 (nginx)
   CGroup: /system.slice/nginx.service
           ├─3448 nginx: master process /usr/sbin/nginx
           └─3450 nginx: worker process
```
### Выводы по 1 части:
Мы запустили nginx на нестандартном порту 3мя разными спосабами
* [x] setsebool
* [x] semanage
* [x] semodule

## 2 Часть
Проверим права named на смену зон:
```
[root@ns01 ~]# getsebool -a | grep named
named_tcp_bind_http_port --> off
named_write_master_zones --> on
```
Проанализируем audit.log:
```
[root@ns01 ~]# cat /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1666871901.448:1901): avc:  denied  { create } for  pid=5103 comm="isc-worker0000" name="named.ddns.lab.view1.jnl" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0
	Was caused by:
		Missing type enforcement (TE) allow rule.
		You can use audit2allow to generate a loadable module to allow this access.
```
Изменим тип контекста безопасности для каталога /etc/named:
```
sudo chcon -R -t named_zone_t /etc/named
```
Перезагрузимся и проверим:
```
[vagrant@client ~]$ nsupdate -k /etc/named.zonetransfer.key
> quit    
[vagrant@client ~]$ nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab       
> update add www.ddns.lab. 60 A 192.168.50.15
> sent
incorrect section name: sent
> send
> quit
[vagrant@client ~]$ dig www.ddns.lab

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.10 <<>> www.ddns.lab
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NXDOMAIN, id: 631
;; flags: qr rd ra ad; QUERY: 1, ANSWER: 0, AUTHORITY: 1, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 512
;; QUESTION SECTION:
;www.ddns.lab.			IN	A

;; AUTHORITY SECTION:
.			86399	IN	SOA	a.root-servers.net. nstld.verisign-grs.com. 2022102700 1800 900 604800 86400

;; Query time: 26 msec
;; SERVER: 10.0.2.3#53(10.0.2.3)
;; WHEN: Thu Oct 27 12:19:53 UTC 2022
;; MSG SIZE  rcvd: 116
```

