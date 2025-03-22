Рассмотрим установления соседства на примере двух роутеров:

![image](https://github.com/user-attachments/assets/cb620f1e-673c-4bda-9b23-e91a5830e9b0)

Дамп будем снимать с int eth1 на R1, его мак:

![image](https://github.com/user-attachments/assets/0eb6da02-6d12-4e9b-b3fc-b4a3a1295a73)

Мак соседнего R2:

![image](https://github.com/user-attachments/assets/b6c22128-c1ba-4de0-a67e-8e1fdea94bb9)

Опишем сразу локальные настройки, на R1:

```bash
interface Ethernet1
   description UP_P1
   no switchport
   ip address 10.0.0.0/31
   isis enable 100 - Активация ISIS на интерфейсе, чтобы задействовать его в обмене маршрутной информацией.
!
interface Ethernet2
!
interface Ethernet3
!
interface Ethernet4
   no switchport
   ip address 192.168.10.1/30 - настроен L3-интерфейс просто для наглядности передачи LSP в дампе.
   isis enable 100 - Пусть для этого интерфейса нет соседа по линку, но мы включаем на нем ISIS для возможности передать маршрут до 192.168.10.1/30.Вторым способом могло быть объявление сети через network в router ISIS.
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
ip routing
!
ip route 192.168.30.0/24 Null0 - Создан статический маршрут с аналогичной целью для дампа.
!
mpls ip
!
!
router isis 100
   net 49.0001.0000.0000.0001.00
   is-type level-1
   redistribute connected 
   redistribute static - передаем Static-маршруты
   !
   address-family ipv4 unicast
!
end
```

Итак, включаем ISIS, затем встает на eth1 у R1:

И смотрим на отправленный нами Hello:

![image](https://github.com/user-attachments/assets/5cd768bd-440d-4291-8321-a09c2cb95b12)

Получаем первый Hello от R2:

![image](https://github.com/user-attachments/assets/79c28aa6-9958-403a-810d-54acf1d02ec7)

Пока что они не видят в качестве соседей друг друга, поэтому соседство не установлено.

Далее R2, получив Hello от R1, добавит в качестве neighbor R1 в свой Hello:

![image](https://github.com/user-attachments/assets/ded654e0-eafb-4fb1-b533-ffcd358e92cd)

И R1 сделал, соответственно, тоже самое:

![image](https://github.com/user-attachments/assets/d943eedc-11d9-462a-8582-eba6cd6a5bd1)

Так как на моих интерфейсах нет команды network ptp isis, которая означала бы отключение выбора DIS. То выбор все-таки произойдет.

В нашем случае будет выбран R2 как Dis, так как приоритет у них одинаковые по умолчанию:

![image](https://github.com/user-attachments/assets/5a972aec-8e64-49f1-8960-f5124d47cefa)

Произойдет выбор по лучшему MAC, а именно:

```bash
Наши адреса:

R1 MAC 1: 50:06:00:49:84:45

R2 MAC 2: 50:06:00:a5:fe:57

Шаг 1. Сравнение первых отличающихся октетов:
Первые три октета (50:06:00) совпадают.

4-й октет:

MAC 1: 49 (шест. 0x49 = дес. 73)MAC 1: a5 (шест. 0xa5 = дес. 165)

MAC 2: a5 (шест. 0xa5 = дес. 165)
```
Соответственно, будет выбран роутер R2 с MAC 2.

Этот выбор нужен, чтобы определить оратора, по аналогии в OSPF это фазы exstart/exchange. То есть выбирается главное устройство, которое будет отправлять CSNP для формирование LSDB, сранивать, есть ли у него все нужные LSP (level1/level2), присылать недостающие LSP соседям. Чтобы не возникло коллизии в широковещательных сетях этим должно заниматься одно устройство, а не несколько. В PTP-сетях этим занимаются оба устройства.

Далее посмотрим на структуру CSNP- сообщения, которое отправляется DIS в нашем случае.

![image](https://github.com/user-attachments/assets/fe6d6071-e846-4160-b5ea-bd5e276afcce)

И сами сообщения LSP, например, от R1, так как там есть еще и статика:

![image](https://github.com/user-attachments/assets/2589f791-c31c-40b8-9a86-cdbd97376ff1)

Здесь наши передаваемые префиксы хранятся в TLV, например, TLV 132 - IP-адреса интерфейсов.

![image](https://github.com/user-attachments/assets/869bd8d4-7db5-46f1-8e9f-e4048e911783)


