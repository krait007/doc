# Configuring OS 

1. Setup centos7 

   SOFTWARE SELECTION:  
   ​    Minimal Install, Add-Ons: Debug tools, Compatibility Libraries, Development tools, Security Tools 

   LANGUAGE: ENGLISH + CHINESE SUPPORT PACKAGE

   INSTALL PACKAGE:  yum install net-tools ntp

2. Disable SELinux

   ```
   setenforce 0 
   vi /etc/selinux/config
   SELINUX=disabled
   reboot
   ```

3. Disable iptables

   ```
   systemctl status firewalld
   systemctl stop firewalld.service
   systemctl disable firewalld.service
   ```

4. Setting OS Parameters

   - Shared Memory

   - Network

   - User Limits

     **vi /etc/sysctl.conf**

     or

     **vi /etc/sysctl.d/20gp-sysctl.conf**

     ```
     kernel.shmmax = 500000000
     kernel.shmmni = 4096
     kernel.shmall = 4000000000
     kernel.sem = 250 512000 100 2048
     kernel.sysrq = 1
     kernel.core_uses_pid = 1
     kernel.msgmnb = 65536
     kernel.msgmax = 65536
     kernel.msgmni = 2048
     net.ipv4.tcp_syncookies = 1
     net.ipv4.ip_forward = 0
     net.ipv4.conf.default.accept_source_route = 0
     net.ipv4.tcp_tw_recycle = 1
     net.ipv4.tcp_max_syn_backlog = 4096
     net.ipv4.conf.all.arp_filter = 1
     net.ipv4.ip_local_port_range = 10000 65535
     net.core.netdev_max_backlog = 10000
     net.core.rmem_max = 2097152
     net.core.wmem_max = 2097152
     vm.overcommit_memory = 2
     ```

      vi /etc/security/limits.conf

     ```
     * soft nofile 65536
     * hard nofile 65536
     * soft nproc 131072
     * hard nproc 131072
     ```

5. Config xfs mount attribute

   vi /etc/fstab

   ```
   /dev/mapper/centos-root /    xfs     defaults        0 0
   =>
   /dev/mapper/centos-root /    xfs     nodev,noatime,inode64,allocsize=16m        0 0
   ```

6. Set blockdev (read-ahead) on a device

   ```
   get blockdev info:
   /sbin/blockdev --getra devname
   example: 
   /sbin/blockdev --getra /dev/sda

   set blockdev attr:
   /sbin/blockdev --setra bytes devname
   example:
   /sbin/blockdev --setra 16384 /dev/sda
   ```
   centos7: if blockdev size reset back to 8192 after system reboot, please set: 

   ```
    echo '/sbin/blockdev --setra 16384 /dev/sda' >> /etc/rc.d/rc.local
    chmod u+x /etc/rc.d/rc.local
   ```

   ​

7. The deadline scheduler option is recommended

   ```
    echo schedulername > /sys/block/devname/queue/scheduler
    example:
    echo deadline > /sys/block/sda/queue/scheduler
   ```

8. Specify the I/O scheduler at boot time 

   ```
   grubby --update-kernel=ALL --args="elevator=deadline"
   get kernel parameter:
   grubby --info=ALL
   ```

9. Disable Transparent Huge Pages (THP)

   ```
   grubby --update-kernel=ALL --args="transparent_hugepage=never"
   reboot os
   check: cat /sys/kernel/mm/*transparent_hugepage/enabled 
   always madvise [never]
   ```

10. Disable IPC object

  ```
  vi /etc/systemd/logind.conf
  RemoveIPC=no
  service systemd-logind restart
  ```

  ​

# Installing the Greenplum Database Software

## *setup master server

1. create gpadmin user

   ```
   groupadd gpadmin
   useradd -g gpadmin -m -d /home/gpadmin -s /bin/bash gpadmin
   echo "gpadmin" | passwd --stdin gpadmin
   ```

2. upload setup package to /home/gpadmin, own user is gpadmin

3. setup the package

   ```
   sh greenplum-db-5.3.0-rhel7-x86_64.bin
   set path: /home/gpadmin/greenplum-db-5.3.0
   ```

4. prepare gpdb data (use root)

   ```
   mkdir /gpdata
   chown gpadmin:gpadmin /gpdata
   ```

5. config gpdb ENV(use gpadmin)

   ```
   echo source /home/gpadmin/greenplum-db/greenplum_path.sh >>~/.bashrc
   ```

6. config /etc/hosts (no dns server)

   ```
   vi /etc/hosts
   192.168.99.103  gpdbm
   192.168.99.104  gpdbs1
   192.168.99.105  gpdbs2
   192.168.99.106  gpdbs3
   192.168.99.107  gpdbs4
   ```
   ​


## *setup segment server

1. prepare host list file ( master server and gpadmin user)

   ```
   mkdir ~/conf
   > ~/conf/hostlist
   echo gpdbm >> ~/conf/hostlist
   echo gpdbs1 >> ~/conf/hostlist
   echo gpdbs2 >> ~/conf/hostlist
   > ~/conf/seglist
   echo gpdbs1 >> ~/conf/seglist
   echo gpdbs2 >> ~/conf/seglist
   ```

2. config ssh key exchange(  master server and gpadmin user )

   ```
   gpssh-exkeys -f ~/conf/hostlist
   ```

3. batch setup greenplum-db to segment server

   ```
   gpseginstall -f ~/conf/seglist
   ```

4. check segment

   ```
   gpssh  -f ~/conf/seglist -e ls -l $GPHOME
   ```
   ​

## *Synchronizing System Clocks

1. master host

   ```
   vi /etc/ntp.conf
   server 202.108.6.95       #cn.ntp.org.cn
   ```

2. segment host

   ```
   vi /etc/ntp.conf
   server gpdbm prefer    #master
   server gpdbms          #standby master
   ```

3. standby master host

   ```
   server mdw prefer
   server 202.108.6.95 
   ```

4. enable ntpd service autostart( all host)

   ```
   systemctl disable chronyd.service 
   systemctl enable ntpd.service
   systemctl start ntpd.service
   ```



## *prepare data storage(use root)

1.  master host

   ```
   source /home/gpadmin/greenplum-db-5.3.x.x/greenplum_path.sh 
   gpssh -h gpdbm -e 'mkdir /gpdata/master'
   gpssh -h gpdbm -e 'chown gpadmin /gpdata/master'
   ```

2. prepare data storage - segment host

   ```
   gpssh -f /home/gpadmin/conf/seglist -e 'mkdir /gpdata/primary'
   gpssh -f /home/gpadmin/conf/seglist -e 'chown gpadmin:gpadmin /gpdata/primary'
   gpssh -f /home/gpadmin/conf/seglist -e 'mkdir /gpdata/mirror'
   gpssh -f /home/gpadmin/conf/seglist -e 'chown gpadmin:gpadmin /gpdata/mirror'
   ```


   ​


## * Validating Your Systems

1. Validating OS Settings

   ```
   gpcheck -f hostlist -m gpdbm
   ```

2. Validating Hardware Performance

   Validating Network Performance

   ```
   gpcheckperf -f hostlist_net_ic -r N -d /tmp     #parallel pair test
   gpcheckperf -f hostlist_net_ic -r n -d /tmp     #serial pair test
   gpcheckperf -f hostlist_net_ic -r M -d /tmp     #full matrix test
   ```

   Validating Disk I/O and Memory Bandwidth

   ```
   gpcheckperf -f hostlist -r ds -D -d /gpdata/primary -d /gpdata/mirror	
   ```

   ​	

## *Configuring Localization Settings

1. The initialization utility, gpinitsystem, will initialize the Greenplum array with the locale setting of its execution environment by default

2. instruct gpinitsystem exactly which locale to use byspecifying the -n locale option. 

   ```
   gpinitsystem -c gp_init_config -n en_US
   ```

3. locale subcategories

   - LC_COLLATE — String sort order
   - LC_CTYPE — Character classification
   - LC_MESSAGES — Language of messages
   - LC_MONETARY — Formatting of currency amounts
   - LC_NUMERIC — Formatting of numbers
   - LC_TIME — Formatting of dates and times

4. you cannot change  categories: LC_COLLATE and LC_CTYPE after  gpinitsystem, they are fixed. 

5. The locale settings influence the following SQL features

   - Sort order in queries using ORDER BY on textual data
   - The ability to use indexes with LIKE clauses
   - The upper, lower, and initcap functions
   - The to_char family of functions

6. **The drawback of using locales other than C or POSIX in Greenplum Database is its performance impact. It slows character handling and prevents ordinary indexes from being used by LIKE. For this reason use locales only if you actually need them.**

7. Setting the Character Set

   - [ ] gpinitsystem defines the default character  by reading the setting of the ENCODING parameter in the gp_init_config file at initialization time. 


   - [ ] The default character set is UNICODE or UTF8.

   - [ ] create a database with a different character

         ```
         CREATE DATABASE dbdemo WITH ENCODING 'GBK';
         ```



# Initializing a Greenplum Database System

1. Creating the Initialization Host File, one segment host name per line

   ```
   su - gpadmin
   cd conf
   vi seglist
   ```

2. Creating the Greenplum Database Configuration File

   ```
   su - gpadmin
   cp $GPHOME/docs/cli_help/gpconfigs/gpinitsystem_config \
         /home/gpadmin/conf/gpinitsystem_config
   vi /home/gpadmin/conf/gpinitsystem_config
   DATA_DIRECTORY
   MASTER_HOSTNAME
   MASTER_DIRECTORY
   ```

3. Running the Initialization Utility

   ```
   cd ~
   gpinitsystem -c conf/gpinitsystem_config -h conf/seglist
   ```

4. Setting Greenplum Environment Variables

   ```
   vi ~/.bashrc
   source /home/gpadmin/greenplum-db/greenplum_path.sh
   export MASTER_DATA_DIRECTORY=/gpdata/master/gpseg-1
   export PGPORT=5432
   export PGUSER=gpadmin
   export PGDATABASE=postgres
   ```

   ​