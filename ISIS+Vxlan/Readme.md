# Основополагающее знание.
Уж очень мне хотелось поднять лабу на Veos для L2VPN с LDP и xconnect'ами...но не тут-то было.

Но и то, что вышло, не лишено для меня смысла. 
Итак, что мы имеем?

# Схема следующая :
![image](https://github.com/user-attachments/assets/a3a34d7e-d4d1-406e-a28e-b89c2b010855)

![image](https://github.com/user-attachments/assets/bafbc6d8-b9da-4683-9813-ef4d0238926b)

# Цель этой работы:
1. Использовать ISIS для underlay-связности между Lo0 наших PE1-PE2. 
2. Обеспечить связность между PC1-PC2, прибегнув к связности на транспорте на уровне L2.
3. Проверить саму связность и посмотреть на выводы всех необходимых таблиц.

# Конфигурации.

Для начала настроим наши PE1/PE2:

```bash
interface Ethernet1
   description UP_P1
   no switchport
   ip address 10.0.0.0/31
   isis enable 100
!
interface Ethernet2
!
interface Ethernet3
!
interface Ethernet4
!
interface Ethernet5
!
interface Ethernet6
!
interface Ethernet7
!
interface Ethernet8
   description To_Client_1
   switchport access vlan 100
!
interface Loopback0
   ip address 10.1.1.1/32
   isis enable 100
!
interface Management1
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 100 vni 1001
   vxlan flood vtep 10.1.1.4
!
ip routing
!
mpls ip
!
router isis 100
   net 49.0001.0000.0000.0001.00
   is-type level-1
   !
   address-family ipv4 unicast
!
end
PE1#
```

Настройка для PE2 будет аналогичная, конечно же, за исключением адресации. 

И настройка для P1/2:
```bash
interface Ethernet1
   description Low_PE1
   no switchport
   ip address 10.0.0.1/31
   isis enable 100
!
interface Ethernet2
   description TO_P2
   no switchport
   ip address 10.0.0.2/31
   isis enable 100
!
interface Ethernet3
!
interface Ethernet4
!
interface Ethernet5
!
interface Ethernet6
!
interface Ethernet7
!
interface Ethernet8
!
interface Loopback0
   ip address 10.1.1.2/32
   isis enable 100
!
interface Management1
!
ip routing
!
mpls ip
!
router isis 100
   net 49.0001.0000.0000.0002.00
   !
   address-family ipv4 unicast
```
С P2 ситуация аналогичная.

#  Умозаключения.

Рассмотрим поближе эти примитивные конфигуарции, но важные для понимания отличия работы isis+vxlan / bgp+evpn+vxlan.

Так как ISIS предоставляет нам связность на underlay уровне, все наши узлы знают друг о друге:

# Routing Table Output

**Gateway of last resort is not set**

| Type | Network      | Metric    | Next Hop   | Interface  |
|------|--------------|-----------|------------|------------|
| C    | 10.0.0.0/31  | Direct    | N/A        | Ethernet1  |
| C    | 10.0.0.2/31  | Direct    | N/A        | Ethernet2  |
| I L1 | 10.0.0.4/31  | [115/20]  | 10.0.0.3   | Ethernet2  |
| I L1 | 10.1.1.1/32  | [115/20]  | 10.0.0.0   | Ethernet1  |
| C    | 10.1.1.2/32  | Direct    | N/A        | Loopback0  |
| I L1 | 10.1.1.3/32  | [115/20]  | 10.0.0.3   | Ethernet2  |
| I L1 | 10.1.1.4/32  | [115/30]  | 10.0.0.3   | Ethernet2  |

Поэтому мы можем поднять vxlan между Lo0 Pe1 и Lo0 Pe2.

Вывод знания о PE2 с PE1:

| Protocol | Description          | Destination    | Metric  | Next Hop  | Interface  |
|----------|----------------------|----------------|---------|-----------|------------|
| I L1     | IS-IS level 1        | 10.1.1.4/32    | [115/40]| 10.0.0.1  | Ethernet1  |

Поднимем Vxlan 1 для overlay-связности
```bash
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 100 vni 1001
   vxlan flood vtep 10.1.1.4
```
Команда  "vxlan flood vtep 10.1.1.4" - это статическое указание соседского Vtep.

 PE1#sh vxlan address-table


| VLAN | Mac Address   | Type    | Prt | VTEP     | Moves | Last Move  |
|------|---------------|---------|-----|----------|-------|------------|
| 100  | 0050.7966.6806| DYNAMIC | Vx1 | 10.1.1.4 | 1     | 0:00:45 ago|

**Total Remote Mac Addresses for this criterion: 1**

 PE1#sh vxlan vtep


| VTEP     | Tunnel Type(s)   |
|----------|-------------------|
| 10.1.1.4 | unicast, flood    |

**Total number of remote VTEPS: 1**
