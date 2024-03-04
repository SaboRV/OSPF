# Цель домашнего задания
Создать домашнюю сетевую лабораторию. Научится настраивать протокол OSPF в Linux-based системах.


# Описание домашнего задания
1. Развернуть 3 виртуальные машины
2. Объединить их разными vlan
- настроить OSPF между машинами на базе Quagga;
- изобразить ассиметричный роутинг;
- сделать один из линков "дорогим", но что бы при этом роутинг был симметричным.

# Введение
OSPF — протокол динамической маршрутизации, использующий концепцию разделения на области в целях масштабирования. 
Административная дистанция OSPF — 110
Основные свойства протокола OSPF:
Быстрая сходимость
Масштабируемость (подходит для маленьких и больших сетей)
Безопасность (поддежка аутентиикации)
Эффективность (испольование алгоритма поиска кратчайшего пути)

При настроенном OSPF маршрутизатор формирует таблицу топологии с использованием результатов вычислений, основанных на алгоритме кратчайшего пути (SPF) Дейкстры. Алгоритм поиска кратчайшего пути основывается на данных о совокупной стоимости доступа к точке назначения. Стоимость доступа определятся на основе скорости интерфейса. 
Чтобы повысить эффективность и масштабируемость OSPF, протокол поддерживает иерархическую маршрутизацию с помощью областей (area). 
Область OSPF (area) — Часть сети, которой ограничивается формирование базы данных о состоянии каналов. Маршрутизаторы, находящиеся в одной и той же области, имеют одну и ту же базу данных о топологии сети. Для определения областей применяются идентификаторы областей.
Протоколы OSPF бвывают 2-х версий: 
    • OSPFv2
    • OSPFv3

Основным отличием протоколов является то, что OSPFv2 работает с IPv4, а OSPFv3 — c IPv6. 
Маршрутизаторы в OSPF клаlссифицируются на основе выполняемой ими функции:

см. Pic_1

- Internal router (внутренний маршрутизатор) — маршрутизатор, все интерфейсы которого находятся в одной и той же области. 
- Backbone router (магистральный маршрутизатор) — это маршрутизатор, который находится в магистральной зоне (area 0).
- ABR (пограничный маргрутизатор области) — маршрутизатор, интерфейсы которого подключены к разным областям.
- ASBR (Граничный маршрутизатор автономной системы) — это маршрутизатор, у которого интерфейс подключен к внешней сети.
Также с помощью OSPF можно настроить ассиметричный роутинг.
Ассиметричная маршрутизация — возможность пересекать сеть в одном направлении, используя один путь, и возвращаться через другой путь.

# РЕШЕНИЕ

## 1. Разворачиваем 3 виртуальные машины

Так как мы планируем настроить OSPF, все 3 виртуальные машины должны быть соединены между собой (разными VLAN), а также иметь одну (или несколько) доолнительных сетей, к которым, далее OSPF сформирует маршруты. Исходя из данных требований, мы можем нарисовать топологию сети:

см. Pic_2

В результате подняты 3 виртуальные машины, которые соединены между собой сетями (10.0.10.0/30, 10.0.11.0/30 и 10.0.12.0/30). У каждого роутера есть дополнительная сеть:
●	на router1 — 192.168.10.0/24
●	на router2 — 192.168.20.0/24
●	на router3 — 192.168.30.0/24

Установлены базовые программы для изменения конфигурационных файлов (vim) и изучения сети (traceroute, tcpdump, net-tools).

Установлен и настроен пакет FRR. Он построен на базе Quagga и продолжает своё развитие.

## Проверим статус FRR:

root@router1:~# sudo systemctl status frr
● frr.service - FRRouting
     Loaded: loaded (/lib/systemd/system/frr.service; enabled; vendor preset: enabled)
     Active: active (running) since Mon 2024-03-04 09:45:15 UTC; 12min ago
       Docs: https://frrouting.readthedocs.io/en/latest/setup.html
    Process: 5653 ExecStart=/usr/lib/frr/frrinit.sh start (code=exited, status=0/SUCCESS)
   Main PID: 5666 (watchfrr)
     Status: "FRR Operational"
      Tasks: 10 (limit: 1117)
     Memory: 17.2M
     CGroup: /system.slice/frr.service
             ├─5666 /usr/lib/frr/watchfrr -d -F traditional zebra mgmtd ospfd staticd
             ├─5677 /usr/lib/frr/zebra -d -F traditional -A 127.0.0.1 -s 90000000
             ├─5682 /usr/lib/frr/mgmtd -d -F traditional -A 127.0.0.1
             ├─5684 /usr/lib/frr/ospfd -d -F traditional -A 127.0.0.1
             └─5687 /usr/lib/frr/staticd -d -F traditional -A 127.0.0.1

Mar 04 09:45:10 router1 mgmtd[5682]: [VTVCM-Y2NW3] Configuration Read in Took: 00:00:00
Mar 04 09:45:10 router1 frrinit.sh[5690]: [5690|mgmtd] Configuration file[/etc/frr/frr.conf] processing >
Mar 04 09:45:10 router1 watchfrr[5666]: [ZJW5C-1EHNT] restart all process 5667 exited with non-zero stat>
Mar 04 09:45:14 router1 watchfrr[5666]: [QDG3Y-BY5TN] staticd state -> up : connect succeeded
Mar 04 09:45:14 router1 watchfrr[5666]: [QDG3Y-BY5TN] mgmtd state -> up : connect succeeded
Mar 04 09:45:15 router1 watchfrr[5666]: [QDG3Y-BY5TN] zebra state -> up : connect succeeded
Mar 04 09:45:15 router1 watchfrr[5666]: [QDG3Y-BY5TN] ospfd state -> up : connect succeeded
Mar 04 09:45:15 router1 watchfrr[5666]: [KWE5Q-QNGFC] all daemons up, doing startup-complete notify
Mar 04 09:45:15 router1 frrinit.sh[5653]:  * Started watchfrr
Mar 04 09:45:15 router1 systemd[1]: Started FRRouting.
lines 1-26/26 (END)

 
### Если мы правильно настроили OSPF, то с любого хоста нам должны быть доступны сети:
192.168.10.0/24
192.168.20.0/24
192.168.30.0/24
10.0.10.0/30 
10.0.11.0/30
10.0.13.0/30

Проверим доступность сетей с хоста router1:

root@router1:~# ping 192.168.30.1
PING 192.168.30.1 (192.168.30.1) 56(84) bytes of data.
64 bytes from 192.168.30.1: icmp_seq=1 ttl=64 time=0.628 ms
64 bytes from 192.168.30.1: icmp_seq=2 ttl=64 time=0.533 ms
64 bytes from 192.168.30.1: icmp_seq=3 ttl=64 time=0.419 ms
64 bytes from 192.168.30.1: icmp_seq=4 ttl=64 time=0.508 ms
64 bytes from 192.168.30.1: icmp_seq=5 ttl=64 time=0.510 ms
^C
--- 192.168.30.1 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4062ms
rtt min/avg/max/mdev = 0.419/0.519/0.628/0.066 ms

Запустим трассировку до адреса 192.168.30.1

root@router1:~# traceroute 192.168.30.1
traceroute to 192.168.30.1 (192.168.30.1), 30 hops max, 60 byte packets
 1  192.168.30.1 (192.168.30.1)  0.400 ms  0.388 ms  0.359 ms

Попробуем отключить интерфейс enp0s9 и немного подождем и снова запустим трассировку до ip-адреса 192.168.30.1

root@router1:~# vtysh

Hello, this is FRRouting (version 9.1).
Copyright 1996-2005 Kunihiro Ishiguro, et al.

router1# show interface brief
Interface       Status  VRF             Addresses
---------       ------  ---             ---------
enp0s3          up      default         10.0.2.15/24
enp0s8          up      default         10.0.10.1/30
enp0s9          up      default         10.0.12.1/30
enp0s10         up      default         192.168.10.1/24
enp0s16         up      default         192.168.50.10/24
lo              up      default         

router1# exit
root@router1:~# ifconfig enp0s9 down
root@router1:~# ip a | grep enp0s9
4: enp0s9: <BROADCAST,MULTICAST> mtu 1500 qdisc fq_codel state DOWN group default qlen 1000
root@router1:~# traceroute 192.168.30.1
traceroute to 192.168.30.1 (192.168.30.1), 30 hops max, 60 byte packets
 1  10.0.10.2 (10.0.10.2)  0.407 ms  0.354 ms  0.340 ms
 2  192.168.30.1 (192.168.30.1)  1.232 ms  1.223 ms  1.055 ms
root@router1:~# 

Как мы видим, после отключения интерфейса сеть 192.168.30.0/24 нам остаётся доступна.

Также мы можем проверить из интерфейса vtysh какие маршруты мы видим на данный момент:

router1# show ip route ospf
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, F - PBR,
       f - OpenFabric,
       > - selected route, * - FIB route, q - queued, r - rejected, b - backup
       t - trapped, o - offload failure

O   10.0.10.0/30 [110/100] is directly connected, enp0s8, weight 1, 00:21:31
O>* 10.0.11.0/30 [110/200] via 10.0.10.2, enp0s8, weight 1, 00:01:50
O>* 10.0.12.0/30 [110/300] via 10.0.10.2, enp0s8, weight 1, 00:01:50
O   192.168.10.0/24 [110/100] is directly connected, enp0s10, weight 1, 00:21:31
O>* 192.168.20.0/24 [110/200] via 10.0.10.2, enp0s8, weight 1, 00:20:51
O>* 192.168.30.0/24 [110/300] via 10.0.10.2, enp0s8, weight 1, 00:01:50
router1# 

# Настройка ассиметричного роутинга

Для настройки ассиметричного роутинга нам необходимо выключить блокировку ассиметричной маршрутизации: sysctl net.ipv4.conf.all.rp_filter=0

Далее, выбираем один из роутеров, на котором изменим «стоимость интерфейса». Например поменяем стоимость интерфейса enp0s8 на router1:

## router1
root@router1:~# sysctl net.ipv4.conf.all.rp_filter=0
net.ipv4.conf.all.rp_filter = 0
root@router1:~# sudo systemctl restart frr
root@router1:~# vtysh

Hello, this is FRRouting (version 9.1).
Copyright 1996-2005 Kunihiro Ishiguro, et al.

router1# conf t
router1(config)# int enp0s8
router1(config-if)# ip ospf cost 1000
router1(config-if)# exit
router1(config)# exit
router1# show ip route ospf
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, F - PBR,
       f - OpenFabric,
       > - selected route, * - FIB route, q - queued, r - rejected, b - backup
       t - trapped, o - offload failure

O   10.0.10.0/30 [110/300] via 10.0.12.2, enp0s9, weight 1, 00:00:17
O>* 10.0.11.0/30 [110/200] via 10.0.12.2, enp0s9, weight 1, 00:00:17
O   10.0.12.0/30 [110/100] is directly connected, enp0s9, weight 1, 00:00:26
O   192.168.10.0/24 [110/100] is directly connected, enp0s10, weight 1, 00:01:01
O>* 192.168.20.0/24 [110/300] via 10.0.12.2, enp0s9, weight 1, 00:00:17
O>* 192.168.30.0/24 [110/200] via 10.0.12.2, enp0s9, weight 1, 00:00:26


## router2
root@router2:~# vtysh

Hello, this is FRRouting (version 9.1).
Copyright 1996-2005 Kunihiro Ishiguro, et al.

router2# show ip route ospf
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, F - PBR,
       f - OpenFabric,
       > - selected route, * - FIB route, q - queued, r - rejected, b - backup
       t - trapped, o - offload failure

O   10.0.10.0/30 [110/100] is directly connected, enp0s8, weight 1, 00:31:02
O   10.0.11.0/30 [110/100] is directly connected, enp0s9, weight 1, 00:31:02
O>* 10.0.12.0/30 [110/200] via 10.0.10.1, enp0s8, weight 1, 00:02:10
  *                        via 10.0.11.1, enp0s9, weight 1, 00:02:10
O>* 192.168.10.0/24 [110/200] via 10.0.10.1, enp0s8, weight 1, 00:02:10
O   192.168.20.0/24 [110/100] is directly connected, enp0s10, weight 1, 00:31:02
O>* 192.168.30.0/24 [110/200] via 10.0.11.1, enp0s9, weight 1, 00:30:22


После внесения данных настроек, мы видим, что маршрут до сети 192.168.20.0/30  теперь пойдёт через router2, но обратный трафик от router2 пойдёт по другому пути. Давайте это проверим:
1) На router1 запускаем пинг от 192.168.10.1 до 192.168.20.1: 
root@router2:~# tcpdump -i enp0s9
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on enp0s9, link-type EN10MB (Ethernet), capture size 262144 bytes
10:18:33.435235 IP 192.168.10.1 > router2: ICMP echo request, id 2, seq 16, length 64
10:18:34.459256 IP 192.168.10.1 > router2: ICMP echo request, id 2, seq 17, length 64
10:18:35.482953 IP 192.168.10.1 > router2: ICMP echo request, id 2, seq 18, length 64
10:18:36.509573 IP 192.168.10.1 > router2: ICMP echo request, id 2, seq 19, length 64
10:18:37.534077 IP 192.168.10.1 > router2: ICMP echo request, id 2, seq 20, length 64
10:18:38.535047 IP 192.168.10.1 > router2: ICMP echo request, id 2, seq 21, length 64
10:18:39.547193 IP 192.168.10.1 > router2: ICMP echo request, id 2, seq 22, length 64

2) На router2 запускаем tcpdump, который будет смотреть трафик только на порту enp0s9:
root@router2:~# tcpdump -i enp0s9
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on enp0s9, link-type EN10MB (Ethernet), capture size 262144 bytes
10:18:33.435235 IP 192.168.10.1 > router2: ICMP echo request, id 2, seq 16, length 64
10:18:34.459256 IP 192.168.10.1 > router2: ICMP echo request, id 2, seq 17, length 64
10:18:35.482953 IP 192.168.10.1 > router2: ICMP echo request, id 2, seq 18, length 64
10:18:36.509573 IP 192.168.10.1 > router2: ICMP echo request, id 2, seq 19, length 64
10:18:37.534077 IP 192.168.10.1 > router2: ICMP echo request, id 2, seq 20, length 64
10:18:38.535047 IP 192.168.10.1 > router2: ICMP echo request, id 2, seq 21, length 64
10:18:39.547193 IP 192.168.10.1 > router2: ICMP echo request, id 2, seq 22, length 64

Видим что данный порт только получает ICMP-трафик с адреса 192.168.10.1

3) На router2 запускаем tcpdump, который будет смотреть трафик только на порту enp0s8:

root@router2:~# tcpdump -i enp0s8
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on enp0s8, link-type EN10MB (Ethernet), capture size 262144 bytes
10:20:33.467195 IP router2 > 192.168.10.1: ICMP echo reply, id 3, seq 20, length 64
10:20:34.491169 IP router2 > 192.168.10.1: ICMP echo reply, id 3, seq 21, length 64
10:20:35.515304 IP router2 > 192.168.10.1: ICMP echo reply, id 3, seq 22, length 64
10:20:36.539427 IP router2 > 192.168.10.1: ICMP echo reply, id 3, seq 23, length 64
10:20:36.676505 IP 10.0.10.1 > ospf-all.mcast.net: OSPFv2, Hello, length 48
10:20:37.563088 IP router2 > 192.168.10.1: ICMP echo reply, id 3, seq 24, length 64
10:20:38.587416 IP router2 > 192.168.10.1: ICMP echo reply, id 3, seq 25, length 64

Видим что данный порт только отправляет ICMP-трафик на адрес 192.168.10.1

Таким образом мы видим ассиметричный роутинг.


# Настройка симметичного роутинга

Так как у нас уже есть один «дорогой» интерфейс, нам потребуется добавить ещё один дорогой интерфейс, чтобы у нас перестала работать ассиметричная маршрутизация. 

Так как в прошлом задании мы заметили что router2 будет отправлять обратно трафик через порт enp0s8, мы также должны сделать его дорогим и далее проверить, что теперь используется симметричная маршрутизация:

Поменяем стоимость интерфейса enp0s8 на router2:
root@router2:~# vtysh

Hello, this is FRRouting (version 9.1).
Copyright 1996-2005 Kunihiro Ishiguro, et al.

router2# conf t
router2(config)# int enp0s8
router2(config-if)# ip ospf cost 1000
router2(config-if)# exit
router2(config)# exit
router2# show ip route ospf
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, F - PBR,
       f - OpenFabric,
       > - selected route, * - FIB route, q - queued, r - rejected, b - backup
       t - trapped, o - offload failure

O   10.0.10.0/30 [110/1000] is directly connected, enp0s8, weight 1, 00:00:12
O   10.0.11.0/30 [110/100] is directly connected, enp0s9, weight 1, 00:01:42
O>* 10.0.12.0/30 [110/200] via 10.0.11.1, enp0s9, weight 1, 00:00:12
O>* 192.168.10.0/24 [110/300] via 10.0.11.1, enp0s9, weight 1, 00:00:12
O   192.168.20.0/24 [110/100] is directly connected, enp0s10, weight 1, 00:02:19
O>* 192.168.30.0/24 [110/200] via 10.0.11.1, enp0s9, weight 1, 00:01:42

После внесения данных настроек, мы видим, что маршрут до сети 192.168.10.0/30  пойдёт через router2.

Давайте это проверим:
На router1 запускаем пинг от 192.168.10.1 до 192.168.20.1: 

Что видно на router2:

root@router2:~# tcpdump -i enp0s9
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on enp0s9, link-type EN10MB (Ethernet), capture size 262144 bytes
10:29:35.364767 IP 192.168.10.1 > router2: ICMP echo request, id 5, seq 22, length 64
10:29:35.364795 IP router2 > 192.168.10.1: ICMP echo reply, id 5, seq 22, length 64
10:29:36.379259 IP 192.168.10.1 > router2: ICMP echo request, id 5, seq 23, length 64
10:29:36.379288 IP router2 > 192.168.10.1: ICMP echo reply, id 5, seq 23, length 64
10:29:37.380811 IP 192.168.10.1 > router2: ICMP echo request, id 5, seq 24, length 64
10:29:37.380836 IP router2 > 192.168.10.1: ICMP echo reply, id 5, seq 24, length 64
10:29:38.382404 IP 192.168.10.1 > router2: ICMP echo request, id 5, seq 25, length 64
10:29:38.382437 IP router2 > 192.168.10.1: ICMP echo reply, id 5, seq 25, length 64
10:29:38.434530 IP 10.0.12.2 > ospf-all.mcast.net: OSPFv2, Hello, length 44
10:29:38.440219 IP 10.0.11.1 > ospf-all.mcast.net: OSPFv2, Hello, length 48
10:29:39.383933 IP 192.168.10.1 > router2: ICMP echo request, id 5, seq 26, length 64
10:29:39.383963 IP router2 > 192.168.10.1: ICMP echo reply, id 5, seq 26, length 64
10:29:40.385299 IP 192.168.10.1 > router2: ICMP echo request, id 5, seq 27, length 64
10:29:40.385322 IP router2 > 192.168.10.1: ICMP echo reply, id 5, seq 27, length 64
10:29:41.003327 IP router2 > ospf-all.mcast.net: OSPFv2, Hello, length 48
10:29:41.386747 IP 192.168.10.1 > router2: ICMP echo request, id 5, seq 28, length 64
10:29:41.386775 IP router2 > 192.168.10.1: ICMP echo reply, id 5, seq 28, length 64

Теперь мы видим, что трафик между роутерами ходит симметрично.























