1М 

SW – root/toor
1. hostnamectl set-hostname hq-sw;exec bash 
2. mcedit /etc/sysconfig/network 
    HOSTNAME= hq-sw
3. timedatectl set-timezone utc+3
ISP – root/toor
1. hostnamectl set-hostname isp;exec bash 
2. mcedit /etc/sysconfig/network
    HOSTNAME= isp

HQ-RTR – admin/admin 
1. en
2. conf
3. hostname hq-rtr
4. ip domain-name au-team.irpo
5. ip name-server 10.10.100.2
6. exit
7. write memory
8. ip route 0.0.0.0/0 172.16.1.1
9. exit
10. write memory

BR-RTR – admin/admin 
1. en
2. conf
3. hostname br-rtr
4. ip domain-name au-team.irpo
5. ip name-server 10.10.100.2
6. exit
7. write memory
8. ip route 0.0.0.0/0 172.16.2.1
9. exit
10. write memory

CLI – resu
1. su -
    toor 
2.. acc 
    ethernet-интерфейсы 
    <img width="250" height="195" alt="изображение" src="https://github.com/user-attachments/assets/462b3a76-f097-4fe8-90e9-e4ee431de811" />


5. mcedit /etc/net/ifaces/ens3/options 

<img width="227" height="175" alt="изображение" src="https://github.com/user-attachments/assets/b2f6235c-f15b-4be6-a361-cbd5f000f5f9" />


6. mcedit /etc/net/ifaces/ens3/resolv.conf 
    nameserver 77.88.8.8
7. systemctl restart network
8. timedatectl set-timezone utc+3
9.  reboot

HQ-SRV – root/toor 
1. hostnamectl set-hostname hq-srv.au-team.irpo;exec bash 
2. mcedit /etc/sysconfig/network 
    HOSTNAME= hq-srv.au-team.irpo
3. mcedit /etc/net/ifaces/ens3/ipv4address
    10.10.100.2/27 
4. mcedit /etc/net/ifaces/ens3/ipv4route 
    default via 10.10.100.1 
5. mcedit /etc/net/ifaces/ens3/resolv.conf 
    nameserver 77.88.8.8 
6. systemctl restart network
7. echo "sshuser ALL=(ALL:ALL) NOPASSWD: ALL" » /etc/sudoers

BR-SRV – root/toor
1. hostnamectl set-hostname br-srv.au-team.irpo;exec bash 
2. mcedit /etc/sysconfig/network 
    HOSTNAME= br-srv.au-team.irpo
3. mcedit /etc/net/ifaces/ens3/ipv4address
    10.20.20.2/28 
4. mcedit /etc/net/ifaces/ens3/ipv4route 
    default via 10.20.20.1
5. mcedit /etc/net/ifaces/ens3/resolv.conf 
    nameserver 77.88.8.8 
6. systemctl restart network 
7. echo "sshuser ALL=(ALL:ALL) NOPASSWD: ALL" » /etc/sudoers 

ISP – root/toor
1. iptables -t nat -A POSTROUTING -s 172.16.1.0/28 -o ens3 -j MASQUERADE
2. iptables -t nat -A POSTROUTING -s 172.16.2.0/28 -o ens3 -j MASQUERADE
3. iptables-save >> /etc/sysconfig/iptables
4. systemctl enable --now iptables
5. systemctl restart network
6. timedatectl set-timezone utc+3 

HQ-SRV – root/toor 
1. mcedit /etc/openssh/sshd_config 
a)<img width="219" height="247" alt="изображение" src="https://github.com/user-attachments/assets/0c2b3c53-bbf8-458a-aa85-dfc01c1fab22" />
 б) <img width="302" height="107" alt="изображение" src="https://github.com/user-attachments/assets/d50e034f-8023-446d-a32d-b9b20bd398c0" />

2. mcedit /etc/openssh/banner 
   Authorized access only 
3. systemctl restart sshd 
4. systemctl restart network 

BR-SRV – root/toor
1. mcedit /etc/openssh/sshd_config 
a)<img width="219" height="247" alt="изображение" src="https://github.com/user-attachments/assets/ad0a69e1-2009-4c88-aa86-4e03a172f557" />
 б) <img width="302" height="107" alt="изображение" src="https://github.com/user-attachments/assets/92229cc8-5b71-445d-a972-076df3290371" />

2. mcedit /etc/openssh/banner 
   Authorized access only 
3. systemctl restart sshd 
4. systemctl restart network
5. timedatectl set-timezone utc+3
 
HQ-RTR – admin/admin 
1. en
2. conf
3. int tunnel.0 
10. description "GRE" 
11. ip address 10.10.10.1/30 
12. ip tunnel 172.16.1.2 172.16.2.2 mode gre 
13. ex
15. router ospf 1 
16. ospf router-id 10.10.10.1 
17. passive-interface default 
18. no passive-interface tunnel.0 
19. network 10.10.10.0/30 area 0 
20. network 10.10.100.0/27 area 0 
21. network 10.10.200.0/28 area 0 
22. network 10.10.30.0/29 area 0 
23. ex 
24. int tunnel.0 
25. ip ospf authentication message-digest 
26. ip ospf message-digest-key 1 md5 P@ssword 
27. exit 
28. write memory 
 
BR-RTR – admin/admin 
1. en
2. conf
3. int tunnel.0 
4. description "GRE" 
5. (hq) ip address 10.10.20.2/30 
6. (hq) ip tunnel 172.16.2.2 172.16.1.2 mode gre 
7. ex
8. router ospf 1 
9. ospf router-id 10.10.10.2 
10. passive-interface default 
18. no passive-interface tunnel.0 
19. network 10.10.10.0/30 area 0 
20. network 10.20.20.0/28 area 0 
21. ex 
22. int tunnel.0 
23. ip ospf authentication message-digest 
24. ip ospf message-digest-key 1 md5 P@ssword 
25. exit 
26. write memory 

HQ-RTR – admin/admin 
1. en
2. conf
3. int isp 
4. ip nat outside 
5. ех 
6. int vl100 
7. ip nat inside 
8. ех 
9. int vl200 
10. ip nat inside 
11. ех 
12. int vl999 
13. ip nat inside 
14. ex 
15. ip nat pool VLAN100 10.10.100.1-10.10.100.30 
16. ip nat pool VLAN200 10.10.200.1-10.10.200.254 
17. ip nat pool VLAN999 10.10.30.1-10.10.30.6 
18. ip nat source dynamic inside-to-outside pool VLAN100 overload interface isp 
19. ip nat source dynamic inside-to-outside pool VLAN200 overload interface isp 
20. ip nat source dynamic inside-to-outside pool VLAN999 overload interface isp 
21. exit
22. write memory 
23. ntp timezone utc+3

BR-RTR – admin/admin 
1. en
2. conf
3. int isp 
4. ip nat outside 
5. ex 
6. int br 
7. ip nat inside 
8. ex 
9. ip nat pool BR-Net 10.20.20.1-10.20.20.16 
10. ip nat source dynamic inside-to-outside pool BR-Net overload interface isp 
11. exit
12. write memory
12. ntp timezone utc+3

HQ-SRV – root/toor 
1. mcedit /var/lib/bind/etc/options.conf 
<img width="418" height="435" alt="изображение" src="https://github.com/user-attachments/assets/ce324ba0-32e3-4597-85f7-66c1e6f08e08" />


2. mcedit /var/lib/bind/etc/rfc1912.conf 
<img width="288" height="166" alt="изображение" src="https://github.com/user-attachments/assets/9d32da45-afe8-46ab-8799-1c873b7268dc" />


3. cp /var/lib/bind/etc/zone/empty /var/lib/bind/etc/zone/au-team.irpo 
4. cp /var/lib/bind/etc/zone/empty /var/lib/bind/etc/zone/100.10.10.in-addr.arpa 
5. cp /var/lib/bind/etc/zone/empty /var/lib/bind/etc/zone/200.10.10.in-addr.arpa 
6. mcedit /var/lib/bind/etc/zone/au-team.irpo 
<img width="250" height="152" alt="изображение" src="https://github.com/user-attachments/assets/255d2e60-6d1a-4bc9-8ce4-29754c0e396b" />


7. mcedit /var/lib/bind/etc/zone/100.10.10.in-add.arpa 
<img width="459" height="142" alt="изображение" src="https://github.com/user-attachments/assets/91d3235a-714d-4e08-b92d-45225488559c" />

 
8. mcedit /var/lib/bind/etc/zone/200.10.10.in-add.arpa 
<img width="457" height="118" alt="изображение" src="https://github.com/user-attachments/assets/308a0bef-6feb-4b3b-867a-00a365c00642" />


9. rndc-confgen > /etc/bind/rndc.key 
10. sed -i '6,$d' /etc/bind/rndc.key 
11. chown -R root:named /var/lib/bind/etc/zone/* 
12. systemctl enable --now bind.service 
13. systemctl status bind. service
14. timedatectl set-timezone utc+3

