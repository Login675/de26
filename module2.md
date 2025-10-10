## Подготовка машин
1) Добавить два жестких диска объемом 1 гигабайт: Перейти в PVE > выбрать HQ-SRV > Hardware > Add > Hard disk > Add
2) Добавить ISO образ: Перейти в PVE > выбрать пункт local (AltPVE) > ISO Images > Upload >
Выбрать путь к ISO > Upload
## BR-SRV
```tml
apt-get update && apt-get install task-samba-dc -y
echo nameserver 192.168.1.10 > /etc/resolv.conf
rm -rf /etc/samba/smb.conf

```
## HQ-SRV
```tml
sed -i '/address=\/br-srv\.au-team\.irpo\/192\.168\.3\.10/a server=/au-team.irpo/192.168.3.10' /etc/dnsmasq.conf
systemctl restart  dnsmasq.service

```
## BR-SRV
```tml
expect -c 'spawn samba-tool domain provision; sleep 1; send "\r\r\r\r\r"; expect "Password:"; send "P@ssw0rd\r"; expect "Retype password:"; send "P@ssw0rd\r"; interact'
mv –f /var/lib/samba/private/krb5.conf /etc/krb5.conf
systemctl enable --now samba
samba-tool user add hquser1 P@ssword
samba-tool user add hquser2 P@ssword
samba-tool user add hquser3 P@ssword
samba-tool user add hquser4 P@ssword
samba-tool user add hquser5 P@ssword
samba-tool group add hq
samba-tool group addmembers hq hquser1,hquser2,hquser3,hquser4,hquser5

```
## HQ-CLI
```tml


```
###RAID
## HQ-SRV
```tml
mdadm --create /dev/md0 --level=0 --raid-devices=2 /dev/sd[b-с]
mdadm  --detail --scan --verbose > /etc/mdadm.conf
apt-get update && apt-get install fdisk -y
echo -e "n\n\n\n\n\nw\n" | fdisk /dev/md0
mkfs.ext4 /dev/md0p1
echo -e "/dev/md0p1\t/raid\text4\tdefaults\t0\t0" | tee -a /etc/fstab
mkdir /raid
mount -a
apt-get install nfs-server -y
mkdir /raid/nfs
chown 99:99 /raid/nfs
chmod 777 /raid/nfs
echo "/raid/nfs 192.168.2.0/28(rw,sync,no_subtree_check)" | tee -a /etc/exports
exportfs -a
exportfs -v
systemctl enable --now nfs
systemctl restart nfs

```
## HQ-CLI
```tml
apt-get update && apt-get install nfs-clients -y
mkdir –p /mnt/nfs
echo -e "192.168.1.10:/raid/nfs\t/mnt/nfs\tnfs\tintr,soft,_netdev,x-systemd.automount\t0\t0" | tee -a /etc/fstab
mount -a
mount -v
touch /mnt/nfs/test

```
### NTP
## ISP
```tml
apt-get install chrony -y
echo -e "server 127.0.0.1 iburst prefer\nhwtimestamp *\nlocal stratum 5\nallow 0/0" > /etc/chrony.conf
systemctl enable --now chronyd
systemctl restart chronyd
chronyc tracking | grep Stratum

```
## HQ-RTR
```tml
en
conf t
ntp server 172.16.1.1
ntp timezone utc+5
ex
show ntp status
write

```
## BR-RTR
```tml
en
conf t
ntp server 172.16.1.1
ntp timezone utc+5
ex
show ntp status
write

```
## HQ-CLI
```tml
apt-get install chrony -y
echo -e "server 172.16.1.1 iburst prefer" > /etc/chrony.conf
systemctl enable --now chronyd
systemctl restart chronyd
chronyc sources

```
## HQ-SRV
```tml
apt-get install chrony -y
echo -e "server 172.16.1.1 iburst prefer" > /etc/chrony.conf
systemctl enable --now chronyd
systemctl restart chronyd
chronyc sources

```
## BR-SRV
```tml
apt-get install chrony -y
echo -e "server 172.16.1.1 iburst prefer" > /etc/chrony.conf
systemctl enable --now chronyd
systemctl restart chronyd
chronyc sources

```


