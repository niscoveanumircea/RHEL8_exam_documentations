# Study notes for EX200 RHCSA (RHEL8)
_by Mircea Niscoveanu

## Exam objectives:

#### Study points for the exam

   RHCSA exam candidates should be able to accomplish the tasks below without assistance. These have been grouped into several categories.
Understand and use essential tools

  - Access a shell prompt and issue commands with correct syntax
  - Use input-output redirection (>, >>, |, 2>, etc.)
  - Use grep and regular expressions to analyze text
  - Access remote systems using SSH
  - Log in and switch users in multiuser targets
  - Archive, compress, unpack, and uncompress files using tar, star, gzip, and bzip2
  - Create and edit text files
  - Create, delete, copy, and move files and directories
  - Create hard and soft links
  - List, set, and change standard ugo/rwx permissions
  - Locate, read, and use system documentation including man, info, and files in /usr/share/doc

#### Operate running systems


  - Boot, reboot, and shut down a system normally
  - Boot systems into different targets manually
  - Interrupt the boot process in order to gain access to a system
  - Identify CPU/memory intensive processes and kill processes
  - Adjust process scheduling
  - Manage tuning profiles
  - Locate and interpret system log files and journals
  - Preserve system journals
  - Start, stop, and check the status of network services
  - Securely transfer files between systems

#### Configure local storage

  - List, create, delete partitions on MBR and GPT disks
  - Create and remove physical volumes
  - Assign physical volumes to volume groups
  - Create and delete logical volumes
  - Configure systems to mount file systems at boot by universally unique ID (UUID) or label
  - Add new partitions and logical volumes, and swap to a system non-destructively*

#### Create and configure file systems

  - Create, mount, unmount, and use vfat, ext4, and xfs file systems
  - Mount and unmount network file systems using NFS
  - Extend existing logical volumes
  - Create and configure set-GID directories for collaboration
  - Configure disk compression
  - Manage layered storage
  - Diagnose and correct file permission problems

#### Deploy, configure, and maintain systems

  - Schedule tasks using at and cron
  - Start and stop services and configure services to start automatically at boot
  - Configure systems to boot into a specific target automatically
  - Configure time service clients
  - Install and update software packages from Red Hat Network, a remote repository, or from the local file system
  - Work with package module streams
  - Modify the system bootloader

#### Manage basic networking

   - Configure IPv4 and IPv6 addresses
   - Configure hostname resolution
   - Configure network services to start automatically at boot
   - Restrict network access using firewall-cmd/firewall

#### Manage users and groups

   - Create, delete, and modify local user accounts
   - Change passwords and adjust password aging for local user accounts
   - Create, delete, and modify local groups and group memberships
   - Configure superuser access
   
#### Manage security

   - Configure firewall settings using firewall-cmd/firewalld
   - Create and use file access control lists
   - Configure key-based authentication for SSH
   - Set enforcing and permissive modes for SELinux
   - List and identify SELinux file and process context
   - Restore default file contexts
   - Use boolean settings to modify system SELinux settings
   - Diagnose and address routine SELinux policy violations
   

### 1. Deploy, configure, and maintain systems

* Check current configuration:
   `$ systemctl get-default`
* Change the current target:
   `$ systemctl isolate name.target`
* Set the default configuration
  `$ systemctl set-default name.target`
* Change target to rescue mode:
   `$ systemctl rescue`
* View targets: 
```
$ ll /etc/systemd/system/default.target
      /etc/systemd/system/default.target -> /lib/systemd/system/graphical.target
```
```
$ ll /lib/systemd/system/runlevel*.target

    lrwxrwxrwx. 1 root root 15 Feb  4 17:01 /lib/systemd/system/runlevel0.target -> poweroff.target
    lrwxrwxrwx. 1 root root 13 Feb  4 17:02 /lib/systemd/system/runlevel1.target -> rescue.target
    lrwxrwxrwx. 1 root root 17 Feb  4 17:01 /lib/systemd/system/runlevel2.target -> multi-user.target
    lrwxrwxrwx. 1 root root 17 Feb  4 17:01 /lib/systemd/system/runlevel3.target -> multi-user.target
    lrwxrwxrwx. 1 root root 17 Feb  4 17:01 /lib/systemd/system/runlevel4.target -> multi-user.target
    lrwxrwxrwx. 1 root root 16 Feb  4 17:01 /lib/systemd/system/runlevel5.target -> graphical.target
    lrwxrwxrwx. 1 root root 13 Feb  4 17:01 /lib/systemd/system/runlevel6.target -> reboot.target
```
#### YUM Repositories

`$ yum-config-manager --add-repo https://dl.fedoraproject.org/pub/epel/8/Everything/x86_64/`

```
$cat local.repo
   [repository_name]
   name = It is like a description
   baseurl = URL
   enabled = 1
   gpgcheck = 1
   gpgkey = file:///etc/pki/rpm-rpg/repo_key (it can be downloaded from the repo - https://dl.fedoraproject.org/pub/epel/RPM-GPG-KEY-EPEL-8)
```
* List packages from a repository: `$ dnf|yum repository-packages epel list`.
* List modules: `$ dnf|yum module list`.
* Enable/disable module: `$ dnf|yum module enable|disable postgresql:12`.
* Install/reset module: `$ dnf|yum module reset|install`.
* Install GUI|Graphical Interface:
   ```
   $ yum grouplist
   $ yum groupinstall "Graphical Administration Tools|Server with GUI"
   ```

#### Updating KERNEL

      ```
      $ yum clean all
      $ yum list kernel
      $ yum install kernel
      ```
OR using RPM

   `$ rpm -ivh kernel-3.10.0-327.4.5.el7.x86_64.rpm`
   
The name of your kernel package may vary. At this point you may realize, upon running the above command, you receive an error relating to failed dependencies. To resolve this, we need to update our `linux-firmware` and `dracut` packages:

   `$ yum install linux-firmware dracut`

With the dependencies resolved, rerun the installation command. If you now **navigate to the /boot directory, you can view the newest kernel installed under vmlinuz-3.10.0327.4.5.el7.x86_64. You can also see the older kernel located alongside it. If vmlinuz-3.10.0-327.4.5.el7.x86_64 is not located in the /boot directory, run dracut to regenerate it for the latest kernel.**

Ensure the new kernel is installed by rebooting the system and once more running the `uname -r` command. It should output the version number of the newly-downloaded kernel.

```
$ yum updateinfo all
Last metadata expiration check: 0:11:40 ago on Thu 20 Feb 2020 10:50:49 PM CET.
Updates Information Summary: all
    3 New Package notice(s)
    1 Security notice(s)
        1 Low Security notice(s)
    1 Bugfix notice(s)
    2 other notice(s)
```

### 2. Manage basic networking

CLI: `nmcli`

GUI: `nmtui`

Show devices | interfaces:
```
# nmcli device status
DEVICE  TYPE      STATE      CONNECTION
ens33   ethernet  connected  ens33
lo      loopback  unmanaged  --
```
Show connections | configurations:
```
# ls -1 /etc/sysconfig/network-scripts/
ifcfg-ens33

# nmcli connection show
NAME   UUID                                  TYPE      DEVICE
ens33  1a883146-2c88-43ea-ab1b-d9c3e23a87d0  ethernet  ens33
```

### 3. LDAP configuration

**Packages:** `nss-pam-ldapd` , `pam_krb5` , `autofs` , `nfs-utils` , `openldap-clients` 


CLI: `authconfig` or install GUI: `authconfig-gtk`

`$ authconfig --enableldap --enableldapauth --enablemkhomedir --enableldaptls --ldaploadcacert=http://ldap.linuxacademy.com/pub/cert.pem --ldapserver=ldap.linuxacademy.com --ldapbasedn="dc=linuxacademy,dc=com" --update`

```
$ realmd` - for joining to an AD Domain
$ realmd discover <AD server>`
$ realmd join <AD server>`
$ ssh -l test@ad.linuxacademy.com IP_ADDRESS
```

##### Auto-Mount a Home Directory

Install the `autofs`, `openldap-clients` and `nfs-utils` packages:

`$ yum install -y autofs nfs-utils openldap-clients`

Create one config  `/etc/auto.master.d/ldap.autofs` file (similar to /etc/auto.master), and add the following line:

`/home/guests /etc/auto.ldap`

Save and exit.

Now create the `/etc/auto.ldap` file, and add the following:

`* -rw ldap.linuxacademy.com:/home/guests/&`

Save and exit.

Additionally, we need to enable the ability to use LDAP credentials through SSH. Open the `/etc/pam.d/sshd` file, and add the following lines to the top of the file:

```
auth sufficient pam_ldap.so
auth sufficient pam_permit.so
```

Start and enable the automounter daemon, then restart SSH:

```
$ systemctl start autofs && systemctl enable autofs
$ systemctl restart sshd
```


### 4. Configure local storage

MBR - supports 4 partions -> `$ fdisk`.

GBT - unlimited partitions and more disk size -> `$ gdisk`.

EXT:

`$ resize2fs` -> resize ext2/ext3/ext4 filesystems

`$ xfs_growfs` -> resize xfs filesystems

XFS:

`$ tune2fs` -> configures ext filesystems / label / UUID etc etc

`$ xfs_admin` -> configures xfs filesystems / label / UUID etc etc

### 5. Create and configure file systems

#### SAMBA

[server]
```
$ yum install -y samba
$ vi /etc/samba/smb/conf - edit config
$ systemctl start smb - start samba service
$ useradd samba_user - add a new linux user for samba
$ smbpasswd -a samba_user
```

[client]
```
$ yum install -y samba-client cifs-utils
$ mount -t cifs //server_ip/samba_shared_dir /mnt/samba_mountpoint -ousername=samba_user,password=samba_password
```

The files written on the client with any user will appear on the server in the ownership of the user "samba_user".

#### Network File System
[server]

```
$ yum install nfs-utils
$ systemctl start rpcbind
$ systemctl start nfs-server

```
/etc/exports: `/nfs-share CLIENT_IP(*,rw,sync, permissions etc)`  + `$ exportfs -a` to export al exports in file

[client]

```
$ yum install -y nfs-utils
$ systemctl start rpcbind.service
```

Show file system exports on the client: `$ showmount -e SERVER_IP`

**Mounting NFS**

```
$ mount -t nfs SERVER_IP:/nfs-share /mounted-nfs-share	
$ vi /etc/fstab : SERVER_IP:/nfs-share /mounted-nfs-share      nfs4    defaults 0 0
```

### 6. Manage users and groups

`$ chage` - set password aging
`$ authconfig --minpasslength X` - set min password length

or

file `/etc/security/pwquality.conf`- set password configuration

`$ passwd -e [user]` - force expire password

### 7 RHCSA 8 Updates

#### Adjust Process Priorities

Priority is specified in **NI** column.

```
   PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
  5966 root      20   0       0      0      0 I   0.3   0.0   0:03.22 kworker/0:1-events_power_efficient
  7187 root      20   0   63912   4304   3616 R   0.3   0.5   0:00.02 top
```

`$ nice` - sets priority when starting processes

Set process priority to : 5

`$ nice 5 cat /dev/urandom > /dev/null &`

```
   PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
  7235 root      25   5    7420   1940   1724 R  98.3   0.2   2:30.32 cat /dev/urandom
  5966 root      20   0       0      0      0 I   0.3   0.0   0:03.48 [kworker/0:1-xfs-cil/dm-0]
```

For process that already started:

`$ renice -n PRIORITY -p PID`


#### Persistent journals

Normally, `journald` journals don't persist through reboots.

Enable persistence:
```
$ mkdir /var/logs/journal
$ systemctl restart system-journald.service
```


#### Working with Virtual Data Optimizer (VDO)

* Install required package: 

```
$ yum install vdo kmod-kvdo
$ systemctl start vdo.service
```

* Create a VDO volume:

`$ vdo create --name=vdo_volume_1 --device=/dev/devName --vdoLogicalSize=vol_size`

vdoLogicalSize - 10 times the disk space if used for storage for VMs / containers

vdoLogicalSize -  3 times the disk space if used for other purposes: web / databases etc

* View information on vdo volumes:

`$ vdostats --human-readable`

* For creating filesystems on top of VDO , we need to make use **DON'T DISCARD BLOCKS** option:

```
$ mkfs.xfs -K /dev/mapper/VDO_NAME
$ mkfs.ext4 -E nodiscard /dev/mapper/VDO_NAME

+

$ udevadm settle -> to make sure the system registers the new device node
```

* Resize VDO:

`$ vdo growLogical --name=vdo_volume_1 --vdoLogicalSize=New_vol_size`

and check:

`$ vdostats --human-readable`

* Mounting persistent filesystem using VDO:

Need to add some extra flags. Before the actual mount, vdo service and others must first start

```
$ cat /etc/fstab
/dev/mapper/NAME  /mount/point   FS_TYPE(xfs,ext etc)    defaults,_netdev,x-systemd.device-timeout=0,x-systemd.requires=vdo.service    0 0  
```


#### Adjust Real Time Process Priorities (chrt)

`$ chrt`

#### AutoFS 

[client]
```
$ yum install -y autofs
$ systectm start autofs.service
```

Create auto.master file (similar to /etc/auto.master):

```
$ vi /etc/auto.master.d/auto.master
/nfs /etc/auto.nfs
```

`/etc/auto.nfs` - is the map file; can have any name


```
$ auto -rw,soft,intr NFS_SERVER_IP:/NFS_SHARE_PATH
+
$ systemctl restart autofs.service
```
