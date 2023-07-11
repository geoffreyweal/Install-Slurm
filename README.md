### Introduction

Install Slurm on CentOS-7 Virtual Cluster.

### Preparation

1. [Connect Virtual Machines](https://github.com/Artlands/Install-Slurm/tree/master/Connect_VM)

2. [Setup NFS Server](https://github.com/Artlands/Install-Slurm/tree/master/Setup_NFS)

### Cluster Server and Computing Nodes
List of master node and computing nodes within the cluster.

|Hostname|IP Addr |
|--------|--------|
|master  |10.0.1.5|
|node1   |10.0.1.6|
|node2   |10.0.1.7|

### (Optional) Delete failed installation of Slurm

Remove database:

```
yum remove mariadb-server mariadb-devel -y
rm -rfv /etc/my.cnf.d
rm -rfv /var/lib/mysql
rm -rfv /var/log/mariadb
```



Remove Slurm and Munge:

```
yum remove slurm munge munge-libs munge-devel -y
rm -rvf /etc/munge
rm -rvf /var/log/munge
rm -rvf /nfsshare/slurm-rpms
rm -rvf /var/log/tallylog
```

Delete the users and corresponding folders:

```
userdel -r slurm
userdel -r munge
```

### Update your system

To update your system:

```
sudo yum update
```

### Create the global users

Slurm and Munge require consistent UID and GID across every node in the cluster. For all the nodes, before you install Slurm or Munge:

```
export MUNGEUSER=1991
groupadd -g $MUNGEUSER munge
useradd  -m -c "MUNGE Uid 'N' Gid Emporium" -d /var/lib/munge -u $MUNGEUSER -g munge  -s /sbin/nologin munge
export SLURMUSER=1992
groupadd -g $SLURMUSER slurm
useradd  -m -c "SLURM workload manager" -d /var/lib/slurm -u $SLURMUSER -g slurm  -s /bin/bash slurm
```

### Setting up MariaDB database: master

Install MariaDB:

```
yum install mariadb-server mariadb-devel -y
```

Start the MariaDB service:

```
systemctl enable mariadb
systemctl start mariadb
systemctl status mariadb
```

Create the Slurm database user:

```
mysql
```

In mariaDB:

```mysql
GRANT ALL ON slurm_acct_db.* TO 'slurm'@'localhost' IDENTIFIED BY '1234' with grant option;
SHOW VARIABLES LIKE 'have_innodb';
FLUSH PRIVILEGES;
CREATE DATABASE slurm_acct_db;
quit;
```

Verify the databases grants for the _slurm_ user:

```
mysql -p -u slurm
```

Type password for slurm: `1234`. In mariaDB:

```mysql
show grants;
quit;
```

Create a new file /etc/my.cnf.d/innodb.cnf containing:
```
[mysqld]
innodb_buffer_pool_size=1024M
innodb_log_file_size=64M
innodb_lock_wait_timeout=900
```

To implement this change you have to shut down the database and move/remove logfiles:
```
systemctl stop mariadb
mv -fv /var/lib/mysql/ib_logfile? /tmp/
systemctl start mariadb
systemctl status mariadb
```

You can check the current setting in ``mysql`` like so:
```sql
mysql
SHOW VARIABLES LIKE 'innodb_buffer_pool_size';
quit;
```

### Install Munge

Get the latest REPL repository:

```
yum install epel-release -y
```

Install Munge:

```
yum install munge munge-libs munge-devel -y
```

Create a secret key on __master__ node. First install rig-tools to properly create the key:

```
yum install rng-tools -y
rngd -r /dev/urandom
/usr/sbin/create-munge-key -r
```

```
dd if=/dev/urandom bs=1 count=1024 > /etc/munge/munge.key
```

```
chown munge: /etc/munge/munge.key
chmod 400 /etc/munge/munge.key
```

Send this key to all of the compute nodes:

```
scp /etc/munge/munge.key root@10.0.1.6:/etc/munge
scp /etc/munge/munge.key root@10.0.1.7:/etc/munge
```

SSH into every node and correct the permissions as well as start the Munge service:

```
chown -R munge: /etc/munge/ /var/log/munge/
chmod 0700 /etc/munge/ /var/log/munge/
```

```
systemctl enable munge
systemctl start munge
systemctl status munge
```

To test Munge, try to access another node with Munge from __master__ node:

```
munge -n
munge -n | munge
munge -n | ssh 10.0.1.6 unmunge
remunge
```

If you encounter no errors, then Munge is working as expected.

### Install Slurm

Install a few dependencies:

```
yum install openssl openssl-devel pam-devel numactl numactl-devel hwloc hwloc-devel lua lua-devel readline-devel rrdtool-devel ncurses-devel man2html libibmad libibumad -y
```

Download the latest version of Slurm in the shared folder:

```
cd /nfsshare
wget https://download.schedmd.com/slurm/slurm-23.02.3.tar.bz2
```

If you don't have `rpmbuild` yet:

```
yum install rpm-build
rpmbuild -ta slurm-23.02.3.tar.bz2
```

Check the rpms created by `rpmbuild`:

```
cd /root/rpmbuild/RPMS/x86_64
ls
```

Move the Slurm rpms for installation for all nodes:

```
mkdir /nfsshare/slurm-rpms
cp * /nfsshare/slurm-rpms
```

On every node, install these rpms:

```
cd /nfsshare/slurm-rpms
yum --nogpgcheck localinstall * -y
```

On the __master__ node:

```
vim /etc/slurm/slurm.conf
```

Paste the slurm.conf in Configs and paste it into `slurm.conf`.

Notice: we manually add lines under #COMPUTE NODES.
```
NodeName=node1 NodeAddr=10.0.1.6 CPUs=32 State=UNKNOWN
NodeName=node2 NodeAddr=10.0.1.7 CPUs=16 State=UNKNOWN
```

You can obtain the ip address of your nodes by typing the following into the terminal for your nodes
```
ifconfig -a
```
Note: These may change over time. In case you have future communication between master and node issues, check this. 

Now the __master__ node has the slurm.conf correctly, we need to send this file to the other compute nodes:

```
scp /etc/slurm/slurm.conf root@10.0.1.6:/etc/slurm/
scp /etc/slurm/slurm.conf root@10.0.1.7:/etc/slurm/
```

On the __master__ node, make sure that the __master__ has all the right configurations and files:

```
mkdir /var/spool/slurm
chown slurm: /var/spool/slurm/
chmod 755 /var/spool/slurm/
touch /var/log/slurmctld.log
chown slurm: /var/log/slurmctld.log
touch /var/log/slurm_jobacct.log
touch /var/log/slurm_jobcomp.log
chown slurm: /var/log/slurm_jobacct.log
chown slurm: /var/log/slurm_jobcomp.log
mkdir -p /var/spool/slurmctld
chmod 777 /var/spool/slurmctld
```

On the computing nodes __node[1-2]__, make sure that all the computing nodes have the right configurations and files:

```
mkdir /var/spool/slurm
chown slurm: /var/spool/slurm
chmod 755 /var/spool/slurm
touch /var/log/slurm/slurmd.log
chown slurm: /var/log/slurm/slurmd.log
```

Use the following command to make sure that `slurmd` is configured properly:

```
slurmd -C
```

You should get something like this:

```
NodeName=node1 CPUs=4 Boards=1 SocketsPerBoard=1 CoresPerSocket=4 ThreadsPerCore=1 RealMemory=990 UpTime=0-07:45:41
```

Disable the firewall on the computing nodes __node[1-2]__:

```
systemctl stop firewalld
systemctl disable firewalld
```

On the __master__ node, open the default ports that Slurm uses:

```
firewall-cmd --permanent --zone=public --add-port=6817/udp
firewall-cmd --permanent --zone=public --add-port=6817/tcp
firewall-cmd --permanent --zone=public --add-port=6818/udp
firewall-cmd --permanent --zone=public --add-port=6818/tcp
firewall-cmd --permanent --zone=public --add-port=6819/udp
firewall-cmd --permanent --zone=public --add-port=6819/tcp
firewall-cmd --reload
```

If the port freeing does not work, stop the firewall for testing.

Sync clocks on the cluster. On every node:

```
yum install ntp -y
chkconfig ntpd on
ntpdate pool.ntp.org
systemctl stop ntpd
systemctl start ntpd
systemctl status ntpd
```


### Setup ``slurmdbd``

Create slurmdbd configuration file:

```
vim /etc/slurm/slurmdbd.conf
```

Some variables for ``slurmdbd.conf`` are:

```
# Authentication info
AuthType=auth/munge

# slurmDBD info
DbdAddr=localhost
DbdHost=localhost
DbdPort=6819
SlurmUser=slurm
DebugLevel=verbose
LogFile=/var/log/slurm/slurmdbd.log
#PidFile=/var/run/slurmdbd.pid

# Database info
StorageType=accounting_storage/mysql
StorageHost=localhost
StoragePass=1234
StorageUser=slurm
StorageLoc=slurm_acct_db
```

Set up files and permissions:

```
chown slurm: /etc/slurm/slurmdbd.conf
chmod 600 /etc/slurm/slurmdbd.conf
touch /var/log/slurmdbd.log
chown slurm: /var/log/slurmdbd.log
```

Try to run _slurndbd_ manually to see the log:

```
slurmdbd -D -vvv
```

Terminate the process by Control+C when the testing is OK.

Start the `slurmdbd` service:

```
systemctl stop slurmdbd
systemctl enable slurmdbd
systemctl start slurmdbd
systemctl status slurmdbd
```

### Time to turn on Slurm

As the final thing to do, try this:


On the computing nodes __node[1-2]__:

```
systemctl stop slurmd.service
systemctl enable slurmd.service
systemctl start slurmd.service
systemctl status slurmd.service
```

On the __master__ node:

```
systemctl stop slurmctld.service
systemctl enable slurmctld.service
systemctl start slurmctld.service
systemctl status slurmctld.service
```

### Final Check 

```
systemctl stop slurmctld.service
systemctl stop slurmctld
systemctl stop slurmdbd
systemctl stop slurmd

systemctl start slurmd
systemctl start slurmdbd
systemctl start slurmctld
systemctl start slurmctld.service

systemctl status slurmd
systemctl status slurmdbd
systemctl status slurmctld
systemctl status slurmctld.service
```

Reference: [slothparadise](https://www.slothparadise.com/how-to-install-slurm-on-centos-7-cluster/), [Niflheim](https://wiki.fysik.dtu.dk/niflheim/Slurm_database), [gabrieleiannetti](https://github.com/gabrieleiannetti/slurm_cluster_wiki/wiki/Installing-a-Slurm-Cluster)
