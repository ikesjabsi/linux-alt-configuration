# l
## part 1: Network Infrastructure (Basic Configuration)

**Characteristic:** Complete infrastructure deployment without any preconfigured settings.

### 1.1 Set Hostnames
ALT Linux (HQ-SRV, BR-SRV, HQ-CLI, ISP)
```bash
hostnamectl set-hostname <hostname>.au-team.irpo
exec bash
```

EcoRouterOS (HQ-RTR, BR-RTR)
```bash
enable
configure terminal
hostname <hostname>
ip domain-name au-team.irpo
write memory
```

### 1.2 IPv4 Addressinginux-alt-configuration

ALT Linux
```bash
echo "<IP-address>/<prefix>" > /etc/net/ifaces/<interface>/ipv4address
echo "default via <gateway-ip>" > /etc/net/ifaces/<interface>/ipv4route
systemctl restart network
```

EcoRouterOS
```bash
configure terminal
interface <name>
 ip address <IP>/<prefix>
 exit
port <portname>
 service-instance <name>
  encapsulation untagged|dot1q <VID>
  connect ip interface <name>
  exit
 exit
write memory
```

### 1.3 Internet Access (ISP)
```bash
# DHCP on the external interface
echo "BOOTPROTO=dhcp" > /etc/net/ifaces/<ext-interface>/options

# Enable IP forwarding
sysctl -w net.ipv4.ip_forward=1

# NAT using iptables
iptables -t nat -A POSTROUTING -o <ext-if> -j MASQUERADE
```

### 1.4 User Management
ALT Linux
```bash
useradd <username> -u <UID>
passwd <username>
usermod -aG wheel <username>
echo "<username> ALL=(ALL:ALL) NOPASSWD: ALL" >> /etc/sudoers
```

EcoRouterOS
```bash
configure terminal
username <name>
 password <password>
 role admin
 exit
write memory
```

### 1.5 SSH Service (Hardened)
```bash
# /etc/openssh/sshd_config
Port 2026
AllowUsers <username>
MaxAuthTries 2
Banner /etc/openssh/banner

systemctl restart sshd
```

### 1.6 Tunnel (GRE or IP-in-IP)
```bash
configure terminal
interface tunnel.0
 ip address <IP>/<prefix>
 ip tunnel <source-IP> <destination-IP> mode gre|ipip
 exit
write memory
```

### 1.7 Dynamic Routing (OSPF)
```bash
configure terminal
router ospf 1
 passive-interface default
 no passive-interface tunnel.0
 network <network>/<prefix> area 0
 area 0 authentication
 interface tunnel.0
  ip ospf authentication-key <key>
  exit
 exit
write memory
```

### 1.8 DHCP Server (EcoRouterOS)
```bash
configure terminal
ip pool <name> <start-IP>-<end-IP>
dhcp-server 1
 pool <name> 1
  mask <prefix>
  gateway <gateway-IP>
  dns <dns-server-IP>
  domain-name <domain>
  exit
 exit
interface <interface>
 dhcp-server 1
 exit
write memory
```

### 1.9 DNS Server (BIND on ALT Linux)
```bash
apt-get update && apt-get install -y bind bind-utils

# /var/lib/bind/etc/options.conf
listen-on { <local-IP>; };
forwarders { <public-DNS>; };
allow-query { <allowed-networks>; };

# Zone configuration in /var/lib/bind/etc/rfc1912.conf
zone "<domain>" { type master; file "master/<zone>"; };

systemctl enable --now bind
```

### 1.10 Time Zone
```bash
# ALT Linux
timedatectl set-timezone Europe/Moscow

# EcoRouterOS
configure terminal
ntp timezone utc+3
```

## part 2: Network Administration (Preconfigured Environment)

**Characteristic:** The environment is partially prepared (IP addresses, routing, NAT, DNS, and DHCP are already configured).

### 2.1 Samba Domain Controller (BR-SRV)
```bash
apt-get update && apt-get install -y task-samba-dc

systemctl stop krb5kdc slapd bind
systemctl disable krb5kdc slapd bind

rm -f /etc/samba/smb.conf
rm -rf /var/lib/samba

samba-tool domain provision --use-rfc2307 --interactive

cp /var/lib/samba/private/krb5.conf /etc/krb5.conf
systemctl enable --now samba
```

### 2.2 Users and Groups in the Domain
```bash
samba-tool group add <group>
samba-tool user add <user> <password>
samba-tool group addmembers <group> <user>
```

### 2.3 Join Domain (HQ-CLI)
```bash
# Point DNS to BR-SRV
echo "nameserver 192.168.0.2" > /etc/net/ifaces/<if>/resolv.conf

apt-get update && apt-get install -y task-auth-ad-sssd
# GUI configuration via Central Control System (CCS)

apt-get install -y libnss-role
roleadd <domain-group> wheel

echo "%wheel ALL=(ALL:ALL) NOPASSWD: /bin/cat, /bin/grep, /usr/bin/id" >> /etc/sudoers
```

### 2.4 RAID 0 (HQ-SRV)
```bash
apt-get install -y mdadm
mdadm --create --verbose /dev/md0 -l 0 -n 2 /dev/sdb /dev/sdc
mkfs.ext4 /dev/md0
mkdir /raid
echo "/dev/md0 /raid ext4 defaults 0 0" >> /etc/fstab
mount -av
```

### 2.5 NFS Share (HQ-SRV → HQ-CLI)
```bash
# Server
apt-get install -y nfs-server nfs-utils
mkdir /raid/nfs
echo "/raid/nfs 192.168.200.0/24(rw,no_root_squash)" >> /etc/exports
systemctl enable --now nfs-server

# Client
apt-get install -y nfs-utils nfs-clients
mkdir /mnt/nfs
echo "192.168.100.2:/raid/nfs /mnt/nfs nfs defaults 0 0" >> /etc/fstab
mount -av
```

### 2.6 Ansible (Control Node: BR-SRV)
```bash
apt-get update && apt-get install -y ansible sshpass
ansible-galaxy collection install ansible.netcommon cisco.ios
```
Inventory (`/etc/ansible/hosts`):
```ini
[servers]
hq-srv ansible_host=192.168.100.2
hq-cli ansible_host=192.168.200.2
[routers]
hq-rtr ansible_host=192.168.100.1
br-rtr ansible_host=192.168.0.1
[all:vars]
ansible_user=sshuser
```

### 2.7 Docker Containers (testapp on BR-SRV)
```bash
apt-get install -y docker-engine docker-compose-v2
systemctl enable --now docker

docker load < /mnt/docker/site_latest.tar
docker load < /mnt/docker/mariadb_latest.tar

docker-compose up -d
```

### 2.8 LAMP Web Application (HQ-SRV)
```bash
apt-get install -y lamp-server
systemctl enable --now mariadb httpd2

mysql -u root
CREATE DATABASE webdb;
CREATE USER 'web'@'localhost' IDENTIFIED BY 'P@ssw0rd';
GRANT ALL PRIVILEGES ON webdb.* TO 'web'@'localhost';
EXIT;

mysql -u web -p webdb < /mnt/web/dump.sql
```

### 2.9 Static NAT (Port Forwarding)
```bash
configure terminal
ip nat source static tcp <local-IP> <local-port> <external-IP> <external-port>
write memory
```

### 2.10 Reverse Proxy (Nginx on ISP)
```bash
apt-get update && apt-get install -y nginx

# Configuration in /etc/nginx/sites-available.d/default.conf
server {
    listen 80;
    server_name <domain>;
    location / {
        proxy_pass http://<destination-IP>:<port>;
    }
}

systemctl enable --now nginx
```

### 2.11 HTTP Basic Authentication (Nginx)
```bash
htpasswd -c /etc/nginx/.htpasswd <username>
# Add auth_basic and auth_basic_user_file inside the location block
systemctl reload nginx
```

### 2.12 Yandex Browser (HQ-CLI)
```bash
apt-get update && apt-get install -y yandex-browser-stable
```

## part 3: Advanced Administration (Security, Monitoring, Backup)

**Characteristic:** Advanced topics including migration, GOST certificates, IPSec, firewall configuration, logging, monitoring, inventory collection, Fail2ban, and backup management.

### 3.1 Bulk User Import (CSV → Samba)
```bash
mount /dev/cdrom /mnt
cp /mnt/Users.csv /root
iconv -f UTF-8 -t UTF-8//IGNORE /root/Users.csv > /root/Users_fixed.csv

# Import script (see original document)
bash /root/import.sh
```

### 3.2 GOST Certificates (CA on HQ-SRV, HTTPS on ISP)
```bash
# Create CA
control openssl-gost enabled
openssl genpkey -algorithm gost2012_256 -pkeyopt paramset:TCB -out ca.key
openssl req -new -x509 -md_gost12_256 -days 90 -key ca.key -out ca.crt

# Server certificates
openssl genpkey -algorithm gost2012_256 -pkeyopt paramset:A -out <name>.key
openssl req -new -md_gost12_256 -key <name>.key -out <name>.csr
openssl x509 -req -in <name>.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out <name>.crt -days 30

# Copy to ISP
scp *.crt *.key root@172.16.1.1:/etc/nginx/

# Nginx configuration: port 443 with SSL parameters
```

### 3.3 IPSec (Secure Tunnel Between HQ-RTR and BR-RTR)
```bash
configure terminal
crypto-ipsec ike enable
crypto-ipsec profile IPSEC ike-v2
 mode tunnel
 ike-phase1 proposal aes256-sha256-modp2048 auth pre-shared-key <key>
 ike-phase2 protocol esp proposal aes256-sha256
 local-ts <local-IP> remote-ts <remote-IP>
exit
crypto-map CMAP 10 match peer <remote-IP> set crypto-ipsec profile IPSEC
filter-map ipv4 FMAP 10 match gre host <local> host <remote> set crypto-map CMAP peer <remote>
interface tunnel.0 set filter-map in FMAP 10
write memory
```

### 3.4 Firewall (EcoRouterOS)
```bash
# Remove default allow rule
no filter-map ipv4 FMAP 30

# Permit rules for specific protocols (HTTP, HTTPS, DNS, NTP, ICMP, Kerberos, LDAP, SMB)
filter-map ipv4 FMAP 21
 match tcp any any eq 88
 match udp any any eq dns
 # ... (additional protocols according to requirements)
 set accept
exit
```

### 3.5 NTP Server (chrony on ISP)
```bash
apt-get install -y chrony
# /etc/chrony.conf: server <public-ntp>, local stratum 5, allow 0.0.0.0/0
systemctl restart chronyd
```

### 3.6 CUPS Print Server (HQ-SRV)
```bash
apt-get install -y cups cups-pdf
# /etc/cups/cupsd.conf: allow access in <Location /> and <Location /admin>
systemctl enable --now cups
```

### 3.7 Rsyslog (Central Log Server: HQ-SRV)
```bash
# Server
module(load="imudp")
input(type="imudp" port="514")
if $fromhost-ip == '<client-IP>' then /opt/<hostname>/log

# Client (BR-SRV, BR-RTR)
*.warn @@<server-IP>:514
```

### 3.8 Monitoring (Prometheus + Grafana on HQ-SRV)
```bash
apt-get install -y grafana prometheus prometheus-node_exporter
# prometheus.yml: targets [localhost:9090, <hq-srv>:9100, <br-srv>:9100]
systemctl enable --now grafana-server prometheus prometheus-node_exporter
```

### 3.9 Inventory Collection with Ansible (Playbook)
```yaml
- name: Get host info
  hosts: hq-srv, hq-cli
  tasks:
    - name: Save info
      copy:
        dest: "/etc/ansible/PC-INFO/{{ ansible_hostname }}.yml"
        content: |
          Hostname: {{ ansible_hostname }}
          IP_Address: {{ ansible_default_ipv4.address }}
      delegate_to: localhost
```

### 3.10 Fail2ban (SSH Protection)
```bash
apt-get install -y fail2ban
# /etc/fail2ban/jail.local
[sshd]
enabled = true
port = 2026
maxretry = 3
bantime = 60
logpath = %(sshd_log)s

systemctl enable --now fail2ban
```

### 3.11 Cyber Backup (Installation and Basic Configuration)
```bash
# Kernel update required (update-kernel && reboot)
# Installation via ISO: Management Server, Agent for Linux, Agent for MySQL
# User: iproadmin / Password: P@ssw0rd
# Storage node on HQ-CLI using /backup directory
# Two backup plans: /etc (files) and webdb (MySQL)
```
