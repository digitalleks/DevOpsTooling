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

Next we set up the Webservers in same subnet as the NFS on AWS:














