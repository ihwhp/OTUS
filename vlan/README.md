## ДЗ по теме VLAN, LACP
*box 'almalinux/8' version '8.7.20221112'*  

```
The legacy scripts are deprecated in RHEL 8 and will be removed in a future major version of RHEL.  
If you still use the legacy network scripts, for example, because you upgraded from an earlier version to RHEL 8,  
Red Hat recommends that you migrate your configuration to NetworkManager. 
```
https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/configuring_and_managing_networking/assembly_legacy-network-scripts-support-in-rhel_configuring-and-managing-networking  
  
### 1.VLAN
  
**На клиентах:**  
Сконфигурировано через network-scripts. Установлен пакет `dnf install network-scripts`.  
**На серверах:**  
Сконфигурировано через NetworkManager.   
  
![tcpdump_vlan](https://user-images.githubusercontent.com/105001717/207634188-cae5e215-f6e0-4c44-b6c1-50d258e7a654.png)  

Не смотря на одинаковые подсети, хосты изолированы друг от друга; icmp-пакеты прилетают с тагом vlan

### 2.Bond  
  
Bond сконфигурирован через network-scripts  
Посмотреть статус: `cat /proc/net/bonding/bond0`  
  
![bond-up-down](https://user-images.githubusercontent.com/105001717/207634296-4a586730-0714-4f3a-94ed-b3ce8e72289b.png)  

При выключении физического интерфейса icmp-пакеты не теряются.

