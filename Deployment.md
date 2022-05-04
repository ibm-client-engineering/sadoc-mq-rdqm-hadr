# Installing and Configuring IBM MQ and RDQM

## Purpose of this doc

This document serves as a documented process for installing IBM MQ and RDMQ. It assumes some basic familiarlity with Linux command line.

## Installation of MQ

In order to build out our setup, we are assuming three hosts that live in three different pods in a Dallas datacenter and three hosts in three pods living in a Washington DC datacenter. Our bastion host will also live in WDC and that's where we'll base our primary HA stack. We also assume root access via ssh to all hosts here. The OS on the mq hosts for our purposes will be **Red Hat 8.4**. We will also assume they are properly subscribed.

```
10.241.1.4      wdc-bastion      

# WDC mq hosts
10.241.0.4	    wdc1-mq1
10.241.64.4     wdc2-mq1
10.241.128.4	wdc3-mq1

# DAL mq hosts
10.240.0.4	    dal1-mq1
10.240.64.4	    dal2-mq1
10.240.128.4	dal3-mq1
```

Above is what the hosts file on our bastion host should look like. We would connect to the bastion host via a public ip. We won't go into configuring ssh proxyjump in this document.

## Host layout

Each of the above hosts barring the bastion have two extra disks attached:

```
Disk /dev/vda: 100 GiB, 107374182400 bytes, 209715200 sectors
Disk /dev/vdb: 100 GiB, 107374182400 bytes, 209715200 sectors
Disk /dev/vde: 25 GiB, 26843545600 bytes, 52428800 sectors
```

- `/dev/vda` is our boot volume
- `/dev/vdb` will become our logical volume for `/var/mqm`
- `/dev/vde` will be our RDQM managed volume

IBM MQ installation recommends multiple separate disks for various aspects of MQ to increase performance and minimize overall I/O during heavy operations. For our purposes, we will only be going with one volume for MQ.

### System preparation

1. Create the mqm userid and group. This step occurs on all host minus the bastion host.

```
groupadd -g 1001 mqm
useradd -g mqm -u 1001 -m -c "MQM User" mqm 
```
2. Update mqm user's bashrc and bash profile. These paths don't exist as of yet until the actual installation of MQ.
```
echo "export MQ_INSTALLATION_PATH=/opt/mqm" >> ~mqm/.bashrc
echo '. $MQ_INSTALLATION_PATH/bin/setmqenv -s' >> ~mqm/.bashrc
echo 'export PATH=$PATH:/opt/mqm/bin:/opt/mqm/samp/bin:' >> ~mqm/.bash_profile
```

3. Update `/etc/sudoers` with the correct permissions for mqm user
```
echo "%mqm ALL=(ALL) NOPASSWD: /opt/mqm/bin/crtmqm,/opt/mqm/bin/dltmqm,/opt/mqm/bin/rdqmadm,/opt/mqm/bin/rdqmstatus" >> /etc/sudoers
``` 
4. Install lvm2 if it's not already there
```
dnf -y install lvm2
```
5. Setup the volume group and logical volume for `/var/mqm`
```
pvcreate /dev/vdb
vgcreate MQStorageVG /dev/vdb
lvcreate -n MQStorageLV -l100%VG MQStorageVG
```
6. Create the `/var/mqm` directory and format the storage volume
```
mkdir /var/mqm
mkfs.xfs /dev/MQStorageVG/MQStorageLV
mount /dev/MQStorageVG/MQStorageLV /var/mqm
```
7. Make sure `/var/mqm` is fully owned by the mqm user and that everything is setup in `/etc/fstab`
```
chown -R mqm:mqm /var/mqm
chmod 755 /var/mqm
echo "/dev/MQStorageVG/MQStorageLV      /var/mqm        xfs     defaults        1 2" >> /etc/fstab
```
8. Configure the storage volume for RDQM. **It's critical that the volume group is named `drbdpool`.**
```
parted -s -a optimal /dev/vde mklabel gpt 'mkpart primary ext4 1 -1'
pvcreate /dev/vde1
vgcreate drbdpool /dev/vde1
```
9. Add the following settings to `/etc/sysctl.conf`
```
kernel.shmmni = 4096
kernel.shmall = 2097152
kernel.shmmax = 268435456
kernel.sem = 32 4096 32 128
fs.file-max = 524288

sysctl -p
```
10. Set the ulimit for the mqm user by adding the following to `/etc/security/limits.conf`
```
# For MQM User
mqm       hard  nofile     10240
mqm       soft  nofile     10240
```

## Installing MQ

This requires you to go to the following link and retrieving IBM MQ version 9.2.5:

https://w3-03.ibm.com/software/xl/download/ticket.wss

Once you have the package, **IBM_MQ_9.2.5_LINUX_X86-64.tar.gz** you will need to upload it to all six hosts. This document will assume you have done this.







