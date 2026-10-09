# Задание 1 КИМ 2027 — исправленный алгоритм по каждой ВМ

Редакция от 9 октября 2026 года. КИМ 09.02.06-3-2027 Том 1 Задание 1.

Платформы: ISP, HQ-SW, HQ-SRV, BR-SRV, HQ-CLI — ALT Linux; HQ-RTR и BR-RTR — EcoRouter 3.2.6.2; BR-FW — Ideco NGFW Novum 21.7.130.

Это последовательность настройки учебного стенда. Команды ниже подготовлены для выполнения в консолях соответствующих ВМ; они не были выполнены на ваших ВМ. Проверены исходный алгоритм, изображения, требования КИМ и документация производителей. Доступ к консолям ВМ и фактические результаты `show version`, `show port`, `ip -br link` не предоставлены. Поэтому имена портов ниже необходимо сопоставить со стендом до ввода команд. Номер версии ALT Linux не указан; используется etcnet, как в исходном алгоритме.

**Актуальный алгоритм — этот текст.** Исходный CHECKIN.docx содержит ошибки и сохраняется как [CHECKIN_legacy_2026.docx](https://github.com/wrwex/1337/blob/main/CHECKIN_legacy_2026.docx), только для истории. Пароль GitHub не относится к экзаменационным настройкам и в этом документе не хранится. `P@ssw0rd` ниже — пароль, прямо заданный в КИМ.

## 1 Что исправлено по устройствам

| ВМ | Ошибки или пропуски исходного алгоритма | Исправление |
|---|---|---|
| ISP | Нет WAN DHCP, адресов офисных интерфейсов, forwarding; повторное добавление снимков iptables; некорректная Linux-зона utc+3 | Полная адресация etcnet, WAN DHCP, постоянный forwarding, PAT, разрешенный транзит GRE, сохранение правил, Europe/Moscow |
| HQ-SW | Только имя; нет VLAN и trunk/access | VLAN-aware Linux bridge, trunk 100/200/999, access 100 и 200, адрес управления VLAN 999, автозапуск |
| HQ-RTR | Нет интерфейсов и привязки портов; нет пользователя и DHCP; неверный ключ OSPF; NAT-пул VLAN200 больше /28 | Созданы интерфейсы и service-instance, net_admin, PAT, DHCP и GRE; OSPF MD5 P@ssw0rd; точные пулы |
| BR-RTR | GRE вне сети 10.10.10.0/24; нет интерфейса к BR-FW и пользователя; неверный ключ OSPF; неверная модель BR-SRV напрямую за роутером | Отдельный транзит к BR-FW, согласованные GRE и OSPF, net_admin, PAT трафика филиала |
| BR-FW | Устройство полностью отсутствует | Два физических интерфейса, маршрут по умолчанию, GRE к BR-RTR, OSPF, межсетевые правила, DNS, имя и зона времени |
| HQ-SRV | Не создан sshuser/UID; символ » вместо >>; SSH 2026; закомментированный forwarders; нет A BR-FW и PTR BR-SRV; опечатки in-add | Пользователь UID 2027, sudoers.d, SSH 2027; полные DNS-зоны, проверка BIND и доступа |
| BR-SRV | Не создан sshuser/UID; неверный SSH-порт; внешний DNS; не учтен BR-FW | Шлюз BR-FW, DNS HQ-SRV, пользователь, sudo и SSH 2027 |
| HQ-CLI | DHCP-клиент есть, сервера нет; вручную указан внешний DNS, A-запись не согласована с арендой | DHCP HQ-RTR выдает адрес .3, /28, шлюз .1, DNS HQ-SRV и суффикс; A-запись соответствует |

## 2 Решения, которые нужно зафиксировать до настройки

### 2.1 Часовой пояс

В примере выбран экзамен в Москве: ALT Linux — `Europe/Moscow`, EcoRouter — `UTC+3`, Ideco — соответствующий пункт часового пояса. Если экзамен проходит в другом месте, заменить эти значения на зону места проведения. `timedatectl set-timezone utc+3` в ALT Linux не использовать. HQ-SW освобожден КИМ от обязательной настройки часового пояса, но установка правильной зоны допустима.

### 2.2 Минимальные подсети

Основной вариант использует /31 для двухточечных соединений: два адреса и отсутствие отдельного network/broadcast по RFC 3021. Адрес с последним октетом 0 является допустимым адресом узла в таком /31. /32 не подходит как общая подсеть двух узлов.

Перед окончательной настройкой проверить принятие /31 CLI EcoRouter и формой Ideco, двусторонний ping и OSPF. У Ideco поддержка маски /31 отмечена производителем начиная с 16.5.31. Если конкретная сборка или требования экспертов предусматривают /30, применить **весь** вариант замены из раздела 13. Нельзя заменить маску только на одном конце.

### 2.3 Неоднозначность OSPF в КИМ и выбранный вариант

КИМ одновременно требует OSPF на BR-FW через сторону BR-RTR и разрешает OSPF на HQ-RTR/BR-RTR только на туннелях. Прямое соседство OSPF на Ethernet BR-RTR–BR-FW нарушает второе ограничение.

В данной инструкции предложен **дополнительный GRE BR-RTR–BR-FW поверх существующего физического соединения**. В результате HQ-RTR использует OSPF на tunnel.0, BR-RTR — на tunnel.0 и tunnel.1, BR-FW — только на GRE к BR-RTR. Физическая схема КИМ и число ВМ сохраняются. Ideco 21 документирует OSPF на GRE. Это техническая интерпретация условия, которую нужно согласовать с экспертами: сам КИМ прямо второй GRE не предписывает.

Если эксперты разрешают OSPF на физическом транзите, есть более простой альтернативный вариант в разделе 14. Эти варианты взаимоисключающие: не создавать оба OSPF-соседства BR-RTR–BR-FW одновременно.

## 3 Схема подключения и адресная таблица

```text
Internet — ISP — HQ-RTR — trunk — HQ-SW — access VLAN100 — HQ-SRV
             │                         └ access VLAN200 — HQ-CLI
             └ BR-RTR — FW-NET — BR-FW — BR-NET — BR-SRV

GRE tunnel.0: HQ-RTR ↔ BR-RTR через ISP
GRE tunnel.1: BR-RTR ↔ BR-FW через FW-NET
Управление HQ-SW: VLAN999 через общий trunk к HQ-RTR
```

### 3.1 Принятая нумерация портов

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

Это выбранные имена, а не результат инвентаризации ваших ВМ. В EcoRouter фактические порты могут называться иначе, например te0/te1. Проверить `show port`; в Linux — `ip -br link`. При отличиях заменить имена во всех командах соответствующей ВМ. В настройках гипервизора соединять именно указанные сегменты; на trunk не применять принудительную access VLAN со стороны гипервизора.

### 3.2 Адреса и шлюзы для таблицы 2 КИМ

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

Офисные подсети содержат 32, 16, 8 и 16 адресов соответственно, как допускает КИМ. Все статические адреса приватные. Адрес провайдера на ISP назначается DHCP согласно отдельному требованию КИМ.

## 4 Общий порядок между ВМ

1. Зафиксировать версии, сетевые карты, MAC-адреса, выбранную схему OSPF и адресную таблицу. Создать снимки ВМ или резервные копии конфигураций.
2. Настроить ISP: WAN DHCP, внутренние интерфейсы, forwarding, PAT и транзит между роутерами.
3. Настроить физические интерфейсы HQ-RTR и BR-RTR, пользователей, маршруты по умолчанию и PAT.
4. Настроить HQ-SW и VLAN; затем базовые адреса HQ-SRV и BR-SRV.
5. Настроить BR-FW: Ethernet, имя, время, default route, правила и GRE к BR-RTR.
6. Создать GRE на EcoRouter и OSPF на трех устройствах; проверить соседства и маршруты.
7. Создать пользователей и SSH на серверах. Установить BIND на HQ-SRV, настроить зоны.
8. Настроить DHCP на HQ-RTR и клиент DHCP на HQ-CLI; проверить имя клиента в DNS.
9. Перевести серверы, роутеры, BR-FW и HQ-SW на DNS HQ-SRV, проверить Интернет и межофисный доступ.
10. Сохранить настройки, проверить восстановление после перезагрузки, оформить отчеты КИМ.

Разделы ниже сгруппированы по ВМ. У каждой ВМ есть этапы A, B и при необходимости C; не завершать поздний этап до настройки его зависимости на другой ВМ. Все Linux-блоки выполняются в Bash от root через консоль ВМ. Команды `su -` требуют текущий пароль root самой ВМ. **Не вводить эти блоки в PowerShell Windows.**

## 5 ISP — ALT Linux

### A Проверка и резервная копия

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

Сохранить существующие строки `/etc/sysconfig/network`, изменив только HOSTNAME:

```bash
if grep -q '^HOSTNAME=' /etc/sysconfig/network; then
  sed -i 's/^HOSTNAME=.*/HOSTNAME=isp.au-team.irpo/' /etc/sysconfig/network
else
  printf '%s\n' 'HOSTNAME=isp.au-team.irpo' >> /etc/sysconfig/network
fi
```

Основная инструкция рассчитана на etcnet (`network.service`). Если карты сейчас управляются NetworkManager или systemd-networkd, сначала через консоль перевести их на etcnet. Не оставлять два менеджера управляющими одной картой. Установку пакетов выполнить при действующем временном доступе к репозиторию либо с экзаменационного носителя.

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

На свежем стенде файлы `ipv4route` офисных интерфейсов должны быть пустыми: на ens4/ens5 не нужен default route. На WAN не оставлять старые статические ipv4address/ipv4route, применяемые другим менеджером. Для проверки и ручной правки:

```bash
ls -l /etc/net/ifaces/ens3 /etc/net/ifaces/ens4 /etc/net/ifaces/ens5
systemctl enable network
systemctl restart network
ip -br address
ip route
cat /etc/resolv.conf
ping -c 3 77.88.8.8
```

WAN должен получить адрес, default route и DNS по DHCP. Если DNS от провайдера не выдан, задать в настройках WAN DNS `77.88.8.8` средствами используемого DHCP-клиента; не угадывать WAN-шлюз. Если default route не получен, сначала устранить проблему DHCP.

### B Forwarding и PAT

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

Между WAN-адресами роутеров разрешен транзит, включая GRE — IP-протокол 47, а не TCP/UDP-порт 47. На этом пути NAT не применяется: MASQUERADE ограничен выходом ens3. Полный снимок правил сохраняется через `>`, не через добавление `>>`. Эти команды добавляют необходимые правила без очистки чужих цепочек. Если стенд уже использует firewalld/nftables как управляющий сервис, сначала согласовать один способ управления правилами.

## 6 HQ-RTR — EcoRouter 3.2.6.2

### A Физические интерфейсы, пользователь и PAT

Сначала проверить фактические порты и исходную конфигурацию. Сохранить вывод running-config в отчет или резервную копию.

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

VLAN 100/200/999 привязаны к одному физическому ge1. `rewrite pop 1` снимает VLAN при передаче на L3; обратное добавление тега определяется service-instance. Не заменять это тремя отдельными адаптерами ВМ.

Блок использует синтаксис NAT `inside-to-outside` исходного алгоритма и практикума EcoRouter того поколения. В новой документации встречается `inside`. Перед вставкой на конкретной сборке посмотреть `ip nat source dynamic ?`: если доступны другие ключевые слова, использовать принимаемый CLI вариант с теми же пулом, overload и outside-интерфейсом. Аналогично проверять `username`/контекст ролей; максимальная роль — admin. Нельзя без проверки подставлять синтаксис Cisco `privilege 15`.

### B GRE и OSPF после настройки BR-RTR и ISP

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

Ожидается сосед BR-RTR с Router ID 172.16.2.2 в FULL. На VLAN нет активного соседства, но сети объявляются как пассивные. После настройки BR-FW должен появиться OSPF-маршрут 10.20.20.0/28.

### C DHCP и окончательный DNS

В КИМ только один клиент HQ-CLI. Для воспроизводимой A-записи выбран динамический пул из одного адреса .3. Клиент получает его по DHCP; адрес шлюза .1 не входит в пул. Это не ручное назначение статического IP клиенту. Если нужны дополнительные клиенты, расширять пул с резервированием адреса HQ-CLI по MAC либо обновлением его A-записи.

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

Для CLI с контекстным `ip pool <NAME>` вместо однострочного диапазона: войти в `ip pool HQ-CLI`, затем `range 10.10.200.3-10.10.200.3`, выйти сначала из диапазона, затем из пула. Определить число выходов по приглашению CLI; DHCP-пул создается в глобальном config. После входа net_admin проверить возможность `configure terminal` и `write memory`.

## 7 BR-RTR — EcoRouter 3.2.6.2

### A Физические интерфейсы и пользователь

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

BR-RTR не получает адрес 10.20.20.1: этот адрес принадлежит BR-FW в сети сервера. Встречавшееся в исходном алгоритме объявление этой сети непосредственно на BR-RTR без BR-FW удалено.

### B Два GRE, OSPF и PAT филиала

Перед этим BR-FW должен получить физический адрес 10.20.30.1/31. Проверить `ping 10.20.30.1`.

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

Ожидаются два соседа FULL: HQ-RTR на tunnel.0 и BR-FW на tunnel.1. На физическом br OSPF не активируется. `ip nat inside` нужен также на tunnel.1: трафик BR-SRV к Интернету может приходить после декапсуляции на этот интерфейс. Внутренний межофисный трафик выходит через GRE, где нет `ip nat outside`, поэтому не должен переводиться в WAN-адрес. У самого BR-FW default route остается через физическую сторону BR-RTR.

### C DNS и проверки

После запуска DNS на HQ-SRV:

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

Не добавлять статические маршруты к офисным сетям для сокрытия неработающего OSPF. На HQ-RTR маршрут BR-NET должен быть динамическим, на BR-FW — динамические маршруты HQ. Старые настройки GRE 10.10.20.2/30 и ключ P@ssword нужно убрать при переносе на уже настроенный стенд.

## 8 HQ-SW — ALT Linux

Используется VLAN-aware bridge Linux, который пропускает только настроенные VLAN. В отличие от объединения портов в обычный мост он разделяет сервер, клиента и управление.

### A Подготовка физических карт

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

`ip` и `bridge` должны быть установлены до смены сети; при отсутствии поставить пакет iproute2 через имеющееся подключение/носитель. Для имени в старом файле:

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

Эти три карты используются как L2-порты: на них не должно оставаться IP/default route или активных DHCP-подключений другого менеджера. Проверить текущие файлы и настройки менеджеров. Адрес будет только на mgmt999.

### B Мост и постоянный автозапуск

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

Результат: ens3 — tagged 100/200/999; ens4 — untagged 100; ens5 — untagged 200; локальное управление — только 999. На свежем мосту благодаря default_pvid 0 нет автоматически разрешенного VLAN1. Если br0 существовал ранее, проверить `bridge vlan show` и убрать только лишние членства, относящиеся к этой конфигурации; не считать старый мост автоматически чистым. При последующей ручной перезагрузке network заново запустить `systemctl restart kim-hq-switch.service`. IPv4 forwarding для функции L2-коммутации не нужен.

### C DNS управления после запуска HQ-SRV

```bash
printf '%s\n' 'search au-team.irpo' 'nameserver 10.10.100.2' > /etc/resolv.conf
cat /etc/resolv.conf
systemctl status kim-hq-switch.service --no-pager
ping -c 3 10.10.100.2
```

В этом варианте на картах коммутатора нет DHCP, который перезапишет resolv.conf. Если файл контролируется отдельным resolver-сервисом, настроить постоянный DNS через него и проверить сохранение после перезагрузки.

## 9 BR-FW — Ideco NGFW Novum 21.7.130

**Постоянные настройки выполнять в штатном веб-интерфейсе Ideco.** Linux-команды etcnet, iptables и ручная правка FRR не являются эквивалентом поддерживаемой конфигурации Ideco и могут быть перезаписаны ее управляющими службами. Ниже приведены все необходимые значения и порядок действий. Точные названия пункта меню проверять в контексте своей установки 21.7.130.

### A Ethernet и общие настройки

1. Открыть локальную консоль ВМ, узнать текущий адрес управления и перейти в веб-интерфейс. Учетные данные — администратора самой Ideco, не GitHub.
2. Сохранить резервную копию средствами Ideco. Записать версию 21.7.130 и сопоставить физические карты по MAC.
3. В настройках сервера задать имя `br-fw.au-team.irpo`, часовой пояс места экзамена; в этом примере Москва.
4. В **Сервисы → Сетевые интерфейсы → Интерфейсы** создать/изменить локальный Ethernet `FW-NET`: карта к BR-RTR, `10.20.30.1/31`.
5. Создать локальный Ethernet `BR-NET`: карта к BR-SRV, `10.20.20.1/28`. На нем не создавать DHCP-сервер — BR-SRV статический.
6. Оба интерфейса в этой схеме — локальные LAN. Это позволяет использовать физический транзит и GRE внутри частной сети без автоматического SNAT WAN. Не превращать соединение к BR-RTR в внешнего провайдера только ради имени WAN.
7. В **Маршрутизация → Статическая** задать default route `0.0.0.0/0` через `10.20.30.0` на FW-NET. Проверить, что он появляется в действующей таблице, а не только в неактивном правиле. При работе с отдельными VCE контекст должен быть тем же, где находятся интерфейсы и OSPF.
8. Пока HQ-SRV не готов, DNS можно временно задать 77.88.8.8. После настройки OSPF и BIND заменить на 10.10.100.2.

### B GRE к BR-RTR

В том же разделе интерфейсов добавить **GRE** с именем `GRE-BR-RTR`:

| Параметр | Значение |
|---|---|
| Локальный адрес транспортного соединения / source | 10.20.30.1 |
| Удаленный адрес транспортного соединения / destination | 10.20.30.0 |
| IPv4-адрес внутри GRE | 10.10.10.3/31 |
| Удаленный IPv4 внутри GRE, если поле присутствует | 10.10.10.2 |
| MTU | 1400 |
| Состояние | включен |

Название и подписи полей сверить с формой: не перепутать адреса транспортных концов и внутренние адреса туннеля. Никакой IPsec КИМ здесь не требует; GRE сам по себе не шифрует трафик.

### C OSPF

В **Маршрутизация → OSPF**:

1. Оставить/назначить уникальный Router ID; записать фактическое значение. Он не должен совпадать с 172.16.1.2 или 172.16.2.2. Если поле только автоматически заполняется, менять его произвольной командой нельзя; достаточно проверить уникальность.
2. Установить MD5, Key ID `1`, пароль `P@ssw0rd`. Это согласовано с tunnel.1 BR-RTR; тот же ключ применяется и к основному туннелю офисов.
3. Добавить только интерфейс `GRE-BR-RTR`, Area `0`, тип области `Normal`, cost `10` при доступности поля. Hello `10`, Dead `40`; эти интервалы должны совпадать с BR-RTR.
4. Тип сети GRE — point-to-point. Ethernet FW-NET и BR-NET в активный OSPF не добавлять.
5. В дополнительной конфигурации включить **Redistribute connected**, чтобы BR-NET объявлялась через GRE без Hello к серверу. Выключить **Redistribute default** и **Redistribute static**: Ideco не должна объявлять себя default gateway для HQ-RTR/BR-RTR.
6. В фильтре анонсируемых сетей разрешить свои `10.20.20.0/28`, `10.20.30.0/31`, `10.10.10.2/31`. Входящие сети — разрешить офисные маршруты HQ, не разрешать нежелательный default route. При использовании поля «Любой, кроме 0.0.0.0/0» оно подходит для входящих маршрутов данного стенда.
7. Сохранить и включить модуль OSPF. Проверить соседство FULL с BR-RTR и маршруты 10.10.100.0/27, 10.10.200.0/28, 10.10.30.0/29 через GRE.

Не объявлять HQ-сети как собственные connected. OSPF должен получать их от HQ через BR-RTR. Не отключать Redistribute connected, оставляя в активном OSPF только GRE: тогда BR-SRV может оказаться неанонсированным.

### D Правила трафика и NAT

В **Правила трафика → Файрвол** создать включенные правила с журналированием и ограничением на заданные сети:

| Назначение правила | Источник | Назначение / протокол |
|---|---|---|
| Доступ к самому BR-FW для GRE | 10.20.30.0 на FW-NET | Локальный BR-FW 10.20.30.1, IP GRE 47 |
| OSPF в туннеле | Сосед BR-RTR на GRE-BR-RTR | Локальное устройство и OSPF multicast, IP OSPF 89 |
| HQ к серверу филиала | 10.10.100.0/27, 10.10.200.0/28, 10.10.30.0/29 через GRE | 10.20.20.0/28, разрешить требуемый межофисный трафик |
| Филиал к HQ | 10.20.20.0/28 | Указанные HQ-подсети через GRE, разрешить |
| BR-SRV к Интернету | 10.20.20.0/28 | Через FW-NET к BR-RTR, разрешить исходящий трафик |
| Проверка самого BR-FW | Адреса управления и соседей стенда | ICMP к BR-FW, только необходимые источники |

Правила к самому устройству относятся к локальному входу, а между сетями — к транзиту. Не пытаться разрешить OSPF правилом TCP/UDP-порта 89: это номер IP-протокола. Для ответного трафика использовать штатное отслеживание состояний. В интерфейсе, где отдельная таблица исходящего трафика устройства отсутствует, проверить штатные правила/диагностику, а не создавать неподдерживаемую таблицу.

**SNAT на BR-FW для этого варианта не настраивать.** Он должен сохранять адреса BR-SRV при обмене с HQ; обязательный PAT в сторону ISP делает BR-RTR. Проверить отсутствие ранее созданного общего SNAT, захватывающего BR-NET→GRE или BR-NET→FW-NET. Если есть, ограничить/исключить эти конкретные потоки. Не отключать межсетевой экран целиком.

После запуска HQ-SRV задать DNS 10.10.100.2 в настройках самой Ideco. Встроенный DNS Ideco не является основным сервером по КИМ. Проверить DNS-запросы от BR-SRV к HQ-SRV по UDP **и** TCP 53; также TCP 2027 для SSH к серверам. При необходимости интернет-авторизации в лицензии Ideco разрешить доступ только объекту BR-SRV/учебной подсети штатными средствами и проверить, что это не включило автоматический SNAT.

### E Что проверить и сохранить

Снять скриншоты Ethernet и GRE, ключа/режима OSPF без излишнего раскрытия секрета, соседей, действующих маршрутов, правил и DNS. Проверить ping к 10.20.30.0, 10.10.10.2, 10.20.20.2 и 10.10.100.2 через диагностику Ideco. Сохранить конфигурацию/резервную копию и проверить восстановление после перезагрузки.

Установка серверного sshuser или net_admin на BR-FW КИМ не предписана. Доступ к консоли/SSH самой Ideco следует использовать только если он уже предусмотрен настройками администратора; не выдавать команды FRR `write memory` как поддерживаемый способ постоянного изменения Ideco.

## 10 HQ-SRV — ALT Linux

### A IPv4 и установка пакетов

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

Внешний DNS здесь временный для установки пакетов. После запуска BIND заменить его на собственный DNS. В свежей серверной установке etcnet должен быть единственным менеджером ens3.

### B sshuser и sudo

```bash
getent passwd sshuser
getent passwd 2027
```

Если обе проверки не нашли записи, создать:

```bash
useradd -m -u 2027 -s /bin/bash sshuser
passwd sshuser
```

На обоих запросах passwd ввести `P@ssw0rd`. Если пользователь уже существует с UID 2027 — только установить требуемый пароль. Если UID занят другим пользователем либо sshuser имеет иной UID — сначала разобраться с существующей учетной записью и владельцами файлов; не назначать повторный UID и не менять владельцев автоматически.

```bash
install -d -m 0750 /etc/sudoers.d
printf '%s\n' 'sshuser ALL=(ALL:ALL) NOPASSWD: ALL' > /etc/sudoers.d/sshuser
chmod 0440 /etc/sudoers.d/sshuser
visudo -c
id sshuser
su - sshuser -c 'sudo -n id'
```

Если штатный `/etc/sudoers` не включает `/etc/sudoers.d`, через `visudo` добавить `#includedir /etc/sudoers.d` как директиву sudoers и повторить проверку. В этом синтаксисе `#includedir` — директива, а не обычный комментарий.

### C SSH 2027

На чистом экзаменационном сервере сделать резервную копию и записать явную минимальную конфигурацию:

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
sshd -T | rg '^(port|allowusers|maxauthtries|banner|permitrootlogin) '
systemctl enable --now sshd
systemctl restart sshd
ss -lntp | grep ':2027'
```

Если `rg` не установлен на ВМ, заменить только команду фильтрации на `sshd -T | grep -E '^(port|allowusers|maxauthtries|banner|permitrootlogin) '`. Если `sshd -t` возвращает ошибку, исправить ее до перезапуска. В уже используемом сервере сохранить необходимые штатные настройки SSH и изменить только заданные директивы, проверив include/Match на конфликтующие значения.

При активном iptables разрешить вход TCP 2027 от учебных офисных подсетей и сохранить правила управляющим сервисом. Например если iptables уже используется на ВМ:

```bash
for s in 10.10.100.0/27 10.10.200.0/28 10.10.30.0/29 10.20.20.0/28; do
  iptables -C INPUT -s "$s" -p tcp --dport 2027 -j ACCEPT || iptables -I INPUT 1 -s "$s" -p tcp --dport 2027 -j ACCEPT
done
```

Если firewall не установлен/не активен, эти четыре правила не являются отдельным требованием устанавливать его. Если его контролирует другой сервис, внести эквивалент через этот сервис. Не сохранять командой `iptables-save` в файл неиспользуемого сервиса.

### D DNS BIND

```bash
tar -czf /root/kim-backup/hq-srv-bind-before.tgz /var/lib/bind/etc /etc/bind
systemctl cat bind.service
ls -l /etc/bind/named.conf /var/lib/bind/etc/named.conf
mcedit /var/lib/bind/etc/options.conf
```

В options.conf сохранить штатную структуру файла и остальные необходимые директивы. **Заменить существующие значения** следующих параметров, убрать `//` перед ними и не создавать дубли:

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

Если options.conf подключен внутри блока `options { ... };` основного файла, дополнительные внешние скобки не добавлять. Если блок содержится в самом options.conf, сохранить его. При mcedit обычно F2 сохраняет, F10 выходит. Основной named.conf должен подключать options.conf и rfc1912.conf; проверить существующие include. На ALT пути `/etc/bind` могут ссылаться на chroot-каталог: не создавать вторую независимую конфигурацию по другому пути.

В rfc1912.conf добавить зоны или исправить уже существующие одноименные определения, сохранив штатные localhost-зоны. Каждую зону объявлять один раз:

```bash
mcedit /var/lib/bind/etc/rfc1912.conf
```

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

Относительные file должны разрешаться в `/var/lib/bind/etc/zone` через штатный `directory`/chroot пакета ALT, как в исходной конфигурации. Если directory в установленном пакете иной, указать правильный путь **внутри chroot**, а не добавлять host-путь `/var/lib/bind` второй раз.

Создать файлы зон:

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

A br-rtr может указывать на выбранный доступный интерфейс самого BR-RTR. Здесь это 10.20.30.0; A br-fw — 10.20.20.1. Адреса обязательно фиксируются в таблице 2. По таблице 3 PTR нужны HQ-SRV и BR-SRV; для других устройств PTR не обязателен. Зоны reverse /24 являются административными DNS-зонами и не меняют сетевые маски /27 и /28.

Проверить конфигурацию перед запуском. Для стандартного chroot ALT `/var/lib/bind` и файла внутри него `/etc/named.conf`:

```bash
named-checkconf -t /var/lib/bind /etc/named.conf
```

Если `systemctl cat bind.service`/штатный launcher показывает иной путь или chroot, использовать именно его в named-checkconf; не обходить ошибку путем запуска BIND с другой, неиспользуемой конфигурацией. Проверка зон отдельно выше остается обязательной.

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

При активном iptables разрешить из нужных офисных подсетей UDP/TCP 53 так же, как TCP 2027, и сохранить правила используемым firewall-сервисом. Проверка DNS должна проходить с BR-SRV и HQ-CLI, а не только localhost. Увеличивать SOA serial при последующих изменениях зон. Настройка rndc КИМ не требуется; некорректное обрезание вывода rndc-confgen из исходного алгоритма исключено.

## 11 BR-SRV — ALT Linux

### A Статическая сеть

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

Шлюз — BR-FW, не BR-RTR. Доступ к Интернету появляется после правил Ideco, маршрутизации и PAT BR-RTR/ISP.

### B Пользователь и SSH

Выполнить те же проверки существующей записи, что на HQ-SRV:

```bash
getent passwd sshuser
getent passwd 2027
```

Если отсутствуют:

```bash
useradd -m -u 2027 -s /bin/bash sshuser
passwd sshuser
```

Пароль в двух запросах — `P@ssw0rd`. Для уже существующего пользователя применять условия из раздела HQ-SRV.

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

При необходимости включения sudoers.d и firewall повторить указанные в разделе HQ-SRV проверки. Оставить на BR-SRV именно тот же UID 2027, но это независимая локальная запись другой ВМ.

### C DNS после запуска HQ-SRV

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

Для проверки ограничения двух попыток при парольном входе использовать `ssh -o PubkeyAuthentication=no -o PreferredAuthentications=password -p 2027 sshuser@10.10.100.2`. Автоматически предлагаемые ключи иначе могут расходовать попытки раньше ввода пароля.

## 12 HQ-CLI — ALT Linux с графической оболочкой

Для клиента сохраняется NetworkManager, если он уже управляет картой. Это не требует отключения графической сети. Сервер DHCP HQ-RTR должен быть настроен раньше.

### A Имя и DHCP через NetworkManager

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

Найти существующее Ethernet-соединение ens3 и подставить его имя вместо `HQ-CLI-LAN`. Если такого соединения нет, создать его:

```bash
nmcli connection add type ethernet ifname ens3 con-name HQ-CLI-LAN
```

Изменить выбранное соединение:

```bash
nmcli connection modify HQ-CLI-LAN ipv4.method auto ipv4.addresses '' ipv4.gateway '' ipv4.dns '' ipv4.dns-search '' ipv4.ignore-auto-dns no ipv4.ignore-auto-routes no connection.autoconnect yes
nmcli connection up HQ-CLI-LAN
ip -br address
ip route
nmcli device show ens3
cat /etc/resolv.conf
```

Старое ручное подключение на этой же карте отключить и убрать из autoconnect, если оно конкурирует с выбранным; не оставлять два одновременно активных IP-профиля. Не записывать поверх NetworkManager файл `/etc/net/ifaces/ens3/resolv.conf` с 77.88.8.8. В DHCP-параметрах должны присутствовать адрес 10.10.200.3/28, шлюз 10.10.200.1, DNS 10.10.100.2, домен au-team.irpo.

### Альтернатива для клиента, уже использующего etcnet

Не выполнять этот блок вместе с NetworkManager-вариантом на той же карте.

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

Перед применением убрать из этой конфигурации старые ручные DNS/маршруты средствами выбранного менеджера; DHCP должен дать все параметры, а не только IP. Файлы не удалять вслепую без резервной копии.

### B Проверка сервиса

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

Для каждого SSH-сервера проверить успешный вход sshuser, отказ root/другому пользователю и отображение баннера. Если установлен bind-utils, выполнить:

```bash
dig @10.10.100.2 hq-cli.au-team.irpo A +short
dig @10.10.100.2 -x 10.10.100.2 +short
dig @10.10.100.2 -x 10.20.20.2 +short
dig @10.10.100.2 example.org A +short
```

## 13 Вариант /30 если /31 не поддерживается или так согласовано с экспертами

Менять все связанные интерфейсы, GRE и OSPF network синхронно. Вариант /30 — технический запасной вариант, а не безусловно самая экономная маска по RFC 3021.

| Соединение | Сеть /30 | Первый конец | Второй конец |
|---|---|---|---|
| HQ-RTR ↔ BR-RTR GRE0 | 10.10.10.0/30 | HQ 10.10.10.1/30 | BR 10.10.10.2/30 |
| BR-RTR ↔ BR-FW Ethernet | 10.20.30.0/30 | BR 10.20.30.1/30 | FW 10.20.30.2/30 |
| BR-RTR ↔ BR-FW GRE1 | 10.10.10.4/30 | BR 10.10.10.5/30 | FW 10.10.10.6/30 |

В транспортных параметрах GRE1 заменить source/destination на 10.20.30.1 и 10.20.30.2 зеркально; default route Ideco — через 10.20.30.1. В OSPF заменить объявления /31 на соответствующие /30. Пул PAT BR-FW-NET — 10.20.30.1–10.20.30.2. A br-rtr в DNS — 10.20.30.1. В ACL DNS и фильтрах Ideco заменить FW-NET на /30, а GRE1 на 10.10.10.4/30. При уже примененной /31-конфигурации сначала удалить старые адреса и старые OSPF network через CLI `no`, сверяя текущее running-config; не оставить два разных адреса на одном интерфейсе.

## 14 Альтернатива без второго GRE по согласованной трактовке OSPF

Использовать только если эксперты разрешили активный OSPF на Ethernet BR-RTR–BR-FW. Тогда GRE0 HQ↔BR сохраняется, GRE1 не создается. На BR-RTR:

```text
enable
configure terminal
interface br
ip ospf network broadcast
ip ospf authentication message-digest
ip ospf message-digest-key 1 md5 P@ssw0rd
exit
router ospf 1
no passive-interface br
network 10.20.30.0/31 area 0
exit
exit
write memory
```

Применять как замену GRE1-блока с чистой конфигурации, а не добавление к нему. На Ideco в активном OSPF выбирается **только FW-NET**, Area0, Ethernet broadcast; MD5 тот же. Анонс BR-NET — через Redistribute connected, без Hello на BR-NET. В PAT BR-RTR трафик BR-SRV приходит на br; он уже marked inside. Удалить зависимость от tunnel.1 и его network, если переносите ранее настроенную схему. Этот вариант проще, но требует согласования буквального ограничения КИМ «только на интерфейсах туннеля».

## 15 Итоговая проверка и обязательный отчет

| Что проверять | Подтверждение |
|---|---|
| Имена | FQDN серверов, клиента, роутеров и BR-FW; hostname/show hostname и скриншот Ideco |
| Адресация | Таблица 2 с каждым интерфейсом, допустимыми размерами подсетей и шлюзами |
| VLAN | bridge vlan show на HQ-SW, service-instance на ge1 HQ-RTR; 100/200/999 разделены |
| GRE | Источник/назначение транспортных концов, внутренние адреса, ping, MTU; обе стороны одинаковой подсети |
| OSPF | FULL на требуемых соседствах; офисные маршруты в таблицах; MD5 и ключ КИМ; нет Hello на серверных VLAN |
| Пользователи | id sshuser = UID2027 на обоих серверах; sudo -n id = root; net_admin имеет admin |
| SSH | sshd -t успешен, слушает 2027, AllowUsers sshuser, MaxAuthTries2, баннер; реальные вход/отказ |
| DHCP | HQ-CLI получил .3/28, шлюз .1, DNS10.10.100.2, suffix au-team.irpo; роутер не попадает в пул |
| DNS | A каждого имени таблицы 3; PTR обоих серверов; запросы из обеих сетей; пересылка внешних имен |
| NAT | Интернет с HQ-SRV, HQ-CLI, BR-SRV, HQ-SW и BR-FW; таблицы трансляций на роутерах/ISP |
| Сохранение | После перезагрузки есть VLAN, адреса, OSPF, службы и правила; DHCP повторно работает |

Дополнительно проверить Интернет TCP/HTTPS, если ping ограничен провайдером: успешный ICMP не заменяет проверку веб-доступа. Для проверки TCP/HTTPS использовать браузер HQ-CLI или `curl -I https://example.org`, если curl установлен. У GRE не полагаться только на статус UP: он не подтверждает ответ удаленного конца.

По каждому пункту КИМ, где требуется отчет, создать документ с фактическим индексом пункта и кратким названием. Включить схему, таблицу адресов, реальные команды/выводы, скриншоты коммутации, GRE, OSPF и DHCP. Кадрировать изображения так, чтобы значения были читаемы. Итоговый файл назвать `ФамилияУчастникаЗадание1` с выбранным расширением. Шаблоны и точную балльную оценку проверить по приложениям 1–4, которые не были предоставлены.

**Не объявлять стенд выполненным только по этой инструкции.** Приведенные проверки нужно выполнить на ВМ и зафиксировать результаты. Подготовка текста команд не подтверждает состояние конфигурации реальных устройств.

## 16 Источники и область проверки

Основной источник требований — предоставленный КИМ 09.02.06-3-2027 Том 1 Задание 1, включая рисунок топологии и таблицы 2–3. Его формулировки и ограничения имеют приоритет над чужими учебными примерами.

Исходный алгоритм проверен в [редакции dc1507c](https://github.com/wrwex/1337/tree/dc1507c6ce02c0ed6f497b255d8a45722ab11c6f), включая README и изображения CHECKIN.docx.

Использованы первичные технические источники:

- [ALT Linux etcnet](https://docs.altlinux.org/ru-RU/alt-server/11.0/html/alt-server/etcnet.html) — конфигурационные файлы интерфейсов.
- [EcoRouter service-instance и VLAN](https://docs.ecorouter.ru/Руководство/05-Сервисные-интерфейсы/03-Операции-над-метками-в-сервисных-интерфейсах) — привязка L2/L3 и снятие тегов.
- [EcoRouter локальные пользователи](https://docs.ecorouter.ru/Руководство/03-Локальная-авторизация/03-Настройка-учётных-записей-пользователей) — пароль, роли и активация.
- [EcoRouter GRE](https://docs.ecorouter.ru/Руководство/23-Настройка-туннелирования/01-GRE) — транспортные адреса и MTU.
- [EcoRouter DHCP](https://docs.ecorouter.ru/Руководство/11-DHCP/03-Настройка-DHCP-сервера) — пулы и привязка к интерфейсу.
- [EcoRouter NAT](https://docs.ecorouter.ru/Руководство/25-Встроенный-NAT/) — inside/outside, пул исходных адресов и PAT.
- [EcoRouter NTP](https://docs.ecorouter.ru/Руководство/17-NTP/) — синтаксис часового пояса.
- [Практикум Базальт СПО 2025](https://kurs.basealt.ru/pluginfile.php/63899/coursecat/description/Demo_Exam_090206_SysNet_Admin.pdf?time=1759318761948), раздел DHCP, печатные страницы 55–57 — синтаксис того поколения EcoRouter; требования 2025 года не подменяют КИМ 2027.
- [Ideco 21 OSPF](https://docs.ideco.ru/pdf/v21/ru-ngfw-settings-routing-ospf.pdf) — GRE OSPF, аутентификация, перераспределение и фильтры.
- [Ideco 21 интерфейсы](https://docs.ideco.ru/pdf/v21/ru-ngfw-settings-services-connection-to-provider-interfaces.pdf) — Ethernet и GRE.
- [Изменения Ideco](https://ideco.ru/changelog) — поддержка /31 в 16.5.31 и последующих поколениях.
- [RFC 3021](https://www.rfc-editor.org/info/rfc3021/) — /31 на IPv4-соединениях точка–точка.
- [BIND конфигурации и зоны](https://bind9.readthedocs.io/en/latest/chapter3.html) — SOA, NS, A и PTR.

Документация EcoRouter развивается и не является доказательством принятия каждой команды именно вашей сборкой 3.2.6.2. Для спорных форм указаны контроль CLI и альтернативы. Инструкция охватывает все восемь ВМ; полное соответствие экзаменационной трактовке и работоспособность подтверждаются после согласования схемы и проверки на стенде.
