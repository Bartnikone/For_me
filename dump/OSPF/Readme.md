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
