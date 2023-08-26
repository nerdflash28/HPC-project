
# Warewulf:
## step 1 : setting up the OS
```bash
setenforce 0
sed -i 's/^SELINUX=.*$/SELINUX=disabled/g' /etc/selinux/config
systemctl stop firewalld
systemctl disable firewalld
nmcli con up ens33
nmcli con up ens34
hostnamectl set-hostname master
echo "`ip a | grep ens34 | tail -1 | cut -d' ' -f6|cut -c 1-12` master" >> /etc/hosts
su
```

## step 2 : installing warewulf
```bash
# adding the repository
yum install -y https://repo.ctrliq.com/rhel/8/ciq-release.rpm

# installing the warewulf
yum install -y warewulf
```

## step 3 : making changes to warwulf.conf
```bash
vim /etc/warewulf/warewulf.conf
    Line no 2: change ip address
    Line no 4: change network
    Line no 14: change DHCP starting ip address
    Line no 15: change DHCP ending ip address

# configure the warewulf
wwctl configure --all

# check if all the services are running or not
systemctl status dhcpd tftp nfs-server warewulfd

# if services are not running then restart the services
systemctl start dhcpd tftp nfs-server warewulfd

# enable the services
systemctl enable dhcpd tftp nfs-server warewulfd

# check logs if necessary
cat /var/log/warewulfd.log
```
![](./images/warewulf_conf.jpg)
![](./images/wwctl-configure-all.jpg)
![](./images/dhcp-service.jpg)
![](./images/nfs-server-service.jpg)

## step 3 : booting the node using container
```bash
# import container from docker hub
wwctl container import docker://warewulf/rocky rocky-8

# sync host uid & gid with the container
wwctl overlay build
wwctl container syncuser --write rocky-8

# to check container list
wwctl container list

# to modify the container
wwctl container exec rocky-8 /bin/bash
    dnf install -y passwd
    passwd root
        new password : root
        re enter password : root
    exit
```

## Step 4 : Add a node 
```bash
# to add a node without knowing MAC address
wwctl node add node1 --ipaddr 10.10.10.232 --discoverable ens34
wwctl node add node1 --ipaddr 10.10.10.245 --discoverable ens34
wwctl node set --container rocky-8 node1
```
\# Note : `if this error shows: hub_port_status failed (err = 110)`  
\# Solution : disable the USB device or remove  
![](./images/solution-for-error.jpg)

# Installing Slurm 

## Step 1: Install munge service on both master & container
```bash
dnf install epel-release -y
dnf config-manager --set-enabled powertools
dnf install munge* -y
dnf install rng-tools.x86_64 -y
dnf repolist
systemctl start rngd
systemctl enable rngd
systemctl status rngd
```
![](./images/epel-release.jpg)
![](./images/yum-repolist.jpg)
![](./images/dnf-install-munge.jpg)
![](./images/dnf-install-rng-tools.jpg)
![](./images/rngd-service.jpg)

## Step 2: generate munge key
```bash
/usr/sbin/create-munge-key -r

# copy the monge key to container
wwctl container exec --bind /root/Documents/temp:/root/temp rocky-8 /bin/bash

# copy the munge file 
cp /etc/munge/munge.key root/Documents/temp
```
![](./images/copy-munge%20key.jpg)

## resolving Error ( glibc )
```bash
# install language to resolve this error
dnf install langpacks-en glibc-all-langpacks -y
```
![](./images/error-glibc.jpg)


## Step 3 : setting up the munge key in container
```bash
# copy the munge key in the default location
cp 
chown munge:munge /etc/munge/munge.key
```

## Step 4: Installing Slurmd 
```bash
# first download the tar file 
wget https://download.schedmd.com/slurm/slurm-20.11.9.tar.bz2

# download the rpm-build package
dnf install rpm-build make -y
# build the packages from tar file
rpmbuild -ta slurm-20.11.9.tar.bz2
# to install the dependencies
dnf install pam-devel python3 readline-devel perl-ExtUtils-MakeMaker gcc mysql-devel -y
# now try to build the package again
rpmbuild -ta slurm-20.11.9.tar.bz2
``` 
![](./images/slurm-package-get.jpg)
![](./images/slurm-dependencies.jpg)
![](./images/dependencies-install.jpg)

## Step 5: Create Slurm User for SLURM Service
```bash
export SLURMUSER=900;
# add group
groupadd -g $SLURMUSER slurm;
# create user
useradd -m -c "SLURM workload manager" -d /var/lib/slurm -u $SLURMUSER -g slurm -s /bin/bash slurm;

export MUNGEUSER=973;
# add group
groupadd -g $MUNGEUSER munge;
# create user
useradd -m -c "Runs Uid 'N' Gid Emporium" -d /var/lib/munge -u $MUNGEUSER -g munge -s /sbin/nologin munge;


munge:x:975:973:Runs Uid 'N' Gid Emporium:/var/run/munge:/sbin/nologin
useradd -m -c "Runs Uid 'N' Gid Emporium" -d /var/lib/munge -u 97 -g munge -s /sbin/nologin 

cat /etc/passwd | grep slurm;
```
![](./images/create-slurm-user.jpg)
![](./images/slurm-user-container.jpg)

## Step 6: installing packages from rpm builds
```bash
# navigate to the directory
cd /root/rpmbuild/RPMS/x86_64/

# installing local packages
dnf --nogpgcheck localinstall * -y

# to confirm if packages has been installed or not
rpm -qa | grep slurm | wc -l

# copy packages to shared folder with container
cp * /root/test/;
```
![](./images/rpm-packages.jpg)
![](./images/confirm-rpm-packages.jpg)
![](./images/installing-slurm-packages-on-container.jpg)

## Step 7: creating repositories for slurm configuration
```bash
# create folder 
mkdir /var/spool/slurm

# change ownership of the slurm service on both client and master
chown slurm:slurm /var/spool/slurm

# change mod
chmod 755 /var/spool/slurm

# create folder 
mkdir /var/log/slurm

# change ownership of the slurm service on both client and master
chown slurm:slurm /var/log/slurm

# create log files
touch /var/log/slurm_jobacct.log /var/log/slurm_jobcomp.log

# change ownership
chown slurm:slurm /var/log/slurm_jobacct.log /var/log/slurm_jobcomp.log
```
![](./images/changing-permissions-slurms.jpg)

## Step 8: Edit the configuration file 

```bash
cp /etc/slurm/slurm.conf.example /etc/slurm/slurm.conf
vi /etc/slurm/slurm.conf
    edit : clusterName=warewulf-cluster
    edit : clusterMachine=master
    edit : slurmUser=slurm
    edit : #COMPUTE NODES
            #NodeName=node[1-2] Procs=1 State=UNKOWN
            NodeName=node1 CPUs=2 Boards=1 SocketsPerBoard=2 CoresPerSocket=1 ThreadsPerCore=1 RealMemory=7928
            ParitionName=standard NODES=ALL Default=YES MaxTime=INFINITE State=UP
            :wq # save and exit

# run on client and get the satus of the node and paste this information in the configuration file
slurmd -C

chown slurm:slurm /etc/slurm/slurm.conf
# copy this config file to all the nodes
cp /etc/slurm/slurm.conf /root/Documents/slurm/

# start slurmctld service on master
systemctl start slurmctld;systemctl enable slurmctld;
# start slurmd service on node1
systemctl start slurmd;systemctl enable slurmd;
```
![](./images/slurm_conf.jpg)

## Step 9: to sync the the user with containers we use the command
```bash
# to rebuild the overlays 
wwctl overlay build
wwctl container syncuser --write rocky-8

# to debug error in slurmd 
slurmd -Dvv
``` 
---
# Installing Prometheus

## Step 1: downloading the zip file
```bash
# download the tar file 
wget https://github.com/prometheus/prometheus/releases/download/v2.45.0/prometheus-2.45.0.linux-amd64.tar.gz
```
![](./images/prometheus/1.jpg)

## Step 2: Unzip the package
```bash
# unziping the tar file
tar -xvf prometheus-*.tar.gz
```
![](./images/prometheus/2.jpg)

## Step 3: prepare the prometheus.yml file
```bash
cd prometheus-*
vim prometheus.yml 
    Line no 3: change time from 15s to 10s
    Line no 4: change time from 15s to 10s
    Line no 31: create new job name : node
    Line no 36-37: create targets entries 
```
![](./images/prometheus/2.jpg)
![](./images/prometheus/3.jpg)
![](./images/prometheus/4.jpg)

## Step 4: setup the warewulf container
```bash


```
![](./images/prometheus/5.jpg)















































































