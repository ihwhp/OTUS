## Домашнее задание DNS
*box 'centos/7' version '2004.01'*

Полезные ссылки:  
https://bind9.readthedocs.io/ - Руководство по BIND9  
https://linux.die.net/man/8/named_selinux - man SELinux for named  
https://docs.ansible.com/ansible/2.9/modules/sefcontext_module.html#sefcontext-module  

### 1. Исправляем конфигурационные файлы

#### Изначально имеем:
```
● playbook.yml — это Ansible-playbook, в котором содержатся инструкции по
настройке нашего стенда
● client-motd — файл, содержимое которого будет появляться перед
пользователем, который подключился по SSH
● named.ddns.lab и named.dns.lab — файлы описания зон ddns.lab и dns.lab
соответсвенно
● master-named.conf и slave-named.conf — конфигурационные файлы, в которых
хранятся настройки DNS-сервера
● client-resolv.conf и servers-resolv.conf — файлы, в которых содержатся
IP-адреса DNS-серверов
```
#### В соответствии с методичкой:
+ Создадим шаблон из `servers-resolv.conf` на `servers-resolv.conf.j2` и внесем изменение в `playbook`  
+ Редактируем `master-named.conf` и `slave-named.conf` и внесем изменение в `playbook`  
+ Создаем новые зоны и вносим изменение в `playbook`  

### 2. ~~Split-DNS~~ Multiple views

Добавляем `acl` в `master-named.conf` и `slave-named.conf`:  
```
acl client { 192.168.50.15; };
acl client2 { 192.168.50.16; };
```
**А можно и не создавать `acl`, вместо этого указать `match-clients { *client_ip_or_subтet*; }` непоссредственно в секции `view`**  
**TSIG key можно не генерировать(из руководства BIND)**
```
 An optional TSIG key can also be specified with each address to cause the notify messages to be signed;  
 this can be useful when sending notifies to multiple views.  
 In place of explicit addresses, one or more named masters lists can be used.
```
В файлах `master-named.conf` и `slave-named.conf` внесем зоны в соответствующие секции `view`  
**Зона `any` должна всегда находиться в самом низу**

### 3. Проверяем  
  
#### Проверка с client:
```
[vagrant@client ~]$ dig @192.168.50.10 web1.dns.lab

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.10 <<>> @192.168.50.10 web1.dns.lab
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 17357
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 2, ADDITIONAL: 3

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;web1.dns.lab.			IN	A

;; ANSWER SECTION:
web1.dns.lab.		3600	IN	A	192.168.50.15

;; AUTHORITY SECTION:
dns.lab.		3600	IN	NS	ns01.dns.lab.
dns.lab.		3600	IN	NS	ns02.dns.lab.

;; ADDITIONAL SECTION:
ns01.dns.lab.		3600	IN	A	192.168.50.10
ns02.dns.lab.		3600	IN	A	192.168.50.11

;; Query time: 2 msec
;; SERVER: 192.168.50.10#53(192.168.50.10)
;; WHEN: Tue Dec 13 10:10:58 UTC 2022
;; MSG SIZE  rcvd: 127

[vagrant@client ~]$ dig @192.168.50.10 web2.dns.lab

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.10 <<>> @192.168.50.10 web2.dns.lab
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NXDOMAIN, id: 6386
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 0, AUTHORITY: 1, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;web2.dns.lab.			IN	A

;; AUTHORITY SECTION:
dns.lab.		600	IN	SOA	ns01.dns.lab. root.dns.lab. 2711201407 3600 600 86400 600

;; Query time: 1 msec
;; SERVER: 192.168.50.10#53(192.168.50.10)
;; WHEN: Tue Dec 13 10:11:03 UTC 2022
;; MSG SIZE  rcvd: 87
```

**Ответа на запроc web2 нет**

### Проверяем с client2:  
```
[vagrant@client2 ~]$ dig @192.168.50.10 web1.dns.lab

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.10 <<>> @192.168.50.10 web1.dns.lab
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 59394
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 2, ADDITIONAL: 3

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;web1.dns.lab.			IN	A

;; ANSWER SECTION:
web1.dns.lab.		3600	IN	A	192.168.50.15

;; AUTHORITY SECTION:
dns.lab.		3600	IN	NS	ns02.dns.lab.
dns.lab.		3600	IN	NS	ns01.dns.lab.

;; ADDITIONAL SECTION:
ns01.dns.lab.		3600	IN	A	192.168.50.10
ns02.dns.lab.		3600	IN	A	192.168.50.11

;; Query time: 1 msec
;; SERVER: 192.168.50.10#53(192.168.50.10)
;; WHEN: Tue Dec 13 11:33:08 UTC 2022
;; MSG SIZE  rcvd: 127

[vagrant@client2 ~]$ dig @192.168.50.10 web2.dns.lab

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.10 <<>> @192.168.50.10 web2.dns.lab
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 14621
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 2, ADDITIONAL: 3

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;web2.dns.lab.			IN	A

;; ANSWER SECTION:
web2.dns.lab.		3600	IN	A	192.168.50.16

;; AUTHORITY SECTION:
dns.lab.		3600	IN	NS	ns01.dns.lab.
dns.lab.		3600	IN	NS	ns02.dns.lab.

;; ADDITIONAL SECTION:
ns01.dns.lab.		3600	IN	A	192.168.50.10
ns02.dns.lab.		3600	IN	A	192.168.50.11

;; Query time: 1 msec
;; SERVER: 192.168.50.10#53(192.168.50.10)
;; WHEN: Tue Dec 13 11:33:11 UTC 2022
;; MSG SIZE  rcvd: 127
```
**Ответ есть на оба запроса**

### Проверим работу ns02:  
```
[vagrant@client2 ~]$ dig @192.168.50.11 web1.dns.lab

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.10 <<>> @192.168.50.11 web1.dns.lab
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NXDOMAIN, id: 34754
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 0, AUTHORITY: 1, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;web1.dns.lab.			IN	A

;; AUTHORITY SECTION:
dns.lab.		600	IN	SOA	ns01.dns.lab. root.dns.lab. 2711201407 3600 600 86400 600

;; Query time: 2 msec
;; SERVER: 192.168.50.11#53(192.168.50.11)
;; WHEN: Tue Dec 13 10:13:21 UTC 2022
;; MSG SIZE  rcvd: 87
```
**(!)Видим, что ns02 не отвечает**  
Проверим службу `named`:  
```
● named.service - Berkeley Internet Name Domain (DNS)
   Loaded: loaded (/usr/lib/systemd/system/named.service; enabled; vendor preset: disabled)
   Active: active (running) since Tue 2022-12-13 10:22:09 UTC; 1s ago
  Process: 3383 ExecStop=/bin/sh -c /usr/sbin/rndc stop > /dev/null 2>&1 || /bin/kill -TERM $MAINPID (code=exited, status=0/SUCCESS)
  Process: 3396 ExecStart=/usr/sbin/named -u named -c ${NAMEDCONF} $OPTIONS (code=exited, status=0/SUCCESS)
  Process: 3394 ExecStartPre=/bin/bash -c if [ ! "$DISABLE_ZONE_CHECKING" == "yes" ]; then /usr/sbin/named-checkconf -z "$NAMEDCONF"; else echo "Checking of zone files is disabled"; fi (code=exited, status=0/SUCCESS)
 Main PID: 3398 (named)
   CGroup: /system.slice/named.service
           └─3398 /usr/sbin/named -u named -c /etc/named.conf

Dec 13 10:22:09 ns02 named[3398]: transfer of 'newdns.lab/IN/default' from 192.168.50.10#53: Transfer completed: 1 messages,...tes/sec)
Dec 13 10:22:09 ns02 named[3398]: zone newdns.lab/IN/default: sending notifies (serial 2711201408)
Dec 13 10:22:09 ns02 named[3398]: dumping master file: /etc/named/tmp-IAAq1ndObW: open: permission denied
Dec 13 10:22:09 ns02 named[3398]: zone 50.168.192.in-addr.arpa/IN/default: transferred serial 2711201407: TSIG 'zonetransfer.key'
Dec 13 10:22:09 ns02 named[3398]: transfer of '50.168.192.in-addr.arpa/IN/default' from 192.168.50.10#53: Transfer status: success
Dec 13 10:22:09 ns02 named[3398]: transfer of '50.168.192.in-addr.arpa/IN/default' from 192.168.50.10#53: Transfer completed...tes/sec)
Dec 13 10:22:09 ns02 named[3398]: zone 50.168.192.in-addr.arpa/IN/default: sending notifies (serial 2711201407)
Dec 13 10:22:09 ns02 named[3398]: dumping master file: /etc/named/tmp-kUjDv2ERy3: open: permission denied
Dec 13 10:22:10 ns02 named[3398]: client @0x7f70f403c3e0 192.168.50.11#44021: view default: received notify for zone 'dns.lab'
Dec 13 10:22:10 ns02 named[3398]: zone dns.lab/IN/default: refused notify from non-master: 192.168.50.11#44021
```
Убеждаемся, что зоны приходят, **но**  прочитать их служба не может  
Проверим *audit.log* на наличие событий:  
```
[root@ns02 audit]# cat audit.log | grep kUjDv2ERy3
type=AVC msg=audit(1670926929.679:1158): avc:  denied  { create } for  pid=3398 comm="isc-worker0000" name="tmp-kUjDv2ERy3" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0
type=AVC msg=audit(1670926929.679:1158): avc:  denied  { create } for  pid=3398 comm="isc-worker0000" name="tmp-kUjDv2ERy3" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0
```
Видим, что SELinux запрещает службе доступ к файлам  
Исправим командой `chcon -Rv --type=named_zone_t /etc/named`, перезапустим службу и убедимся, что проблема решена: 
```
● named.service - Berkeley Internet Name Domain (DNS)
   Loaded: loaded (/usr/lib/systemd/system/named.service; enabled; vendor preset: disabled)
   Active: active (running) since Tue 2022-12-13 10:25:01 UTC; 1s ago
  Process: 3419 ExecStop=/bin/sh -c /usr/sbin/rndc stop > /dev/null 2>&1 || /bin/kill -TERM $MAINPID (code=exited, status=0/SUCCESS)
  Process: 3432 ExecStart=/usr/sbin/named -u named -c ${NAMEDCONF} $OPTIONS (code=exited, status=0/SUCCESS)
  Process: 3430 ExecStartPre=/bin/bash -c if [ ! "$DISABLE_ZONE_CHECKING" == "yes" ]; then /usr/sbin/named-checkconf -z "$NAMEDCONF"; else echo "Checking of zone files is disabled"; fi (code=exited, status=0/SUCCESS)
 Main PID: 3434 (named)
   CGroup: /system.slice/named.service
           └─3434 /usr/sbin/named -u named -c /etc/named.conf

Dec 13 10:25:02 ns02 named[3434]: zone ddns.lab/IN/default: transferred serial 2711201407: TSIG 'zonetransfer.key'
Dec 13 10:25:02 ns02 named[3434]: transfer of 'ddns.lab/IN/default' from 192.168.50.10#53: Transfer status: success
Dec 13 10:25:02 ns02 named[3434]: transfer of 'ddns.lab/IN/default' from 192.168.50.10#53: Transfer completed: 1 messages, 6...tes/sec)
Dec 13 10:25:02 ns02 named[3434]: zone newdns.lab/IN/default: sending notifies (serial 2711201408)
Dec 13 10:25:02 ns02 named[3434]: zone ddns.lab/IN/default: sending notifies (serial 2711201407)
Dec 13 10:25:02 ns02 named[3434]: transfer of 'dns.lab/IN/default' from 192.168.50.10#53: connected using 192.168.50.11#4430...sfer.key
Dec 13 10:25:02 ns02 named[3434]: zone dns.lab/IN/default: transferred serial 2711201410: TSIG 'zonetransfer.key'
Dec 13 10:25:02 ns02 named[3434]: transfer of 'dns.lab/IN/default' from 192.168.50.10#53: Transfer status: success
Dec 13 10:25:02 ns02 named[3434]: transfer of 'dns.lab/IN/default' from 192.168.50.10#53: Transfer completed: 1 messages, 8 ...tes/sec)
Dec 13 10:25:02 ns02 named[3434]: zone dns.lab/IN/default: sending notifies (serial 2711201410)
```
```
[vagrant@client2 ~]$ dig @192.168.50.11 web2.dns.lab

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.10 <<>> @192.168.50.11 web2.dns.lab
; (1 server found)
;; global options: +cmd
;; connection timed out; no servers could be reached
[vagrant@client2 ~]$ dig @192.168.50.11 web2.dns.lab

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.10 <<>> @192.168.50.11 web2.dns.lab
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 34907
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 2, ADDITIONAL: 3

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;web2.dns.lab.			IN	A

;; ANSWER SECTION:
web2.dns.lab.		3600	IN	A	192.168.50.16

;; AUTHORITY SECTION:
dns.lab.		3600	IN	NS	ns01.dns.lab.
dns.lab.		3600	IN	NS	ns02.dns.lab.

;; ADDITIONAL SECTION:
ns01.dns.lab.		3600	IN	A	192.168.50.10
ns02.dns.lab.		3600	IN	A	192.168.50.11

;; Query time: 1 msec
;; SERVER: 192.168.50.11#53(192.168.50.11)
;; WHEN: Tue Dec 13 10:44:45 UTC 2022
;; MSG SIZE  rcvd: 127
```
Добавим в *provisioning.yml*:  
```
  - name: selinux /etc/named/
    sefcontext:
      target: '/etc/named(/.*)?'
      setype: named_zone_t
      state: present
      reload: yes
    register: selinux
    
  - name: apply  file context 
    command: restorecon -Rv /etc/named
    when: selinux.changed
```