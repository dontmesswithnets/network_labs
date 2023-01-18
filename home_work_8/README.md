# VxLAN 4. Route-type 5

## Цель:

* Реализовать передачу суммарных префиксов через EVPN route-type 5

_В этой самостоятельной работе мы ожидаем, что вы самостоятельно:_

* Анонсируете суммарные префиксы клиентов в Overlay сеть
* Настроите маршрутизацию между клиентами через суммарный префикс
* План работы, адресное пространство, схема сети, настройки - зафиксируете в документации

## План работы:

_Схема сети будет выглядеть так:_

![image](topology.JPG)

_В этой схеме у нас по легенде будет два VRF, хосты в которых будут общаться между собой через Firewall (тоже arista vEOS). Host-01 и host-03 находятся в одном VRF, в одном vlan и, следовательно, будут доступны друг другу через L2VNI. Host-02 и host-04 находятся в одном VRF, но в разных широковещательных доменах, их мы соберем через L3VNI. Ну и наконец VRF'ы "A" и "B" будут видеть друг друга через Firewall. Чуть ниже станет понятнее, как это организовать_

## Базовые настройки

<br/>

* Spine-01

```
interface Ethernet1
   description leaf-01
   no switchport
   ip address 169.254.0.0/31
   ip ospf area 0.0.0.0

interface Ethernet2
   description leaf-02
   no switchport
   ip address 169.254.0.2/31
   ip ospf area 0.0.0.0

interface Ethernet3
   description border_leaf-01
   no switchport
   ip address 169.254.0.4/31
   ip ospf area 0.0.0.0

interface Loopback0
   description router-id
   ip address 10.0.2.1/32
   ip ospf area 0.0.0.0

ip routing

mpls ip

peer-filter LEAF_AS_RANGE
   10 match as-range 65000-65010 result accept

router bgp 65000
   router-id 10.0.2.1
   bgp listen range 10.0.0.0/16 peer-group OVERLAY peer-filter LEAF_AS_RANGE
   neighbor OVERLAY peer group
   neighbor OVERLAY update-source Loopback0
   neighbor OVERLAY route-reflector-client
   neighbor OVERLAY send-community
   
   address-family evpn
      neighbor OVERLAY activate
   
   address-family ipv4
      no neighbor OVERLAY activate

router ospf 1
   router-id 10.0.2.1
```

_В этот раз я использую OSPF (все в backbone area) для Underlay и IBGP для Overlay. Как видно из конфига выше - Spine-01 является Route Reflector'ом в этой топологии. Лупбеки разлетаются по фабрике благодаря OSPF, а VTEP'ы обмениваются NLRI по MPBGP EVPN. На спайне нет других настроек, кроме приведенных выше_

* Leaf-01

```
vlan 10
   name PROD

vlan 20
   name DEV

vrf instance A

vrf instance B

interface Ethernet1
   description spine-01
   no switchport
   ip address 169.254.0.1/31
   ip ospf area 0.0.0.0

interface Ethernet2
   switchport access vlan 10

interface Ethernet3
   switchport access vlan 20

interface Loopback0
   description router-id
   ip address 10.0.1.1/32
   ip ospf area 0.0.0.0

interface Loopback1
   description vtep
   ip address 10.0.1.10/32
   ip ospf area 0.0.0.0

interface Vlan10
   vrf A
   ip address virtual 192.168.0.1/24

interface Vlan20
   vrf B
   ip address virtual 192.168.1.1/24

interface Vxlan1
   vxlan source-interface Loopback1
   vxlan vlan 10 vni 100
   vxlan vlan 20 vni 200
   vxlan vrf A vni 5000
   vxlan vrf B vni 7500

ip virtual-router mac-address 00:00:11:11:22:22

ip routing
ip routing vrf A
ip routing vrf B

mpls ip

router bgp 65000
   router-id 10.0.1.1
   neighbor OVERLAY peer group
   neighbor OVERLAY remote-as 65000
   neighbor OVERLAY update-source Loopback0
   neighbor OVERLAY send-community
   neighbor 10.0.2.1 peer group OVERLAY
   
   vlan 10
      rd 10.0.1.10:100
      route-target both 100:10
      redistribute learned
   
   vlan 20
      rd 10.0.1.10:200
      route-target both 200:20
      redistribute learned
   
   address-family evpn
      neighbor OVERLAY activate
   
   address-family ipv4
      no neighbor OVERLAY activate
   
   vrf A
      rd 10.0.1.1:5000
      route-target import evpn 5000:500
      route-target export evpn 5000:500
   
   vrf B
      rd 10.0.1.1:7500
      route-target import evpn 7500:750
      route-target export evpn 7500:750

router ospf 1
   router-id 10.0.1.1
   passive-interface Ethernet2
   passive-interface Ethernet3
```

* Leaf-02

```
vlan 10
   name PROD

vlan 30
   name STAGE

vrf instance A

vrf instance B

interface Ethernet1
   description spine-01
   no switchport
   ip address 169.254.0.3/31
   ip ospf area 0.0.0.0

interface Ethernet2
   switchport access vlan 10

interface Ethernet3
   switchport access vlan 30

interface Loopback0
   description router-id
   ip address 10.0.1.2/32
   ip ospf area 0.0.0.0

interface Loopback1
   description vtep
   ip address 10.0.1.20/32
   ip ospf area 0.0.0.0

interface Vlan10
   vrf A
   ip address virtual 192.168.0.1/24

interface Vlan30
   vrf B
   ip address virtual 192.168.2.1/24

interface Vxlan1
   vxlan source-interface Loopback1
   vxlan vlan 10 vni 100
   vxlan vlan 30 vni 300
   vxlan vrf A vni 5000
   vxlan vrf B vni 7500

ip virtual-router mac-address 00:00:11:11:22:22

ip routing
ip routing vrf A
ip routing vrf B

mpls ip

router bgp 65000
   router-id 10.0.1.2
   neighbor OVERLAY peer group
   neighbor OVERLAY remote-as 65000
   neighbor OVERLAY update-source Loopback0
   neighbor OVERLAY send-community
   neighbor 10.0.2.1 peer group OVERLAY
   
   vlan 10
      rd 10.0.1.20:100
      route-target both 100:10
      redistribute learned
   
   vlan 30
      rd 10.0.1.20:300
      route-target both 300:30
      redistribute learned
   
   address-family evpn
      neighbor OVERLAY activate
   
   address-family ipv4
      no neighbor OVERLAY activate
   
   vrf A
      rd 10.0.1.2:5000
      route-target import evpn 5000:500
      route-target export evpn 5000:500
   
   vrf B
      rd 10.0.1.2:7500
      route-target import evpn 7500:750
      route-target export evpn 7500:750

router ospf 1
   router-id 10.0.1.2
   passive-interface Ethernet2
   passive-interface Ethernet3
```

_В приведенных выше конфигах нет ничего из того, что я НЕ делал в лабе "Vxlan L3VNI". Тут есть необходимые настройки для обмена трафиком между host-01 и host-03, а также между host-02 и host-04. Проверим связность_

* с host-01 пингуем host-03

```
VPCS> show ip

NAME        : VPCS[1]
IP/MASK     : 192.168.0.2/24
GATEWAY     : 192.168.0.1
DNS         :
MAC         : 00:50:79:66:68:07
LPORT       : 20000
RHOST:PORT  : 127.0.0.1:30000
MTU         : 1500

VPCS> ping 192.168.0.3

84 bytes from 192.168.0.3 icmp_seq=1 ttl=64 time=18.318 ms
84 bytes from 192.168.0.3 icmp_seq=2 ttl=64 time=15.351 ms
84 bytes from 192.168.0.3 icmp_seq=3 ttl=64 time=11.949 ms
84 bytes from 192.168.0.3 icmp_seq=4 ttl=64 time=12.816 ms
84 bytes from 192.168.0.3 icmp_seq=5 ttl=64 time=10.646 ms
```

* с host-02 пингуем host-04

```
VPCS> show ip

NAME        : VPCS[1]
IP/MASK     : 192.168.1.2/24
GATEWAY     : 192.168.1.1
DNS         :
MAC         : 00:50:79:66:68:08
LPORT       : 20000
RHOST:PORT  : 127.0.0.1:30000
MTU         : 1500

VPCS> ping 192.168.2.2

84 bytes from 192.168.2.2 icmp_seq=1 ttl=62 time=13.945 ms
84 bytes from 192.168.2.2 icmp_seq=2 ttl=62 time=16.739 ms
84 bytes from 192.168.2.2 icmp_seq=3 ttl=62 time=12.348 ms
84 bytes from 192.168.2.2 icmp_seq=4 ttl=62 time=14.742 ms
84 bytes from 192.168.2.2 icmp_seq=5 ttl=62 time=12.514 ms
```

## Настройки для route-type 5 (IP-prefix)

_Здесь у нас появляется Borderleaf и Firewall. Задумка следующая: так как это два разных VRF'а, нужно иметь возможность контроллировать хождение трафика между ними. В такой топологии привычнее представить, что каждый VRF это отдельный клиент, но так как конкретно в этой лабе у нас нет возможности сделать Firewall в "зоне клиента", то применим отдельный "железный" Firewall. В рамках лабораторной работы фильтровать трафик я не буду, но такая возможность есть_

_Borderleaf устанавливает соседство по Overlay и благодаря настроенному L3VNI с route-target import/export политикой получает arp записи от leaf-01/02, а дальше уже транслирует их на Firewall, НО уже в агегированном до /24 виде. В качестве альтернативы команде "allowas-in" я использую локальную подмену AS при пиринге с Firewall'ом, это позволит "разворачивать" маршруты обратно в фабрику. В результате до leaf-01/02 приходят уже анонсы с /24 маской, что позволит хостам из разных VRF обмениваться трафиком_

## ___Зачем это все нужно?___

_Вообще, все будет работать и без агрегации префиксов и использования route-type 5, так как Borderleaf все равно будет отправлять на Firewall route-type 2 (ip-mac), а тот благополучно поделится ими через второй Dot1q интерфейс. Но если представить реальную ситуацию, при которой в одном vlan вполне может быть тысяча и больше хостов, это может сулить проблемы, так как эти arp записи также занимают место в TCAM памяти, да и выглядит это не совсем удобно. И решая эту проблему в рамках лабораторной работы я буду применять агрегацию префиксов и передачу их в виде route-type 5 маршрутов_

* Borderleaf

```
vrf instance A

vrf instance B

interface Ethernet1
   description spine-01
   no switchport
   ip address 169.254.0.5/31
   ip ospf area 0.0.0.0

interface Ethernet2
   description firewall-01
   no switchport

interface Ethernet2.100
   encapsulation dot1q vlan 100
   vrf A
   ip address 169.254.0.7/31

interface Ethernet2.200
   encapsulation dot1q vlan 200
   vrf B
   ip address 169.254.0.9/31

interface Loopback0
   description router-id
   ip address 10.0.3.1/32
   ip ospf area 0.0.0.0

interface Loopback1
   description vtep
   ip address 10.0.3.10/32
   ip ospf area 0.0.0.0

interface Vxlan1
   vxlan source-interface Loopback1
   vxlan vrf A vni 5000
   vxlan vrf B vni 7500

ip virtual-router mac-address 00:00:11:11:22:22

ip routing
ip routing vrf A
ip routing vrf B

mpls ip

router bgp 65000
   router-id 10.0.3.1
   neighbor OVERLAY peer group
   neighbor OVERLAY remote-as 65000
   neighbor OVERLAY update-source Loopback0
   neighbor OVERLAY send-community
   neighbor 10.0.2.1 peer group OVERLAY
   
   address-family evpn
      neighbor OVERLAY activate
   
   address-family ipv4
      no neighbor OVERLAY activate
   
   vrf A
      rd 10.0.3.10:5000
      route-target import evpn 5000:500
      route-target export evpn 5000:500
      neighbor 169.254.0.6 remote-as 4259905000
      neighbor 169.254.0.6 local-as 4259840100 no-prepend replace-as
      aggregate-address 192.168.0.0/24 summary-only
   
   vrf B
      rd 10.0.3.10:7500
      route-target import evpn 7500:750
      route-target export evpn 7500:750
      neighbor 169.254.0.8 remote-as 4259905000
      neighbor 169.254.0.8 local-as 4259840200 no-prepend replace-as
      aggregate-address 192.168.1.0/24 summary-only
      aggregate-address 192.168.2.0/24 summary-only

router ospf 1
   router-id 1.0.3.1
   passive-interface Ethernet2
   passive-interface Ethernet3
```

* Firewall

```
interface Ethernet1
   no switchport
!
interface Ethernet1.100
   encapsulation dot1q vlan 100
   ip address 169.254.0.6/31
!
interface Ethernet1.200
   encapsulation dot1q vlan 200
   ip address 169.254.0.8/31

interface Loopback0
   description router-id
   ip address 10.0.4.1/32

ip routing

mpls ip

router bgp 4259905000
   router-id 10.0.4.1
   neighbor 169.254.0.7 remote-as 4259840100
   neighbor 169.254.0.9 remote-as 4259840200
```

_Получается, что получив анонс от соседа, Firewall отправляет этот анонс другому (по его мнению) соседу. Ведь для него это EBGP с двумя разными AS. А Borderleaf, получив такой маршрут от соседа, по политике route-target export evpn уже отправляет апдейт лифам через спайн_

_Если сейчас посмотреть на вывод команды show ip bgp summary на Firewall, то он окажется пустым, ведь Borderleaf при "молчании" хостов может изучить только imet (route-type 3), иными словами подписку на BUM трафик в определенных VNI. А такие машруты он не отправляет в сторону Firewall, так как они пирятся в address-family ipv4, а для такого типа маршрутов нужно EVPN_

```
firewall-01#sh ip bgp summary
BGP summary information for VRF default
Router identifier 10.0.4.1, local AS number 4259905000
Neighbor Status Codes: m - Under maintenance
  Neighbor    V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  169.254.0.7 4 4259840100       236       247    0    0 01:16:31 Estab   0      0
  169.254.0.9 4 4259840200       246       241    0    0 01:16:25 Estab   0      0
```

_Однако, после обмена трафиком между хостами с разных VRF здесь должны появиться /24 префиксы, проверим_

* с host-01 пингуем host-02 и host-04

```
VPCS> show ip

NAME        : VPCS[1]
IP/MASK     : 192.168.0.2/24
GATEWAY     : 192.168.0.1
DNS         :
MAC         : 00:50:79:66:68:07
LPORT       : 20000
RHOST:PORT  : 127.0.0.1:30000
MTU         : 1500

VPCS> ping 192.168.1.2

84 bytes from 192.168.1.2 icmp_seq=1 ttl=59 time=32.932 ms
84 bytes from 192.168.1.2 icmp_seq=2 ttl=59 time=25.375 ms
84 bytes from 192.168.1.2 icmp_seq=3 ttl=59 time=29.875 ms
84 bytes from 192.168.1.2 icmp_seq=4 ttl=59 time=28.580 ms
84 bytes from 192.168.1.2 icmp_seq=5 ttl=59 time=30.418 ms

VPCS> ping 192.168.2.2

84 bytes from 192.168.2.2 icmp_seq=1 ttl=59 time=29.331 ms
84 bytes from 192.168.2.2 icmp_seq=2 ttl=59 time=26.177 ms
84 bytes from 192.168.2.2 icmp_seq=3 ttl=59 time=26.627 ms
84 bytes from 192.168.2.2 icmp_seq=4 ttl=59 time=28.398 ms
84 bytes from 192.168.2.2 icmp_seq=5 ttl=59 time=28.717 ms
```

_Как видим, связность между хостами из разных VRF появилась, что и являлось целью лабораторной. Проверим теперь выводы команд_

* На Borderleaf

```
borderleaf-01#sh bgp evpn
BGP routing table information for VRF default
Router identifier 10.0.3.1, local AS number 65000
Route status codes: s - suppressed, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >     RD: 10.0.1.10:100 mac-ip 0050.7966.6807
                                 10.0.1.10             -       100     0       i Or-ID: 10.0.1.1 C-LST: 10.0.2.1
 * >     RD: 10.0.1.10:100 mac-ip 0050.7966.6807 192.168.0.2
                                 10.0.1.10             -       100     0       i Or-ID: 10.0.1.1 C-LST: 10.0.2.1
 * >     RD: 10.0.1.10:200 mac-ip 0050.7966.6808
                                 10.0.1.10             -       100     0       i Or-ID: 10.0.1.1 C-LST: 10.0.2.1
 * >     RD: 10.0.1.10:200 mac-ip 0050.7966.6808 192.168.1.2
                                 10.0.1.10             -       100     0       i Or-ID: 10.0.1.1 C-LST: 10.0.2.1
 * >     RD: 10.0.1.20:100 mac-ip 0050.7966.6809
                                 10.0.1.20             -       100     0       i Or-ID: 10.0.1.2 C-LST: 10.0.2.1
 * >     RD: 10.0.1.20:100 mac-ip 0050.7966.6809 192.168.0.3
                                 10.0.1.20             -       100     0       i Or-ID: 10.0.1.2 C-LST: 10.0.2.1
 * >     RD: 10.0.1.20:300 mac-ip 0050.7966.680a
                                 10.0.1.20             -       100     0       i Or-ID: 10.0.1.2 C-LST: 10.0.2.1
 * >     RD: 10.0.1.20:300 mac-ip 0050.7966.680a 192.168.2.2
                                 10.0.1.20             -       100     0       i Or-ID: 10.0.1.2 C-LST: 10.0.2.1
 * >     RD: 10.0.1.10:100 imet 10.0.1.10
                                 10.0.1.10             -       100     0       i Or-ID: 10.0.1.1 C-LST: 10.0.2.1
 * >     RD: 10.0.1.10:200 imet 10.0.1.10
                                 10.0.1.10             -       100     0       i Or-ID: 10.0.1.1 C-LST: 10.0.2.1
 * >     RD: 10.0.1.20:100 imet 10.0.1.20
                                 10.0.1.20             -       100     0       i Or-ID: 10.0.1.2 C-LST: 10.0.2.1
 * >     RD: 10.0.1.20:300 imet 10.0.1.20
                                 10.0.1.20             -       100     0       i Or-ID: 10.0.1.2 C-LST: 10.0.2.1
 * >     RD: 10.0.3.10:5000 ip-prefix 192.168.0.0/24
                                 -                     -       -       0       i
 * >     RD: 10.0.3.10:7500 ip-prefix 192.168.0.0/24
                                 -                     -       100     0       4259905000 4259840100 i
 * >     RD: 10.0.3.10:5000 ip-prefix 192.168.1.0/24
                                 -                     -       100     0       4259905000 4259840200 i
 * >     RD: 10.0.3.10:7500 ip-prefix 192.168.1.0/24
                                 -                     -       -       0       i
 * >     RD: 10.0.3.10:5000 ip-prefix 192.168.2.0/24
                                 -                     -       100     0       4259905000 4259840200 i
 * >     RD: 10.0.3.10:7500 ip-prefix 192.168.2.0/24
                                 -                     -       -       0       i
```

```
borderleaf-01#sh bgp evpn route-type ip-prefix ipv4
BGP routing table information for VRF default
Router identifier 10.0.3.1, local AS number 65000
Route status codes: s - suppressed, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >     RD: 10.0.3.10:5000 ip-prefix 192.168.0.0/24
                                 -                     -       -       0       i
 * >     RD: 10.0.3.10:7500 ip-prefix 192.168.0.0/24
                                 -                     -       100     0       4259905000 4259840100 i
 * >     RD: 10.0.3.10:5000 ip-prefix 192.168.1.0/24
                                 -                     -       100     0       4259905000 4259840200 i
 * >     RD: 10.0.3.10:7500 ip-prefix 192.168.1.0/24
                                 -                     -       -       0       i
 * >     RD: 10.0.3.10:5000 ip-prefix 192.168.2.0/24
                                 -                     -       100     0       4259905000 4259840200 i
 * >     RD: 10.0.3.10:7500 ip-prefix 192.168.2.0/24
                                 -                     -       -       0       i
```

* На Firewall

```
firewall-01#sh ip bgp
BGP routing table information for VRF default
Router identifier 10.0.4.1, local AS number 4259905000
Route status codes: s - suppressed, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >      192.168.0.0/24         169.254.0.7           0       -          100     0       4259840100 i
 * >      192.168.1.0/24         169.254.0.9           0       -          100     0       4259840200 i
 * >      192.168.2.0/24         169.254.0.9           0       -          100     0       4259840200 i
```

_Все успешно работает. Конфиги будут лежать_ [здесь](https://github.com/dontmesswithnets/study_otus/tree/main/Fouth_month/configs)

<br/>

# Тоже самое на NXOS 9K (9.3.10)

_Схема точно такая же, как и на аристах, однако, в этот раз решил вообще убрать L2VNI связность между host-01 и host-03, ибо это слишком легко_

_Так как в качестве хостов я использую образы vIOS Cisco роутеры, то линки от лифов настрою в режиме trunk_

_По легенде в моей схеме host-01 и host-03 будут в одном VRF, но в разных широковещательных доменах, host-02 и host-04 аналогично в своем VRF и в разных широковещательных доменах_

_Во всем остальном повторение сценария с аристами_

## Настройки хостов

* host-01

```
interface GigabitEthernet0/0
 no ip address
!
interface GigabitEthernet0/0.10
 encapsulation dot1Q 10
 ip address 192.168.0.2 255.255.255.0
!
ip route 0.0.0.0 0.0.0.0 192.168.0.1
```

* host-02

```
interface GigabitEthernet0/0
 no ip address
!
interface GigabitEthernet0/0.20
 encapsulation dot1Q 20
 ip address 192.168.1.2 255.255.255.0
!
ip route 0.0.0.0 0.0.0.0 192.168.1.1
```

* host-03

```
interface GigabitEthernet0/0
 no ip address
!
interface GigabitEthernet0/0.30
 encapsulation dot1Q 30
 ip address 192.168.2.2 255.255.255.0
!
ip route 0.0.0.0 0.0.0.0 192.168.2.1
```

* host-04

```
interface GigabitEthernet0/0
 no ip address
!
interface GigabitEthernet0/0.40
 encapsulation dot1Q 40
 ip address 192.168.3.2 255.255.255.0
!
ip route 0.0.0.0 0.0.0.0 192.168.3.1
```

* Необходимые фичи

```
feature ospf
feature bgp
feature fabric forwarding
feature interface-vlan
feature vn-segment-vlan-based
feature nv overlay
nv overlay evpn
```

_После этого можем приступать к базовым настройкам интерфейсов (выглядит это примерно так)_

* leaf-01

```
interface Ethernet1/1
  description spine-01
  no switchport
  ip address 169.254.0.1/31
  no ip ospf passive-interface
  ip router ospf 1 area 0.0.0.0
  no shutdown

interface Ethernet1/2
  description host-01
  switchport mode trunk
  switchport trunk allowed vlan 10

interface Ethernet1/3
  description host-02
  switchport mode trunk
  switchport trunk allowed vlan 20

interface loopback0
  description router-id
  ip address 10.0.1.1/32
  ip router ospf 1 area 0.0.0.0

interface loopback1
  description vtep
  ip address 10.0.1.10/32
  ip router ospf 1 area 0.0.0.0
```

* leaf-02

```
interface Ethernet1/1
  description spine-01
  no switchport
  ip address 169.254.0.3/31
  no ip ospf passive-interface
  ip router ospf 1 area 0.0.0.0
  no shutdown

interface Ethernet1/2
  description host-03
  switchport mode trunk
  switchport trunk allowed vlan 30

interface Ethernet1/3
  description host-04
  switchport mode trunk
  switchport trunk allowed vlan 40

interface loopback0
  description router-id
  ip address 10.0.1.2/32
  ip router ospf 1 area 0.0.0.0

interface loopback1
  description vtep
  ip address 10.0.1.20/32
  ip router ospf 1 area 0.0.0.0
```

* spine-01

```
interface Ethernet1/1
  description leaf-01
  no switchport
  ip address 169.254.0.0/31
  ip router ospf 1 area 0.0.0.0
  no shutdown

interface Ethernet1/2
  description leaf-02
  no switchport
  ip address 169.254.0.2/31
  ip router ospf 1 area 0.0.0.0
  no shutdown

interface Ethernet1/3
  description borderleaf-01
  no switchport
  ip address 169.254.0.4/31
  ip router ospf 1 area 0.0.0.0
  no shutdown

interface loopback0
  description router-id
  ip address 10.0.2.1/32
  ip router ospf 1 area 0.0.0.0
```

* borderleaf-01

```
interface Ethernet1/1
  description spine-01
  no switchport
  ip address 169.254.0.5/31
  ip router ospf 1 area 0.0.0.0
  no shutdown

interface Ethernet1/2
  description firewall-01
  no switchport
  ip ospf passive-interface
  no shutdown

interface Ethernet1/2.100
  encapsulation dot1q 100
  ip address 169.254.0.7/31

interface Ethernet1/2.200
  encapsulation dot1q 200
  ip address 169.254.0.9/31

interface loopback0
  description router-id
  ip address 10.0.3.1/32
  ip router ospf 1 area 0.0.0.0

interface loopback1
  description vtep
  ip address 10.0.3.10/32
  ip router ospf 1 area 0.0.0.0

```

* firewall-01

```
interface GigabitEthernet0/0
 no ip address

interface GigabitEthernet0/0.100
 encapsulation dot1Q 100
 ip address 169.254.0.6 255.255.255.254

interface GigabitEthernet0/0.200
 encapsulation dot1Q 200
 ip address 169.254.0.8 255.255.255.254
```

_После этого можно утверждать, что на уровне Underlay у меня все готово_

* spine-01

```
spine-01# sh ip ospf neighbors
 OSPF Process ID 1 VRF default
 Total number of neighbors: 3
 Neighbor ID     Pri State            Up Time  Address         Interface
 10.0.1.1          1 FULL/DR          00:10:26 169.254.0.1     Eth1/1
 10.0.1.2          1 FULL/DR          00:10:32 169.254.0.3     Eth1/2
 10.0.3.1          1 FULL/BDR         00:08:16 169.254.0.5     Eth1/3
spine-01# sh ip route ospf-1
IP Route Table for VRF "default"
'*' denotes best ucast next-hop
'**' denotes best mcast next-hop
'[x/y]' denotes [preference/metric]
'%<string>' in via output denotes VRF <string>

10.0.1.1/32, ubest/mbest: 1/0
    *via 169.254.0.1, Eth1/1, [110/41], 00:10:28, ospf-1, intra
10.0.1.2/32, ubest/mbest: 1/0
    *via 169.254.0.3, Eth1/2, [110/41], 00:10:34, ospf-1, intra
10.0.1.10/32, ubest/mbest: 1/0
    *via 169.254.0.1, Eth1/1, [110/41], 00:10:28, ospf-1, intra
10.0.1.20/32, ubest/mbest: 1/0
    *via 169.254.0.3, Eth1/2, [110/41], 00:10:34, ospf-1, intra
10.0.3.1/32, ubest/mbest: 1/0
    *via 169.254.0.5, Eth1/3, [110/41], 00:01:13, ospf-1, intra
10.0.3.10/32, ubest/mbest: 1/0
    *via 169.254.0.5, Eth1/3, [110/41], 00:01:08, ospf-1, intra
```

_Все знают адреса лупбеков своих соседей, а значит, пришло время построить BGP сессии со спайном в AF l2vpn EVPN_

* leaf-01

```
router bgp 65000
  router-id 10.0.1.1
  template peer SPINE
    remote-as 65000
    update-source loopback0
    address-family l2vpn evpn
      send-community
      send-community extended
  neighbor 10.0.2.1
    inherit peer SPINE
```

* leaf-02

```
router bgp 65000
  router-id 10.0.1.2
  template peer SPINE
    remote-as 65000
    update-source loopback0
    address-family l2vpn evpn
      send-community
      send-community extended
  neighbor 10.0.2.1
    inherit peer SPINE
```

* spine-01

```
router bgp 65000
  router-id 10.0.2.1
  template peer LEAF
    remote-as 65000
    update-source loopback0
    address-family l2vpn evpn
      send-community
      send-community extended
      route-reflector-client
  neighbor 10.0.0.0/16
    inherit peer LEAF
```

* borderleaf-01

```
router bgp 65000
  router-id 10.0.3.1
  template peer SPINE
    remote-as 65000
    update-source loopback0
    address-family l2vpn evpn
      send-community
      send-community extended
  neighbor 10.0.2.1
    inherit peer SPINE
```

_После этого можно проверять состояние BGP сессий на spine-01_

```
spine-01# sh bgp l2vpn evpn summary
BGP summary information for VRF default, address family L2VPN EVPN
BGP router identifier 10.0.2.1, local AS number 65000
BGP table version is 5, L2VPN EVPN config peers 4, capable peers 3
0 network entries and 0 paths using 0 bytes of memory
BGP attribute entries [0/0], BGP AS path entries [0/0]
BGP community entries [0/0], BGP clusterlist entries [0/0]

Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
10.0.1.1        4 65000       8       8        5    0    0 00:02:05 0
10.0.1.2        4 65000       7       7        5    0    0 00:01:11 0
10.0.3.1        4 65000       6       6        5    0    0 00:00:42 0
```

_Все работает, апдейтов мы, естественно, пока не видим_

_Далее выполню настройки в следующем порядке_

* создам два vlan'а для L3VNI на каждом лифе
* создам два VRF'а на каждом лифе с полной настройкой для L3VNI
* создам два SVI интерфейса на каждом лифе и примаплю каждый к своему VRF
* настрою L3VNI для связности между хостами в своих VRF

_Также, пора задать системный МАС для фабрики (один на каждом лифе)_

```
leaf-01(config)# fabric forwarding anycast-gateway-mac 00:00:11:11:22:22
```

_Сейчас на лифах созданы следующие vlan'ы_

* leaf-01

```
vlan 10,20,88,99
vlan 10
  name PROD
  vn-segment 100
vlan 20
  name DEV
  vn-segment 200
vlan 88
  name L3VNI_for_VRF_PROD
  vn-segment 888
vlan 99
  name L3VNI_for_VRF_DEV
  vn-segment 999
```

* leaf-02

```
vlan 30,40,88,99
vlan 30
  name PROD
  vn-segment 300
vlan 40
  name DEV
  vn-segment 400
vlan 88
  name L3VNI_for_VRF_PROD
  vn-segment 888
vlan 99
  name L3VNI_for_VRF_DEV
  vn-segment 999
```

_Создал соответствующие два VRF'а_

* leaf-01

```
vrf context PROD
  rd 10.0.1.10:888
  address-family ipv4 unicast
    route-target import 888:88
    route-target import 888:88 evpn
    route-target export 888:88
    route-target export 888:88 evpn
  vni 888

vrf context DEV
  rd 10.0.1.10:999
  address-family ipv4 unicast
    route-target import 999:99
    route-target import 999:99 evpn
    route-target export 999:99
    route-target export 999:99 evpn
  vni 999
```

* leaf-02

```
vrf context PROD
  rd 10.0.1.20:888
  address-family ipv4 unicast
    route-target import 888:88
    route-target import 888:88 evpn
    route-target export 888:88
    route-target export 888:88 evpn
  vni 888

vrf context DEV
  rd 10.0.1.20:999
  address-family ipv4 unicast
    route-target import 999:99
    route-target import 999:99 evpn
    route-target export 999:99
    route-target export 999:99 evpn
  vni 999
```

_Теперь SVI интерфейсы_

* leaf-01

```
interface Vlan10
  no shutdown
  vrf member PROD
  ip address 192.168.0.1/24
  fabric forwarding mode anycast-gateway

interface Vlan20
  no shutdown
  vrf member DEV
  ip address 192.168.1.1/24
  fabric forwarding mode anycast-gateway

interface Vlan88
  no shutdown
  vrf member PROD
  ip forward

interface Vlan99
  no shutdown
  vrf member DEV
  ip forward
```

* leaf-02

```
interface Vlan30
  no shutdown
  vrf member PROD
  ip address 192.168.2.1/24
  fabric forwarding mode anycast-gateway

interface Vlan40
  no shutdown
  vrf member DEV
  ip address 192.168.3.1/24
  fabric forwarding mode anycast-gateway

interface Vlan88
  no shutdown
  vrf member PROD
  ip forward

interface Vlan99
  no shutdown
  vrf member DEV
  ip forward
```

_Совсем забыл про NVE интерфейс, исправялем_

* leaf-01

```
interface nve1
  no shutdown
  host-reachability protocol bgp
  source-interface loopback1
  member vni 100
    ingress-replication protocol bgp
  member vni 200
    ingress-replication protocol bgp
  member vni 888 associate-vrf
  member vni 999 associate-vrf
```

* leaf-02

```
interface nve1
  no shutdown
  host-reachability protocol bgp
  source-interface loopback1
  member vni 300
    ingress-replication protocol bgp
  member vni 400
    ingress-replication protocol bgp
  member vni 888 associate-vrf
  member vni 999 associate-vrf
```

_После этого можно заметить, что spine-01 получил маршруты type 3_

* spine-01

```
spine-01# sh bgp l2vpn evpn summary
BGP summary information for VRF default, address family L2VPN EVPN
BGP router identifier 10.0.2.1, local AS number 65000
BGP table version is 20, L2VPN EVPN config peers 4, capable peers 3
4 network entries and 4 paths using 976 bytes of memory
BGP attribute entries [4/688], BGP AS path entries [0/0]
BGP community entries [0/0], BGP clusterlist entries [0/0]

Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
10.0.1.1        4 65000      34      38       20    0    0 00:22:12 2
10.0.1.2        4 65000      33      35       20    0    0 00:21:18 2
10.0.3.1        4 65000      26      25       20    0    0 00:20:49 0
```

_(!) Можно было заметить, что я не настраивал секцию EVPN, в рамках L3VNI это необязательно, так как VTEP сразу форвардит пакеты в L3VNI_

_Проверим связность хостов в рамках одного VRF'а_

* с host-01 пингуем host-03 и наоборот

```
host-01#ping 192.168.2.2
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 192.168.2.2, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 6/6/8 ms

host-03#ping 192.168.0.2
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 192.168.0.2, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 6/7/12 ms
```

* с host-02 пингуем host-04 и наоборот

```
host-02#ping 192.168.3.2
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 192.168.3.2, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 6/7/11 ms

host-04#ping 192.168.1.2
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 192.168.1.2, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 6/6/9 ms
```

_После этого снова идем на спайн_

```
spine-01# sh bgp l2vpn evpn summary
BGP summary information for VRF default, address family L2VPN EVPN
BGP router identifier 10.0.2.1, local AS number 65000
BGP table version is 29, L2VPN EVPN config peers 4, capable peers 3
12 network entries and 12 paths using 2928 bytes of memory
BGP attribute entries [12/2064], BGP AS path entries [0/0]
BGP community entries [0/0], BGP clusterlist entries [0/0]

Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
10.0.1.1        4 65000      47      46       29    0    0 00:31:38 6
10.0.1.2        4 65000      45      45       29    0    0 00:30:44 6
10.0.3.1        4 65000      36      33       29    0    0 00:30:15 0
```

_Я не буду вставлять сюда полный вывод маршрутной информации, там добавились записи route-type 2 (ip-mac)_

<br/>

## Настройка взаимодействия между разными VRF

<br/>

_Реализовывать будем так же, как и на аристе, но тут есть свои тонкости_

* borderleaf-01

```
vlan 88,99
vlan 88
  name L3VNI_for_VRF_PROD
  vn-segment 888
vlan 99
  name L3VNI_for_VRF_DEV
  vn-segment 999
```

```
vrf context PROD
  rd 10.0.3.10:888
  address-family ipv4 unicast
    route-target import 888:88
    route-target import 888:88 evpn
    route-target export 888:88
    route-target export 888:88 evpn
  vni 888

vrf context DEV
  rd 10.0.3.10:999
  address-family ipv4 unicast
    route-target import 999:99
    route-target import 999:99 evpn
    route-target export 999:99
    route-target export 999:99 evpn
  vni 999
```

_(!) Не забываем добавить интерфейсы Eth 1/2.100 и 1/2.200 в соответствующие VRF'ы_

```
interface Ethernet1/2.100
  encapsulation dot1q 100
  vrf member PROD
  ip address 169.254.0.7/31
  no shutdown

interface Ethernet1/2.200
  encapsulation dot1q 200
  vrf member DEV
  ip address 169.254.0.9/31
  no shutdown
```

```
interface Vlan88
  no shutdown
  vrf member PROD
  ip forward

interface Vlan99
  no shutdown
  vrf member DEV
  ip forward
```

```
interface nve1
  no shutdown
  host-reachability protocol bgp
  source-interface loopback1
  member vni 888 associate-vrf
  member vni 999 associate-vrf
```

_Теперь осталось настроить BGP сессии в AF ipv4 unicast между borderleaf-01 и firewall-01_

* borderleaf-01

```
  vrf DEV
    address-family ipv4 unicast
      aggregate-address 192.168.1.0/24 summary-only
      aggregate-address 192.168.3.0/24 summary-only
    neighbor 169.254.0.8
      remote-as 65000.65000
      local-as 65000.2 no-prepend replace-as
      address-family ipv4 unicast
  vrf PROD
    address-family ipv4 unicast
      aggregate-address 192.168.0.0/24 summary-only
      aggregate-address 192.168.2.0/24 summary-only
    neighbor 169.254.0.6
      remote-as 65000.65000
      local-as 65000.1 no-prepend replace-as
      address-family ipv4 unicast
```

* firewall-01

```
router bgp 4259905000
 neighbor 169.254.0.7 remote-as 4259840001
 neighbor 169.254.0.9 remote-as 4259840002
```

_После этого идем проверять_

* firewall-01

```
Router#sh ip bgp summary
BGP router identifier 169.254.0.8, local AS number 4259905000
BGP table version is 5, main routing table version 5
4 network entries using 576 bytes of memory
4 path entries using 320 bytes of memory
2/2 BGP path/bestpath attribute entries using 304 bytes of memory
2 BGP AS-PATH entries using 48 bytes of memory
0 BGP route-map cache entries using 0 bytes of memory
0 BGP filter-list cache entries using 0 bytes of memory
BGP using 1248 total bytes of memory
BGP activity 4/0 prefixes, 4/0 paths, scan interval 60 secs

Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
169.254.0.7     4   4259840001      10      10        5    0    0 00:05:42        2
169.254.0.9     4   4259840002      10      11        5    0    0 00:05:32        2
Router#sh ip bgp
BGP table version is 5, local router ID is 169.254.0.8
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal,
              r RIB-failure, S Stale, m multipath, b backup-path, f RT-Filter,
              x best-external, a additional-path, c RIB-compressed,
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI validation codes: V valid, I invalid, N Not found

     Network          Next Hop            Metric LocPrf Weight Path
 *>  192.168.0.0      169.254.0.7                            0 4259840001 i
 *>  192.168.1.0      169.254.0.9                            0 4259840002 i
 *>  192.168.2.0      169.254.0.7                            0 4259840001 i
 *>  192.168.3.0      169.254.0.9                            0 4259840002 i
```

_Как видим, borderleaf-01 отдает уже агрегированные маршруты в сторону firewall-01, а тот уже разворачивает их обратно в фабрику_

_Проверим связность хостов в разных VRF_

* с host-01 пингуем host-02 и host-04

```
host-01#ping 192.168.1.2
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 192.168.1.2, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 11/13/18 ms
host-01#ping 192.168.3.2
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 192.168.3.2, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 13/16/21 ms
```

* с host-02 пингуем host-01 и host-03

```
host-02#ping 192.168.0.2
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 192.168.0.2, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 10/12/18 ms
host-02#ping 192.168.2.2
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 192.168.2.2, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 11/13/16 ms
```

_Все работает!_

_Итоговая таблица маршрутизации l2vpn evpn на spine-01_

```
spine-01# sh bgp l2vpn evpn summary
BGP summary information for VRF default, address family L2VPN EVPN
BGP router identifier 10.0.2.1, local AS number 65000
BGP table version is 50, L2VPN EVPN config peers 4, capable peers 3
20 network entries and 20 paths using 4880 bytes of memory
BGP attribute entries [16/2752], BGP AS path entries [2/20]
BGP community entries [0/0], BGP clusterlist entries [0/0]

Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
10.0.1.1        4 65000      94      88       50    0    0 01:15:01 6
10.0.1.2        4 65000      91      88       50    0    0 01:14:07 6
10.0.3.1        4 65000      87      88       50    0    0 01:13:38 8

```

_А это маршруты type 5_

```
spine-01# sh bgp l2vpn evpn route-type 5
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 10.0.3.10:888
BGP routing table entry for [5]:[0]:[0]:[24]:[192.168.0.0]/224, version 35
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not i
n HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    10.0.3.10 (metric 41) from 10.0.3.1 (10.0.3.1)
      Origin IGP, MED not set, localpref 100, weight 0
      Aggregated by 0.0.0.0, aggregator AS 65000, atomic-aggregate set
      Received label 888
      Extcommunity: RT:888:88 ENCAP:8 Router MAC:5008.0000.1b08

  Path-id 1 advertised to peers:
    10.0.1.1           10.0.1.2
BGP routing table entry for [5]:[0]:[0]:[24]:[192.168.1.0]/224, version 41
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not i
n HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  Gateway IP: 0.0.0.0
  AS-Path: 4259905000 4259840002 , path sourced external to AS
    10.0.3.10 (metric 41) from 10.0.3.1 (10.0.3.1)
      Origin IGP, MED not set, localpref 100, weight 0
      Aggregated by 0.0.0.0, aggregator AS 65000, atomic-aggregate set
      Received label 888
      Extcommunity: RT:888:88 ENCAP:8 Router MAC:5008.0000.1b08

  Path-id 1 advertised to peers:
    10.0.1.1           10.0.1.2
BGP routing table entry for [5]:[0]:[0]:[24]:[192.168.2.0]/224, version 36
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not i
n HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    10.0.3.10 (metric 41) from 10.0.3.1 (10.0.3.1)
      Origin IGP, MED not set, localpref 100, weight 0
      Aggregated by 0.0.0.0, aggregator AS 65000, atomic-aggregate set
      Received label 888
      Extcommunity: RT:888:88 ENCAP:8 Router MAC:5008.0000.1b08

  Path-id 1 advertised to peers:
    10.0.1.1           10.0.1.2
BGP routing table entry for [5]:[0]:[0]:[24]:[192.168.3.0]/224, version 42
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not i
n HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  Gateway IP: 0.0.0.0
  AS-Path: 4259905000 4259840002 , path sourced external to AS
    10.0.3.10 (metric 41) from 10.0.3.1 (10.0.3.1)
      Origin IGP, MED not set, localpref 100, weight 0
      Aggregated by 0.0.0.0, aggregator AS 65000, atomic-aggregate set
      Received label 888
      Extcommunity: RT:888:88 ENCAP:8 Router MAC:5008.0000.1b08

  Path-id 1 advertised to peers:
    10.0.1.1           10.0.1.2

Route Distinguisher: 10.0.3.10:999
BGP routing table entry for [5]:[0]:[0]:[24]:[192.168.0.0]/224, version 39
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not i
n HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  Gateway IP: 0.0.0.0
  AS-Path: 4259905000 4259840001 , path sourced external to AS
    10.0.3.10 (metric 41) from 10.0.3.1 (10.0.3.1)
      Origin IGP, MED not set, localpref 100, weight 0
      Aggregated by 0.0.0.0, aggregator AS 65000, atomic-aggregate set
      Received label 999
      Extcommunity: RT:999:99 ENCAP:8 Router MAC:5008.0000.1b08

  Path-id 1 advertised to peers:
    10.0.1.1           10.0.1.2
BGP routing table entry for [5]:[0]:[0]:[24]:[192.168.1.0]/224, version 37
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not i
n HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    10.0.3.10 (metric 41) from 10.0.3.1 (10.0.3.1)
      Origin IGP, MED not set, localpref 100, weight 0
      Aggregated by 0.0.0.0, aggregator AS 65000, atomic-aggregate set
      Received label 999
      Extcommunity: RT:999:99 ENCAP:8 Router MAC:5008.0000.1b08

  Path-id 1 advertised to peers:
    10.0.1.1           10.0.1.2
BGP routing table entry for [5]:[0]:[0]:[24]:[192.168.2.0]/224, version 40
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not i
n HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  Gateway IP: 0.0.0.0
  AS-Path: 4259905000 4259840001 , path sourced external to AS
    10.0.3.10 (metric 41) from 10.0.3.1 (10.0.3.1)
      Origin IGP, MED not set, localpref 100, weight 0
      Aggregated by 0.0.0.0, aggregator AS 65000, atomic-aggregate set
      Received label 999
      Extcommunity: RT:999:99 ENCAP:8 Router MAC:5008.0000.1b08

  Path-id 1 advertised to peers:
    10.0.1.1           10.0.1.2
BGP routing table entry for [5]:[0]:[0]:[24]:[192.168.3.0]/224, version 38
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not i
n HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    10.0.3.10 (metric 41) from 10.0.3.1 (10.0.3.1)
      Origin IGP, MED not set, localpref 100, weight 0
      Aggregated by 0.0.0.0, aggregator AS 65000, atomic-aggregate set
      Received label 999
      Extcommunity: RT:999:99 ENCAP:8 Router MAC:5008.0000.1b08

  Path-id 1 advertised to peers:
    10.0.1.1           10.0.1.2
```

_Конфиги лежат_ [здесь](https://github.com/dontmesswithnets/study_otus/tree/main/Fouth_month/configs_nxos)

