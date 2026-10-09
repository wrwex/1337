# Задание 1 Команды и порядок настройки ВМ

ALT Linux: ISP, HQ-SW, HQ-SRV, BR-SRV, HQ-CLI. EcoRouter 3.2.6.2: HQ-RTR, BR-RTR. Ideco NGFW Novum 21.7.130: BR-FW.

Выполнять этапы 1–12 последовательно через консоли соответствующих ВМ. Linux-команды вводить в Bash от root; EcoRouter-команды — в CLI маршрутизатора. На Ideco выполнять указанные действия в веб-интерфейсе.

Имена интерфейсов сверить перед настройкой: ALT Linux — `ip -br link`, EcoRouter — `show port`, Ideco — MAC-адреса сетевых карт. При отличиях заменить имена во всех командах устройства. Для площадки в Москве использовать Europe/Moscow и UTC+3; для другой площадки — ее часовой пояс.

Схема использует GRE HQ-RTR–BR-RTR и GRE BR-RTR–BR-FW. Подсети двухточечных соединений — /31. До настройки подтвердить выбранную схему OSPF с экспертами и поддержку /31 на обоих концах.

### Подключение портов

| ВМ | Порт в примере | Куда подключен |
|---|---|---|
| ISP | ens3 | Internet, DHCP |
| ISP | ens4 | HQ-RTR |
| ISP | ens5 | BR-RTR |
| HQ-RTR | ge0 | ISP, без VLAN |
| HQ-RTR | ge1 | HQ-SW, tagged VLAN 100/200/999 |
| BR-RTR | ge0 | ISP, без VLAN |
| BR-RTR | ge1 | BR-FW, без VLAN |
| HQ-SW | ens3 | trunk к HQ-RTR |
| HQ-SW | ens4 | access к HQ-SRV |
| HQ-SW | ens5 | access к HQ-CLI |
| HQ-SRV, BR-SRV, HQ-CLI | ens3 | соответствующая локальная сеть |
| BR-FW | физический порт FW-NET | BR-RTR, локальный Ethernet |
| BR-FW | физический порт BR-NET | BR-SRV, локальный Ethernet |

### Адреса устройств

| Устройство | Интерфейс / роль | IPv4 | Шлюз по умолчанию |
|---|---|---|---|
| ISP | ens3 Internet | DHCP провайдера | DHCP провайдера |
| ISP | ens4 ISP-HQ | 172.16.1.1/28 | — |
| ISP | ens5 ISP-BR | 172.16.2.1/28 | — |
| HQ-RTR | isp | 172.16.1.2/28 | 172.16.1.1 |
| HQ-RTR | vl100 | 10.10.100.1/27 | — |
| HQ-RTR | vl200 | 10.10.200.1/28 | — |
| HQ-RTR | vl999 | 10.10.30.1/29 | — |
| HQ-RTR | tunnel.0 | 10.10.10.0/31 | — |
| HQ-SW | mgmt999 | 10.10.30.2/29 | 10.10.30.1 |
| HQ-SRV | ens3 | 10.10.100.2/27 | 10.10.100.1 |
| HQ-CLI | ens3, DHCP | 10.10.200.3/28 | 10.10.200.1 |
| BR-RTR | isp | 172.16.2.2/28 | 172.16.2.1 |
| BR-RTR | br | 10.20.30.0/31 | — |
| BR-RTR | tunnel.0 | 10.10.10.1/31 | — |
| BR-RTR | tunnel.1 | 10.10.10.2/31 | — |
| BR-FW | FW-NET | 10.20.30.1/31 | 10.20.30.0 |
| BR-FW | GRE к BR-RTR | 10.10.10.3/31 | — |
| BR-FW | BR-NET | 10.20.20.1/28 | — |
| BR-SRV | ens3 | 10.20.20.2/28 | 10.20.20.1 |

## 1 ISP — ALT Linux

### 1.1 Проверить интерфейсы и задать имя и часовой пояс

Сетевую конфигурацию ISP обслуживает etcnet. Перед записью файлов переключить выбранные карты на etcnet; одна карта должна управляться одним сетевым менеджером.

```bash
su -
cat /etc/os-release
ip -br link
ip -br address
ip route
systemctl is-active network
systemctl is-active NetworkManager
systemctl is-active systemd-networkd
mkdir -p /root/kim-backup
tar -czf /root/kim-backup/isp-etcnet-before.tgz /etc/net /etc/sysconfig/network
hostnamectl set-hostname isp.au-team.irpo
timedatectl set-timezone Europe/Moscow
```

```bash
if grep -q '^HOSTNAME=' /etc/sysconfig/network; then
  sed -i 's/^HOSTNAME=.*/HOSTNAME=isp.au-team.irpo/' /etc/sysconfig/network
else
  printf '%s\n' 'HOSTNAME=isp.au-team.irpo' >> /etc/sysconfig/network
fi
```

### 1.2 Установить пакеты и настроить WAN DHCP и офисные интерфейсы

ens3 получает адрес, маршрут по умолчанию и DNS от провайдера. ens4 и ens5 имеют статические адреса. Пакеты установить через действующий доступ к репозиторию или экзаменационный носитель. Старые статические адреса и маршруты выбранных карт предварительно сохранить и убрать из действующей конфигурации.

```bash
apt-get update
apt-get install -y etcnet iptables
mkdir -p /etc/net/ifaces/ens3 /etc/net/ifaces/ens4 /etc/net/ifaces/ens5
cat > /etc/net/ifaces/ens3/options <<'EOF'
TYPE=eth
BOOTPROTO=dhcp
ONBOOT=yes
DISABLED=no
CONFIG_IPV4=yes
CONFIG_IPV6=no
CONFIG_WIRELESS=no
NM_CONTROLLED=no
SYSTEMD_CONTROLLED=no
EOF
for n in ens4 ens5; do
  cat > /etc/net/ifaces/$n/options <<'EOF'
TYPE=eth
BOOTPROTO=static
ONBOOT=yes
DISABLED=no
CONFIG_IPV4=yes
CONFIG_IPV6=no
CONFIG_WIRELESS=no
NM_CONTROLLED=no
SYSTEMD_CONTROLLED=no
EOF
done
printf '%s\n' '172.16.1.1/28' > /etc/net/ifaces/ens4/ipv4address
printf '%s\n' '172.16.2.1/28' > /etc/net/ifaces/ens5/ipv4address
```

### 1.3 Применить адресацию и проверить выход в Интернет

```bash
ls -l /etc/net/ifaces/ens3 /etc/net/ifaces/ens4 /etc/net/ifaces/ens5
systemctl enable network
systemctl restart network
ip -br address
ip route
cat /etc/resolv.conf
ping -c 3 77.88.8.8
```

Проверить наличие WAN-адреса, маршрута default и DNS от DHCP. При отсутствии параметров восстановить DHCP-подключение к провайдеру до следующего этапа.

### 1.4 Включить маршрутизацию, PAT и транзит между офисными WAN

Правила iptables разрешают исходящий трафик и обмен WAN-адресов роутеров. Полный снимок правил сохраняется в файл, который загружает iptables.service.

```bash
cat > /etc/sysctl.d/90-kim-forward.conf <<'EOF'
net.ipv4.ip_forward = 1
EOF
sysctl --system
sysctl net.ipv4.ip_forward
systemctl enable --now iptables
iptables-save > /root/kim-backup/isp-iptables-before.rules
iptables -t nat -C POSTROUTING -s 172.16.1.0/28 -o ens3 -j MASQUERADE || iptables -t nat -A POSTROUTING -s 172.16.1.0/28 -o ens3 -j MASQUERADE
iptables -t nat -C POSTROUTING -s 172.16.2.0/28 -o ens3 -j MASQUERADE || iptables -t nat -A POSTROUTING -s 172.16.2.0/28 -o ens3 -j MASQUERADE
iptables -C FORWARD -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT || iptables -I FORWARD 1 -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -C FORWARD -i ens4 -o ens3 -s 172.16.1.0/28 -j ACCEPT || iptables -I FORWARD 1 -i ens4 -o ens3 -s 172.16.1.0/28 -j ACCEPT
iptables -C FORWARD -i ens5 -o ens3 -s 172.16.2.0/28 -j ACCEPT || iptables -I FORWARD 1 -i ens5 -o ens3 -s 172.16.2.0/28 -j ACCEPT
iptables -C FORWARD -i ens4 -o ens5 -s 172.16.1.2 -d 172.16.2.2 -j ACCEPT || iptables -I FORWARD 1 -i ens4 -o ens5 -s 172.16.1.2 -d 172.16.2.2 -j ACCEPT
iptables -C FORWARD -i ens5 -o ens4 -s 172.16.2.2 -d 172.16.1.2 -j ACCEPT || iptables -I FORWARD 1 -i ens5 -o ens4 -s 172.16.2.2 -d 172.16.1.2 -j ACCEPT
iptables-save > /etc/sysconfig/iptables
iptables -t nat -L POSTROUTING -n -v
iptables -L FORWARD -n -v
systemctl is-enabled iptables
```

## 2 HQ-RTR — EcoRouter — базовая настройка

Создать net_admin с ролью admin, интерфейс ISP и VLAN 100/200/999. Привязать три VLAN к одному ge1. Настроить default route и PAT.

```text
enable
show version
show port
show interface
show running-config
configure terminal
hostname hq-rtr.au-team.irpo
ip domain-name au-team.irpo
ip name-server 77.88.8.8
ntp timezone UTC+3
username net_admin
password P@ssw0rd
role admin
activate
exit
interface isp
ip address 172.16.1.2/28
ip nat outside
exit
interface vl100
ip address 10.10.100.1/27
ip nat inside
exit
interface vl200
ip address 10.10.200.1/28
ip nat inside
exit
interface vl999
ip address 10.10.30.1/29
ip nat inside
exit
port ge0
service-instance ISP
encapsulation untagged
connect ip interface isp
exit
exit
port ge1
service-instance VLAN100
encapsulation dot1q 100 exact
rewrite pop 1
connect ip interface vl100
exit
service-instance VLAN200
encapsulation dot1q 200 exact
rewrite pop 1
connect ip interface vl200
exit
service-instance VLAN999
encapsulation dot1q 999 exact
rewrite pop 1
connect ip interface vl999
exit
exit
ip route 0.0.0.0/0 172.16.1.1
ip nat pool VLAN100 10.10.100.1-10.10.100.30
ip nat pool VLAN200 10.10.200.1-10.10.200.14
ip nat pool VLAN999 10.10.30.1-10.10.30.6
ip nat source dynamic inside-to-outside pool VLAN100 overload interface isp
ip nat source dynamic inside-to-outside pool VLAN200 overload interface isp
ip nat source dynamic inside-to-outside pool VLAN999 overload interface isp
exit
write memory
show hostname
show users localdb
show ip route
show interface
ping 172.16.1.1
ping 77.88.8.8
```

Если CLI показывает другую форму команды, проверить доступные параметры через `?` в текущем контексте. Для PAT сохранить пул исходных адресов, overload и интерфейс isp. На сборке с ключевым словом inside использовать его вместо inside-to-outside.

## 3 BR-RTR — EcoRouter — базовая настройка

Создать net_admin с ролью admin. ge0 соединить с ISP, ge1 — с BR-FW. Настроить адресацию и default route.

```text
enable
show version
show port
show running-config
configure terminal
hostname br-rtr.au-team.irpo
ip domain-name au-team.irpo
ip name-server 77.88.8.8
ntp timezone UTC+3
username net_admin
password P@ssw0rd
role admin
activate
exit
interface isp
ip address 172.16.2.2/28
ip nat outside
exit
interface br
ip address 10.20.30.0/31
ip nat inside
exit
port ge0
service-instance ISP
encapsulation untagged
connect ip interface isp
exit
exit
port ge1
service-instance FW
encapsulation untagged
connect ip interface br
exit
exit
ip route 0.0.0.0/0 172.16.2.1
exit
write memory
show ip route
ping 172.16.2.1
ping 172.16.1.2
ping 77.88.8.8
```

## 4 HQ-SW — ALT Linux

### 4.1 Подготовить физические карты

Проверить наличие ip и bridge. При отсутствии установить iproute2 до смены сети. Карты ens3/ens4/ens5 использовать без собственного IPv4 и DHCP; управление будет на mgmt999.

```bash
su -
cat /etc/os-release
ip -br link
mkdir -p /root/kim-backup
tar -czf /root/kim-backup/hq-sw-etcnet-before.tgz /etc/net /etc/sysconfig/network
hostnamectl set-hostname hq-sw.au-team.irpo
timedatectl set-timezone Europe/Moscow
command -v ip
command -v bridge
```

```bash
if grep -q '^HOSTNAME=' /etc/sysconfig/network; then
  sed -i 's/^HOSTNAME=.*/HOSTNAME=hq-sw.au-team.irpo/' /etc/sysconfig/network
else
  printf '%s\n' 'HOSTNAME=hq-sw.au-team.irpo' >> /etc/sysconfig/network
fi
for n in ens3 ens4 ens5; do
  mkdir -p /etc/net/ifaces/$n
  cat > /etc/net/ifaces/$n/options <<'EOF'
TYPE=eth
BOOTPROTO=static
ONBOOT=yes
DISABLED=no
CONFIG_IPV4=no
CONFIG_IPV6=no
CONFIG_WIRELESS=no
NM_CONTROLLED=no
SYSTEMD_CONTROLLED=no
EOF
done
```

### 4.2 Создать VLAN-мост и службу автозапуска

ens3 — tagged VLAN 100/200/999; ens4 — access VLAN 100; ens5 — access VLAN 200. mgmt999 получает 10.10.30.2/29. Для существующего br0 перед применением проверить и привести членство VLAN к этим значениям.

```bash
install -d /usr/local/sbin
cat > /usr/local/sbin/kim-hq-switch <<'EOF'
#!/bin/bash
set -euo pipefail
ip link show br0 >/dev/null 2>&1 || ip link add br0 type bridge vlan_filtering 1 vlan_default_pvid 0
ip link set br0 type bridge vlan_filtering 1 vlan_default_pvid 0
for n in ens3 ens4 ens5; do
  ip link set "$n" master br0
  ip link set "$n" up
done
bridge vlan add dev ens3 vid 100
bridge vlan add dev ens3 vid 200
bridge vlan add dev ens3 vid 999
bridge vlan add dev ens4 vid 100 pvid untagged
bridge vlan add dev ens5 vid 200 pvid untagged
bridge vlan add dev br0 vid 999 self
ip link set br0 up
ip link show mgmt999 >/dev/null 2>&1 || ip link add link br0 name mgmt999 type vlan id 999
ip address replace 10.10.30.2/29 dev mgmt999
ip link set mgmt999 up
ip route replace default via 10.10.30.1 dev mgmt999
EOF
chmod 0755 /usr/local/sbin/kim-hq-switch
cat > /etc/systemd/system/kim-hq-switch.service <<'EOF'
[Unit]
Description=HQ switch VLAN 100 200 999
Requires=network.service
After=network.service

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/kim-hq-switch
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF
bash -n /usr/local/sbin/kim-hq-switch
systemctl daemon-reload
systemctl enable network
systemctl restart network
systemctl enable --now kim-hq-switch.service
bridge vlan show
ip -br address
ip route
ping -c 3 10.10.30.1
```

После ручного перезапуска network повторно выполнить `systemctl restart kim-hq-switch.service`.

## 5 BR-FW — Ideco 21.7.130

### 5.1 Настроить Ethernet и маршрут по умолчанию

1. Через локальную консоль узнать адрес управления и войти в веб-интерфейс Ideco.
2. Создать резервную копию настроек.
3. Задать имя `br-fw.au-team.irpo` и часовой пояс площадки.
4. В **Сервисы → Сетевые интерфейсы → Интерфейсы** создать локальный Ethernet `FW-NET`: карта к BR-RTR, адрес `10.20.30.1/31`.
5. Создать локальный Ethernet `BR-NET`: карта к BR-SRV, адрес `10.20.20.1/28`.
6. В **Маршрутизация → Статическая** добавить действующий маршрут `0.0.0.0/0` через `10.20.30.0` на FW-NET. При использовании VCE выбрать контекст с этими интерфейсами.
7. Для начальной установки задать DNS `77.88.8.8`.

### 5.2 Создать GRE к BR-RTR

В разделе интерфейсов добавить и включить GRE `GRE-BR-RTR`:

| Поле | Значение |
|---|---|
| Локальный транспортный адрес source | 10.20.30.1 |
| Удаленный транспортный адрес destination | 10.20.30.0 |
| IPv4 внутри GRE | 10.10.10.3/31 |
| Удаленный IPv4 внутри GRE, если поле присутствует | 10.10.10.2 |
| MTU | 1400 |

### 5.3 Настроить OSPF

В **Маршрутизация → OSPF** установить:

| Параметр | Значение |
|---|---|
| Router ID | уникальный ID BR-FW; проверить автоматически выбранное значение |
| Аутентификация | MD5 |
| Key ID | 1 |
| Пароль | P@ssw0rd |
| Активный интерфейс | только GRE-BR-RTR |
| Area | 0 |
| Тип области | Normal |
| Тип сети | point-to-point |
| Hello / Dead | 10 / 40 секунд |
| Redistribute connected | включить |
| Redistribute default | выключить |
| Redistribute static | выключить |
| Анонсируемые сети | 10.20.20.0/28, 10.20.30.0/31, 10.10.10.2/31 |
| Входящие сети | офисные сети; исключить 0.0.0.0/0 |

Сохранить параметры и включить OSPF. Ethernet FW-NET и BR-NET не добавлять как активные OSPF-интерфейсы. После этапа 7 проверить FULL с BR-RTR и маршруты HQ через GRE.

### 5.4 Настроить правила трафика

В **Правила трафика → Файрвол** создать и включить разрешающие правила:

| Поток | Источник | Назначение / протокол |
|---|---|---|
| GRE к самому BR-FW | 10.20.30.0 на FW-NET | BR-FW 10.20.30.1, IP-протокол 47 |
| OSPF в GRE | BR-RTR на GRE-BR-RTR | BR-FW и OSPF multicast, IP-протокол 89 |
| HQ → BR-SRV | 10.10.100.0/27, 10.10.200.0/28, 10.10.30.0/29 | 10.20.20.0/28 |
| BR-SRV → HQ | 10.20.20.0/28 | три HQ-подсети через GRE |
| BR-SRV → Интернет | 10.20.20.0/28 | выход FW-NET к BR-RTR |
| Диагностика BR-FW | адреса соседей и управления стенда | ICMP к устройству |

Правила к устройству применять к локальному входу; правила между сетями — к транзиту. Разрешить ответный трафик через штатное отслеживание состояний. Сохранить исходные адреса трафика BR-NET→GRE и BR-NET→FW-NET: SNAT на этих потоках не применять. PAT для Интернета выполняет BR-RTR. Включить нужную интернет-авторизацию объекта BR-SRV штатными средствами Ideco, если она требуется настройками лицензии; сохранить указанный режим без SNAT.

Проверить доступ к 10.20.30.0 и 10.20.20.2 через диагностику Ideco. Сохранить настройки.

## 6 HQ-RTR — EcoRouter — GRE и OSPF

После создания офисных WAN-интерфейсов настроить GRE к BR-RTR. Объявить HQ-сети в OSPF, включить MD5, оставить активным только tunnel.0.

```text
enable
ping 172.16.2.2
configure terminal
interface tunnel.0
description GRE-HQ-BR
ip address 10.10.10.0/31
ip mtu 1400
ip tunnel 172.16.1.2 172.16.2.2 mode gre
ip ospf network point-to-point
ip ospf authentication message-digest
ip ospf message-digest-key 1 md5 P@ssw0rd
exit
router ospf 1
ospf router-id 172.16.1.2
passive-interface default
no passive-interface tunnel.0
network 10.10.10.0/31 area 0
network 10.10.100.0/27 area 0
network 10.10.200.0/28 area 0
network 10.10.30.0/29 area 0
exit
exit
write memory
show interface tunnel.0
show ip ospf neighbor
show ip route
ping 10.10.10.1
```

## 7 BR-RTR — EcoRouter — GRE, OSPF и PAT

Проверить физическую доступность BR-FW командой `ping 10.20.30.1`. Создать tunnel.0 к HQ-RTR и tunnel.1 к BR-FW. Настроить MD5, объявить туннельные сети и создать PAT филиала.

```text
enable
configure terminal
interface tunnel.0
description GRE-BR-HQ
ip address 10.10.10.1/31
ip mtu 1400
ip tunnel 172.16.2.2 172.16.1.2 mode gre
ip ospf network point-to-point
ip ospf authentication message-digest
ip ospf message-digest-key 1 md5 P@ssw0rd
exit
interface tunnel.1
description GRE-BR-FW
ip address 10.10.10.2/31
ip mtu 1400
ip tunnel 10.20.30.0 10.20.30.1 mode gre
ip nat inside
ip ospf network point-to-point
ip ospf authentication message-digest
ip ospf message-digest-key 1 md5 P@ssw0rd
exit
router ospf 1
ospf router-id 172.16.2.2
passive-interface default
no passive-interface tunnel.0
no passive-interface tunnel.1
network 10.10.10.0/31 area 0
network 10.10.10.2/31 area 0
exit
ip nat pool BR-SRV-NET 10.20.20.1-10.20.20.14
ip nat pool BR-FW-NET 10.20.30.0-10.20.30.1
ip nat source dynamic inside-to-outside pool BR-SRV-NET overload interface isp
ip nat source dynamic inside-to-outside pool BR-FW-NET overload interface isp
exit
write memory
show interface tunnel.0
show interface tunnel.1
show ip ospf neighbor
show ip route
ping 10.10.10.0
ping 10.10.10.3
```

Проверить два соседства FULL: HQ-RTR на tunnel.0 и BR-FW на tunnel.1. На Ideco должны появиться маршруты HQ, на HQ-RTR — маршрут 10.20.20.0/28.

## 8 HQ-SRV — ALT Linux

### 8.1 Настроить адрес, имя, часовой пояс и пакеты

Адрес 10.10.100.2/27, шлюз 10.10.100.1. Использовать etcnet для ens3. Временный DNS 77.88.8.8 используется для установки пакетов.

```bash
su -
cat /etc/os-release
ip -br link
mkdir -p /root/kim-backup
tar -czf /root/kim-backup/hq-srv-network-before.tgz /etc/net /etc/sysconfig/network
hostnamectl set-hostname hq-srv.au-team.irpo
timedatectl set-timezone Europe/Moscow
if grep -q '^HOSTNAME=' /etc/sysconfig/network; then
  sed -i 's/^HOSTNAME=.*/HOSTNAME=hq-srv.au-team.irpo/' /etc/sysconfig/network
else
  printf '%s\n' 'HOSTNAME=hq-srv.au-team.irpo' >> /etc/sysconfig/network
fi
mkdir -p /etc/net/ifaces/ens3
cat > /etc/net/ifaces/ens3/options <<'EOF'
TYPE=eth
BOOTPROTO=static
ONBOOT=yes
DISABLED=no
CONFIG_IPV4=yes
CONFIG_IPV6=no
CONFIG_WIRELESS=no
NM_CONTROLLED=no
SYSTEMD_CONTROLLED=no
EOF
printf '%s\n' '10.10.100.2/27' > /etc/net/ifaces/ens3/ipv4address
printf '%s\n' 'default via 10.10.100.1' > /etc/net/ifaces/ens3/ipv4route
printf '%s\n' 'nameserver 77.88.8.8' > /etc/net/ifaces/ens3/resolv.conf
systemctl enable network
systemctl restart network
ping -c 3 10.10.100.1
ping -c 3 77.88.8.8
apt-get update
apt-get install -y sudo openssh-server bind bind-utils mc
```

### 8.2 Создать sshuser и разрешить sudo

Проверить имя sshuser и UID 2027. Если обе записи отсутствуют, выполнить useradd. Для существующего sshuser UID должен быть 2027; занятый другим пользователем UID предварительно разрешить вручную.

```bash
getent passwd sshuser
getent passwd 2027
```

```bash
useradd -m -u 2027 -s /bin/bash sshuser
passwd sshuser
```

В обоих запросах passwd ввести `P@ssw0rd`.

```bash
install -d -m 0750 /etc/sudoers.d
printf '%s\n' 'sshuser ALL=(ALL:ALL) NOPASSWD: ALL' > /etc/sudoers.d/sshuser
chmod 0440 /etc/sudoers.d/sshuser
visudo -c
id sshuser
su - sshuser -c 'sudo -n id'
```

Если sudoers.d не подключен штатным sudoers, открыть `visudo`, добавить директиву `#includedir /etc/sudoers.d` и повторить `visudo -c` и `sudo -n id`.

### 8.3 Настроить SSH

SSH слушает 2027, допускает sshuser, ограничивает попытки входа до 2 и показывает баннер. Проверку sshd -t завершить успешно до перезапуска.

```bash
cp -a /etc/openssh/sshd_config /root/kim-backup/hq-srv-sshd_config.before
ssh-keygen -A
printf '%s\n' 'Authorized access only' > /etc/openssh/banner
cat > /etc/openssh/sshd_config <<'EOF'
Port 2027
AddressFamily inet
ListenAddress 0.0.0.0
PermitRootLogin no
PubkeyAuthentication yes
PasswordAuthentication yes
UsePAM yes
AllowUsers sshuser
MaxAuthTries 2
Banner /etc/openssh/banner
Subsystem sftp internal-sftp
EOF
sshd -t
sshd -T | grep -E '^(port|allowusers|maxauthtries|banner|permitrootlogin) '
systemctl enable --now sshd
systemctl restart sshd
ss -lntp | grep ':2027'
```

Если на сервере активен iptables, разрешить SSH от офисных сетей и сохранить правила управляющим сервисом:

```bash
for s in 10.10.100.0/27 10.10.200.0/28 10.10.30.0/29 10.20.20.0/28; do
  iptables -C INPUT -s "$s" -p tcp --dport 2027 -j ACCEPT || iptables -I INPUT 1 -s "$s" -p tcp --dport 2027 -j ACCEPT
done
```

### 8.4 Настроить параметры BIND

Сохранить конфигурацию, проверить путь/chroot службы и открыть options.conf.

```bash
tar -czf /root/kim-backup/hq-srv-bind-before.tgz /var/lib/bind/etc /etc/bind
systemctl cat bind.service
ls -l /etc/bind/named.conf /var/lib/bind/etc/named.conf
mcedit /var/lib/bind/etc/options.conf
```

В штатном блоке options установить следующие директивы без дублирования. Сохранить файл F2, выйти F10.

```text
listen-on { 127.0.0.1; 10.10.100.2; };
listen-on-v6 { none; };
forward only;
forwarders { 77.88.8.8; };
recursion yes;
allow-query { 127.0.0.1; 10.10.100.0/27; 10.10.200.0/28; 10.10.30.0/29; 10.20.20.0/28; 10.20.30.0/31; 10.10.10.0/24; 172.16.1.0/28; 172.16.2.0/28; };
allow-recursion { 127.0.0.1; 10.10.100.0/27; 10.10.200.0/28; 10.10.30.0/29; 10.20.20.0/28; 10.20.30.0/31; 10.10.10.0/24; 172.16.1.0/28; 172.16.2.0/28; };
allow-query-cache { 127.0.0.1; 10.10.100.0/27; 10.10.200.0/28; 10.10.30.0/29; 10.20.20.0/28; 10.20.30.0/31; 10.10.10.0/24; 172.16.1.0/28; 172.16.2.0/28; };
```

### 8.5 Объявить зоны

```bash
mcedit /var/lib/bind/etc/rfc1912.conf
```

Добавить эти три определения по одному в rfc1912.conf; сохранить штатные localhost-зоны. Относительные file должны указывать в каталог зон текущего directory внутри chroot.

```text
zone "au-team.irpo" {
    type master;
    file "au-team.irpo";
};
zone "100.10.10.in-addr.arpa" {
    type master;
    file "100.10.10.in-addr.arpa";
};
zone "20.20.10.in-addr.arpa" {
    type master;
    file "20.20.10.in-addr.arpa";
};
```

### 8.6 Записать зоны и проверить их

Создать прямую зону и PTR-зоны двух серверов. Полные имена SOA, NS и PTR завершаются точкой.

```bash
install -d -m 0750 -o root -g named /var/lib/bind/etc/zone
cat > /var/lib/bind/etc/zone/au-team.irpo <<'EOF'
$TTL 3600
@ IN SOA hq-srv.au-team.irpo. hostmaster.au-team.irpo. (
  2026100901 3600 900 604800 3600
)
@      IN NS hq-srv.au-team.irpo.
hq-rtr IN A 10.10.100.1
br-rtr IN A 10.20.30.0
br-fw  IN A 10.20.20.1
hq-srv IN A 10.10.100.2
hq-cli IN A 10.10.200.3
br-srv IN A 10.20.20.2
docker IN A 172.16.1.1
web    IN A 172.16.2.1
hq-sw  IN A 10.10.30.2
EOF
cat > /var/lib/bind/etc/zone/100.10.10.in-addr.arpa <<'EOF'
$TTL 3600
@ IN SOA hq-srv.au-team.irpo. hostmaster.au-team.irpo. (
  2026100901 3600 900 604800 3600
)
@ IN NS hq-srv.au-team.irpo.
2 IN PTR hq-srv.au-team.irpo.
EOF
cat > /var/lib/bind/etc/zone/20.20.10.in-addr.arpa <<'EOF'
$TTL 3600
@ IN SOA hq-srv.au-team.irpo. hostmaster.au-team.irpo. (
  2026100901 3600 900 604800 3600
)
@ IN NS hq-srv.au-team.irpo.
2 IN PTR br-srv.au-team.irpo.
EOF
chown root:named /var/lib/bind/etc/zone/au-team.irpo /var/lib/bind/etc/zone/100.10.10.in-addr.arpa /var/lib/bind/etc/zone/20.20.10.in-addr.arpa
chmod 0640 /var/lib/bind/etc/zone/au-team.irpo /var/lib/bind/etc/zone/100.10.10.in-addr.arpa /var/lib/bind/etc/zone/20.20.10.in-addr.arpa
named-checkzone au-team.irpo /var/lib/bind/etc/zone/au-team.irpo
named-checkzone 100.10.10.in-addr.arpa /var/lib/bind/etc/zone/100.10.10.in-addr.arpa
named-checkzone 20.20.10.in-addr.arpa /var/lib/bind/etc/zone/20.20.10.in-addr.arpa
```

Проверить весь конфигурационный файл. Для штатного chroot /var/lib/bind и файла /etc/named.conf внутри него выполнить:

```bash
named-checkconf -t /var/lib/bind /etc/named.conf
```

Если служба использует другой путь, взять его из systemctl cat bind.service для этой проверки.

### 8.7 Разрешить DNS при активном iptables

Выполнить этот блок только если iptables является управляющим firewall сервера; сохранить правила используемым сервисом.

```bash
for s in 10.10.100.0/27 10.10.200.0/28 10.10.30.0/29 10.20.20.0/28 10.20.30.0/31 10.10.10.0/24 172.16.1.0/28 172.16.2.0/28; do
  for p in udp tcp; do
    iptables -C INPUT -s "$s" -p "$p" --dport 53 -j ACCEPT || iptables -I INPUT 1 -s "$s" -p "$p" --dport 53 -j ACCEPT
  done
done
```

### 8.8 Запустить DNS и назначить серверу собственный resolver

```bash
systemctl enable --now bind.service
systemctl restart bind.service
systemctl status bind.service --no-pager
journalctl -u bind.service -n 40 --no-pager
ss -lntup | grep ':53'
dig @127.0.0.1 hq-srv.au-team.irpo A +short
dig @127.0.0.1 br-fw.au-team.irpo A +short
dig @127.0.0.1 -x 10.10.100.2 +short
dig @127.0.0.1 -x 10.20.20.2 +short
dig @127.0.0.1 example.org A +short
printf '%s\n' 'search au-team.irpo' 'nameserver 10.10.100.2' > /etc/net/ifaces/ens3/resolv.conf
systemctl restart network
```

## 9 BR-SRV — ALT Linux

### 9.1 Настроить адрес, имя, часовой пояс и пакеты

Адрес 10.20.20.2/28, шлюз BR-FW10.20.20.1. ens3 обслуживает etcnet. К этому этапу BR-FW, BR-RTR и ISP должны обеспечивать Интернет.

```bash
su -
cat /etc/os-release
ip -br link
mkdir -p /root/kim-backup
tar -czf /root/kim-backup/br-srv-network-before.tgz /etc/net /etc/sysconfig/network
hostnamectl set-hostname br-srv.au-team.irpo
timedatectl set-timezone Europe/Moscow
if grep -q '^HOSTNAME=' /etc/sysconfig/network; then
  sed -i 's/^HOSTNAME=.*/HOSTNAME=br-srv.au-team.irpo/' /etc/sysconfig/network
else
  printf '%s\n' 'HOSTNAME=br-srv.au-team.irpo' >> /etc/sysconfig/network
fi
mkdir -p /etc/net/ifaces/ens3
cat > /etc/net/ifaces/ens3/options <<'EOF'
TYPE=eth
BOOTPROTO=static
ONBOOT=yes
DISABLED=no
CONFIG_IPV4=yes
CONFIG_IPV6=no
CONFIG_WIRELESS=no
NM_CONTROLLED=no
SYSTEMD_CONTROLLED=no
EOF
printf '%s\n' '10.20.20.2/28' > /etc/net/ifaces/ens3/ipv4address
printf '%s\n' 'default via 10.20.20.1' > /etc/net/ifaces/ens3/ipv4route
printf '%s\n' 'nameserver 77.88.8.8' > /etc/net/ifaces/ens3/resolv.conf
systemctl enable network
systemctl restart network
ping -c 3 10.20.20.1
ping -c 3 77.88.8.8
apt-get update
apt-get install -y sudo openssh-server bind-utils mc
```

### 9.2 Создать sshuser и настроить sudo и SSH

Проверить существующие записи. Создать sshuser только при отсутствии имени и UID 2027. В passwd ввести P@ssw0rd два раза.

```bash
getent passwd sshuser
getent passwd 2027
```

```bash
useradd -m -u 2027 -s /bin/bash sshuser
passwd sshuser
```

```bash
install -d -m 0750 /etc/sudoers.d
printf '%s\n' 'sshuser ALL=(ALL:ALL) NOPASSWD: ALL' > /etc/sudoers.d/sshuser
chmod 0440 /etc/sudoers.d/sshuser
visudo -c
id sshuser
su - sshuser -c 'sudo -n id'
cp -a /etc/openssh/sshd_config /root/kim-backup/br-srv-sshd_config.before
ssh-keygen -A
printf '%s\n' 'Authorized access only' > /etc/openssh/banner
cat > /etc/openssh/sshd_config <<'EOF'
Port 2027
AddressFamily inet
ListenAddress 0.0.0.0
PermitRootLogin no
PubkeyAuthentication yes
PasswordAuthentication yes
UsePAM yes
AllowUsers sshuser
MaxAuthTries 2
Banner /etc/openssh/banner
Subsystem sftp internal-sftp
EOF
sshd -t
sshd -T | grep -E '^(port|allowusers|maxauthtries|banner|permitrootlogin) '
systemctl enable --now sshd
systemctl restart sshd
ss -lntp | grep ':2027'
```

Подключить sudoers.d через visudo при необходимости. Если активен iptables, разрешить TCP 2027 от офисных сетей блоком из этапа 8.3 и сохранить правила используемым firewall-сервисом.

### 9.3 Назначить DNS HQ-SRV и проверить доступ между офисами

```bash
printf '%s\n' 'search au-team.irpo' 'nameserver 10.10.100.2' > /etc/net/ifaces/ens3/resolv.conf
systemctl restart network
ip -br address
ip route
cat /etc/resolv.conf
ping -c 3 10.10.100.2
dig @10.10.100.2 hq-srv.au-team.irpo A +short
dig @10.10.100.2 br-fw.au-team.irpo A +short
dig @10.10.100.2 -x 10.20.20.2 +short
dig @10.10.100.2 example.org A +short
dig +tcp @10.10.100.2 hq-srv.au-team.irpo A +short
ssh -p 2027 sshuser@10.10.100.2
```

## 10 HQ-RTR — EcoRouter — DHCP и DNS

Создать DHCP-пул единственного клиента HQ-CLI: 10.10.200.3. Выдать /28, шлюз 10.10.200.1, DNS 10.10.100.2 и суффикс au-team.irpo. Привязать сервер к vl200, назначить роутеру DNS HQ-SRV и сохранить.

```text
enable
configure terminal
ip pool HQ-CLI 10.10.200.3-10.10.200.3
dhcp-server 1
pool HQ-CLI 1
mask 255.255.255.240
gateway 10.10.200.1
dns 10.10.100.2
domain-name au-team.irpo
exit
exit
interface vl200
dhcp-server 1
exit
no ip name-server 77.88.8.8
ip name-server 10.10.100.2
exit
write memory
show dhcp-server 1 detailed
show dhcp-server clients vl200
show ip nat translations
```

## 11 HQ-CLI — ALT Linux

### 11.1 Задать имя и выбрать сетевое подключение

```bash
su -
cat /etc/os-release
hostnamectl set-hostname hq-cli.au-team.irpo
timedatectl set-timezone Europe/Moscow
if grep -q '^HOSTNAME=' /etc/sysconfig/network; then
  sed -i 's/^HOSTNAME=.*/HOSTNAME=hq-cli.au-team.irpo/' /etc/sysconfig/network
else
  printf '%s\n' 'HOSTNAME=hq-cli.au-team.irpo' >> /etc/sysconfig/network
fi
ip -br link
nmcli device status
nmcli connection show
```

Для карты ens3 выбрать существующий Ethernet-профиль и подставить его имя вместо HQ-CLI-LAN. Если профиля нет, создать:

```bash
nmcli connection add type ethernet ifname ens3 con-name HQ-CLI-LAN
```

### 11.2 Включить DHCP через NetworkManager

Очистить ручные параметры выбранного профиля и принимать DNS и маршруты из DHCP. Конкурирующие подключения этой же карты отключить от autoconnect.

```bash
nmcli connection modify HQ-CLI-LAN ipv4.method auto ipv4.addresses '' ipv4.gateway '' ipv4.dns '' ipv4.dns-search '' ipv4.ignore-auto-dns no ipv4.ignore-auto-routes no connection.autoconnect yes
nmcli connection up HQ-CLI-LAN
ip -br address
ip route
nmcli device show ens3
cat /etc/resolv.conf
```

### 11.3 Вариант для карты под управлением etcnet

Выполнить вместо этапа 11.2, если ens3 уже управляется etcnet. Убрать старые ручные параметры из действующей конфигурации; использовать только один менеджер карты.

```bash
mkdir -p /etc/net/ifaces/ens3
cat > /etc/net/ifaces/ens3/options <<'EOF'
TYPE=eth
BOOTPROTO=dhcp
ONBOOT=yes
DISABLED=no
CONFIG_IPV4=yes
CONFIG_IPV6=no
CONFIG_WIRELESS=no
NM_CONTROLLED=no
SYSTEMD_CONTROLLED=no
EOF
systemctl enable network
systemctl restart network
ip -br address
ip route
cat /etc/resolv.conf
```

### 11.4 Проверить адресацию, DNS и SSH

Ожидаются 10.10.200.3/28, default via10.10.200.1, DNS 10.10.100.2 и search au-team.irpo. При SSH ввести пароль sshuser P@ssw0rd.

```bash
ping -c 3 10.10.200.1
ping -c 3 10.10.100.2
ping -c 3 10.20.20.2
ping -c 3 77.88.8.8
getent hosts hq-srv.au-team.irpo
getent hosts br-srv.au-team.irpo
getent hosts hq-cli.au-team.irpo
ssh -p 2027 sshuser@hq-srv.au-team.irpo
ssh -p 2027 sshuser@br-srv.au-team.irpo
```

При установленном bind-utils проверить A, PTR и внешнее имя:

```bash
dig @10.10.100.2 hq-cli.au-team.irpo A +short
dig @10.10.100.2 -x 10.10.100.2 +short
dig @10.10.100.2 -x 10.20.20.2 +short
dig @10.10.100.2 example.org A +short
```

## 12 Окончательная настройка DNS и сохранение

### 12.1 BR-RTR

Назначить HQ-SRV как DNS, сохранить конфигурацию и проверить маршруты, пользователей и PAT.

```text
enable
configure terminal
no ip name-server 77.88.8.8
ip name-server 10.10.100.2
exit
write memory
show ip route
show ip nat translations
show users localdb
ping 10.10.100.2
ping 10.20.20.2
```

### 12.2 HQ-SW

Назначить DNS управления и проверить службу коммутации. Если resolv.conf управляется resolver-сервисом, сохранить DNS через него.

```bash
printf '%s\n' 'search au-team.irpo' 'nameserver 10.10.100.2' > /etc/resolv.conf
cat /etc/resolv.conf
systemctl status kim-hq-switch.service --no-pager
ping -c 3 10.10.100.2
```

### 12.3 BR-FW

В настройках самой Ideco задать DNS `10.10.100.2` и сохранить. Проверить через диагностику:

1. Доступ к 10.20.30.0, 10.10.10.2, 10.20.20.2 и 10.10.100.2.
2. FULL соседство OSPF и маршруты трех HQ-подсетей через GRE.
3. Разрешение hq-srv.au-team.irpo и br-srv.au-team.irpo.
4. Прохождение DNS по UDP/TCP 53 и SSH к серверам по TCP 2027.

Создать резервную копию конфигурации Ideco. На обоих EcoRouter выполнить `write memory`. После перезагрузки ВМ проверить восстановление адресов, VLAN, GRE/OSPF, DHCP, DNS, SSH и доступа к Интернету.
