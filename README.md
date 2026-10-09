# Задание 1

## 1 ISP — ALT Linux

```bash
su -
```

```bash
cat /etc/os-release
ip -br link
ip -br address
ip route
mkdir -p /root/kim-backup
tar -czf /root/kim-backup/isp-etcnet-before.tgz /etc/net /etc/sysconfig/network
hostnamectl set-hostname isp.au-team.irpo
timedatectl set-timezone Europe/Moscow
if grep -q '^HOSTNAME=' /etc/sysconfig/network; then
  sed -i 's/^HOSTNAME=.*/HOSTNAME=isp.au-team.irpo/' /etc/sysconfig/network
else
  printf '%s\n' 'HOSTNAME=isp.au-team.irpo' >> /etc/sysconfig/network
fi
apt-get update
apt-get install -y etcnet iptables
mkdir -p /etc/net/ifaces/ens3 /etc/net/ifaces/ens4 /etc/net/ifaces/ens5
for f in ipv4address ipv4route resolv.conf; do
  : > /etc/net/ifaces/ens3/$f
done
: > /etc/net/ifaces/ens4/ipv4route
: > /etc/net/ifaces/ens5/ipv4route
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
ls -l /etc/net/ifaces/ens3 /etc/net/ifaces/ens4 /etc/net/ifaces/ens5
systemctl enable network
systemctl restart network
ip -br address
ip route
cat /etc/resolv.conf
ping -c 3 77.88.8.8
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

## 2 HQ-RTR — EcoRouter 3.2.6.2

```text
enable
show version
show port
configure terminal
hostname hq-rtr
ip domain-name au-team.irpo
ip name-server 10.10.100.2
ntp timezone utc+3
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
ip pool HQ-CLI 10.10.200.3-10.10.200.3
dhcp-server 1
pool HQ-CLI 1
mask 28
gateway 10.10.200.1
dns 10.10.100.2
domain-name au-team.irpo
exit
exit
interface vl200
dhcp-server 1
exit
exit
write memory
show interface
show ip route
show running-config dhcp-server 1
show users localdb
```

## 3 BR-RTR — EcoRouter 3.2.6.2

```text
enable
show version
show port
configure terminal
hostname br-rtr
ip domain-name au-team.irpo
ip name-server 10.10.100.2
ntp timezone utc+3
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
network 10.20.30.0/31 area 0
exit
ip nat pool BR-SRV-NET 10.20.20.1-10.20.20.14
ip nat pool BR-FW-NET 10.20.30.0-10.20.30.1
ip nat source dynamic inside-to-outside pool BR-SRV-NET overload interface isp
ip nat source dynamic inside-to-outside pool BR-FW-NET overload interface isp
exit
write memory
show interface
show ip route
show users localdb
```

## 4 HQ-SW — ALT Linux

```bash
su -
```

```bash
ip -br link
mkdir -p /root/kim-backup
tar -czf /root/kim-backup/hq-sw-network-before.tgz /etc/net /etc/sysconfig/network
hostnamectl set-hostname hq-sw.au-team.irpo
sed -i '/^HOSTNAME=/d' /etc/sysconfig/network
printf '%s\n' 'HOSTNAME=hq-sw.au-team.irpo' >> /etc/sysconfig/network
apt-get update
apt-get install -y openvswitch iproute2
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
systemctl enable --now openvswitch
SW_BR="$(ovs-vsctl iface-to-br ens3 2>/dev/null || true)"
SW_BR="${SW_BR:-hq-sw}"
ovs-vsctl --may-exist add-br "$SW_BR"
ovs-vsctl --may-exist add-port "$SW_BR" ens3
ovs-vsctl set Port ens3 vlan_mode=trunk trunks=100,200,999
ovs-vsctl clear Port ens3 tag
ovs-vsctl --may-exist add-port "$SW_BR" ens4
ovs-vsctl set Port ens4 vlan_mode=access tag=100
ovs-vsctl clear Port ens4 trunks
ovs-vsctl --may-exist add-port "$SW_BR" ens5
ovs-vsctl set Port ens5 vlan_mode=access tag=200
ovs-vsctl clear Port ens5 trunks
ovs-vsctl --may-exist add-port "$SW_BR" MGMT
ovs-vsctl set Interface MGMT type=internal
ovs-vsctl set Port MGMT vlan_mode=access tag=999
ovs-vsctl clear Port MGMT trunks
install -d /usr/local/sbin
cat > /usr/local/sbin/kim-hq-switch <<'EOF'
#!/bin/bash
set -euo pipefail
for n in ens3 ens4 ens5; do
  ip address flush dev "$n" scope global
  ip link set "$n" up
done
SW_BR="$(ovs-vsctl iface-to-br ens3)"
ip address flush dev "$SW_BR" scope global
ip link set "$SW_BR" up
ip address replace 10.10.30.2/29 dev MGMT
ip link set MGMT up
ip route replace default via 10.10.30.1 dev MGMT
EOF
chmod 0755 /usr/local/sbin/kim-hq-switch
cat > /etc/systemd/system/kim-hq-switch.service <<'EOF'
[Unit]
Description=HQ-SW
Requires=network.service openvswitch.service
After=network.service openvswitch.service

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
systemctl restart kim-hq-switch.service
printf '%s\n' 'search au-team.irpo' 'nameserver 10.10.100.2' > /etc/resolv.conf
ovs-vsctl show
ip -br address
ip route
ping -c 3 10.10.30.1
```

## 5 BR-FW — Ideco 21.7.130 — веб-интерфейс

### 5.1 Система

```text
Имя устройства: br-fw.au-team.irpo
Часовой пояс: Europe/Moscow
DNS: 10.10.100.2
Сохранить
```

### 5.2 Сервисы → Сетевые интерфейсы → Интерфейсы

```text
FW-NET
Тип: Ethernet
Назначение: локальная сеть
Сетевой адаптер: к BR-RTR
IPv4: 10.20.30.1/31
Включить
Сохранить

BR-NET
Тип: Ethernet
Назначение: локальная сеть
Сетевой адаптер: к BR-SRV
IPv4: 10.20.20.1/28
Включить
Сохранить
```

### 5.3 Маршрутизация → Статическая

```text
Сеть назначения: 0.0.0.0/0
Шлюз: 10.20.30.0
Интерфейс: FW-NET
Включить
Сохранить
```

### 5.4 Сервисы → Сетевые интерфейсы → Интерфейсы → GRE

```text
Имя: GRE-BR-RTR
Локальный адрес: 10.20.30.1
Удаленный адрес: 10.20.30.0
IPv4: 10.10.10.3/31
MTU: 1400
Включить
Сохранить
```

### 5.5 Маршрутизация → OSPF

```text
Аутентификация соседей: MD5
Key ID: 1
Пароль: P@ssw0rd

Интерфейс: GRE-BR-RTR
Область: 0
Тип области: Normal
Тип сети: Point-to-point
Hello: 10
Dead: 40

Redistribute connected: включено
Redistribute default: выключено
Redistribute static: выключено

Фильтр исходящих маршрутов:
10.20.20.0/28 — разрешить
10.20.30.0/31 — разрешить
10.10.10.2/31 — разрешить

Фильтр входящих маршрутов:
10.10.100.0/27 — разрешить
10.10.200.0/28 — разрешить
10.10.30.0/29 — разрешить
10.10.10.0/31 — разрешить
0.0.0.0/0 — запретить

Сохранить
Включить OSPF
```

### 5.6 Правила трафика → Файрвол

```text
К устройству:
10.20.30.0 → 10.20.30.1; IP-протокол 47; разрешить
10.10.10.2 → BR-FW, 224.0.0.5, 224.0.0.6; IP-протокол 89; разрешить
10.20.30.0, 10.20.20.0/28, 10.10.100.0/27, 10.10.200.0/28, 10.10.30.0/29 → BR-FW; ICMP; разрешить

Транзит:
10.10.100.0/27, 10.10.200.0/28, 10.10.30.0/29 → 10.20.20.0/28; разрешить
10.20.20.0/28 → 10.10.100.0/27, 10.10.200.0/28, 10.10.30.0/29; разрешить
10.20.20.0/28 → Интернет через FW-NET; разрешить
Ответный трафик установленных соединений: разрешить

SNAT BR-NET → GRE-BR-RTR: выключено
SNAT BR-NET → FW-NET: выключено
BR-SRV 10.20.20.2: доступ в Интернет включен

Сохранить
```

## 6 HQ-SRV — ALT Linux

```bash
su -
```

```bash
set -e
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

```bash
if ! getent passwd sshuser >/dev/null; then
  useradd -m -u 2027 -s /bin/bash sshuser
fi
if [ "$(id -u sshuser)" -ne 2027 ]; then
  usermod -u 2027 sshuser
fi
test "$(id -u sshuser)" -eq 2027
printf '%s\n' 'sshuser:P@ssw0rd' | chpasswd
install -d -m 0750 /etc/sudoers.d
printf '%s\n' 'sshuser ALL=(ALL:ALL) NOPASSWD: ALL' > /etc/sudoers.d/sshuser
chmod 0440 /etc/sudoers.d/sshuser
grep -Eq '^[[:space:]]*([#@]includedir)[[:space:]]+/etc/sudoers\.d/?([[:space:]]|$)' /etc/sudoers || printf '\n%s\n' '#includedir /etc/sudoers.d' >> /etc/sudoers
visudo -c
id sshuser
su - sshuser -c 'sudo -n id'
```

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

```bash
if systemctl is-active --quiet iptables.service; then
  iptables -C INPUT -i lo -j ACCEPT || iptables -I INPUT 1 -i lo -j ACCEPT
  iptables -C INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT || iptables -I INPUT 1 -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
  for s in 10.10.100.0/27 10.10.200.0/28 10.10.30.0/29 10.20.20.0/28; do
    iptables -C INPUT -s "$s" -p tcp --dport 2027 -j ACCEPT || iptables -I INPUT 1 -s "$s" -p tcp --dport 2027 -j ACCEPT
  done

  for s in 10.10.100.0/27 10.10.200.0/28 10.10.30.0/29 10.20.20.0/28 10.20.30.0/31 10.10.10.0/24 172.16.1.0/28 172.16.2.0/28; do
    for p in udp tcp; do
      iptables -C INPUT -s "$s" -p "$p" --dport 53 -j ACCEPT || iptables -I INPUT 1 -s "$s" -p "$p" --dport 53 -j ACCEPT
    done
  done
  iptables-save > /etc/sysconfig/iptables
fi
```

```bash
tar -czf /root/kim-backup/hq-srv-bind-before.tgz /var/lib/bind/etc /etc/bind
BIND_ROOT=/var/lib/bind
BIND_CONFIG="$(readlink -f /etc/named.conf)"
BIND_OPTIONS="$(readlink -f /etc/bind/options.conf)"
BIND_DIR="$(dirname "$BIND_OPTIONS")"
BIND_ZONE_DIR="$BIND_DIR/zone"
BIND_LOCAL="$BIND_DIR/local.conf"
for p in "$BIND_CONFIG" "$BIND_OPTIONS" "$BIND_ZONE_DIR" "$BIND_LOCAL"; do
  case "$p" in "$BIND_ROOT"/*) ;; *) exit 1 ;; esac
done
install -d -m 0750 -o root -g named "$BIND_ZONE_DIR"
cat > "$BIND_OPTIONS" <<EOF
acl office {
    127.0.0.1;
    10.10.100.0/27;
    10.10.200.0/28;
    10.10.30.0/29;
    10.20.20.0/28;
    10.20.30.0/31;
    10.10.10.0/24;
    172.16.1.0/28;
    172.16.2.0/28;
};
options {
    directory "${BIND_ZONE_DIR#"$BIND_ROOT"}";
    pid-file none;
    listen-on { 127.0.0.1; 10.10.100.2; };
    listen-on-v6 { none; };
    forward only;
    forwarders { 77.88.8.8; };
    recursion yes;
    allow-query { office; };
    allow-recursion { office; };
    allow-query-cache { office; };
    allow-transfer { none; };
};
EOF
cat > "$BIND_LOCAL" <<'EOF'
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
EOF
cat > "$BIND_CONFIG" <<EOF
include "${BIND_OPTIONS#"$BIND_ROOT"}";
include "${BIND_LOCAL#"$BIND_ROOT"}";
EOF
```

```bash
cat > "$BIND_ZONE_DIR"/au-team.irpo <<'EOF'
$TTL 3600
@ IN SOA hq-srv.au-team.irpo. hostmaster.au-team.irpo. (
  2026100902 3600 900 604800 3600
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
cat > "$BIND_ZONE_DIR"/100.10.10.in-addr.arpa <<'EOF'
$TTL 3600
@ IN SOA hq-srv.au-team.irpo. hostmaster.au-team.irpo. (
  2026100902 3600 900 604800 3600
)
@ IN NS hq-srv.au-team.irpo.
2 IN PTR hq-srv.au-team.irpo.
EOF
cat > "$BIND_ZONE_DIR"/20.20.10.in-addr.arpa <<'EOF'
$TTL 3600
@ IN SOA hq-srv.au-team.irpo. hostmaster.au-team.irpo. (
  2026100902 3600 900 604800 3600
)
@ IN NS hq-srv.au-team.irpo.
2 IN PTR br-srv.au-team.irpo.
EOF
chown root:named "$BIND_ZONE_DIR"/au-team.irpo "$BIND_ZONE_DIR"/100.10.10.in-addr.arpa "$BIND_ZONE_DIR"/20.20.10.in-addr.arpa
chmod 0640 "$BIND_ZONE_DIR"/au-team.irpo "$BIND_ZONE_DIR"/100.10.10.in-addr.arpa "$BIND_ZONE_DIR"/20.20.10.in-addr.arpa
named-checkzone au-team.irpo "$BIND_ZONE_DIR"/au-team.irpo
named-checkzone 100.10.10.in-addr.arpa "$BIND_ZONE_DIR"/100.10.10.in-addr.arpa
named-checkzone 20.20.10.in-addr.arpa "$BIND_ZONE_DIR"/20.20.10.in-addr.arpa
```

```bash
chown root:named "$BIND_CONFIG" "$BIND_OPTIONS" "$BIND_LOCAL"
chmod 0640 "$BIND_CONFIG" "$BIND_OPTIONS" "$BIND_LOCAL"
named-checkconf -t "$BIND_ROOT" -z "${BIND_CONFIG#"$BIND_ROOT"}"
systemctl enable --now bind.service
systemctl restart bind.service
systemctl status bind.service --no-pager
ss -lntup | grep ':53'
dig @127.0.0.1 hq-srv.au-team.irpo A +short
dig @127.0.0.1 br-fw.au-team.irpo A +short
dig @127.0.0.1 -x 10.10.100.2 +short
dig @127.0.0.1 -x 10.20.20.2 +short
dig @127.0.0.1 example.org A +short
printf '%s\n' 'search au-team.irpo' 'nameserver 10.10.100.2' > /etc/net/ifaces/ens3/resolv.conf
systemctl restart network
```

## 7 BR-SRV — ALT Linux

```bash
su -
```

```bash
set -e
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

```bash
if ! getent passwd sshuser >/dev/null; then
  useradd -m -u 2027 -s /bin/bash sshuser
fi
if [ "$(id -u sshuser)" -ne 2027 ]; then
  usermod -u 2027 sshuser
fi
test "$(id -u sshuser)" -eq 2027
printf '%s\n' 'sshuser:P@ssw0rd' | chpasswd
install -d -m 0750 /etc/sudoers.d
printf '%s\n' 'sshuser ALL=(ALL:ALL) NOPASSWD: ALL' > /etc/sudoers.d/sshuser
chmod 0440 /etc/sudoers.d/sshuser
grep -Eq '^[[:space:]]*([#@]includedir)[[:space:]]+/etc/sudoers\.d/?([[:space:]]|$)' /etc/sudoers || printf '\n%s\n' '#includedir /etc/sudoers.d' >> /etc/sudoers
visudo -c
id sshuser
su - sshuser -c 'sudo -n id'
```

```bash
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

```bash
if systemctl is-active --quiet iptables.service; then
  iptables -C INPUT -i lo -j ACCEPT || iptables -I INPUT 1 -i lo -j ACCEPT
  iptables -C INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT || iptables -I INPUT 1 -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
  for s in 10.10.100.0/27 10.10.200.0/28 10.10.30.0/29 10.20.20.0/28; do
    iptables -C INPUT -s "$s" -p tcp --dport 2027 -j ACCEPT || iptables -I INPUT 1 -s "$s" -p tcp --dport 2027 -j ACCEPT
  done
  iptables-save > /etc/sysconfig/iptables
fi
```

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
ssh -p 2027 sshuser@10.10.100.2 'hostname; sudo -n id'
```

## 8 HQ-CLI — ALT Linux

```bash
su -
```

```bash
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

```bash
if ! nmcli -t -f NAME connection show | grep -Fxq HQ-CLI-LAN; then
  nmcli connection add type ethernet ifname ens3 con-name HQ-CLI-LAN
fi
ACTIVE_CON="$(nmcli -g GENERAL.CONNECTION device show ens3)"
if [ -n "$ACTIVE_CON" ] && [ "$ACTIVE_CON" != '--' ] && [ "$ACTIVE_CON" != 'HQ-CLI-LAN' ]; then
  nmcli connection modify "$ACTIVE_CON" connection.autoconnect no
fi
if [ -f /etc/net/ifaces/ens3/options ]; then
  sed -i '/^NM_CONTROLLED=/d; /^SYSTEMD_CONTROLLED=/d; /^DISABLED=/d' /etc/net/ifaces/ens3/options
  printf '%s\n' 'NM_CONTROLLED=yes' 'SYSTEMD_CONTROLLED=no' 'DISABLED=no' >> /etc/net/ifaces/ens3/options
fi
nmcli device set ens3 managed yes
nmcli connection modify HQ-CLI-LAN ipv4.method auto ipv4.addresses '' ipv4.gateway '' ipv4.dns '' ipv4.dns-search '' ipv4.ignore-auto-dns no ipv4.ignore-auto-routes no connection.autoconnect yes connection.autoconnect-priority 100
nmcli connection up HQ-CLI-LAN
ip -br address
ip route
nmcli device show ens3
cat /etc/resolv.conf
apt-get update
apt-get install -y bind-utils openssh-clients
```

```bash
ping -c 3 10.10.200.1
ping -c 3 10.10.100.2
ping -c 3 10.20.20.2
ping -c 3 77.88.8.8
getent hosts hq-srv.au-team.irpo
getent hosts br-srv.au-team.irpo
getent hosts hq-cli.au-team.irpo
ssh -p 2027 sshuser@hq-srv.au-team.irpo 'hostname; sudo -n id'
ssh -p 2027 sshuser@br-srv.au-team.irpo 'hostname; sudo -n id'
dig @10.10.100.2 hq-cli.au-team.irpo A +short
dig @10.10.100.2 -x 10.10.100.2 +short
dig @10.10.100.2 -x 10.20.20.2 +short
dig @10.10.100.2 example.org A +short
```

## 9 HQ-RTR — EcoRouter 3.2.6.2

```text
enable
show ip ospf neighbor
show ip route
show dhcp-server clients vl200
show ip nat translations
ping 10.10.10.1
ping 10.10.100.2
ping 10.20.20.2
write memory
```

## 10 BR-RTR — EcoRouter 3.2.6.2

```text
enable
show ip ospf neighbor
show ip route
show ip nat translations
ping 10.10.10.0
ping 10.10.10.3
ping 10.10.100.2
ping 10.20.20.2
write memory
```

