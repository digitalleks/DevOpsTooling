### DevOpsTooling
#DevOps Tooling Website Solution
###NFS Server Preparation
This projects starts with the preparation of the web server on AWS. Launch an EC2 instance that will serve as "NFS Server".\
<img width="706" alt="NFS Instance" src="https://user-images.githubusercontent.com/61512079/184509277-0fb52d8f-e18f-40d7-b084-b7d44b7fd1a7.PNG">

Next we create EBS volume and attach it to the NFS Server instance created:\
<img width="764" alt="EBS" src="https://user-images.githubusercontent.com/61512079/184509547-6ecc9057-3af3-4ae5-9d6f-2c4cbc013d2e.PNG">
Volume is attached to the Instance as follow:\
<img width="464" alt="EBS1" src="https://user-images.githubusercontent.com/61512079/184510036-29e367c8-f7a6-4d34-9210-f68f045ff228.PNG">\
<img width="438" alt="EBS2" src="https://user-images.githubusercontent.com/61512079/184510071-d6e27c35-c836-4dca-8cc7-fde0098c0b4d.PNG"> \
<img width="438" alt="EBS2" src="https://user-images.githubusercontent.com/61512079/184510082-621c9bfd-2321-4dea-9958-316dcdcc1047.PNG">

GPT partition is created with gdisk as follow:
```bash
sudo gdisk /dev/xvdf
sudo gdisk /dev/xvdg
sudo gdisk /dev/xvdh
```
For each of the gdisk command, follow the prompt by first Selecting 'n' for new partition and '8e00' as the GUID:\
<img width="485" alt="LVM partition1" src="https://user-images.githubusercontent.com/61512079/184512130-90ceedf0-c8c1-495b-ae21-e4bc9de6a8e0.PNG">\
<img width="290" alt="LVM partition2" src="https://user-images.githubusercontent.com/61512079/184512191-005aba59-2ddf-4a65-9a46-0ef598dc4c7d.PNG">

Next is the creation of the LVM components,  physical volume, volume group and logical volume as follow : \
```bash
sudo pvcreate /dev/xvdf1
sudo pvcreate /dev/xvdg1
sudo pvcreate /dev/xvdh1
```
<img width="354" alt="LVM PV" src="https://user-images.githubusercontent.com/61512079/184513148-519ca9b1-b08b-4887-bb1a-26fe0e95edfa.PNG">\

Create volume group for all the physical volume:
```bash
sudo vgcreate nfs-server-vg /dev/xvdh1 /dev/xvdg1 /dev/xvdf1
```
<img width="320" alt="VG" src="https://user-images.githubusercontent.com/61512079/184514900-f861653d-af3c-48b4-8950-709e881ebc96.PNG">

The following three logical volumes are created: lv-opt, lv-apps and lv-logs. 
```bash
sudo lvcreate -n lv-opt -L 9.5G nfs-server-vg
sudo lvcreate -n lv-apps -L 9.5G nfs-server-vg
sudo lvcreate -n lv-logs -L 9.5G nfs-server-vg
```
The logical volume created is confirmed as follow:\
<img width="392" alt="LVM LV" src="https://user-images.githubusercontent.com/61512079/184515367-72faacb4-056c-4d47-b08b-d0746c39c914.PNG">

Next we format the logical volumes with xfs file system:
```bash
sudo mkfs.xfs /dev/nfs-server-vg/lv-opt
sudo mkfs.xfs /dev/nfs-server-vg/lv-apps
sudo mkfs.xfs /dev/nfs-server-vg/lv-logs
```
Output of the xfs formating is shown below:\
<img width="494" alt="XFS FORMATING" src="https://user-images.githubusercontent.com/61512079/184515615-18c6cfbc-54bf-463d-ad47-a4990deccb60.PNG"> \

Next, we create the following  directories to mount the formated volume: /mnt/apps, /mnt/logs and /mnt/opt 
```bash
sudo mkdir /mnt/apps /mnt/logs /mnt/opt
```
Then we mount the volume to the created directories:
```bash
sudo mount /dev/nfs-server-vg/lv-opt /mnt/opt
sudo mount /dev/nfs-server-vg/lv-apps /mnt/apps
sudo mount /dev/nfs-server-vg/lv-logs /mnt/logs
```
To make the mounting persisitent, we edit the /etc/fstab and put in the respective details:
```bash
sudo vi /etc/fstab
```
fstab entries is modified as shown:\
<img width="624" alt="fstab-mount" src="https://user-images.githubusercontent.com/61512079/184516029-f4370774-0221-40df-b104-20fe966bb1a2.PNG">\
 Reload the system daemon and confirm the directories are mounted:
 ```bash
sudo mount -a
sudo systemctl daemon-reload
df -h
```
Output confirming the directories are mounted:\
<img width="459" alt="mounted" src="https://user-images.githubusercontent.com/61512079/184516075-9798be9c-2654-449d-95ec-653a9251d83b.PNG">

Next is the installation of NFS Server:
```bash
sudo yum -y update
sudo yum install nfs-utils -y
sudo systemctl start nfs-server.service
sudo systemctl enable nfs-server.service
sudo systemctl status nfs-server.service
```
After the installation and starting of service, NFS is confirmed running as shown below:\
<img width="754" alt="NFS SERVICE" src="https://user-images.githubusercontent.com/61512079/184516346-f8a7d8e9-57f2-44fd-b92f-99f03c180532.PNG">

Next permission is set for web servers within the same subnet to access and write to the NFS server:
```bash
sudo chown -R nobody: /mnt/apps
sudo chown -R nobody: /mnt/logs
sudo chown -R nobody: /mnt/opt

sudo chmod -R 777 /mnt/apps
sudo chmod -R 777 /mnt/logs
sudo chmod -R 777 /mnt/opt

sudo systemctl restart nfs-server.service
```

To make all the mount NFS directories accessible from within the network, we add the subnet CIDR in export directory as shown:
```bash
sudo vi /etc/exports
```
To verify the port used by NFS and open it on the EC2 instance, run the following:
```bash
rpcinfo -p | grep nfs
```
<img width="350" alt="NFS Port" src="https://user-images.githubusercontent.com/61512079/184516771-94c38848-8f8b-410e-a989-ef1e61192461.PNG">\
TCP port 111, UDP port 111 and TCP port 2049 are opened in the NFS Server security group:\
<img width="688" alt="NFS Inbound Rule" src="https://user-images.githubusercontent.com/61512079/184516799-28b727a4-81b5-40e0-9137-e43c5e46ce5e.PNG">

Next we set up the Webservers in same subnet as the NFS on AWS:\
<img width="529" alt="webserver1" src="https://user-images.githubusercontent.com/61512079/184556782-887d1ce5-62de-4930-b648-45195fe8f8b9.PNG">\
Installation of the NFS client:
```bash
sudo yum install nfs-utils nfs4-acl-tools -y
```
Next we created directory /var/www and mount to the NFS Server apps directory:
```bash
sudo mkdir /var/www
sudo mount -t nfs -o rw,nosuid 172.31.43.227:/mnt/apps /var/www
```
The mount was verified successfuly as shown below:\
<img width="405" alt="Web-mount-nfs" src="https://user-images.githubusercontent.com/61512079/184556893-493536e7-b2fe-4236-982b-eea428980513.PNG">

Next we installed Remi’s repository, Apache and PHP:
```bash
sudo yum install httpd -y
sudo dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
sudo dnf install dnf-utils http://rpms.remirepo.net/enterprise/remi-release-8.rpm
sudo dnf module reset php
sudo dnf module enable php:remi-7.4
sudo dnf install php php-opcache php-gd php-curl php-mysqlnd
sudo systemctl start php-fpm
sudo systemctl enable php-fpm
setsebool -P httpd_execmem 1
```
Output of the installation is shown below:\
<img width="506" alt="EPEL1" src="https://user-images.githubusercontent.com/61512079/184557108-4cae788d-6c46-4cb2-881c-8add0254b2aa.PNG">\
<img width="355" alt="Remi-Release" src="https://user-images.githubusercontent.com/61512079/184557116-1b2a1c7e-f17f-48f7-9b08-2eee06e02701.PNG">\
<img width="939" alt="Remi-Release1" src="https://user-images.githubusercontent.com/61512079/184557123-83a6a5b6-ac41-4dda-9b52-d85883df435f.PNG">\
<img width="922" alt="PHP-Install" src="https://user-images.githubusercontent.com/61512079/184557130-95d42b09-250f-43c0-a138-ffa51c60cc8b.PNG">\
<img width="719" alt="PHP-enable" src="https://user-images.githubusercontent.com/61512079/184557135-271cbd80-22c5-4906-9001-16e250ea8276.PNG">\

The apache files created on the webserver was confirmed to be on the mounted directory by creating a test file and seeing the same on the NFS server mount directory:\
```bash
cd /var/www
sudo touch test.txt
```
<img width="288" alt="NFS Test" src="https://user-images.githubusercontent.com/61512079/184557226-503570a7-cd17-4a85-963b-7f814981a504.PNG">\

The same file is accessible on webserver2 as shown:\
<img width="232" alt="Access-test-file" src="https://user-images.githubusercontent.com/61512079/184557257-a83d4b2b-679e-44dd-8380-90124afc8bf6.PNG">\

Next we mount the webserver log directory to the NFS log directory:
```bash
sudo mount -t nfs -o rw,nosuid 172.31.43.227:/mnt/logs /var/log
```
<img width="378" alt="Log-mount" src="https://user-images.githubusercontent.com/61512079/184557312-dba33b89-e69c-4380-8338-731bd8513677.PNG">\
Make the mount persisitent in the /etc/fstab file.

The next step is the creation of Database server to be used by the webserver.  An EC2 instance with Ubuntu  is created.  
The we update and install mysql server as follow:
```bash
sudo apt-get update
sudo apt install mysql-server
```
After the installation,  mysql service is started and enabled:
```bash
sudo systemctl start mysql
sudo systemctl enable mysql
```
The MYSQL Service is confirmed running as follow:\
<img width="543" alt="MYSQL SERVER STATUS" src="https://user-images.githubusercontent.com/61512079/184669174-29986a74-ab55-4f43-a20f-5375071ec37f.PNG">\

Then we log on to the mysql database and create a user as follow:
```bash
sudo mysql
```

```mysql
CREATE DATABASE tooling;
CREATE USER 'webaccess'@'%' IDENTIFIED BY 'P@ssw0rd';
GRANT ALL PRIVILEGES ON tooling.* TO 'webaccess'@'%' WITH GRANT OPTION;
```
The created user is confirmed as follow:\
<img width="263" alt="Webaccess user" src="https://user-images.githubusercontent.com/61512079/184669941-0edd941f-d853-44f6-b503-cbf22428d63a.PNG">\

The bind address for the database is modified to 0.0.0.0:
```bash
sudo vi /etc/mysql/mysql.conf.d/mysqld.cnf
```

Next we restart the mysql service:
```bash
  sudo systemctl restart mysql
 ```
 
A website tooling code is fork from DAREY.IO GIT account to my git account.  Next I installed git on my Webserver instance:
```bash
sudo yum install git -y
```
After the git installed, the content of the forked directory is cloned using the git command below:
```bash
sudo git clone https://github.com/digitalleks/tooling
```
This confirmed as follow:\
<img width="421" alt="git clone" src="https://user-images.githubusercontent.com/61512079/184662657-581d8509-0e7a-46f3-8843-bb07b436b88b.PNG">\

Then the content of the clone directory is copied into our /var/www/html directory using the command below:
```bash
sudo cp -R ~/tooling/html/. /var/www/html
```
<img width="439" alt="html-confirmed" src="https://user-images.githubusercontent.com/61512079/184663104-739cbcb1-cd89-4b9f-af9b-f87dfebf9cd8.PNG">\


Necessary permissions are granted as follow:
```bash
sudo setsebool -P httpd_can_network_connect=1
sudo setsebool -P httpd_can_network_connect_db=1
sudo setsebool -P httpd_execmem=1
sudo setsebool -P httpd_use_nfs 1
```
Also, SELLinux is disabled using the command below:
```bash
sudo setenforce 0
```
This is made permanent by editing "SELINUX" entry to "disabled" in /etc/sysconfig/selinux :
```bash
sudo vi /etc/sysconfig/selinux
```
After this, the apache is restarted.
```bash
sudo systemctl restart httpd
```

For access to the database from the webserver, we installed mysql client on the websierver as follow:
```bash
sudo yum install mysql
```
After that, we apply the tooling script to the database using this command(where 172.31.41.49 is the IP address of the Database server):
```bash
sudo mysql -h 172.31.41.49 -u webaccess -p tooling < tooling-db.sql
```
This is confirmed successful with no error as shown below:\
<img width="632" alt="script-apply" src="https://user-images.githubusercontent.com/61512079/184663968-be913121-fa12-403f-b6bb-dc02d0eaffd6.PNG">\

Finally, we modify the Database IP in the function.php file in the html directory:
```bash
sudo vi /var/www/html/functions.php
```
The entries are edited as shown below:\
<img width="610" alt="function-php-edit" src="https://user-images.githubusercontent.com/61512079/184665069-3f3d3eae-636b-4538-8b48-d8ed87b70576.PNG">\

Finally,  we test the access by using the public IP of the Webserver to access the index.php page on the browser as confirmed below:\
<img width="506" alt="Login_Page" src="https://user-images.githubusercontent.com/61512079/184665434-58da7f7e-8102-4d0b-8794-c4541e40d289.PNG">\
The username and password is supllied as "admin/admin' and access is confirmed as shown below:\
<img width="762" alt="propitix Tool test" src="https://user-images.githubusercontent.com/61512079/184665666-db2a7fc1-c95f-45cb-933f-d049d80b4d55.PNG">






























