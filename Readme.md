# Openstack Integration ceph and application 3 tier (Web app, Database, and object store)
## Topology Openstack : 
![](https://raw.githubusercontent.com/Razor-Sec/openstackwith3tierapps/main/Pasted%20image%2020210727164010.png)

## Topology Ceph : 
![](https://github.com/Razor-Sec/openstackwith3tierapps/blob/main/Pasted%20image%2020210727164035.png?raw=true)

## Topology Subnet : 
![[https://github.com/Razor-Sec/openstackwith3tierapps/blob/main/Pasted%20image%2020210727164213.png?raw=true]]

## Pre-requirement
-   9 Node (3 controller+network, 3 compute , 3 Ceph)
-   Internet access on network
-   ubuntu 18.04
-   Ceph Octopus
-   Openstack Ussuri

## Note : 
-  Run command on regular user
-  Run command on deployer

## Permission user :
Note : 
- run this on all node

```bash
$ sudo su
# echo "<your username> ALL=(ALL) NOPASSWD:ALL" | tee /etc/sudoers.d/<your username>
```

## Ceph Deploy

### Install Dependencies
```bash
$ sudo apt update -y;sudo apt upgrade -y
```

### Add /etc/hosts
```bash
# Controller
10.10.10.11 controller1
10.10.10.12 controller2
10.10.10.13 controller3

# Compute
10.10.10.21 compute1
10.10.10.22 compute2
10.10.10.23 compute3

# ceph 
10.10.10.31 ceph1
10.10.10.32 ceph2
10.10.10.33 ceph3
```

### Generate ssh keypair
```bash
ssh-keygen -t rsa
```

### Checking ping and copy ssh id to all ceph node
```bash
ping -c3 ceph1
ping -c3 ceph2
ping -c3 ceph3

ssh-copy-id student@ceph1
ssh-copy-id student@ceph2
ssh-copy-id student@ceph3
```

### Install depedency : 
Note : 
- Run command on ceph1 or ceph deployer

```bash
$ sudo apt-get install python3-pip -y
$ git clone [https://github.com/ceph/ceph-ansible.git](https://github.com/ceph/ceph-ansible.git)  
$ cd ceph-ansible  
$ git checkout stable-5.0
```

### Install requirement : 
```bash 
sudo apt install ansible 
```

### Config hosts ceph : 

```bash
vim /etc/ansible/hosts
...
[mons]
ceph[1:3]

# MDS Nodes
[mdss]
ceph[1:3]

# RGW
[rgws]
ceph[1:3]

# Manager Daemon Nodes
[mgrs]
ceph[1:3]

# set OSD (Object Storage Daemon) Node
[osds]
ceph[1:3]

# Grafana server
[grafana-server]
ceph[1:3]
...
```

### Copy file configuration
```bash
$ cp site.yml.sample site.yml  
$ cd group_vars/  
$ cp all.yml.sample all.yml  
$ cp mons.yml.sample mons.yml  
$ cp osds.yml.sample osds.yml  
$ cp mgrs.yml.sample mgrs.yml
```

### config file all.yml
```bash
# Inventory host group variables

mon_group_name: mons
osd_group_name: osds
rgw_group_name: rgws
mds_group_name: mdss
nfs_group_name: nfss
rbdmirror_group_name: rbdmirrors
client_group_name: clients
iscsi_gw_group_name: iscsigws
mgr_group_name: mgrs
rgwloadbalancer_group_name: rgwloadbalancers
grafana_server_group_name: grafana-server

# Ceph packages
ceph_origin: repository
ceph_repository: community
ceph_repository_type: cdn
ceph_stable_release: octopus


# Interface options
monitor_interface: enp1s0
radosgw_interface: enp1s0

## OSD options
journal_size: 5120 # OSD journal size in MB
public_network: 10.10.10.31/24
cluster_network: 10.10.10.31/24



# DASHBOARD
dashboard_protocol: http
dashboard_port: 8443
dashboard_admin_user: admin
dashboard_admin_password: passwordforceph
grafana_admin_user: admin
grafana_admin_password: passwordforceph
```

### Config OSD
```bash
cp osds.yml.sample osds.yml
vim osds.yml
...
devices:
  - /dev/vdb

osd_auto_discovery: false
...
```

### install ceph 
```bash
$ cd ..
$ sudo apt install python-pip
$ sudo pip install -U pip
$ sudo pip install -r requirements.txt
$ ansible-playbook site.yml
```

### Fix health and vuln 
```bash
ceph -s
```

Note : if health is warning run this command : 
```bash
$ sudo ceph config set mon $ sudo ceph mon_warn_on_insecure_global_id_reclaim_allowed false
$ sudo ceph ceph config set mon auth_expose_insecure_global_id_reclaim false
ceph -s
```

### manage pool and keyring 
Note : Run on ceph1 or ceph deployer :

#### Create pool
```bash
$ sudo ceph osd pool create volumes  
$ sudo ceph osd pool create images  
$ sudo ceph osd pool create backups  
$ sudo ceph osd pool create vms
``` 
#### Set pool for rbd
```bash
$ sudo rbd pool init volumes  
$ sudo rbd pool init images  
$ sudo rbd pool init backups  
$ sudo rbd pool init vms
```
#### Create keyring 
```bash
$ sudo ceph auth get-or-create client.glance mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=images' -o /etc/ceph/ceph.client.glance.keyring
$ sudo auth get-or-create client.cinder mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=volumes, allow rwx pool=images' -o /etc/ceph/ceph.client.cinder.keyring
$ ceph auth get-or-create client.nova mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=vms, allow rx pool=images' -o /etc/ceph/ceph.client.nova.keyring
$ ceph auth get-or-create client.cinder-backup mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=backups' -o /etc/ceph/ceph.client.cinder-backup.keyring
```

### Prepare keyring ceph
in this case using python3 http server for copy the keyring (Optional) , you can use scp for copy keyring
Note : run this on ceph1 or ceph deployer
```bash
$ cd /etc/ceph
$ python3 -m http.server 9001
```
Note : run this on controller1 or kolla-deployer
```bash
$ mkdir ceph_keyring
$ cd ceph_keyring
$ wget ceph1:9001/ceph.conf
$ wget ceph1:9001/ceph.client.cinder-backup.keyring
$ wget ceph1:9001/ceph.client.cinder.keyring
$ wget ceph1:9001/ceph.client.glance.keyring
$ wget ceph1:9001/ceph.client.nova.keyring
```

## Deploy Openstack using kolla
Note : 
- run command on controller1 or kolla deployer 

### Copy ssh id to all node : 
```bash
$ ssh-keygen -t rsa
$ ssh-copy-id student@controller1
$ ssh-copy-id student@controller2
$ ssh-copy-id student@controller3
$ ssh-copy-id student@compute1
$ ssh-copy-id student@compute2
$ ssh-copy-id student@compute3
$ ssh-copy-id student@ceph1
$ ssh-copy-id student@ceph2
$ ssh-copy-id student@ceph3
```

### verification ssh
```bash
$ ssh controller1
$ ssh controller2
$ ssh controller3
$ ssh compute1
$ ssh compute2
$ ssh compute3
$ ssh ceph1
$ ssh ceph2
$ ssh ceph3
```

### Update repository and dependencies
```bash
$ sudo apt update
$ sudo apt-get install python3-dev libffi-dev gcc libssl-dev python3-selinux python3-setuptools python3-venv -y
```

### Create a virtual environment
```bash
$ python3 -m venv kolla-venv
$ source kolla-venv/bin/activate
(kolla-venv) student@controller1:~$ pip install -U pip
(kolla-venv) student@controller1:~$ pip install ansible==2.9.13
(kolla-venv) student@controller1:~$ pip install kolla-ansible==10.2.0
```

### Create kolla directory
```bash
(kolla-venv) student@controller1:~$ sudo mkdir -p /etc/kolla
(kolla-venv) student@controller1:~$ sudo chown $USER:$USER /etc/kolla
(kolla-venv) student@controller1:~$ cp -r kolla-venv/share/kolla-ansible/etc_examples/kolla/* /etc/kolla
(kolla-venv) student@controller1:~$ cp kolla-venv/share/kolla-ansible/ansible/inventory/* .
```

### Configure multinode
```bash
(kolla-venv) student@controller1:~$ nano multinode
...
#### 
[control] 
controller[1:3]
 
[network] 
controller[1:3]
 
[compute] 
compute[1:3] 
 
[monitoring] 
controller[1:3]
 
[storage] 
controller[1:3]
 
[deployment] 
localhost       ansible_connection=local
...
```

### Configure ansible
```bash
(kolla-venv) student@controller1:~$ sudo mkdir -p /etc/ansible
(kolla-venv) student@controller1:~$ sudo nano /etc/ansible/ansible.cfg
...
[defaults]
host_key_checking=False
pipelining=True
forks=100
interpreter_python=/usr/bin/python3
...
```
### Test Connection : 
```bash
(kolla-venv) student@controller1:~$ ansible -i multinode all -m ping
```

### Create password kolla : 
```bash
(kolla-venv) student@controller1:~$ kolla-genpwd
```

### Configure globals.yml
```bash
nano /etc/kolla/globals.yml
...

kolla_base_distro: "ubuntu"
kolla_install_type: "source"
openstack_release: "ussuri"
network_interface: "enp1s0"
neutron_external_interface: "enp2s0"
kolla_internal_vip_address: "10.10.10.100"
kolla_external_vip_address: "10.10.10.101"
enable_cinder: "yes"
nova_compute_virt_type: "kvm"
enable_openstack_core: "yes"
kolla_enable_tls_internal: "yes"
kolla_enable_tls_external: "yes"
kolla_copy_ca_into_containers: "yes"
kolla_enable_tls_backend: "yes"
openstack_cacert: "/etc/ssl/certs/ca-certificates.crt"
ceph_cinder_keyring: "ceph.client.cinder.keyring"
ceph_glance_keyring: "ceph.client.glance.keyring"
ceph_nova_keyring: "ceph.client.nova.keyring"
ceph_cinder_backup_keyring: "ceph.client.cinder-backup.keyring"
glance_backend_ceph: "yes"
cinder_backend_ceph: "yes"
nova_backend_ceph: "yes"
enable_docker_repo: false
docker_apt_package: docker.io
...

```
### Generate TLS Certificate
```bash
kolla-ansible -i multinode certificates
```

### Configuration directory Kolla Ansible
```
$ mkdir /etc/kolla/config  
$ mkdir /etc/kolla/config/nova  
$ mkdir /etc/kolla/config/glance  
$ mkdir -p /etc/kolla/config/cinder/cinder-volume  
$ mkdir /etc/kolla/config/cinder/cinder-backup
```

### Copy keyring
```bash
$ cp ceph_keyring/ceph.conf /etc/kolla/config/cinder/  
$ cp ceph_keyring/ceph.conf /etc/kolla/config/nova/  
$ cp ceph_keyring/ceph.conf /etc/kolla/config/glance/# cp /etc/ceph/ceph.client.glance.keyring /etc/kolla/config/glance/  
$ cp ceph_keyring/ceph.client.nova.keyring /etc/kolla/config/nova/  
$ cp ceph_keyring/ceph.client.cinder.keyring /etc/kolla/config/nova/  
$ cp ceph_keyring/ceph.client.cinder.keyring /etc/kolla/config/cinder/cinder-volume/  
$ cp ceph_keyring/ceph.client.cinder.keyring /etc/kolla/config/cinder/cinder-backup/  
$ cp ceph_keyring/ceph.client.cinder-backup.keyring /etc/kolla/config/cinder/cinder-backup
```

### Deploy OpenStack
```bash
(kolla-venv) student@controller1:~$ kolla-ansible -i ./multinode bootstrap-servers
(kolla-venv) student@controller1:~$ kolla-ansible -i ./multinode prechecks
(kolla-venv) student@controller1:~$ kolla-ansible -i ./multinode deploy
(kolla-venv) student@controller1:~$ kolla-ansible -i ./multinode post-deploy
```

### Install openstack client
```bash
(kolla-venv) student@controller1:~$ cd ~
(kolla-venv) student@controller1:~$ cd ~ apt install python3-venv
(kolla-venv) student@controller1:~$ python3 -m venv osclient
(kolla-venv) student@controller1:~$ python3 -m venv osclient
(kolla-venv) student@controller1:~$ pip3 install python-openstackclient
```

### Test Service OpenStack
```bash
(kolla-venv) student@controller1:~$ source /etc/kolla/admin-openrc.sh
(kolla-venv) student@controller1:~$ openstack service list
(kolla-venv) student@controller1:~$ openstack compute service list
(kolla-venv) student@controller1:~$ openstack volume service list
(kolla-venv) student@controller1:~$ openstack network agent list
```

## Deployment Apps (Webapps , Database , and Object store)

### Topology apps

![[Pasted image 20210819210945.png]]

### Requirement

![[Pasted image 20210819211337.png]]

## Deployment webapps
### List service
- Nginx 
- PHP-FPM
- MariaDB
- Lsyncd
- HAproxy
- S3FS
- KeepAlived

### Create instance web server
```bash
$ openstack server create --flavor midrange --image focal-server-cloudimg-amd64
--key-name controller-key-1 --security-group security-group-db --network internal-net
web-prod01
$ openstack server create --flavor midrange --image focal-server-cloudimg-amd64
--key-name controller-key-1 --security-group security-group-db --network internal-net
web-prod02
$ openstack server create --flavor midrange --image focal-server-cloudimg-amd64
--key-name controller-key-1 --security-group security-group-db --network internal-net
web-prod03
```

### Attach floating ip
```bash
$ openstack floating ip create --floating-ip-address 10.10.20.101 external-net
$ openstack server add floating ip web-prod01 10.10.20.101
$ openstack floating ip create --floating-ip-address 10.10.20.7 external-net
$ openstack server add floating ip web-prod02 10.10.20.7
$ openstack floating ip create --floating-ip-address 10.10.20.208 external-net
$ openstack server add floating ip web-prod03 10.10.20.208
```

### Test Login ssh web
```bash
student@controller1:~$ ssh ubuntu@10.10.20.101
student@controller1:~$ ssh ubuntu@10.10.20.7
student@controller1:~$ ssh ubuntu@10.10.20.208
```

### install depedency on all instance web
```bash
$ apt update
$ apt upgrade -y
$ apt install nginx -y
$ systemctl start nginx
$ systemctl status nginx
$ systemctl enable nginx

### Install php 7.4

$ apt install software-properties-common -y
$ add-apt-repository ppa:ondrej/php
$ apt install php7.4-fpm php7.4-common php7.4-mysql php7.4-xml php7.4-xmlrpc
php7.4-curl php7.4-gd php7.4-imagick php7.4-cli php7.4-dev php7.4-imap
php7.4-mbstring php7.4-soap php7.4-zip php7.4-bcmath -y
```

### Check and enable php-fpm
```bash
$ systemctl status php7.4-fpm
$ systemctl enable php7.4-fpm
```

### Download and extrack CMS Wordpress
```bash
$ wget -c https://wordpress.org/latest.tar.gz -O wordpress.tar.gz
$ tar xzvf wordpress.tar.gz
```

### setup directory for web
```bash
$ mv wordpress /var/www/wordpress
$ chmod -R 755 /var/www/wordpress
$ chown -R www-data:www-data /var/www/wordpress
```

### Configuration nginx for wordpress
```bash
$ vim /etc/nginx/sites-available/wordpress.conf
```

```bash
server {
            listen 80;
            root /var/www/html/wordpress/;
            index index.php index.html;
            server_name _;

            access_log /var/log/nginx/access.log;
            error_log /var/log/nginx/error.log;

            location / {
                         try_files $uri $uri/ /index.php?$query_string;
            }

            location ~ \.php$ {
                         try_files $fastcgi_script_name =404;
                         include fastcgi_params;
                         fastcgi_pass    unix:/run/php/php7.4-fpm.sock;
                         fastcgi_index   index.php;
                         fastcgi_param DOCUMENT_ROOT    $realpath_root;
                         fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
            }
            
            location ~ /\.ht {
                         deny all;
            }

            location = /favicon.ico {
                         log_not_found off;
                         access_log off;
            }

            location = /robots.txt {
                         allow all;
                         log_not_found off;
                         access_log off;
           }
       
            location ~* \.(js|css|png|jpg|jpeg|gif|ico)$ {
                         expires max;
                         log_not_found off;
           }
}
```

### Add nginx server block and verification 
```bash
$ cd /etc/nginx/sites-enabled
$ ln -s /etc/nginx/sites-available/wordpress.conf .
$ nginx -t
$ systemctl restart nginx
$ systemctl status nginx
```

### Configuration wordpress 
```bash
$ cd /var/www/html/wordpress
$ mv wp-config-sample.php wp-config.php
$ vim wp-config.php
...
define( 'DB_NAME', '<wp-name>' );
define( 'DB_USER', '<wp-user-db>' );
define( 'DB_PASSWORD', '<wp-password-db>' );
define( 'DB_HOST', '<wp-host-db>' );
define( 'DB_CHARSET', 'utf8' );
...

```

### access ip and install wordpress
- Choose language
- fill title information , username, password, email, and click install wordpress
- Login wordpress
- done

### Configuration bucket in ceph 
* Note : login on node ceph1
```bash
$ sudo radosgw-admin user create --uid=userprod --display-name=userprod

#### Remember your key
```

### install S3 client aws cli
```bash
$ sudo apt install -y awscli
$ sudo aws configure --profile=userprod

#### input your key 
```

### Create bucket
```bash
$ sudo aws --profile userprod --endpoint-url http://ceph1:8080/ s3api create-bucket --bucket wp-content
```

### Install and configuration s3fs 
* Note : login on instance web-prod01

```bash
$ apt update && apt install s3fs
$ echo "<accesskey>:<secretkey>" > /etc/.passwd-s3fs
$ chmod 600 /etc/.passwd-s3fs
$ mkdir -p /var/www/html/wordpress/wp-content-bak
$ cp -r /var/www/html/wordpress/wp-content/* /var/www/html/wordpress/wp-content-bak/
$ rm -rf /var/www/html/wordpress/wp-content/*

##### Makesure directory of wp-content has been moved to wp-content-bak

$ /usr/bin/s3fs wp-content /var/www/html/wordpress/wp-content -o
passwd_file=/etc/.passwd-s3fs -o url=http://10.10.10.31:8080 -o
use_path_request_style -o umask=0022,uid=33,gid=33 -o allow_other
$ df -hT

##### Makesure S3FS already mounted

$ umount /var/www/html/wordpress/wp-content

```

### Create service systemd (Autorun after reboot)
* Note : login on instance web-prod01

```bash
$ sudo vim /etc/systemd/system/s3fs-service.service
...
[Unit]
Description=Script untuk menjalankan otomatis s3fs ketika mesin restart

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/bin/s3fs wp-content /var/www/html/wordpress/wp-content -o passwd_file=/etc/.passwd-s3fs -o url=http://10.10.10.33:8080 -o use_path_request_style -o umask=0022,uid=33,gid=33 -o allow_other
ExecStop=/bin/fusermount -u /var/www/html/wordpress/wp-content

[Install]
WantedBy=default.target
...

```

### Start, Enable and makesure Status S3FS
```bash
$ systemctl start s3fs-service.service
$ systemctl enable s3fs-service.service
$ systemctl status s3fs-service.service
```

### Check status directory mount
```bash
$ df -hT
$ cp -r /var/www/html/wordpress/wp-content-bak/* /var/www/html/wordpress/wp-content/
```

### Install and configuration S3FS di Node web-prod02 dan web-prod03
```bash
$ apt update && apt install s3fs
$ echo "<accesskey>:<secretkey>" > /etc/.passwd-s3fs
$ chmod 600 /etc/.passwd-s3fs
$ mkdir -p /var/www/html/wordpress/wp-content-bak
$ mv /var/www/html/wordpress/wp-content/* /var/www/html/wordpress/wp-content-bak/
$ vim /etc/systemd/system/s3fs-service.service

...
[Unit]
Description=Script untuk menjalankan otomatis s3fs ketika mesin restart

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/bin/s3fs wp-content /var/www/html/wordpress/wp-content -o passwd_file=/etc/.passwd-s3fs -o url=http://10.10.10.33:8080 -o use_path_request_style -o umask=0022,uid=33,gid=33 -o allow_other
ExecStop=/bin/fusermount -u /var/www/html/wordpress/wp-content

[Install]
WantedBy=default.target
...

$ systemctl start s3fs-service.service
$ systemctl enable s3fs-service.service
$ systemctl status s3fs-service.service
$ df -hT
```

### Install and configuration Lsyncd
* Note : run on all web instance

```bash
$ apt update && apt install syncd
$ ssh-keygen -t rsa
$ ssh-copy-id root@192.168.10.21
$ ssh-copy-id root@192.168.10.22
$ ssh-copy-id root@192.168.10.23
$ mkdir /etc/lsyncd
$ systemctl enable lsyncd
$ vim /etc/lsyncd/lsyncd.conf.lua
...
settings {
        logfile         = "/var/log/lsyncd/lsyncd.log",
        statusFile      = "/tmp/lsyncd.stat",
        statusInterval  = 1,
}
sync {
        default.rsync,
        source  = "/var/www/html/wordpress",
        target  = "192.168.10.21:/var/www/html/wordpress",

}
sync {
        default.rsync,
        source  = "/var/www/html/wordpress",
        target  = "192.168.10.22:/var/www/html/wordpress",

}
sync {
        default.rsync,
        source  = "/var/www/html/wordpress",
        target  = "192.168.10.23:/var/www/html/wordpress",

}

rsync = {
        update  = true,
        perms   = true,
        owner   = true,
        group   = true,
        rsh     = "/usr/bin/ssh -l root -i /root/.ssh/id_rsa"
}
...

$ mkdir -p /var/log/lsyncd/
$ lsyncd /etc/lsyncd/lsyncd.conf.lua
$ systemctl restart lsyncd
$ systemctl status lsyncd
```

### Test Sync directory
```bash
$ touch /var/www/html/wordpress/test

##### Makesure data test was in all instance web

```

## Deployment Database

### create instance 3 instance db
```bash
$ openstack server create --flavor midrange --image bionic-server-cloudimg-amd64
--key-name controller-key-1 --security-group security-group-db --network internal-net db-prod01
$ openstack server create --flavor midrange --image bionic-server-cloudimg-amd64
--key-name controller-key-1 --security-group security-group-db --network internal-net db-prod02
$ openstack server create --flavor midrange --image bionic-server-cloudimg-amd64
--key-name controller-key-1 --security-group security-group-db --network internal-net db-prod03
$ openstack server create --flavor midrange --image bionic-server-cloudimg-amd64
--key-name controller-key-1 --security-group security-group-db --network internal-net db-prod04
```

### Attach floating ip to all db instance

```bash
$ openstack floating ip create --floating-ip-address 10.10.20.100 external-net
$ openstack server add floating ip db-prod01 10.10.20.100
$ openstack floating ip create --floating-ip-address 10.10.20.102 external-net
$ openstack server add floating ip db-prod02 10.10.20.102
$ openstack floating ip create --floating-ip-address 10.10.20.103 external-net
$ openstack server add floating ip db-prod03 10.10.20.103
$ openstack floating ip create --floating-ip-address 10.10.20.104 external-net
$ openstack server add floating ip db-prod04 10.10.20.104
```

### Login ssh db instance

```bash
student@controller1:~$ ssh ubuntu@10.10.20.100
student@controller1:~$ ssh ubuntu@10.10.20.102
student@controller1:~$ ssh ubuntu@10.10.20.103
student@controller1:~$ ssh ubuntu@10.10.20.104
```

### Update and Configuration mariadb to all db instance 
```bash
$ sudo apt update -y
$ sudo apt-get remove mariadb-server
$ sudo apt-get install software-properties-common
$ sudo apt-key adv --fetch-keys 'https://mariadb.org/mariadb_release_signing_key.asc'
$ sudo add-apt-repository 'deb [arch=amd64,arm64,ppc64el]
https://mariadb.mirror.liquidtelecom.com/repo/10.5/ubuntu bionic main'
$ sudo apt update
$ sudo apt install mariadb-server mariadb-client -y

##### Configuration password root mariadb

$ sudo mysql_secure_installation
Switch to unix_socket authentication [Y/n] Y
Change the root password? [Y/n] Y
New password: #Input Password MariaDB Root
Re-enter new password: #Konfirmasi Password MariaDB Root
Remove anonymous users? [Y/n]
Disallow root login remotely? [Y/n]
Remove test database and access to it? [Y/n]
Reload privilege tables now? [Y/n]

##### Test it

$ mysql -u root -p
```

### Configuraiton galera cluster  : 
* Instance db-prod01
```bash
$ sudo vim /etc/mysql/mariadb.conf.d/60-galera.cnf
...
[mysqld]
bind-address=0.0.0.0
default_storage_engine=InnoDB
binlog_format=row
innodb_autoinc_lock_mode=2
server-id = 1
report_host = master
log_bin = /var/lib/mysql/mariadb-bin
log_bin_index = /var/lib/mysql/mariadb-bin.index
relay_log = /var/lib/mysql/relay-bin
relay_log_index = /var/lib/mysql/relay-bin.index

[galera]

# Galera cluster configuration
wsrep_on=ON
wsrep_provider=/usr/lib/galera/libgalera_smm.so
wsrep_cluster_address="gcomm://192.168.10.15,192.168.10.16,192.168.10.17"
wsrep_cluster_name="mariadb-galera-cluster"
wsrep_sst_method=rsync

# Cluster node configuration
wsrep_node_address="192.168.10.15"
wsrep_node_name="galera-db-01"
...
```
* Instance db-prod02
```bash
$ sudo vim /etc/mysql/mariadb.conf.d/60-galera.cnf
...
[mysqld]                                                         
bind-address=0.0.0.0                                             
default_storage_engine=InnoDB                                    
binlog_format=row                                                
innodb_autoinc_lock_mode=2                                       
server-id = 1                                                    
report_host = master                                             
log_bin = /var/lib/mysql/mariadb-bin                             
log_bin_index = /var/lib/mysql/mariadb-bin.index                 
relay_log = /var/lib/mysql/relay-bin                             
relay_log_index = /var/lib/mysql/relay-bin.index                 
                                                                 

[galera]

# Galera cluster configuration                                                 
wsrep_on=ON                                                                    
wsrep_provider=/usr/lib/galera/libgalera_smm.so                                
wsrep_cluster_address="gcomm://192.168.10.15,192.168.10.16,192.168.10.17"      
wsrep_cluster_name="mariadb-galera-cluster"                                    
wsrep_sst_method=rsync                                                         
                                                                               
# Cluster node configuration                                                   
wsrep_node_address="192.168.10.16"                                             
wsrep_node_name="galera-db-02"
...
```
* Instance db-prod03
```bash
$ sudo vim /etc/mysql/mariadb.conf.d/60-galera.cnf
...
[mysqld]                                         
bind-address=0.0.0.0                             
default_storage_engine=InnoDB                    
binlog_format=row                                
innodb_autoinc_lock_mode=2                       
server-id = 1                                    
report_host = master                             
log_bin = /var/lib/mysql/mariadb-bin             
log_bin_index = /var/lib/mysql/mariadb-bin.index 
relay_log = /var/lib/mysql/relay-bin             
relay_log_index = /var/lib/mysql/relay-bin.index 

[galera]

# Galera cluster configuration                                                 
wsrep_on=ON                                                                    
wsrep_provider=/usr/lib/galera/libgalera_smm.so                                
wsrep_cluster_address="gcomm://192.168.10.15,192.168.10.16,192.168.10.17"      
wsrep_cluster_name="mariadb-galera-cluster"                                    
wsrep_sst_method=rsync                                                         
                                                                               
# Cluster node configuration                                                   
wsrep_node_address="192.168.10.17"                                             
wsrep_node_name="galera-db-03"
...
```

### Restart mariadb service on all db instance
```bash
$ sudo systemctl start mariadb
$ sudo systemctl stop mariadb
```

### Create galera cluster in node db-prod01
```bash
$ sudo galera_new_cluster

##### Check status of galera cluster in db-prod01

$ mysql -u root -p -e "show status like 'wsrep_%'"

##### Check amount of galera cluster

$ mysql -u root -p -e "show status like 'wsrep_cluster_size'"

##### Start MariaDB in db-prod02 and db-prod03

$ sudo systemctl start mariadb
$ sudo systemctl start mariadb

##### Check again amount of galera cluster. makesure the value is 3 which meant 3 node cluster

$ mysql -u root -p -e "show status like 'wsrep_cluster_size'"

##### Test galera cluster

$ mysql -u root -p
MariaDB [(none)]> create database galera_test;
```
![[Pasted image 20210819231112.png]]

### Configuration MariaDB Slave Replikasi
```bash
##### addition slave configuration  in db-prod04
$ sudo vim /etc/mysql/mariadb.conf.d/50-server.cnf
...
[mysqld]
server-id = 2
report_host = slave
log_bin = /var/lib/mysql/mariadb-bin
log_bin_index = /var/lib/mysql/mariadb-bin.index
relay_log = /var/lib/mysql/relay-bin
relay_log_index = /var/lib/mysql/relay-bin.index
...

##### restart database MariaDB

$ sudo systemctl restart mariadb  
$ sudo systemctl status  mariadb

##### Create user replication in db-prod01

$ sudo mysql -u root -p  
  
MariaDB [(none)]>  
MariaDB [(none)]> GRANT REPLICATION SLAVE ON *.* TO 'user'@'192.168.10.18' IDENTIFIED BY 'secret';  
Query OK, 0 rows affected (0.002 sec)  
  
MariaDB [(none)]> FLUSH PRIVILEGES;  
Query OK, 0 rows affected (0.001 sec)  
MariaDB [(none)]>

##### save your username and password then makesure you have correct input ip of slave instance

#### Lock lead opration read and write in db-prod01

MariaDB [(none)]> FLUSH TABLES WITH READ LOCK;
Query OK, 0 rows affected (0.001 sec)
MariaDB [(none)]>

##### Check master stats

MariaDB [(none)]> SHOW MASTER STATUS\G                                                              *************************** 1. row ***************************                                       	 
      	File: mariadb-bin.000004                                                                 	 
  	Position: 838                                                                                	 
Binlog_Do_DB:                                                                                    	 
Binlog_Ignore_DB:                                                                                    	 
1 row in set (0.000 sec)

MariaDB [(none)]>

##### note : save file and posisition later for connect slave to master

##### Dump database master (Galera cluster) in db-prod01
$ mysqldump --all-databases --user=root --password --master-data >> backupdbmaster.sql

##### Login and unluck database master (Galera Cluster) in db-prod01

$ sudo mysql -u root -p
MariaDB [(none)]> UNLOCK TABLES;
Query OK, 0 rows affected (0.000 sec)
MariaDB [(none)]> exit
Bye

##### Copy file dump database to slave

$ scp backupdbmaster.sql ubuntu@192.168.10.18:/home/ubuntu/

##### Restore file dump db master to slave

$ cd /home/ubuntu/
$ mysql -u root -p < backupdbmaster.sql

##### Restart Database slave

$ sudo systemctl restart mariadb

##### Connect database slave to master

$ sudo mysql -u root -p

MariaDB [(none)]> stop slave;
Query OK, 0 rows affected, 1 warning (0.000 sec)
MariaDB [(none)]>

MariaDB [(none)]> change master to master_host='192.168.10.15',master_user='user',master_password='secret',master_log_file='mariadb-bin.000004',master_log_pos=838;
Query OK, 0 rows affected (0.016 sec)
MariaDB [(none)]>

...
Captions : 
- master_host: Fill in IP Instance Master(db-prod01 node)
- master_user: Fill in the previously created user
- master_password: Fill in the password that was previously created in master
- master_log_file: Fill with the log file obtained in master status
- master_log_pos: Fill with the position number obtained in the master status
...

##### Start slave
MariaDB [(none)]> start slave;
Query OK, 0 rows affected (0.003 sec)
MariaDB [(none)]>

##### check Status Slave
MariaDB [(none)]> show slave status \G
```
### Test replication

![[Pasted image 20210819233646.png]]

## Install and configuration Keepalived HAproxy High Avaibility in Openstack

### Setup ip port in openstack 
```bash
##### Eksekusi di Node Controller

##### Membuat IP Port

$ neutron port-create --name vip-port internal-net
$ neutron port-create --name lb-master-port --allowed-address-pair ip_address=192.168.10.51 internal-net
$ neutron port-create --name lb-backup-port --allowed-address-pair ip_address=192.168.10.51 internal-net

# Create firewall rule

$ neutron security-group-rule-create --protocol 112 security-group-lb
$ neutron security-group-rule-create --protocol icmp security-group-lb
$ neutron security-group-rule-create --protocol tcp --port-range-min 22 --port-range-max 22 security-group-lb
$ neutron security-group-rule-create --protocol tcp --port-range-min 80 --port-range-max 80 security-group-lb
$ neutron security-group-rule-create --protocol tcp --port-range-min 443 --port-range-max 443 security-group-lb

##### Update port security group

$ neutron port-update --security-group security-group-lb lb-master-port
$ neutron port-update --security-group security-group-lb lb-backup-port

##### Create 3 floating IP (lb-master, lb-backup and vip-port)

$ neutron floatingip-create floating
$ neutron floatingip-create floating
$ neutron floatingip-create floating

##### Floating the IP

$ neutron floatingip-associate <ID first floating IP> <ID vip-port>
$ neutron floatingip-associate <ID second floating IP> <ID lb-master-port>
$ neutron floatingip-associate <ID third floating IP> <ID lb-backup-port>

##### create 2 instance (lb-master and lb-backup) dan  setup up dengan lb-master-port dan lb-backup-port


$ nova boot --flavor <your flavor> --image <image ID> --key-name <name of keypair> --nic port-id=<ID lb-master-port> lb-master
$ nova boot --flavor <your flavor> --image <image ID> --key-name <name of keypair> --nic port-id=<ID lb-backup-port> lb-backup

##### SSH to Instance for Configuration HAProxy and Keepalived

student@controller1:~$ ssh ubuntu@<ip lb-master>
student@controller1:~$ ssh ubuntu@<ip lb-backup>
```

### Install and Konfigurasi Haproxy (HTTP/HTTPS)
Note : run on all LB instance

```bash
##### Update package

$ sudo su
$ apt update
$ apt upgrade -y
$ apt install curl -y
$ curl https://haproxy.debian.net/bernat.debian.org.gpg | apt-key add -
$ apt install software-properties-common
$ add-apt-repository ppa:vbernat/haproxy-2.0
$ apt update
$ apt install haproxy=2.0.\* -y
$ haproxy -v

##### Generate SSL Certificate untuk HAProxy

$ openssl genrsa -out /etc/ssl/private/haproxy.key 2048
$ openssl req -new -key /etc/ssl/private/haproxy.key -out /etc/ssl/certs/haproxy.csr
$ openssl x509 -req -days 365 -in /etc/ssl/certs/haproxy.csr -signkey /etc/ssl/private/haproxy.key -out /etc/ssl/certs/haproxy.crt
$ cat /etc/ssl/private/haproxy.key /etc/ssl/certs/haproxy.crt >> /etc/ssl/certs/haproxy.pem

# Configuration HAProxy

$ vim /etc/haproxy/haproxy.cfg
...
global
        log /dev/log    local0
        log /dev/log    local1 notice
        chroot /var/lib/haproxy
        stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
        stats timeout 30s
        user haproxy
        group haproxy
        daemon

        # Default SSL material locations
        ca-base /etc/ssl/certs
        crt-base /etc/ssl/private

        # See: https://ssl-config.mozilla.org/#server=haproxy&server-version=2.0.3&config=intermediate
        ssl-default-bind-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384
        ssl-default-bind-ciphersuites TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256
        ssl-default-bind-options ssl-min-ver TLSv1.2 no-tls-tickets

        # add SSL
        tune.ssl.default-dh-param 2048

defaults
        log     global
        option  dontlognull
        timeout connect 5000
        timeout client  50000
        timeout server  50000
        errorfile 400 /etc/haproxy/errors/400.http
        errorfile 403 /etc/haproxy/errors/403.http
        errorfile 408 /etc/haproxy/errors/408.http
        errorfile 500 /etc/haproxy/errors/500.http
        errorfile 502 /etc/haproxy/errors/502.http
        errorfile 503 /etc/haproxy/errors/503.http
        errorfile 504 /etc/haproxy/errors/504.http

        #enable stats
        stats enable                    # enable statistics reports
        stats hide-version              # Hide the version of HAProxy
        stats uri /haproxy_stats        # Statistics URL
        stats refresh 30s               # HAProxy refresh time
        stats show-node                 # Shows the hostname of the node
        stats auth admin:P@ssword       # Authentication for Stats page

frontend http_frontend
        bind 192.168.10.51:80
        default_backend http_backend

backend http_backend
        mode http
        option httplog
        balance roundrobin
        server web-prod01 192.168.10.21:80 check
        server web-prod02 192.168.10.22:80 check
        server web-prod03 192.168.10.23:80 check

frontend https_frontend
        bind 192.168.10.51:443 ssl crt /etc/ssl/certs/haproxy.pem
        default_backend https_backend

backend https_backend
        mode tcp
        option tcplog
        balance roundrobin
        server web-prod01 192.168.10.21:80 check
        server web-prod02 192.168.10.22:80 check
        server web-prod03 192.168.10.23:80 check
...

##### Test configuration Haproxy

$ haproxy -c -f /etc/haproxy/haproxy.cfg

##### Allow port 443

$ ufw allow 80
$ ufw allow 443

##### Running HAProxy

$ systemctl restart haproxy
$ systemctl enable haproxy
```

### Install and configuration keepalived 
```bash
##### Install keepalived

$ apt install keepalived -y

##### Configuration IP Forwarding and non-local binding

$ sed -i 's/#net.ipv4.ip_forward=1/net.ipv4.ip_forward=1/' /etc/sysctl.conf
$ echo "net.ipv4.ip_nonlocal_bind = 1" >> /etc/sysctl.conf
$ sysctl -p

##### Configuration Keepalived

##### in Node LB-Master

$ vim /etc/keepalived/keepalived.conf
...
vrrp_script chk_haproxy {
    script "/usr/bin/killall -0 haproxy"
    interval 2
    weight 2
}

# Configuration for Virtual Interface
vrrp_instance LB_VIP {
    interface ens3
    state MASTER        # set to BACKUP on the peer machine
    priority 101        # higher value than backup set to  99 on the peer machine
    virtual_router_id 51

    smtp_alert          # Enable Notifications Via Email

    authentication {
        auth_type AH
        auth_pass myP@ssword    # Password for accessing vrrpd. Same on all devices
    }
    unicast_src_ip 192.168.10.218 # Private IP address of master
    unicast_peer {
        192.168.10.129          # Private IP address of the backup haproxy
   }

    # The virtual ip address shared between the two loadbalancers
    virtual_ipaddress {
        192.168.10.51
    }

    # Use the Defined Script to Check whether to initiate a fail over
    track_script {
        chk_haproxy
    }
}
...

##### in Node LB-Backup

$ vim /etc/keepalived/keepalived.conf
...
vrrp_script chk_haproxy {
    script "/usr/bin/killall -0 haproxy"
    interval 2
    weight 2
}

# Configuration for Virtual Interface
vrrp_instance LB_VIP {
    interface ens3
    state BACKUP        # set to BACKUP on the peer machine
    priority 100        # lower value than master,      set to  99 on the peer machine
    virtual_router_id 51

    smtp_alert          # Enable Notifications Via Email

    authentication {
        auth_type AH
        auth_pass myP@ssword    # Password for accessing vrrpd. Same on all devices
    }
    unicast_src_ip 192.168.10.129 # Private IP address of the backup haproxy
    unicast_peer {
        192.168.10.218          # Private IP address of master
   }

    # The virtual ip address shared between the two loadbalancers
    virtual_ipaddress {
        192.168.10.51
    }

    # Use the Defined Script to Check whether to initiate a fail over
    track_script {
        chk_haproxy
    }
}
...

##### Running Keepalived

$ systemctl enable --now keepalived
$ systemctl status keepalived

##### Testing VIP LB

##### Access floating ip from ip port for web apps in browser

URL HTTP: http://10.10.20.223
URL HTTPS: https://10.10.20.223

##### Testing VIP Keepalived
##### Check IP on LB-Master

$ ip a

##### Check IP on LB-Backup, nothing the VIP until master in DOWN

$ ip a

##### Test HA, execute on Master

$ systemctl stop keepalived
$ systemctl stop haproxy

##### Check on LB-Backup, there is any VIP


$ systemctl status keepalived
$ ip a

##### Access Floating IP from IP Port for Web Apps with Browser

URL HTTP: http://10.10.20.223
URL HTTPS: https://10.10.20.223

```

# DONE :D
