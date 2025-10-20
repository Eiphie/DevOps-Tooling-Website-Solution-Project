# DevOps Tooling Website Solution
A DevOps Tooling Website Solution is a web-based platform or portal that uses integrated tools, dashboards, and workflows to assist teams in implementing, managing, and automating DevOps practices.
## Project SetUp
### NFS Server Setup
- Launch EC2 on Redhat
```
sudo dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-10.noarch.rpm
```
```
sudo dnf install gdisk
```
- Using gdisk, On each of the 3 disk create a seperate single partition
```
lsblk
```
```
sudo gdisk /dev/xvdf
```
Command (? for help): n
Command (? for help): w

- Install lvm2 package sudo yum install lvm2, then check available partion lvmdiskscan and mark all 3 disk as PVs to be used by LVM
```
sudo yum install lvm2
```
```
lvmdiskscan
```
```
sudo pvcreate /dev/xvdf1
```

- Confirm each PVs was created successfully with sudo pvs and add each PVs to VG, Name the VG webdata-vg
```
sudo pvs
```
```
sudo vgcreate nfs-vg /dev/xvdf1 /dev/xvdg1 /dev/xvdh1
```

Create 3 LVs namely; lv-apps,lv-logs and lv-opt, and split the available PV size for all then confirm LVs was created successfully
```
sudo lvcreate -n lv-apps -L 18G nfs-vg
sudo lvcreate -n lv-logs -L 18G nfs-vg
sudo lvcreate -n lv-opt -L 18G nfs-vg
```
```
sudo lvs
```
```
sudo vgdisplay
```
```
sudo lsblk
```

Format all 3 LVs with xfs filesystem
```
sudo mkfs -t xfs /dev/nfs-vg/lv-apps
sudo mkfs -t xfs /dev/nfs-vg/lv-logs
sudo mkfs -t xfs /dev/nfs-vg/lv-opt
```
```
ls /mnt
find /mnt
```

- Create /mnt/app, /mnt/logs, /mnt/opt directory to store website files also mount all 3 directories on 3 matching LVs
```
sudo mkdir -p /mnt/apps
sudo mkdir -p /mnt/logs
sudo mkdir -p /mnt/opt
```
```
sudo mount /dev/nfs-vg/lv-apps /mnt/apps
sudo mount /dev/nfs-vg/lv-logs /mnt/logs
sudo mount /dev/nfs-vg/lv-opt /mnt/opt
```

- Install NFS Server, set it up to launch upon reboot, and confirm that it is running.
```
sudo yum -y update
```
```
sudo yum install nfs-utils -y
```
```
sudo systemctl start nfs-server.service
sudo systemctl enable nfs-server.service
sudo systemctl status nfs-server.service
```

- Update etc/fstab file with UUID of device, then test configuration and reload daemon
```
sudo blkid
sudo vi /etc/fsta
sudo mount -a
sudo systemctl daemon-reload
```

- enable the web server to read, write, and run files on NFS by setting up permissions.  Configure NFS access for clients in the same subnet, then use a security group to open the port that the NFS is using.
```
sudo chown -R nobody: /mnt/apps
sudo chown -R nobody: /mnt/logs
sudo chown -R nobody: /mnt/opt
```
```
sudo chmod -R 777 /mnt/apps
sudo chmod -R 777 /mnt/logs
sudo chmod -R 777 /mnt/opt
```
```
sudo vi /etc/exports
sudo exportfs -arv
```
```
rpcinfo -p | grep nfs
```

### Database Server Setup
- Launch EC2 on Ubuntu
```
sudo apt update
sudo apt install mysql-server -y
sudo systemctl enable mysql
sudo systemctl start mysql
```

Set `bind-address = 0.0.0.0` to allow remote connection
```
sudo vim /etc/mysql/mysql.conf.d/mysqld.cnf
```
```
sudo systemctl restart mysql
sudo mysql
```

### Web Servers Setup
- Launch 3 EC2 on Redhat and set up the NFS client, then mount /var/www and aim for the apps' export from the NFS server.
```
sudo yum install nfs-utils nfs4-acl-tools -y
sudo mkdir /var/www
sudo mount -t nfs -o rw,nosuid <NFS-Server-Private-IP-Address>:/mnt/apps /var/www
```

- Make sure the NFS was successfully mounted and is running, and that the modifications will remain on the web server following a reboot.
```
df -h
```
Edit /etc/fstab and add the volume mount `<NFS-Server-Private-IP-Address>:/mnt/apps /var/www nfs defaults 0 0`
```
sudo vi /etc/fstab
```

Enter Apache, PHP, and Remi's repository for each of the three web servers that were created.
```
sudo yum -y update
sudo yum -y install httpd php php-mysqlnd php-fpm
sudo systemctl enable httpd
sudo systemctl start httpd
sudo dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-10.noarch.rpm
sudo dnf install https://rpms.remirepo.net/enterprise/remi-release-10.rpm
sudo yum module list php
sudo yum module reset php
sudo yum module enable php:remi-8.5
sudo yum install php php-opcache php-gd php-curl php-mysqlnd
sudo systemctl start php-fpm
sudo systemctl enable php-fpm
sudo setsebool -P httpd_execmem 1
sudo systemctl restart httpd
```

- Check that the Web Server's /var/www directory contains Apache files and directories and in /mnt/apps on the NFS server.  It indicates that NFS is mounted correctly if you see the same files, Try creating a new file called "touch test.txt" from one server and seeing if the same file is reachable from different web servers.


