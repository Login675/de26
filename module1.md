
## ISP
```tml
cp /etc/apt/sources.list.d/alt.list /etc/apt/sources.list.d/alt.list.bak
sed -i 's|^rpm.*ftp\.altlinux|# &|g' /etc/apt/sources.list.d/alt.list
cat >> /etc/apt/sources.list.d/alt.list <<EOF
rpm [p11] http://192.168.0.222/mirror p11/branch/x86_64 classic
rpm [p11] http://192.168.0.222/mirror p11/branch/noarch classic
rpm [p11] http://192.168.0.222/mirror p11/branch/x86_64-i586 classic
EOF
hostnamectl set-hostname ISP
mkdir /etc/net/ifaces/ens{20,21,22}
echo -e "BOOTPROTO=dhcp\nDISABLED=no\nTYPE=eth\nCONFIG_IPV4=yes" > /etc/net/ifaces/ens20/options
echo -e "BOOTPROTO=static\nDISABLED=no\nTYPE=eth\nCONFIG_IPV4=yes" > /etc/net/ifaces/ens21/options
echo -e "BOOTPROTO=static\nDISABLED=no\nTYPE=eth\nCONFIG_IPV4=yes" > /etc/net/ifaces/ens22/options
echo 172.16.1.1/28 > /etc/net/ifaces/ens21/ipv4address
echo 172.16.2.1/28 > /etc/net/ifaces/ens22/ipv4address
sed -i 's/^\(net\.ipv4\.ip_forward\s*=\s*\).*/\1 1/' /etc/net/sysctl.conf
systemctl restart network
apt-get update && apt-get install iptables -y
iptables -t nat -A POSTROUTING -o ens20 -s 172.16.1.0/28 -j MASQUERADE
iptables -t nat -A POSTROUTING -o ens20 -s 172.16.2.0/28 -j MASQUERADE
iptables-save > /etc/sysconfig/iptables
systemctl enable --now iptables
apt-get update && apt-get reinstall tzdata -y
timedatectl set-timezone Asia/Yekaterinburg
exec bash

```
## HQ-RTR
```tml
en
conf t
hostname hq-rtr
ip domain-name au-team.irpo
interface int0
description “to isp”
ip address 172.16.1.4/28
exit
port te0
service-instance te0/int0
encapsulation untagged
exit
exit
interface int0
connect port te0 service-instance te0/int0
exit
interface int1
description “to hq-srv”
ip address 192.168.1.1/27
exit
interface int2
description “to hq-cli”
ip address 192.168.2.1/28
exit
port te1
service-instance te1/int1
encapsulation dot1q 100
rewrite pop 1
exit
service-instance te1/int2
encapsulation dot1q 200
rewrite pop 1
exit
exit
interface int1
connect port te1 service-instance te1/int1
exit
interface int2
connect port te1 service-instance te1/int2
exit
interface int3
description “999”
ip address 192.168.99.1/29
exit
port te1
service-instance te1/int3
encapsulation dot1q 999
rewrite pop 1
exit
exit
interface int3
connect port te1 service-instance te1/int3
exit
int tunnel.0
ip address 172.16.0.1/30
ip mtu 1400
ip tunnel 172.16.1.4 172.16.2.5 mode gre
ip ospf authentication-key ecorouter
exit
ip route 0.0.0.0 0.0.0.0 172.16.1.1
username net_admin
password P@ssw0rd
role admin
exit
router ospf 1
network 172.16.0.1/30 area 0
network 192.168.1.0/27 area 0
network 192.168.2.0/28 area 0
passive-interface default
no passive-interface tunnel.0
area 0 authentication
exit
interface int1
ip nat inside
exit
interface int2
ip nat inside
exit
interface int0
ip nat outside
exit
ip name-server 8.8.8.8
ip nat pool NAT_POOL 192.168.1.1-192.168.1.254,192.168.2.1-192.168.2.254
ip nat source dynamic inside-to-outside pool NAT_POOL overload interface int0
ip pool cli_pool 192.168.2.10-192.168.2.10
dhcp-server 1
pool cli_pool 1
mask 255.255.255.240
gateway 192.168.2.1
dns 192.168.1.10
domain-name au-team.irpo
exit
exit
interface int2
dhcp-server 1
exit
ntp timezone utc+5
exit
write

```
## BR-RTR
```tml
en
conf t
hostname br-rtr
ip domain-name au-team.irpo
interface int0
description “to isp”
ip address 172.16.2.5/28
exit
port te0
service-instance te0/int0
encapsulation untagged
exit
exit
interface int0
connect port te0 service-instance te0/int0
exit
interface int1
description “to br-srv”
ip address 192.168.3.1/28
exit
port te1
service-instance te1/int1
encapsulation untagged
exit
exit
interface int1
connect port te1 service-instance te1/int1
exit
int tunnel.0
ip address 172.16.0.2/30
ip mtu 1400
ip tunnel 172.16.2.5 172.16.1.4 mode gre
ip ospf authentication-key ecorouter
exit
username net_admin
password P@ssw0rd
role admin
exit
ip route 0.0.0.0 0.0.0.0 172.16.2.1
router ospf 1
network 172.16.0.2/30 area 0
network 192.168.3.0/28 area 0
passive-interface default
no passive-interface tunnel.0
area 0 authentication
exit
interface int1
ip nat inside
exit
interface int0
ip nat outside
exit
ip name-server 8.8.8.8
ip nat pool NAT_POOL 192.168.3.1-192.168.3.254
ip nat source dynamic inside-to-outside pool NAT_POOL overload interface int0
ntp timezone utc+5
exit
write

```

## HQ-SRV
```tml
hostnamectl hostname hq-srv.au-team.irpo
timedatectl set-timezone Asia/Yekaterinburg
adduser sshuser -u 2026 && echo "P@ssw0rd" | passwd --stdin sshuser
sed -i 's/# WHEEL_USERS ALL=(ALL:ALL) NOPASSWD: ALL/ WHEEL_USERS ALL=(ALL:ALL) NOPASSWD: ALL/g' /etc/sudoers
gpasswd -a "sshuser" wheel
echo Authorized access only > /etc/openssh/banner
sed -i '1i\Port 2026\nAllowUsers sshuser\nMaxAuthTries 2\nPasswordAuthentication yes\nBanner /etc/openssh/banner' /etc/openssh/sshd_config
systemctl restart sshd
mkdir /etc/net/ifaces/ens20
echo -e "DISABLED=no\nTYPE=eth\nBOOTPROTO=static\nCONFIG_IPV4=yes" >> /etc/net/ifaces/ens20/options
echo 192.168.1.10/27 > /etc/net/ifaces/ens20/ipv4address
echo default via 192.168.1.1 > /etc/net/ifaces/ens20/ipv4route
echo nameserver 8.8.8.8 > /etc/resolv.conf
systemctl restart network
apt-get update && apt-get install dnsmasq -y
sed -i '1i\no-resolv\ndomain=au-team.irpo\nserver=8.8.8.8\ninterface=*\naddress=/hq-rtr.au-team.irpo/192.168.1.1\nptr-record=1.1.168.192.in-addr.arpa,hq-rtr.au-team.irpo\naddress=/docker.au-team.irpo/172.16.1.1\naddress=/web.au-team.irpo/172.16.2.1\naddress=/br-rtr.au-team.irpo/192.168.3.1\naddress=/hq-srv.au-team.irpo/192.168.1.10\nptr-record=10.1.168.192.in-addr.arpa,hq-srv.au-team.irpo\naddress=/hq-cli.au-team.irpo/192.168.2.10\nptr-record=10.2.168.192.in-addr.arpa,hq-cli.au-team.irpo\naddress=/br-srv.au-team.irpo/192.168.3.10' /etc/dnsmasq.conf
echo -e "192.168.1.1\thq-rtr.au-team.irpo" >> /etc/hosts
systemctl enable --now dnsmasq
systemctl restart dnsmasq
exec bash

```

## HQ-CLI
```tml
cp /etc/apt/sources.list.d/alt.list /etc/apt/sources.list.d/alt.list.bak
sed -i 's|^rpm.*ftp\.altlinux|# &|g' /etc/apt/sources.list.d/alt.list
cat >> /etc/apt/sources.list.d/alt.list <<EOF
rpm [p10] http://192.168.0.222/mirror p10/branch/x86_64 classic
rpm [p10] http://192.168.0.222/mirror p10/branch/noarch classic
rpm [p10] http://192.168.0.222/mirror p10/branch/x86_64-i586 classic
EOF
hostnamectl set-hostname hq-cli.au-team.irpo
mkdir /etc/net/ifaces/ens20
echo -e "BOOTPROTO=dhcp\nDISABLED=no\nTYPE=eth\nCONFIG_IPV4=yes" > /etc/net/ifaces/ens20/options
systemctl restart network
timedatectl set-timezone Asia/Yekaterinburg
exec bash
apt-get update

```

## BR-SRV
```tml
cp /etc/apt/sources.list.d/alt.list /etc/apt/sources.list.d/alt.list.bak
sed -i 's|^rpm.*ftp\.altlinux|# &|g' /etc/apt/sources.list.d/alt.list
cat >> /etc/apt/sources.list.d/alt.list <<EOF
rpm [p10] http://192.168.0.222/mirror p10/branch/x86_64 classic
rpm [p10] http://192.168.0.222/mirror p10/branch/noarch classic
rpm [p10] http://192.168.0.222/mirror p10/branch/x86_64-i586 classic
EOF
hostnamectl set-hostname br-srv.au-team.irpo
mkdir /etc/net/ifaces/ens20
echo -e "BOOTPROTO=static\nDISABLED=no\nTYPE=eth\nCONFIG_IPV4=yes" > /etc/net/ifaces/ens20/options
echo 192.168.3.10/28 > /etc/net/ifaces/ens20/ipv4address
echo default via 192.168.3.1 > /etc/net/ifaces/ens20/ipv4route
systemctl restart network
useradd -u 2026 sshuser
echo 'sshuser:P@ssw0rd' | chpasswd
sed -i 's/^\s*#\s*\(WHEEL_USERS\s\+ALL=(ALL:ALL)\s\+NOPASSWD:\s\+ALL\)/\1/' /etc/sudoers
gpasswd -a "sshuser" wheel
echo -e "Port 2026\nAllowUsers sshuser\nMaxAuthTries 2\nPasswordAuthentication yes\nBanner /etc/openssh/banner" > /etc/openssh/sshd_config
echo Aauthorized access only > /etc/openssh/banner
systemctl restart sshd
echo nameserver 8.8.8.8 > /etc/resolv.conf
timedatectl set-timezone Asia/Yekaterinburg
exec bash

```
