В этой статье рассмотрим сообщения eBGP на все том же простеньком примере из двух роутеров:

![image](https://github.com/user-attachments/assets/77de54de-7b9a-49bf-b560-3043bd4a2dc8)

Настройки с R1:

```bash
R1#sh run
!
interface Ethernet1
   description UP_P1
   no switchport
   ip address 10.0.0.0/31
   ip ospf area 0.0.0.0
   isis enable 100
!
interface Ethernet2
!
interface Ethernet3
!
interface Ethernet4
   no switchport
   ip address 192.168.10.1/30
   isis enable 100
!
interface Loopback0
   ip address 10.1.1.1/32
   isis enable 100
!
interface Management1
!
ip routing
!
ip route 192.168.30.0/24 Null0 - создан для наглядности передачи в сообщениях Update
!
mpls ip
!
router bgp 65000
   neighbor 10.0.0.1 remote-as 65001
   redistribute connected - передаем подключенные сети
   redistribute static - передаем статические маршруты
!
end
```

Для R2 настройки будут идентичными:

```bash
interface Ethernet1
   description Low_PE1
   no switchport
   ip address 10.0.0.1/31
   ip ospf area 0.0.0.0
   isis enable 100
!
router bgp 65001
   neighbor 10.0.0.0 remote-as 65000
```

