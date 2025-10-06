```tml
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
