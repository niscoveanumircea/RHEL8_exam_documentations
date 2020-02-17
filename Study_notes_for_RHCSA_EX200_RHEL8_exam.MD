# Study notes for EX200 RHCSA (RHEL8)
_by Mircea Niscoveanu

## Exam objectives:

#### Study points for the exam

   RHCSA exam candidates should be able to accomplish the tasks below without assistance. These have been grouped into several categories.
Understand and use essential tools

Access a shell prompt and issue commands with correct syntax
Use input-output redirection (>, >>, |, 2>, etc.)
Use grep and regular expressions to analyze text
Access remote systems using SSH
Log in and switch users in multiuser targets
Archive, compress, unpack, and uncompress files using tar, star, gzip, and bzip2
Create and edit text files
Create, delete, copy, and move files and directories
Create hard and soft links
List, set, and change standard ugo/rwx permissions
Locate, read, and use system documentation including man, info, and files in /usr/share/doc

#### Operate running systems

        Boot, reboot, and shut down a system normally
        Boot systems into different targets manually
        Interrupt the boot process in order to gain access to a system
        Identify CPU/memory intensive processes and kill processes
        Adjust process scheduling
        Manage tuning profiles
        Locate and interpret system log files and journals
        Preserve system journals
        Start, stop, and check the status of network services
        Securely transfer files between systems

#### Configure local storage

        List, create, delete partitions on MBR and GPT disks
        Create and remove physical volumes
        Assign physical volumes to volume groups
        Create and delete logical volumes
        Configure systems to mount file systems at boot by universally unique ID (UUID) or label
        Add new partitions and logical volumes, and swap to a system non-destructively

#### Create and configure file systems

        Create, mount, unmount, and use vfat, ext4, and xfs file systems
        Mount and unmount network file systems using NFS
        Extend existing logical volumes
        Create and configure set-GID directories for collaboration
        Configure disk compression
        Manage layered storage
        Diagnose and correct file permission problems

#### Deploy, configure, and maintain systems

        Schedule tasks using at and cron
        Start and stop services and configure services to start automatically at boot
        Configure systems to boot into a specific target automatically
        Configure time service clients
        Install and update software packages from Red Hat Network, a remote repository, or from the local file system
        Work with package module streams
        Modify the system bootloader

#### Manage basic networking

        Configure IPv4 and IPv6 addresses
        Configure hostname resolution
        Configure network services to start automatically at boot
        Restrict network access using firewall-cmd/firewall

#### Manage users and groups

        Create, delete, and modify local user accounts
        Change passwords and adjust password aging for local user accounts
        Create, delete, and modify local groups and group memberships
        Configure superuser access

#### Manage security

        Configure firewall settings using firewall-cmd/firewalld
        Create and use file access control lists
        Configure key-based authentication for SSH
        Set enforcing and permissive modes for SELinux
        List and identify SELinux file and process context
        Restore default file contexts
        Use boolean settings to modify system SELinux settings
        Diagnose and address routine SELinux policy violations


## 1. Deploy, configure, and maintain systems

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

## 2. Manage basic networking

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

## 3. LDAP configuration

**Packages:** `nss-pam-dap` , `pam_krb5` , `autofs` , `nfs-utils` , `openldap-clients` 

CLI: `authconfig` or install GUI: `authconfig-gtk`

`$ authconfig --enableldap --enableldapauth --enablemkhomedir --enableldaptls --ldaploadcacert=http://ldap.linuxacademy.com/pub/cert.pem --ldapserver=ldap.linuxacademy.com --ldapbasedn="dc=linuxacademy,dc=com" --update`

```
$ realmd` - for joining to an AD Domain
$ realmd discover <AD server>`
$ realmd join <AD server>`
$ ssh -l test@ad.linuxacademy.com IP_ADDRESS
```
