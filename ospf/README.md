## Домашнее задание по теме ospf
+ Схема сети взята из методички  
+ Стенд сделан на образе almalinux8

Сделан Vagrantfile и ansible плейбуки, раворачиваюшие машины-роутеры r1,r2 и r3.  
За роутерами есть приватные подсети 192.168.10.1/24, 192.168.20.1/24 и 192.168.30.1/24,  
которые надо объединить с помощью ospf. Для этого соединяем роутеры сетями и настраиваем FRRoutingи  
  
Проверка соседства:  
```
r1# sho ip os n
Neighbor ID     Pri State           Dead Time Address         Interface                        RXmtL RqstL DBsmL
2.2.2.2           1 Full/Backup       28.116s 10.0.10.2       enp0s8:10.0.10.1                     0     0     0
3.3.3.3           1 Full/Backup       27.316s 10.0.12.2       enp0s9:10.0.12.1                     0     0     0

r2# sho ip ospf neighbor 
Neighbor ID     Pri State           Dead Time Address         Interface                        RXmtL RqstL DBsmL
1.1.1.1           1 Full/DR           23.824s 10.0.10.1       enp0s8:10.0.10.2                     0     0     0
3.3.3.3           1 Full/Backup       26.644s 10.0.11.1       enp0s9:10.0.11.2                     0     0     0

router3# sho ip ospf neighbor 
Neighbor ID     Pri State           Dead Time Address         Interface                        RXmtL RqstL DBsmL
2.2.2.2           1 Full/DR           23.759s 10.0.11.2       enp0s8:10.0.11.1                     0     0     0
1.1.1.1           1 Full/DR           21.811s 10.0.12.1       enp0s9:10.0.12.2                     0     0     0
```

icmp проверка с r3:  
```
[root@router3 ~]# ping 192.168.10.1 -c 1
PING 192.168.10.1 (192.168.10.1) 56(84) bytes of data.
64 bytes from 192.168.10.1: icmp_seq=1 ttl=64 time=0.639 ms
--- 192.168.10.1 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.639/0.639/0.639/0.000 ms
[root@router3 ~]# ping 192.168.20.1 -c 1
PING 192.168.20.1 (192.168.20.1) 56(84) bytes of data.
64 bytes from 192.168.20.1: icmp_seq=1 ttl=64 time=0.724 ms
--- 192.168.20.1 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.724/0.724/0.724/0.000 ms
[root@router3 ~]# ping 10.0.12.2 -c 1
PING 10.0.12.2 (10.0.12.2) 56(84) bytes of data.
64 bytes from 10.0.12.2: icmp_seq=1 ttl=64 time=0.027 ms
--- 10.0.12.2 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.027/0.027/0.027/0.000 ms
```
## Ассиметричный routing  
  
Проверяем маршруты на r1:  
```
r1# sho ip route ospf 
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, D - SHARP,
       F - PBR, f - OpenFabric,
       > - selected route, * - FIB route, q - queued, r - rejected, b - backup

O   10.0.10.0/30 [110/100] is directly connected, enp0s8, weight 1, 00:00:09
O>* 10.0.11.0/30 [110/200] via 10.0.10.2, enp0s8, weight 1, 00:00:09
  *                        via 10.0.12.2, enp0s9, weight 1, 00:00:09
O   10.0.12.0/30 [110/100] is directly connected, enp0s9, weight 1, 03:21:14
O   192.168.10.0/24 [110/100] is directly connected, enp0s10, weight 1, 03:20:56
O>* 192.168.20.0/24 [110/200] via 10.0.10.2, enp0s8, weight 1, 00:00:09
O>* 192.168.30.0/24 [110/200] via 10.0.12.2, enp0s9, weight 1, 00:25:37
```
Изменяем _стоимость_ интерфейса и проверяем снова:  
```
r1(config)# interface enp0s8
r1(config-if)# ip ospf cost 1000
r1(config-if)# exit
r1(config)# exit
r1# sho ip route ospf 
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, D - SHARP,
       F - PBR, f - OpenFabric,
       > - selected route, * - FIB route, q - queued, r - rejected, b - backup

O   10.0.10.0/30 [110/300] via 10.0.12.2, enp0s9, weight 1, 00:00:15
O>* 10.0.11.0/30 [110/200] via 10.0.12.2, enp0s9, weight 1, 00:00:15
O   10.0.12.0/30 [110/100] is directly connected, enp0s9, weight 1, 03:18:55
O   192.168.10.0/24 [110/100] is directly connected, enp0s10, weight 1, 03:18:37
O>* 192.168.20.0/24 [110/300] via 10.0.12.2, enp0s9, weight 1, 00:00:15
O>* 192.168.30.0/24 [110/200] via 10.0.12.2, enp0s9, weight 1, 00:23:18
```
Видим изменения маршрута __O>* 192.168.20.0/24 [110/300] via 10.0.12.2, enp0s9, weight 1, 00:00:15__  

***В ansible-provision можно добавит cost для интерфеса в переменную enp0s_cost***  

## Факультативно настроим BFD для OSPF между r1 и r2  

Включим bfdd в /etc/frr/daemons  

Добавим на интерфейсы `ip ospf bfd`  
и пропишем соответствующие peer:  
```
r1
bfd
 peer 10.0.10.2
  transmit-interval 150
  receive-interval 150
  no shutdown

r2
bfd
 peer 10.0.10.1
  transmit-interval 150
  receive-interval 150
  no shutdown
```
  
Проверим работу bfd:
```
r2# sho bfd peers
BFD Peers:
	peer 10.0.10.1 vrf default
		ID: 3691661061
		Remote ID: 575929340
		Active mode
		Status: up
		Uptime: 6 minute(s), 43 second(s)
		Diagnostics: ok
		Remote diagnostics: ok
		Peer Type: configured
		Local timers:
			Detect-multiplier: 3
			Receive interval: 300ms
			Transmission interval: 300ms
			Echo transmission interval: 50ms
		Remote timers:
			Detect-multiplier: 3
			Receive interval: 150ms
			Transmission interval: 150ms
			Echo transmission interval: 50ms
```
  
При отключении соответствующего интерфеса увиди моментальную перестройку ospf до истечения таймеров:  
```
Neighbor ID     Pri State           Dead Time Address         Interface                        RXmtL RqstL DBsmL
1.1.1.1           1 Full/Backup       26.677s 10.0.10.1       enp0s8:10.0.10.2                     0     0     0
3.3.3.3           1 Full/DR           21.847s 10.0.11.1       enp0s9:10.0.11.2                     0     0     0

r2# sho ip ospf neighbor 

Neighbor ID     Pri State           Dead Time Address         Interface                        RXmtL RqstL DBsmL
3.3.3.3           1 Full/DR           28.812s 10.0.11.1       enp0s9:10.0.11.2                     2     0     0

r2# sho ip ospf neighbor 

Neighbor ID     Pri State           Dead Time Address         Interface                        RXmtL RqstL DBsmL
3.3.3.3           1 Full/DR           28.710s 10.0.11.1       enp0s9:10.0.11.2                     0     0     0

r2# sho ip ospf neighbor 

Neighbor ID     Pri State           Dead Time Address         Interface                        RXmtL RqstL DBsmL
1.1.1.1           1 Init/DROther      28.655s 10.0.10.1       enp0s8:10.0.10.2                     0     0     0
3.3.3.3           1 Full/DR           28.340s 10.0.11.1       enp0s9:10.0.11.2                     0     0     0

```

Внесем в ansible-provisioning и templates соответствующие дополнения
```
 {% if r_hostname == 'r1' %}
 ip ospf bfd
 {% else r_hostaname == 'r2'%}
 ip ospf bfd
 {% else %}
 !ip ospf bfd
 {% endif %}
 ```
 и bfdd.conf.j2