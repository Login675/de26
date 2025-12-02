#### BR-SRV
```tml
mkdir /iso
mount -o loop /dev/sr0 /iso
echo "/dev/sr0	/iso	iso9660	loop,ro,auto	0	0" | sudo tee -a /etc/fstab
cp /iso/playbook/get_hostname_address.yml /etc/ansible/playbook.yml
mkdir /etc/ansible/PC-INFO
cat > /etc/ansible/playbook.yml << 'EOF'
- name: PC-INFO
  hosts: LMX   
  tasks:   
     -  name: получение данных с хоста      
        copy:        
           dest: /etc/ansible/PC-INFO/{{ ansible_hostname }}.txt
           content: |            
               Hostname: {{ ansible_hostname }}            
               IP-Adress: {{ ansible_default_ipv4.address }}      
        delegate_to: 127.0.0.1
EOF
sh -c 'cat <<EOF >> /etc/ansible/hosts
LMX:
 hosts:
  HQ-SRV:
  HQ-CLI:
EOF'
ansible-playbook /etc/ansible/playbook.yml
ls -la /etc/ansible/PC-INFO
cat /etc/ansible/PC-INFO/hq-srv.txt
cat /etc/ansible/PC-INFO/hq-cli.txt
```
