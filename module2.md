## Подготовка машин
1) Добавить два жестких диска объемом 1 гигабайт(HQ-SRV): Перейти в PVE > выбрать HQ-SRV > Hardware > Add > Hard disk > Add
2) Добавить ISO образ(HQ-SRV,BR-SRV): Перейти в PVE > выбрать пункт local (AltPVE) > ISO Images > Upload >
Выбрать путь к ISO > Upload
## Samba
#### HQ-SRV
```tml
echo "server=/au-team.irpo/192.168.3.10" >> /etc/dnsmasq.conf
systemctl restart dnsmasq

```
#### BR-SRV
```tml
apt-get update && apt-get install wget dos2unix task-samba-dc -y
sleep 3
echo nameserver 192.168.1.10 > /etc/resolv.conf
sleep 2
echo 192.168.3.10 br-srv.au-team.irpo >> /etc/hosts
rm -rf /etc/samba/smb.conf
samba-tool domain provision --realm=AU-TEAM.IRPO --domain=AU-TEAM --adminpass=P@ssw0rd --dns-backend=SAMBA_INTERNAL --server-role=dc --option='dns forwarder=192.168.1.10'
mv -f /var/lib/samba/private/krb5.conf /etc/krb5.conf
systemctl enable --now samba.service
samba-tool user add hquser1 P@ssw0rd
samba-tool user add hquser2 P@ssw0rd
samba-tool user add hquser3 P@ssw0rd
samba-tool user add hquser4 P@ssw0rd
samba-tool user add hquser5 P@ssw0rd
samba-tool group add hq
samba-tool group addmembers hq hquser1,hquser2,hquser3,hquser4,hquser5
wget https://raw.githubusercontent.com/sudo-project/sudo/main/docs/schema.ActiveDirectory
dos2unix schema.ActiveDirectory
sed -i 's/DC=X/DC=au-team,DC=irpo/g' schema.ActiveDirectory
head -$(grep -B1 -n '^dn:$' schema.ActiveDirectory | head -1 | grep -oP '\d+') schema.ActiveDirectory > first.ldif
tail +$(grep -B1 -n '^dn:$' schema.ActiveDirectory | head -1 | grep -oP '\d+') schema.ActiveDirectory | sed '/^-/d' > second.ldif
ldbadd -H /var/lib/samba/private/sam.ldb first.ldif --option="dsdb:schema update allowed"=true
ldbmodify -v -H /var/lib/samba/private/sam.ldb second.ldif --option="dsdb:schema update allowed"=true
samba-tool ou add 'ou=sudoers'
cat << EOF > sudoRole-object.ldif
dn: CN=prava_hq,OU=sudoers,DC=au-team,DC=irpo
changetype: add
objectClass: top
objectClass: sudoRole
cn: prava_hq
name: prava_hq
sudoUser: %hq
sudoHost: ALL
sudoCommand: /bin/grep
sudoCommand: /bin/cat
sudoCommand: /usr/bin/id
sudoOption: !authenticate
EOF
ldbadd -H /var/lib/samba/private/sam.ldb sudoRole-object.ldif
echo -e "dn: CN=prava_hq,OU=sudoers,DC=au-team,DC=irpo\nchangetype: modify\nreplace: nTSecurityDescriptor" > ntGen.ldif
ldbsearch  -H /var/lib/samba/private/sam.ldb -s base -b 'CN=prava_hq,OU=sudoers,DC=au-team,DC=irpo' 'nTSecurityDescriptor' | sed -n '/^#/d;s/O:DAG:DAD:AI/O:DAG:DAD:AI\(A\;\;RPLCRC\;\;\;AU\)\(A\;\;RPWPCRCCDCLCLORCWOWDSDDTSW\;\;\;SY\)/;3,$p' | sed ':a;N;$!ba;s/\n\s//g' | sed -e 's/.\{78\}/&\n /g' >> ntGen.ldif
ldbmodify -v -H /var/lib/samba/private/sam.ldb ntGen.ldif

```
#### HQ-CLI
```tml
systemctl restart network
apt-get update && apt-get install bind-utils sudo libsss_sudo -y
system-auth write ad AU-TEAM.IRPO cli AU-TEAM 'administrator' 'P@ssw0rd'

control sudo public
sed -i '19 a\
sudo_provider = ad' /etc/sssd/sssd.conf
sed -i 's/services = nss, pam/services = nss, pam, sudo/' /etc/sssd/sssd.conf
sed -i '28 a\
sudoers: files sss' /etc/nsswitch.conf
rm -rf /var/lib/sss/db/*
sss_cache -E
systemctl restart sssd
sudo -l -U hquser1

```
## RAID
#### HQ-SRV
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
#### HQ-CLI
```tml
apt-get update && apt-get install nfs-clients -y
mkdir –p /mnt/nfs
echo -e "192.168.1.10:/raid/nfs\t/mnt/nfs\tnfs\tintr,soft,_netdev,x-systemd.automount\t0\t0" | tee -a /etc/fstab
mount -a
mount -v
touch /mnt/nfs/test

```
## NTP
#### ISP
```tml
apt-get update && apt-get install chrony -y
echo -e "server 127.0.0.1 iburst prefer\nhwtimestamp *\nlocal stratum 5\nallow 0/0" > /etc/chrony.conf
systemctl enable --now chronyd
systemctl restart chronyd
chronyc tracking | grep Stratum

```
#### HQ-RTR
```tml
en
conf t
ntp server 172.16.1.1
ntp timezone utc+5
ex
show ntp status
write

```
#### BR-RTR
```tml
en
conf t
ntp server 172.16.1.1
ntp timezone utc+5
ex
show ntp status
write

```
#### HQ-CLI
```tml
apt-get update && apt-get install chrony -y
echo -e "server 172.16.1.1 iburst prefer" > /etc/chrony.conf
systemctl enable --now chronyd
systemctl restart chronyd
sleep 2
chronyc sources

```
#### HQ-SRV
```tml
apt-get update && apt-get install chrony -y
echo -e "server 172.16.1.1 iburst prefer" > /etc/chrony.conf
systemctl enable --now chronyd
systemctl restart chronyd
sleep 2
chronyc sources

```
#### BR-SRV
```tml
apt-get update && apt-get install chrony -y
echo -e "server 172.16.1.1 iburst prefer" > /etc/chrony.conf
systemctl enable --now chronyd
systemctl restart chronyd
sleep 2
chronyc sources

```
## Ansible
#### BR-SRV
```tml
apt-get update && apt-get install ansible -y
echo -e "VMs:\n hosts:\n  HQ-SRV:\n   ansible_host: 192.168.1.10\n   ansible_user: sshuser\n   ansible_port: 2026\n  HQ-CLI:\n   ansible_host: 192.168.2.10\n   ansible_user: sshuser\n   ansible_port: 2026\n  HQ-RTR:\n   ansible_host: 192.168.1.1\n   ansible_user: net_admin\n   ansible_password: P@ssw0rd\n   ansible_connection: network_cli\n   ansible_network_os: ios\n  BR-RTR:\n   ansible_host: 192.168.3.1\n   ansible_user: net_admin\n   ansible_password: P@ssw0rd\n   ansible_connection: network_cli\n   ansible_network_os: ios" > /etc/ansible/hosts
sed -i '/^\[defaults\]/a\
ansible_python_interpreter=/usr/bin/python3\
interpreter_python=auto_silent\
ansible_host_key_checking=false' /etc/ansible/ansible.cfg

```
#### HQ-CLI
```tml
useradd sshuser -u 2026
echo 'sshuser:P@ssw0rd' | chpasswd
sed -i 's/^#\s*\(WHEEL\s\+USERS\s\+ALL=(ALL:ALL)\s\+NOPASSWD:\s\+ALL\)/\1/' /etc/sudoers
gpasswd -a “sshuser” wheel
echo -e "Port 2026\nAllowUsers sshuser\nMaxAuthTries 2\nPasswordAuthentication yes\nBanner /etc/openssh/banner" > /etc/openssh/sshd_config
echo Aauthorized access only > /etc/openssh/banner
systemctl restart sshd

```
#### BR-SRV
```tml
apt-get update && apt-get install sshpass -y
ssh-keygen -t rsa -f ~/.ssh/id_rsa -N ""
sshpass -p 'P@ssw0rd' ssh-copy-id -o StrictHostKeyChecking=no -p 2026 sshuser@192.168.1.10
sshpass -p 'P@ssw0rd' ssh-copy-id -o StrictHostKeyChecking=no -p 2026 sshuser@192.168.2.10
ansible all -m ping

```
## Docker
#### BR-SRV
```tml
apt-get update && apt-get install docker-compose docker-engine -y
systemctl enable --now docker
systemctl status docker
mount -o loop /dev/sr0
docker load < /media/ALTLinux/docker/site_latest.tar
docker load < /media/ALTLinux/docker/mariadb_latest.tar
echo -e "services:\n db:\n  image: mariadb\n  container_name: db\n  environment:\n\tDB_NAME: testdb\n\tDB_USER: test\n\tDB_PASS: Passw0rd\n\tMYSQL_ROOT_PASSWORD: Passw0rd\n\tMYSQL_DATABASE: testdb\n\tMYSQL_USER: test\n\tMYSQL_PASSWORD: Passw0rd\n  volumes:\n\t- db_data:/var/lib/mysql\n  networks:\n\t- app_network\n  restart: unless-stopped\n\n testapp:\n  image: site\n  container_name: testapp\n  environment:\n\tDB_TYPE: maria\n\tDB_HOST: db\n\tDB_NAME: testdb\n\tDB_USER: test\n\tDB_PASS: Passw0rd\n\tDB_PORT: 3306\n  ports:\n\t- "8080:8000"\n  networks:\n\t- app_network\n  depends_on:\n\t- db\n  restart: unless-stopped\nvolumes:\n db_data:\n\nnetworks:\n app_network:\n  driver: bridge" > /root/site.yml
docker compose -f site.yml up -d
docker exec -it db mysql -u root -pPassw0rd -e "CREATE DATABASE testdb; CREATE USER 'test'@'%' IDENTIFIED BY 'Passw0rd'; GRANT ALL PRIVILEGES ON testdb.* TO 'test'@'%'; FLUSH PRIVILEGES;"
docker compose -f site.yml down && docker compose -f site.yml up -d
mkdir -p /root/config
echo -e "#! /bin/bash\n\ndocker compose -f /root/site.yml down\nsystemctl restart docker\ndocker compose -f /root/site.yml up -d" > /root/config/autorestart.sh
export EDITOR=vim
crontab -e
# Нажимайте i, и в самом конце пишите
@reboot /root/config/autorestart.sh
# Затем нажимайте ESC > :wq
```
#### HQ-CLI
```tml
systemctl restart network
curl -I http://192.168.3.10:8080

```
## Moodle
#### HQ-SRV
```tml
apt-get update
apt-get install apache2 php8.2 apache2-mod_php8.2 mariadb-server php8.2-{opcache,curl,gd,intl,mysqli,xml,xmlrpc,ldap,zip,soap,mbstring,json,xmlreader,fileinfo,sodium} -y
mount -o loop /dev/sr0
systemctl enable --now httpd2 mysqld
echo -e "\n\n\nP@ssw0rd" | mysql_secure_installation
mariadb -u root -p
CREATE DATABASE webdb;
CREATE USER 'webc'@'localhost' IDENTIFIED BY 'P@ssw0rd';
GRANT ALL PRIVILEGES ON webdb.* TO 'webc'@'localhost';
FLUSH PRIVILEGES;
exit
iconv -f UTF-16LE -t UTF-8 /media/ALTLinux/web/dump.sql > /tmp/dump_utf8.sql
mariadb -u root -p webdb < /tmp/dump_utf8.sql
chmod 777 /var/www/html
cp /media/ALTLinux/web/index.php /var/www/html
cp /media/ALTLinux/web/logo.png /var/www/html
rm -f /var/www/html/index.html
chown apache2:apache2 /var/www/html
systemctl restart httpd2
echo -e "$servername = "localhost";\n$username = "webc";\n$password = "P@ssw0rd";\n$dbname = "webdb";" > /var/www/html/index.php

```
#### HQ-CLI
```tml
systemctl restart network
curl -I http://192.168.1.10
```

## Проброс портов
#### HQ-RTR
```tml
en
conf t
ip nat source static tcp 192.168.1.10 80 172.16.1.4 8080
ip nat source static tcp 192.168.1.10 2026 172.16.1.4 2026
end
wr
```
#### BR-RTR
```tml
en
conf t
ip nat source static tcp 192.168.3.10 8080 172.16.2.5 8080
ip nat source static tcp 192.168.3.10 2026 172.16.2.5 2026
end
wr

```
## Nginx
#### ISP
```tml
apt-get update && apt-get install nginx -y
echo -e "server {\n\tlisten 80;\n\tserver_name web.au-team.irpo;\n\n\tlocation / {\n\t\tproxy_pass http://172.16.1.4:8080;\n\t\tproxy_set_header Host $host;\n\t\tproxy_set_header X-Real-IP $remote_addr;\n\t}\n}\n\nserver {\n\tlisten 80;\n\tserver_name docker.au-tam.irpo;\n\n\tlocation / {\n\t\tproxy_pass http://172.16.2.5:8080;\n\t\tproxy_set_header Host $host;\n\t\tproxy_set_header X-Real-IP $remote_addr;\n\t}\n}" > /etc/nginx/sites-available.d/proxy.conf
ln -s /etc/nginx/sites-available.d/proxy.conf /etc/nginx/sites-enabled.d/
mv /etc/nginx/sites-available.d/default.conf /root/
systemctl enable --now nginx
systemctl restart nginx
curl -I http://web.au-team.irpo
curl -I http://docker.au-team.irpo

```
## web-based аутентификация
#### ISP
```tml
apt-get update && apt-get install apache2-htpasswd -y
htpasswd -bc /etc/nginx/.htpasswd WEB P@ssw0rd
echo -e "server {\n\tlisten 80;\n\tserver_name web.au-team.irpo;\n\n\tlocation / {\n\t\tproxy_pass http://172.16.1.4:8080;\n\t\tproxy_set_header Host $host;\n\t\tproxy_set_header X-Real-IP $remote_addr;\n\t}\n}\n\nserver {\n\tlisten 80;\n\tserver_name docker.au-tam.irpo;\n\tauth_basic "Restricted Access";\n\tauth_basic_user_file /etc/nginx/.htpasswd;\n\tlocation / {\n\t\tproxy_pass http://172.16.2.5:8080;\n\t\tproxy_set_header Host $host;\n\t\tproxy_set_header X-Real-IP $remote_addr;\n\t}\n}" > /etc/nginx/sites-available/proxy.conf
systemctl restart nginx

```
#### HQ-CLI
```tml
systemctl restart network
curl -I http://web.au-team.irpo

```
## Установка Яндекс Браузера
#### HQ-CLI
```tml
apt-get update && apt-get install yandex-browser -y

```
