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

# Но почему мы не можем получить Vtep автоматически, как мы получили бы, используя Bgp+EVPN?

Роль ISIS заключается в нескольких вещах в нашем случае:
1. Обеспечитвает IP-маршрутизацию между всеми узлами в сети.
2. Передает информацию об IP-префиксах и топлогии сети.

Он не может передавать данные о MAC-адресах, Vlan, Vtep, VNI.
ISIS нужен, чтобы построить маршруты между всеми участниками сети. PE1 и PE2 знают, через какие интерфейсы и узлы найти друг друга, но понятия не имеют о Vtep, VNI.

В отличии от него существует BGP EVPN, чем он тут нам мог бы помочь для динамического изучения?
Тогда поговорим об его назначении:
1. Создает виртуальные сети поверх IP-инфраструктуры.
2. Динамечски распространяет MAC-адреса клиентов.
3. Автоматически обнаруживает VTEP.
4. Может использовать расширение MP-BGP для передачи данных о виртуальных сетях, например type 2 MAC-IP-какой MAC за каким VTEP живет. Или Type 3 IMET - обнаруживает соседние VTEP и настраивает VXLAN между ними.
5. Каждый VTEP анонсирует свои IP-адреса(Lo0) через BGP EVPN, чтобы остальные соседи могли знать, куда отправлять vxlan-трафик.

# Почему BGP EVPN может изучать VTEP динамически:

BGP EVPN — это протокол уровня приложений, который умеет передавать данные о виртуальных сетях.

Когда VTEP поднимается, он отправляет через BGP EVPN сообщение: «Я — VTEP с адресом 10.1.1.1 и поддерживаю VNI 1001». Другие VTEP получают эту информацию и автоматически настраивают туннели.

# Чем технически обусловлена эта невозможность передавать Vtep динамически в ISIS?
Формат сообщений:

IS-IS использует TLV-поля для передачи информации о префиксах и метриках, но в стандарте нет TLV для MAC-адресов или VTEP.

BGP EVPN добавляет новые семейства адресов (AFI/SAI) для L2VPN, что позволяет передавать данные о виртуальных сетях.

# Аналогия
Таким образом, можно представить, что:

IS-IS — это карта дорог (underlay), которая показывает, как добраться из точки A в B.

BGP EVPN — это навигатор (overlay), который знает не только дороги, но и адреса магазинов, заправок и того, что в них продается (MAC-адреса, VTEP).

