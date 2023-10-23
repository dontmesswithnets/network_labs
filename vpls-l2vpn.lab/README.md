# VPLS. L2VPN на Cisco CSR

### Основные термины:

* VPLS домен - изолированная виртуальная L2-сеть или отдельный  L2VPN (два разных заказчика будут жить в пределах разных VPLS доменов)
* Virtual Switching Instance (VSI) или VFI (Virtual Forwarding Instance) - Это виртуальный коммутатор в пределах одного устройства (аналог MAC-VRF в VxLAN EVPN)
* VPLS Edge (VE) - участник VPLS домена или PE в терминологии MPLS

_В VPLS домене работают те же правила, что и в классическом L2 коммутаторе, а именно:_

* MAC learning
* Split Horizon

### VPLS Kompella или VPLS Auto-Discovery with BGP-Signalling

_В этой лабораторной работе не будут рассмотрен VPLS Martini (LDP Signalling)_

_Для VPLS в секции BGP используется Address Family L2VPN AFI (25) и VPLS SAFI (65)_

_Для динамического обнаружения соседей используются BGP Communities (на основе route-target). Loop prevention в VPLS работает по следующему принципу: если пакет получен через VSI/VFI интерфейс, то он не будет отправлен обратно в VPLS домен, даже в сторону других VE_



