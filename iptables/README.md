## ДЗ сценарии iptables

### Реализовать knocking port
Импорт правил **iptables** через файл `iptables.rules` и выполнения `iptables-restore`
```
*nat
:PREROUTING ACCEPT [0:0]
:INPUT ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
:POSTROUTING ACCEPT [0:0]
-A POSTROUTING ! -d 192.168.0.0/24 -o eth0 -j MASQUERADE
COMMIT
*filter
:INPUT DROP [0:0]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
:SSH-INPUT - [0:0]
:SSH-INPUTTWO - [0:0]
:TRAFFIC - [0:0]
-A INPUT -m conntrack --ctstate INVALID -j DROP
-A INPUT -i lo -j ACCEPT
-A INPUT -p tcp -m tcp -s 10.0.0.0/8 --dport 22 -m conntrack --ctstate NEW,UNTRACKED -j ACCEPT # Нужно для доступа к ВМ через vagrant ssh
-A INPUT -j TRAFFIC
-A INPUT -j REJECT --reject-with icmp-host-prohibited
-A SSH-INPUT -m recent --set --name SSH1 --mask 255.255.255.255 --rsource -j DROP
-A SSH-INPUTTWO -m recent --set --name SSH2 --mask 255.255.255.255 --rsource -j DROP
-A TRAFFIC -p icmp -m icmp --icmp-type any -j ACCEPT
-A TRAFFIC -m state --state RELATED,ESTABLISHED -j ACCEPT
-A TRAFFIC -p tcp -m state --state NEW -m tcp --dport 22 -m recent --rcheck --seconds 30 --name SSH2 --mask 255.255.255.255 --rsource -j ACCEPT
-A TRAFFIC -p tcp -m state --state NEW -m tcp -m recent --remove --name SSH2 --mask 255.255.255.255 --rsource -j DROP
-A TRAFFIC -p tcp -m state --state NEW -m tcp --dport 9991 -m recent --rcheck --name SSH1 --mask 255.255.255.255 --rsource -j SSH-INPUTTWO
-A TRAFFIC -p tcp -m state --state NEW -m tcp -m recent --remove --name SSH1 --mask 255.255.255.255 --rsource -j DROP
-A TRAFFIC -p tcp -m state --state NEW -m tcp --dport 7777 -m recent --rcheck --name SSH0 --mask 255.255.255.255 --rsource -j SSH-INPUT
-A TRAFFIC -p tcp -m state --state NEW -m tcp -m recent --remove --name SSH0 --mask 255.255.255.255 --rsource -j DROP
-A TRAFFIC -p tcp -m state --state NEW -m tcp --dport 8881 -m recent --set --name SSH0 --mask 255.255.255.255 --rsource -j DROP
-A TRAFFIC -j DROP
COMMIT
```

На **centralRouter** копируется файл `knock.sh` в директорию `/root`  
Для подключения по ssh надо перейти в root(sudo -i) и выполнить:  
```
./knock.sh 192.168.255.11 8881 7777 9991 
```
затем подключится по `ssh`:
```
ssh vagrant@192.168.255.11
```
![iptables_knock](https://user-images.githubusercontent.com/105001717/203731186-108c7be0-4bd9-4198-b7bf-9f9bca5508db.png)

  
## Проброс порта
Порт с локальной машины пробрасываем в Vagrantfile  
```
box.vm.network "forwarded_port", guest: 8080, host: 8888, host_ip: "127.0.0.1", id: "nginx"
```
по адресу 127.0.0.1:8888 мы должны попасть на 80й порт **centralServer**(`nginx`)  
  
**iptables** настрайвается через ansible:
```
    - name: iptables |
      iptables:
        table: nat
        chain: PREROUTING
        in_interface: eth0
        protocol: tcp
        match: tcp
        destination_port: 8080
        jump: DNAT
        to_destination: 192.168.0.2:80
      become: yes

    - name: iptables |
      iptables:
        table: nat
        chain: POSTROUTING
        destination: 192.168.0.2/32
        jump: SNAT
        to_source: 192.168.255.12
      become: yes
```
**Вывод `curl` с хостовой машины**:
![curl](https://user-images.githubusercontent.com/105001717/203731234-658c288e-44c1-4b62-b77c-82fb9998cb38.png)
