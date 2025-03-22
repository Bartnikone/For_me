Рассмотрим установления соседства на примере двух роутеров:

![image](https://github.com/user-attachments/assets/efe8eec1-edb3-42ea-b92f-f5e783f9b748)

Дамп будем снимать с int eth1 на R1, его мак:

![image](https://github.com/user-attachments/assets/9714f77b-4497-4450-ab51-b4f4200e5faa)

Мак соседнего R2:

![image](https://github.com/user-attachments/assets/d1b5252d-ae0d-47a6-95ed-75cb8d03bffd)

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

![image](https://github.com/user-attachments/assets/04d1961f-c183-44f1-90af-243fa0304cdc)

Получаем первый Hello от R2:

![image](https://github.com/user-attachments/assets/475dcf5a-7232-405c-8300-c329d5c19b32)

Пока что они не видят в качестве соседей друг друга, поэтому соседство не установлено.

Далее R2, получив Hello от R1, добавит в качестве neighbor R1 в свой Hello:

![image](https://github.com/user-attachments/assets/efe56107-caa0-4ec6-a657-d685bd7c600b)

И R1 сделал, соответственно, тоже самое:

![image](https://github.com/user-attachments/assets/b5bdc79d-8f4a-459f-a5a1-568daea812ab)

Так как на моих интерфейсах нет команды network ptp isis, которая означала бы отключение выбора DIS. То выбор все-таки произойдет.

В нашем случае будет выбран R2 как Dis, так как приоритет у них одинаковые по умолчанию:

![image](https://github.com/user-attachments/assets/3ce09a24-e52f-4570-a6dd-d15357ebdfae)

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

![image](https://github.com/user-attachments/assets/f8cb7e60-4f2b-4f8f-b1bf-7bf0506a3e70)

И сами сообщения LSP, например, от R1, так как там есть еще и статика:

![image](https://github.com/user-attachments/assets/73b9da70-0fb5-4f4c-b42c-f13683efcd28)

Здесь наши передаваемые префиксы хранятся в TLV, например, TLV 132 - IP-адреса интерфейсов.

![image](https://github.com/user-attachments/assets/41dc1fb3-3517-4172-b959-8311f2be2e66)



