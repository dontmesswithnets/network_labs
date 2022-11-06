# Underlay. IS-IS
## **Цель:**
_Настроить **IS-IS** для **Underlay** сети_

_В этой самостоятельной работе мы ожидаем, что вы самостоятельно:_
- настроите IS-IS в Underlay сети, для IP связности между всеми устройствами NXOS
- составите план работы, адресное пространство, схему сети, настройки
  
## **План работы**
_На интерфейсах между __Underlay__ устройствами мы используем __/31__ адреса. На лупбеке настроен __/32__ адрес_

### **Со стороны спайнов**

* _Сейчас интерфейсы настроены следующим образом_ (пример для dc1-sp-01)
```
interface Ethernet1/1
  description dc1-lf-01 Eth1/6
  no switchport
  no ip redirects
  ip address 169.254.0.0/31
  no shutdown
```
```
interface Ethernet1/2
  description dc1-lf-02 Eth1/6
  no switchport
  no ip redirects
  ip address 169.254.0.2/31
  no shutdown
```
```
interface Ethernet1/3
  description dc1-lf-03 Eth1/6
  no switchport
  no ip redirects
  ip address 169.254.0.4/31
  no shutdown
```
Loopback 0
```
interface loopback0
  ip address 10.1.0.1/32
```
* _На_ dc1-sp-02 _интерфейсы настроены аналогично_ (отличаются лишь последние октеты в IP адресе)

### **Со стороны лифов**

* _Настройки интерфейсов следующие_ (пример для dc1-lf-01)
```
interface Ethernet1/6
  description dc1-sp-01 Eth1/1
  no switchport
  no ip redirects
  ip address 169.254.0.1/31
  no shutdown
```
```
interface Ethernet1/7
  description dc1-sp-02 Eth1/1
  no switchport
  no ip redirects
  ip address 169.254.0.7/31
  no shutdown
```
* _На_ dc1-lf-02 _и_ dc1-lf-03 _интерфейсы настроены аналогично_ (отличаются лишь последние октеты в IP адресе)

### **Проверим базовую IP связность по протоколу ICMP между dc1-sp-01 и dc1-lf-01**

```
dc1-sp-01# ping 169.254.0.1
PING 169.254.0.1 (169.254.0.1): 56 data bytes
64 bytes from 169.254.0.1: icmp_seq=0 ttl=254 time=11.236 ms
64 bytes from 169.254.0.1: icmp_seq=1 ttl=254 time=2.171 ms
64 bytes from 169.254.0.1: icmp_seq=2 ttl=254 time=2.934 ms
64 bytes from 169.254.0.1: icmp_seq=3 ttl=254 time=2.229 ms
64 bytes from 169.254.0.1: icmp_seq=4 ttl=254 time=2.407 ms

--- 169.254.0.1 ping statistics ---
5 packets transmitted, 5 packets received, 0.00% packet loss
round-trip min/avg/max = 2.171/4.195/11.236 ms
```

___Теперь нужно настроить IP связность между всеми Underlay устройствами с помощью протокола IS-IS___
* _Включаем функцию_ ***IS-IS*** _на NXOS, задаем ***name*** и назначаем_ ***NET*** идентификатор (пример для dc1-sp-01)
```
feature isis
router isis Underlay
  net 49.0001.0001.0001.0001.00
```
* _Включаем логирование_ ***IS-IS*** _состояний, а также говорим свитчу формировать исключительно_ ***level-2*** _соседства_
```
log-adjacency-changes
  is-type level-2
```
* _Далее запускаем ***IS-IS*** на интерфейсах, а также задаем тип_ ***point-to-point*** (пример для dc1-sp-1). _Для этого добавляем эти две строчки в конфигурацию всех L3 интерфейсов_ (кроме loopback 0, там только первую)
```
  ip router isis Underlay
  isis network point-to-point
```

### **Теперь можем посмотреть соседства и таблицу маршрутизации для IS-IS instance**

```
dc1-sp-01(config)# sh isis adjacency
IS-IS process: Underlay VRF: default
IS-IS adjacency database:
Legend: '!': No AF level connectivity in given topology
System ID       SNPA            Level  State  Hold Time  Interface
dc1-lf-01       N/A             2      UP     00:00:22   Ethernet1/1
dc1-lf-02       N/A             2      UP     00:00:22   Ethernet1/2
dc1-lf-03       N/A             2      UP     00:00:21   Ethernet1/3

```
```
dc1-sp-01(config)# sh routing isis-Underlay
IP Route Table for VRF "default"
'*' denotes best ucast next-hop
'**' denotes best mcast next-hop
'[x/y]' denotes [preference/metric]
'%<string>' in via output denotes VRF <string>

10.1.0.2/32, ubest/mbest: 2/0
    *via 169.254.0.1, Eth1/1, [115/81], 00:00:56, isis-Underlay, L2
    *via 169.254.0.3, Eth1/2, [115/81], 00:14:09, isis-Underlay, L2
10.2.0.1/32, ubest/mbest: 1/0
    *via 169.254.0.1, Eth1/1, [115/41], 00:13:36, isis-Underlay, L2
10.2.0.2/32, ubest/mbest: 1/0
    *via 169.254.0.3, Eth1/2, [115/41], 00:13:10, isis-Underlay, L2
10.2.0.3/32, ubest/mbest: 2/0
    *via 169.254.0.1, Eth1/1, [115/121], 0.000000, isis-Underlay, L2
    *via 169.254.0.3, Eth1/2, [115/121], 0.000000, isis-Underlay, L2
169.254.0.6/31, ubest/mbest: 1/0
    *via 169.254.0.1, Eth1/1, [115/80], 00:22:50, isis-Underlay, L2
169.254.0.8/31, ubest/mbest: 1/0
    *via 169.254.0.3, Eth1/2, [115/80], 00:20:25, isis-Underlay, L2
169.254.0.10/31, ubest/mbest: 2/0
    *via 169.254.0.1, Eth1/1, [115/120], 0.000000, isis-Underlay, L2
    *via 169.254.0.3, Eth1/2, [115/120], 0.000000, isis-Underlay, L2
```
___На остальных аналогично___

_Теперь попробуем проверить IP связность между_ **dc1-lf-01** и **dc1-lf-03** _(между которыми нет прямого линка) через loopback интерфейсы_
* Для этого выполним команду **Ping**
```
dc1-lf-01# ping 10.2.0.3 source 10.2.0.1
PING 10.2.0.3 (10.2.0.3) from 10.2.0.1: 56 data bytes
64 bytes from 10.2.0.3: icmp_seq=0 ttl=253 time=16.037 ms
64 bytes from 10.2.0.3: icmp_seq=1 ttl=253 time=4.826 ms
64 bytes from 10.2.0.3: icmp_seq=2 ttl=253 time=10.998 ms
64 bytes from 10.2.0.3: icmp_seq=3 ttl=253 time=5.952 ms
64 bytes from 10.2.0.3: icmp_seq=4 ttl=253 time=5.185 ms

--- 10.2.0.3 ping statistics ---
5 packets transmitted, 5 packets received, 0.00% packet loss
round-trip min/avg/max = 4.826/8.599/16.037 ms
```
_Как видим, между всеми **Underlay** устройствами появилась IP связность благодаря настроенному протоколу **IS-IS**_

_Топология_ **Underlay** _с указанием адресации на устройствах_ (включая loopback) ![image](topology.JPG)

_Полные конфиги устройств лежат [здесь](congigs)