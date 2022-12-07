## OpenVPN tap/tun
***box 'almalinux/8' version '8.7.20221112'***  
**Режим работы tun/tap изменяется в конфигурационных файлах сервера и клиента изменятся только в директиве `dev`**
  
Канал работает:  
```
[root@server server]# ping 10.10.10.2 -c 2
PING 10.10.10.2 (10.10.10.2) 56(84) bytes of data.
64 bytes from 10.10.10.2: icmp_seq=1 ttl=64 time=2.02 ms
64 bytes from 10.10.10.2: icmp_seq=2 ttl=64 time=1.37 ms

--- 10.10.10.2 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 1.374/1.699/2.024/0.325 ms


[root@client ~]# ping 10.10.10.1 -c 2
PING 10.10.10.1 (10.10.10.1) 56(84) bytes of data.
64 bytes from 10.10.10.1: icmp_seq=1 ttl=64 time=1.90 ms
64 bytes from 10.10.10.1: icmp_seq=2 ttl=64 time=1.47 ms
--- 10.10.10.1 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1002ms
rtt min/avg/max/mdev = 1.469/1.685/1.902/0.220 ms

```

### iperf в режиме tap:  
```
[root@client ~]# iperf3 -c 10.10.10.1 -t 60 -i 20
Connecting to host 10.10.10.1, port 5201
[  5] local 10.10.10.2 port 42354 connected to 10.10.10.1 port 5201
[ ID] Interval           Transfer     Bitrate         Retr  Cwnd
[  5]   0.00-20.00  sec  80.2 MBytes  33.6 Mbits/sec   36    107 KBytes       
[  5]  20.00-40.00  sec  94.3 MBytes  39.5 Mbits/sec   28    110 KBytes       
[  5]  40.00-60.00  sec  85.6 MBytes  35.9 Mbits/sec   30   80.3 KBytes       
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-60.00  sec   260 MBytes  36.4 Mbits/sec   94             sender
[  5]   0.00-60.04  sec   260 MBytes  36.3 Mbits/sec                  receiver

iperf Done.
```
  
### iperf в режиме tup:  
```
[root@client ~]# iperf3 -c 10.10.10.1 -t 60 -i 20
Connecting to host 10.10.10.1, port 5201
[  5] local 10.10.10.2 port 38332 connected to 10.10.10.1 port 5201
[ ID] Interval           Transfer     Bitrate         Retr  Cwnd
[  5]   0.00-20.03  sec  92.5 MBytes  38.8 Mbits/sec   31    110 KBytes       
[  5]  20.03-40.02  sec  98.9 MBytes  41.5 Mbits/sec   30   87.5 KBytes       
[  5]  40.02-60.02  sec  86.8 MBytes  36.4 Mbits/sec   24   91.5 KBytes       
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-60.02  sec   278 MBytes  38.9 Mbits/sec   85             sender
[  5]   0.00-60.16  sec   278 MBytes  38.8 Mbits/sec                  receiver

iperf Done.
```
## Выводы о работе tun/tap:  
  ###  tap:
```
Работает на L2. 
```
  ###  tun:
```
Работает на L3. 
```
**tun показывает большую производительность относительно tap на том же стенде**
