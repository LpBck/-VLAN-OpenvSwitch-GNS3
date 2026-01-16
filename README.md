# Аннотация
Писал изначально сей текст для себя, но решил закинуть на github, чтобы научится с ним взаимодействовать.
**Здесь мы соберем следующую схему:**
<img width="1410" height="801" alt="Рисунок 1" src="https://github.com/user-attachments/assets/a85edc2b-edc4-4735-9318-8245a5c14b17" />
Как видно из рисунка:
- VM1 и VM2 принадлежат к VLAN 10(подсеть **10.11.0.0/24**). Обе машины подключены к OVS1
- VM3 принадлежит к VLAN 20(подсеть **10.12.0.0/24**) и подключена к OVS1
- VM4 принадлежит к VLAN 20(подсеть **10.12.0.0/24**) и подключена к OVS2
- OVS1 и OVS2 связаны между собой trunk портами. [Хорошая статья, покрывающая теоретическую часть технологии VLAN, разницу между access и trunk портами](https://github.com/sonic-net/SONiC/blob/master/doc/vlan/switchport-mode-support/Switchport%20Mode%20and%20VLAN%20CLI%20Enhancement.md)
# Настройка IP адресов на VM-ах
Мне прикольно выставлять сетевые настройки через файл `/etc/network/interfaces`.
### VM1
Откроем файл через nano
```bash
firstgeorgedroid@debian:/# nano /etc/network/interfaces
```
в нем пропишем следующее:
```bash
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

# Настройка статического адреса на интерфейсе ens4
auto ens4 # Интерфейс автоматически переходит в UP
iface ens4 inet static # Статические ipv4 настройки для ens4
        address 10.11.0.1
        netmask 255.255.255.0
```
### VM2
```bash
secondgeorgedroid@debian:/# nano /etc/network/interfaces
```

```bash
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

# Static config for ens4
auto ens4
iface ens4 inet static
        address 10.11.0.2
        netmask 255.255.255.0
```

### VM3
```bash
firstfentanylenjoyer@debian:/# nano /etc/network/interfaces
```

```bash

# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

# Static config for ens4
auto ens4
iface ens4 inet static
        address 10.12.0.1
        netmask 255.255.255
```

### VM4
```bash
secondfentanylenjoyer@debian:/# nano /etc/network/interfaces
```

```bash

# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

# Static config for ens4
auto ens4
iface ens4 inet static
        address 10.12.0.2
        netmask 255.255.255.0
```
# Настройка OpenvSwitch

### OVS1
Так как все порты OpenvSwitch уже добавлены в OVS бридж br0, будем использовать команду `ovs-vsctl set port` для перевода портов eth0, eth1 и eth2 в режим access и присвоения им соответствующих VLAN-ID
```bash
ovs-vsctl set port eth0 tag=10
ovs-vsctl set port eth1 tag=10
ovs-vsctl set port eth2 tag=20
```

**Настроим eth10 на OpenvSwitch в режиме trunk**, разрешив пропуск тегов с VLAN-ID 10 и 20 (вдруг захочется добавит к OVS2 VLAN 10, если хочется строго по схеме — оставляйте только VLAN-ID 10)
```bash
ovs-vsctl set port eth10 trunks=10,20
```

**Проверим конфигурацию** командой `ovs-vsctl show`:
```bash
OpenvSwitch-1:/$ ovs-vsctl show
e6abe8ab-1dac-45a6-b69c-3f9caa3a00eb
    Bridge br0
        datapath_type: netdev
        Port eth2
            tag: 20
            Interface eth2
        ....
        Port eth10
            trunks: [10, 20]
            Interface eth10
        .....
        Port eth1
            tag: 10
            Interface eth1
        Port eth0
            tag: 10
            Interface eth0
        ....
```

### OVS2
Переведем порт eth0 в режим *access*
```bash
ovs-vsctl set port eth0 tag=20
```
Настроим eth10 на OpenvSwitch в режиме trunk
```bash
vs-vsctl set port eth3 trunks=10,20
```
Проверим настройки
```bash
OpenvSwitch-2:/$ ovs-vsctl show
d3734852-23ea-4345-9e5a-612d98acd18e
    Bridge br0
        datapath_type: netdev
       ......
        Port eth0
            tag: 20
            Interface eth0
        ......
        Port eth10
            trunks: [10, 20]
            Interface eth10

```

# Проверим
Запустим **ping** на **VM2** до **VM1**
```bash
secondgeorgedroid@debian:/home/debian# ping -c 3 10.11.0.1
PING 10.11.0.1 (10.11.0.1) 56(84) bytes of data.
64 bytes from 10.11.0.1: icmp_seq=1 ttl=64 time=7.04 ms
64 bytes from 10.11.0.1: icmp_seq=2 ttl=64 time=4.19 ms
64 bytes from 10.11.0.1: icmp_seq=3 ttl=64 time=3.36 ms

--- 10.11.0.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2004ms
rtt min/avg/max/mdev = 3.359/4.861/7.035/1.573 ms
```
Работает, как надо

Запустим **ping** на **VM2** до **VM3**
```bash
secondgeorgedroid@debian:/home/debian# ping -c 3 10.12.0.1
PING 10.12.0.1 (10.12.0.1) 56(84) bytes of data.
From 10.11.0.2 icmp_seq=1 Destination Host Unreachable
From 10.11.0.2 icmp_seq=2 Destination Host Unreachable
From 10.11.0.2 icmp_seq=3 Destination Host Unreachable

--- 10.12.0.1 ping statistics ---
3 packets transmitted, 0 received, +3 errors, 100% packet loss, time 2044ms
pipe 3
```
Работает, как надо

Запустим **ping** на **VM3** до **VM4**
```bash
firstfentanylenjoyer@debian:/home/debian# ping -c 3 10.12.0.2
PING 10.12.0.2 (10.12.0.2) 56(84) bytes of data.
64 bytes from 10.12.0.2: icmp_seq=1 ttl=64 time=10.2 ms
64 bytes from 10.12.0.2: icmp_seq=2 ttl=64 time=8.26 ms
64 bytes from 10.12.0.2: icmp_seq=3 ttl=64 time=8.99 ms

--- 10.12.0.2 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2005ms
rtt min/avg/max/mdev = 8.262/9.147/10.185/0.792 ms
```
Работает, как надо

# Итог
Простенькая схемка собрана и работает, маленькая победа дня
