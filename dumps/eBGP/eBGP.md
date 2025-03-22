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
Конечно же, BGP строит поверх TCP, поэтому изначально наши роутеры строят TCP-сессию:

![image](https://github.com/user-attachments/assets/7948bae1-bba1-4692-ad83-43af27fd160e)

## Рассмотрим первый SYN и его заголовки:

![image](https://github.com/user-attachments/assets/e33e9f89-5528-43b9-9755-6766c0a79c25)

Затем от R2 к R1 последует Syn-Ack:

![image](https://github.com/user-attachments/assets/bb910d32-4699-4800-b7e0-8bbe92598c4a)

И, наконец, Ack от R1 к R2:

![image](https://github.com/user-attachments/assets/0ba362ec-2406-4ed1-bc3f-af0613fafaa6)

TCP-сессия установлена, и это переход в состояние Open Sent:

## Затем R1 отправляет сообщение OPEN, его структура:

![image](https://github.com/user-attachments/assets/373239a0-97b7-4138-8e28-dd8ba8944ba4)

R2 отправляет ответный OPEN:

![image](https://github.com/user-attachments/assets/98e141a3-56ab-4ac9-9215-1f5e666d62a7)

## R1 сверяет его параметры, AS, Hold Timer, версию протокола и Router ID, если параметры верны, то далее идет переход в стадию Open Confirm:

![image](https://github.com/user-attachments/assets/94167ad2-37d3-41cc-8d78-4c32691ad875)

Если KEEPALIVE не приходят, сессия разрывается.

## После обмена OPEN и KEEPALIVE сессия переходит в статус Established

