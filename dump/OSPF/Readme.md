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

Также в этой разделе сверяется MTU:

![image](https://github.com/user-attachments/assets/198d2872-527b-4635-8d88-106ed2ee1327)

Если MTU будет разным, соседство не установится.

В качестве Master будет выбран R2, так как имеет больший Router ID, дальше после выбора оратора роутеры обмениваются своими списками LSA, R1 отправляет первым список своих LSA:

![image](https://github.com/user-attachments/assets/a6fc6e85-2e99-4e25-b4e0-094cc82fef0c)

Затем список LSA отправляет R2:

![image](https://github.com/user-attachments/assets/e44dee0e-0673-4220-a341-eae99effa9ce)

R1 понимает, что у него нет тех LSA, которые ему прислал R2, переходят на фазу loading, чтобы обменяться недостающими LSA или как в нашем случае, sequence number в поле LSA для R1 меньше, чем те LSA, что прислал R2 :

R1 запрашивает недостающие LSA через LSR:

![image](https://github.com/user-attachments/assets/9fad7b6f-c764-4715-8972-adce2feb2db7)

И пусть эти LSA относятся напрямую к нему, но имеют хуже порядковый номер, поэтому все равно будет перезапрос их у Master.

Далее получаем LSU от R2:

![image](https://github.com/user-attachments/assets/ffffa220-5486-48a6-9731-782e622ff506)

Затем R1 отправляет LSack:

![image](https://github.com/user-attachments/assets/b2c3370e-aabf-45c8-bb62-c4c5c7b14843)

![image](https://github.com/user-attachments/assets/03145da2-c2c4-40ed-a263-a175337207d6)

Затем состояние соседства у роутеров переходи в Full:

| Neighbor ID | Instance | VRF      | Pri | State     | Dead Time | Address  | Interface |
|-------------|----------|----------|-----|-----------|-----------|----------|-----------|
| 1.1.1.1     | 100      | default  | 1   | FULL/BDR  | 00:00:38  | 10.0.0.0 | Ethernet1 |

