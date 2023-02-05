# _Проектная работа_

## _Проектирование сетевой фабрики на основне VxLAN EVPN_

_Цель проекта:_ 
* _спроектировать схему взаимодействия двух независмых друг от друга дата центров_
* _описать предъявляемые требования и применяемые технологии для решения задачи_
* _выполнить конфигурацию оборудования и продемонстрировать работоспособность услуги_

### _Схема взаимодействия (глазами клиента)_

_Для заказчика все выглядит как стандартный L3VPN. То есть у него есть два тенанта и два стыка с оборудованием облачного провайдера. На каждой площадке он получает необходиые маршруты для обмена трафиком со второй площадкой и наоборот. Nexthop во всех случаях - BGP пир со стороны провайдера_

_То есть нам нужно предоставить условному заказчику услугу L3VPN через VxLAN фабрику. Под услугой L3VPN я здесь подразумеваю не MPLS, а тот факт, что зона ответственности клиента заканчивается на ipv4 BGP стыке с облачным провайдером_

### _Требования к проекту_

* _обеспечить сетевую связность между удаленными площадками заказчика_
* _обеспечить заказчику надежный и приватный канал связи_
* _обеспечить отказоустойчивость и балансировку трафика_

### _Как я буду решать второй и третий пункт?_

_Канал связи будет построен таким образом, что трафик не будет выходить за пределы облака. Как это работает? - Не используем публичные адреса для маршрутизации, а значит трафик не может маршрутизироваться через сторонних провайдеров и, соответственно, быть "прослушан" на стороне. На POP площадках подразумевается выделенная линия между портами коммутаторов, размещенных в пределах одного здания, что тоже повышает надежность и конфиденциальность передачи данных_

### _Какие технологии я буду применять?_

* _BGP IPv4 для стыка с клиентом_
* _VxLAN EVPN для маршрутизации через фабрику_
* BGP IPv4 для стыка между ЦОДами

### _Разберем это все на целевой схеме. Если охватывать всю схему, выглядит она так_

![image](main_topology.jpg)

_Мы можем видеть два дата центра, по разные стороны друг от друга. Это ЦОД "LEFT и ЦОД "RIGHT" (не благодарите за гениальные названия)_

_Они связаны через публичные POP площадки (можно представить, что это M9 и 3Data). Идея в том, что оба ЦОДа имеют выход на РОР, а значит организовать выделенный канал проще всего. ДЛя этого всего-то нужно настроить по одному интерфейсу на каждой из площадок с обеих сторон_

_Представим, что с каждой из сторон клиенту дали два влана (и даже представим, что они совпадают). Далее на бордерах создаем VRF, создаем SVI в тех самых вланах, связываем их с клиентским VRF'ом и внутри этого VRF'а поднимаем ipv4 BGP соседство. В рамках этой BGP сессии мы обмениваемся маршрутами из одного ЦОДа в другой_

_Но как передать клиентские префиксы от POP площадки до клиентских виртуальных маршрутизаторов на другом конце дата центра?_

### _Тут мы воспользуемся тем, что изучали весь курс - VxLAN EVPN фабрика_

_Для начала рассмотрим поближе каждый из ЦОДов. Начнем с "LEFT"_

![image](topology_LEFT.jpg)

_Слева в виде сервера у меня представлен ESXi гипервизор (шучу, это обычная ариста) и работающая "внутри" виртуальная машина_

_Для упрощения я опустил некоторые особенности VMware (например такие, что гипервизоры с виртуальными машинами и гипервизоры с виртуальными роутерами это чаще всего физически разные хосты). Представим, что этот гипервизор с подключенным VPCS это виртуальный клиентский роутер (читай VRF внутри EVM) и тенант с виртуальными машинами в одном лице_

_Этому гипервизору нужно как-то передать префикс 172.16.0.0/23 до двух бордеров, которые затем этот префикс через BGP передадут в другой дата центр_

_Для этого мы поднимаем  ipv4 BGP соседство с двумя лифами внутри VRF. LEAF коммутаторы, получив этот префикс импортируют его в EVPN соседство и через VxLAN фабрику отправляют дальше через L3VNI, примапленный к клиентскому VRF_

_Когда бордеры получат этот префикс, они наоборот импортируют его в ipv4 таблицу BGP и передают бордерам с другой POP площадки. А там мы увидим точно такой же алгоритм_

![image](topology_RIGHT.jpg)

_Только здесь у меня два POD'а, но схема маршрутизации трафика абсолютно аналогичная. Через VxLAN фабрику префикс 172.16.0.0/23 доходит до LEAF коммутаторов, которые импортируют маршрут из EVPN таблицы в IPv4 и передают на клиентский виртуальный маршрутизатор_

### _Почему внутри POD'а всего два лифа?_

_На этой схеме рассматривается взаимодействие только в рамках конкретного канала связи. В настоящих дата центрах, безусловно, в одном только POD'е может быть несколько десятков лифов и хостов. Однако, в моем случае используется стандартная схема подключения одного хоста через два резервирующих друг друга лифа. Тут надо заметить, что в таких случаях эти лифы находятся либо в VPC\MLAG паре, либо используется EVPN Multihoming (кстати, оба способа я рассматривал здесь же в других лабах). Но в целях демонстрации подключения виртуального клиентского роутера через BGP к провайдерскому L3VPN я специально опустил эти настройки_

### _Вот и все с теорией. Приступаем к настройкам!_

<br/>

## _Настройки хостов (виртуальных маршрутизаторов внутри ESXi гипервизоров)_

_Вначале создаю 3 влана, один я буду использовать в качестве SVI для имитации маршрутизации виртуальных машин в зоне ответственности клиента. Они то и будут конечными хостами (для примера я использую VPCS образы в каждом POD'е)_

_Другие два, предположим, клиенту выделил провайдер (то есть мы), в них тоже будут созданы SVI, но уже для установления BGP сессии с оборудованием провайдера в рамках switchport trunk интерфейса_

_Ах да, еще необходимо, конечно же, создать VRF и все эти SVI положить внутрь. В нашем примере, этот самый VRF будет играть роль тенанта клиента_

```
vlan 100,1110,1120

vrf instance CUSTOMERS

interface Vlan100
   vrf CUSTOMERS
   ip address 172.16.1.254/23

interface Vlan1110
   mtu 9214
   vrf CUSTOMERS
   ip address 169.254.1.1/30

interface Vlan1120
   mtu 9214
   vrf CUSTOMERS
   ip address 169.254.1.5/30
```

_Далее необходимо настроить интерфейсы в сторону фабрики_

```
interface Port-Channel1
   description LEFT-pd01-leaf-01_Po1
   mtu 9214
   switchport trunk allowed vlan 1110
   switchport mode trunk
!
interface Port-Channel2
   description LEFT-pd01-leaf-02_Po1
   switchport trunk allowed vlan 1120
   switchport mode trunk
!
interface Ethernet1
   description LEFT-pd01-leaf-01_Eth1
   mtu 9214
   channel-group 1 mode active
!
interface Ethernet2
   description LEFT-pd01-leaf-02_Eth1
   mtu 9214
   channel-group 2 mode active
```

_Остается поднять BGP сессию с провайдером и отдать ему нужные сети для анонса в другой ЦОД_

```
ip prefix-list CLIENT_NET seq 5 permit 172.16.0.0/23
!
route-map CLIENT_NET permit 10
   match ip address prefix-list CLIENT_NET
!
peer-filter LEAF
   10 match as-range 4200100102 result accept
!
router bgp 4200100101
   no bgp default ipv4-unicast
   neighbor CUSTOMERS peer group
   !
   address-family ipv4
      neighbor CUSTOMERS activate
   !
   vrf CUSTOMERS
      router-id 10.1.0.1
      bgp listen range 169.254.1.0/29 peer-group CUSTOMERS peer-filter LEAF
      redistribute connected route-map CLIENT_NET
```

_На других двух гипервизорах в дата центре RIGHT настройки аналогичные, отличаются IP адреса на интерфейсах, вланы и анонсируемая сеть. С точки зрения всего остального логика та же. Опустим момент их настройки и будем считать, что они уже ждут TCP SYN'ов на 179 порт_

## _Настройки LEAF_

_Согласно нашей архитектуры, на каждом из двух лифов будет выделен один уникальный влан для пиринга с клиентом. Создадим VRF, SVI и пропишем настройки на интерфейсе_

```
vlan 1110

vrf instance CUSTOMERS

interface Port-Channel1
   description LEFT-pd01-esxi_Po1
   switchport trunk allowed vlan 1110
   switchport mode trunk

interface Ethernet1
   description LEFT-pd01-esxi_Eth1
   mtu 9214
   channel-group 1 mode active

interface Vlan1110
   mtu 9214
   vrf CUSTOMERS
   ip address 169.254.1.2/30   
```

_Поднимем BGP сессию с клиентским маршрутизатором, который уже ждет подключений в режиме "bgp listen range"_

```
interface Loopback0
   description router-id
   ip address 10.1.1.1/32

router bgp 4200100102
    router-id 10.1.1.1
    no bgp default ipv4-unicast
    neighbor CUSTOMERS peer group
    neighbor CUSTOMERS remote-as 4200100101
    neighbor CUSTOMERS maximum-routes 1000

    address-family ipv4
      neighbor CUSTOMERS activate
    
    vrf CUSTOMERS
      rd 10.1.1.1:22999
      route-target import evpn 2:22999
      route-target export evpn 2:22999
      neighbor 169.254.1.1 peer group CUSTOMERS
```

_На втором лифе проделаем все то же самое, но с другим vlan id. После этого BGP соседство должно было установиться и мы должны были увидеть анонсируемый со стороны LEFT-pd01-esxi префикс 172.16.0.0/23_

```
LEFT-pd01-leaf-01#sh ip bgp neighbors 169.254.1.1 routes vrf CUSTOMERS
BGP routing table information for VRF CUSTOMERS
Router identifier 169.254.1.2, local AS number 4200100102
Route status codes: s - suppressed, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >      172.16.0.0/23          169.254.1.1           0       -          100     0       4200100101 i
```

_Так и есть. Все работает. Идем дальше. Предположим, что со стороны ЦОДа RIGHT мы выполнили аналогичные настройки и смотрим на результат_

* pd01

```
RIGHT-pd01-leaf-01#show ip bgp neighbors 169.254.1.1 routes vrf CUSTOMERS
BGP routing table information for VRF CUSTOMERS
Router identifier 169.254.1.2, local AS number 4200200102
Route status codes: s - suppressed, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >      172.16.2.0/24          169.254.1.1           0       -          100     0       4200200101 i
```

* pd02

```
RIGHT-pd02-leaf-01#show ip bgp neighbors 169.254.1.1 routes vrf CUSTOMERS
BGP routing table information for VRF CUSTOMERS
Router identifier 169.254.1.2, local AS number 4200200202
Route status codes: s - suppressed, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >      172.16.3.0/24          169.254.1.1           0       -          100     0       4200200201 i
```

_Отлично, теперь нужно заняться тем, чтобы отдать эти префиксы на бордеры ЦОДов. Приступим к настройкам VxLAN EVPN_

_Процедура стандартная, на лифах создаем лупбек под VTEP, интерфейс VxLAN 1, а также устанавливаем EVPN BGP сессию со спайнами_

```
interface Loopback1
   description vtep
   ip address 10.1.1.10/32

interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vrf CUSTOMERS vni 11999

ip prefix-list LOOPBACKS seq 5 permit 10.1.1.0/24 eq 32

route-map LOOPBACKS permit 10
   match ip address prefix-list LOOPBACKS

router bgp 4200100102
   neighbor SPINE_OVERLAY peer group
   neighbor SPINE_OVERLAY remote-as 4200100103
   neighbor SPINE_OVERLAY update-source Loopback0
   neighbor SPINE_OVERLAY ebgp-multihop 2
   neighbor SPINE_OVERLAY send-community
   neighbor SPINE_UNDERLAY peer group
   neighbor SPINE_UNDERLAY remote-as 4200100103
   neighbor 10.1.2.1 peer group SPINE_OVERLAY
   neighbor 10.1.2.1 description LEFT-pd01-spine-01
   neighbor 10.1.2.2 peer group SPINE_OVERLAY
   neighbor 10.1.2.2 description LEFT-pd01-spine-02
   neighbor 169.254.0.0 peer group SPINE_UNDERLAY
   neighbor 169.254.0.8 peer group SPINE_UNDERLAY
   redistribute connected route-map LOOPBACKS
   !
   address-family evpn
      neighbor SPINE_OVERLAY activate
   !
   address-family ipv4
      neighbor SPINE_UNDERLAY activate
   !
   vrf CUSTOMERS
      rd 10.1.1.1:22999
      route-target import evpn 2:22999
      route-target export evpn 2:22999
```

_Выше я установил BGP сессии в Underlay, затем в рамках него проанонсирова IP адреса лупбеков, а после поднял с них BGP EVPN сессии в Overlay. После того, как сессии в обеих AFI установились, необходимо проэкспортировать полученные от клиента в ipv4 BGP префиксы в EVPN и передать дальше по фабрике - готово_

_Все аналогичные настройки невидимо проделываю еще на пяти лифах и идем дальше_

## _Настройка SPINE_

_Спайны настраиваются проще, им не нужно иметь VxLAN интерфейсы для форвардинга маршрутной информации между VTEP'ами. Достаточно создать по одному лупбеку, настроить интерфейсы в сторону фабрики и поднять BGP сессии в обеих AFI_

```
interface Loopback0
   description router-id
   ip address 10.1.2.1/32

ip prefix-list LOOPBACKS seq 5 permit 10.1.2.0/24 eq 32

route-map LOOPBACKS permit 10
   match ip address prefix-list LOOPBACKS

peer-filter AS_RANGE_UNDERLAY
   10 match as-range 4200100102-4200100199 result accept
   20 match as-range 4200100001 result accept

router bgp 4200100103
   router-id 10.1.2.1
   no bgp default ipv4-unicast
   bgp listen range 169.254.0.0/24 peer-group UNDERLAY peer-filter AS_RANGE_UNDERLAY
   neighbor BORDER_OVERLAY peer group
   neighbor BORDER_OVERLAY remote-as 4200100001
   neighbor BORDER_OVERLAY next-hop-unchanged
   neighbor BORDER_OVERLAY update-source Loopback0
   neighbor BORDER_OVERLAY ebgp-multihop 2
   neighbor BORDER_OVERLAY send-community
   neighbor LEAF_OVERLAY peer group
   neighbor LEAF_OVERLAY next-hop-unchanged
   neighbor LEAF_OVERLAY update-source Loopback0
   neighbor LEAF_OVERLAY ebgp-multihop 2
   neighbor LEAF_OVERLAY send-community
   neighbor UNDERLAY peer group
   neighbor 10.1.1.1 peer group LEAF_OVERLAY
   neighbor 10.1.1.1 remote-as 4200100102
   neighbor 10.1.1.1 description LEFT-pd01-leaf-01
   neighbor 10.1.1.2 peer group LEAF_OVERLAY
   neighbor 10.1.1.2 remote-as 4200100102
   neighbor 10.1.1.2 description LEFT-pd01-leaf-02
   neighbor 10.1.3.1 peer group BORDER_OVERLAY
   neighbor 10.1.3.1 description LEFT-border-01
   neighbor 10.1.3.2 peer group BORDER_OVERLAY
   neighbor 10.1.3.2 description LEFT-border-02
   redistribute connected route-map LOOPBACKS
   
   address-family evpn
      neighbor BORDER_OVERLAY activate
      neighbor LEAF_OVERLAY activate
   
   address-family ipv4
      neighbor UNDERLAY activate
```

_То же самое еще на пяти спайнах и идем проверять установившиеся сессии_

```
LEFT-pd01-spine-01#show bgp evpn summary
BGP summary information for VRF default
Router identifier 10.1.2.1, local AS number 4200100103
Neighbor Status Codes: m - Under maintenance
  Description              Neighbor V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  LEFT-pd01-leaf-01        10.1.1.1 4 4200100102      3090      3088    0    0    1d19h Estab   1      1
  LEFT-pd01-leaf-02        10.1.1.2 4 4200100102      3086      3097    0    0    1d19h Estab   1      1
```

_На текущий момент мы получили route-type 5 префиксы от лифов_

```

LEFT-pd01-spine-01#show bgp evpn
BGP routing table information for VRF default
Router identifier 10.1.2.1, local AS number 4200100103
Route status codes: s - suppressed, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >     RD: 10.1.1.1:22999 ip-prefix 172.16.0.0/23
                                 10.1.1.10             -       100     0       4200100102 4200100101 i
 * >     RD: 10.1.1.2:22999 ip-prefix 172.16.0.0/23
                                 10.1.1.20             -       100     0       4200100102 4200100101 i
```

_На спайнах в другом ЦОД все точно так же (просто поверьте)_

## _Настройки BORDER_

_Осталось настроить бордеры на POP площадках. Начнем_

_настройки для установки BGP EVPN сессии мало чем отличаются от лифов_

```
interface Loopback0
   description router-id
   ip address 10.1.3.1/32

interface Loopback1
   description vtep
   ip address 10.1.3.10/32

interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vrf CUSTOMERS vni 22999

ip prefix-list LOOPBACKS seq 10 permit 10.1.3.0/24 eq 32
!
route-map LOOPBACKS permit 10
   match ip address prefix-list LOOPBACKS

router bgp 4200100001
   router-id 10.1.3.1
   no bgp default ipv4-unicast
   neighbor SPINE_OVERLAY peer group
   neighbor SPINE_OVERLAY update-source Loopback0
   neighbor SPINE_OVERLAY ebgp-multihop 2
   neighbor SPINE_OVERLAY send-community
   neighbor SPINE_UNDERLAY peer group
   neighbor 10.1.2.1 peer group SPINE_OVERLAY
   neighbor 10.1.2.1 remote-as 4200100103
   neighbor 10.1.2.1 description LEFT-pd01-spine-01
   neighbor 10.1.2.2 peer group SPINE_OVERLAY
   neighbor 10.1.2.2 remote-as 4200100103
   neighbor 10.1.2.2 description LEFT-pd01-spine-02
   neighbor 169.254.0.4 peer group SPINE_UNDERLAY
   neighbor 169.254.0.4 remote-as 4200100103
   neighbor 169.254.0.12 peer group SPINE_UNDERLAY
   neighbor 169.254.0.12 remote-as 4200100103
   redistribute connected route-map LOOPBACKS

   address-family evpn
      neighbor SPINE_OVERLAY activate

   address-family ipv4
      neighbor SPINE_UNDERLAY activate

   vrf CUSTOMERS
      rd 10.1.3.1:22999
      route-target import evpn 2:22999
      route-target export evpn 2:22999
```

_Проделываем на еще 3 бордерах и идем проверять сессии на спайн_

```
LEFT-pd01-spine-01#show bgp evpn summary
BGP summary information for VRF default
Router identifier 10.1.2.1, local AS number 4200100103
Neighbor Status Codes: m - Under maintenance
  Description              Neighbor V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  LEFT-pd01-leaf-01        10.1.1.1 4 4200100102      3110      3107    0    0    1d19h Estab   1      1
  LEFT-pd01-leaf-02        10.1.1.2 4 4200100102      3106      3117    0    0    1d19h Estab   1      1
  LEFT-border-01           10.1.3.1 4 4200100001      3106      3092    0    0    1d19h Estab   0      0
  LEFT-border-02           10.1.3.2 4 4200100001      3102      3091    0    0    1d19h Estab   0      0
```

_Так и должно быть. Нам нужно еще поднять IPv4 BGP сессию внутри VRF с другим ЦОДом_

## _Настройки DCI_

_Для начала нужно выделить некоторый влан, в рамках которого будет создан SVI для пиринга внутри VRF. С другой стороны он должен быть таким же, либо нужно применять настройку "switchport vlan translation" для маппинга_

_Пусть это будут вланы 3456 на верхней POP площадке и 3457 на нижней_

_Применим необходиые настройки_

```
vlan 3456

vrf instance CUSTOMERS

interface Port-Channel3
   description Data_Center_RIGHT_border-01_Po3
   mtu 9214
   switchport trunk allowed vlan 3456
   switchport mode trunk

interface Ethernet3
   description Data_Center_RIGHT_border-01_Eth3
   mtu 9214
   channel-group 3 mode active

interface Vlan3456
   mtu 9214
   vrf CUSTOMERS
   ip address 169.254.100.1/30

router bgp 4200100001
   neighbor Direct_Connect_Customer peer group
   neighbor Direct_Connect_Customer remote-as 4200200001
   neighbor Direct_Connect_Customer maximum-routes 1000

   address-family ipv4
      neighbor Direct_Connect_Customer activate
   
   vrf CUSTOMERS
      neighbor 169.254.100.2 peer group Direct_Connect_Customer
```

_С другой стороны аналогично на обеих площадках. А вот теперь можно проверить наличие маршрутов из разных фабрик на гипервизорах_

## _Проверка работоспособности_

* _на LEFT-pd01-esxi_

```
LEAF-pd01-esxi#show ip bgp vrf CUSTOMERS
BGP routing table information for VRF CUSTOMERS
Router identifier 10.1.0.1, local AS number 4200100101
Route status codes: s - suppressed, * - valid, > - active, # - not installed, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

         Network                Next Hop            Metric  LocPref Weight  Path
 * >     172.16.0.0/23          -                     1       0       -       i
 * >     172.16.2.0/24          169.254.1.2           0       100     0       4200100102 4200100103 4200100001 4200200001 4200200103 4200200102 4200200101 i
 *       172.16.2.0/24          169.254.1.6           0       100     0       4200100102 4200100103 4200100001 4200200001 4200200103 4200200102 4200200101 i
 * >     172.16.3.0/24          169.254.1.2           0       100     0       4200100102 4200100103 4200100001 4200200001 4200200203 4200200202 4200200201 i
 *       172.16.3.0/24          169.254.1.6           0       100     0       4200100102 4200100103 4200100001 4200200001 4200200203 4200200202 4200200201 i
```

* _на RIGHT-pd01-esxi_

```
RIGHT-pd01-esxi#show ip bgp vrf CUSTOMERS
BGP routing table information for VRF CUSTOMERS
Router identifier 10.2.0.1, local AS number 4200200101
Route status codes: s - suppressed, * - valid, > - active, # - not installed, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

         Network                Next Hop            Metric  LocPref Weight  Path
 * >     172.16.0.0/23          169.254.1.2           0       100     0       4200200102 4200200103 4200200001 4200100001 4200100103 4200100102 4200100101 i
 *       172.16.0.0/23          169.254.1.6           0       100     0       4200200102 4200200103 4200200001 4200100001 4200100103 4200100102 4200100101 i
 * >     172.16.2.0/24          -                     1       0       -       i
 * >     172.16.3.0/24          169.254.1.2           0       100     0       4200200102 4200200103 4200200001 4200200203 4200200202 4200200201 i
 *       172.16.3.0/24          169.254.1.6           0       100     0       4200200102 4200200103 4200200001 4200200203 4200200202 4200200201 i
```

* _на RIGHT-pd02-esxi_

```
RIGHT-pd02-esxi#show ip bgp vrf CUSTOMERS
BGP routing table information for VRF CUSTOMERS
Router identifier 10.22.0.1, local AS number 4200200201
Route status codes: s - suppressed, * - valid, > - active, # - not installed, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

         Network                Next Hop            Metric  LocPref Weight  Path
 * >     172.16.0.0/23          169.254.1.2           0       100     0       4200200202 4200200203 4200200001 4200100001 4200100103 4200100102 4200100101 i
 *       172.16.0.0/23          169.254.1.6           0       100     0       4200200202 4200200203 4200200001 4200100001 4200100103 4200100102 4200100101 i
 * >     172.16.2.0/24          169.254.1.2           0       100     0       4200200202 4200200203 4200200001 4200200103 4200200102 4200200101 i
 *       172.16.2.0/24          169.254.1.6           0       100     0       4200200202 4200200203 4200200001 4200200103 4200200102 4200200101 i
 * >     172.16.3.0/24          -                     1       0       -       i
```

_Как видим все необходимые маршруты появились в таблице маршрутизации vrf CUSTOMERS_

_Можем даже проверить связность между ВМ в LEFT-pd01-esxi с IP адресом 172.16.1.200 и ВМ в RIGHT-pd02-esxi с IP адресом 172.16.3.200_

```
VPCS> show ip

NAME        : VPCS[1]
IP/MASK     : 172.16.1.200/23
GATEWAY     : 172.16.1.254
DNS         :
MAC         : 00:50:79:66:68:16
LPORT       : 20000
RHOST:PORT  : 127.0.0.1:30000
MTU         : 1500

VPCS> ping 172.16.3.200

84 bytes from 172.16.3.200 icmp_seq=1 ttl=58 time=50.370 ms
84 bytes from 172.16.3.200 icmp_seq=2 ttl=58 time=50.954 ms
84 bytes from 172.16.3.200 icmp_seq=3 ttl=58 time=44.520 ms
84 bytes from 172.16.3.200 icmp_seq=4 ttl=58 time=54.716 ms
84 bytes from 172.16.3.200 icmp_seq=5 ttl=58 time=40.779 ms
```

_Как видим, схема взлетела и все работает. Таким образом нам удалось настроить сетевую связность между разными площадками клиента в рамках L3VPN, которая соответствует всем изложенным вначале требованиям_

<br/>

_[Ссылка](https://github.com/dontmesswithnets/study_otus/tree/main/project.work/configs) на конфиги_
