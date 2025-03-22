Рассмотрим взаимодействие на простом примере:

![image](https://github.com/user-attachments/assets/e5cd2c2a-5076-4a59-981e-31a563c90a66)

Настройки интерфейса со стороны R2:

```bash
R2#sh run int ethernet 1
interface Ethernet1
   description Low_PE1
   no switchport
   ip address 10.0.0.1/31
   ip ospf area 0.0.0.0
   isis enable 100
```
И со стороны R1 соответственно:

```bash
R1#sh run int ethernet 1
interface Ethernet1
   description UP_P1
   no switchport
   ip address 10.0.0.0/31
   ip ospf area 0.0.0.0
   isis enable 100
```

Сначал посылаются Hello пакеты, рассмотрим их структуру на примере сообщения от R2:

![image](https://github.com/user-attachments/assets/17a8b84e-bf60-413d-a882-e495ba73d10f)

R1 сделает тоже самое, отошлет Hello, проинформировав о себе на dest 224.0.0.5

Когда R2 получит этот Hello, он добавит увиденный Router ID в свой Hello:

![image](https://github.com/user-attachments/assets/04337642-ce68-4406-b851-9ea15363b6c0)

После успешного обмена Hello, начинается второй этап, Exstart- Exchange ( обмен DBD database description):

![image](https://github.com/user-attachments/assets/40c0c1bd-9c4a-4f83-bbf4-56726608f7fe)
