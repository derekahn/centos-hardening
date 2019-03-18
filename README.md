# Centos Hardening

Walkthrough in hardening centos

### First and foremost add and configure vim:

```bash
$ sudo yum update
$ sudo yum upgrade
$ sudo yum -y install vim-enhanced
```

### Set vimrc for all users

```bash
$ sudo vi /etc/vimrc
```

```vim
" Map leader to space
let mapleader = "\<Space>"
let g:mapleader = "\<Space>"

colo torte
syntax enable

set autoindent
set smartindent
set backspace=eol,start,indent

set expandtab
set tabstop=2
set shiftwidth=2
set ruler
set ignorecase
set smartcase

" Exit insert mode with `jj`
inoremap jj <ESC>

" Map movement to homerow
map H ^
map L $

" Show current line number
set number

" Show relative line numbers
set relativenumber

" highlights parentheses
set showmatch

" highlights matched words
" if not, specify [ set nohlsearch ]
set hlsearch

" highlights parentheses
set showmatch

" change colors for comments if it's set [ syntax on ]
highlight Comment ctermfg=LightCyan

" Visualize break ( $ ) or tab ( ^I )
set list

" wrap lines
" if not, specify [ set nowrap ]
set wrap
```

## Protect that bootloader

Protection through authentication

```bash
# Switch to root user
$ sudo su

# Set a password you won't forget
$ grub2-setpassword
Enter password:
Confirm password:

```

### Software maintenance

```bash
# switch to root
$ sudo su

# C2S/CIS: CCE-26989-4 (High)
$ gpgcheck=1

$ yum update

$ yum check-update

$ yum --security upgrade

$ yum -y install cronie

$ yum -y install psacct

$ yum remove cronie-anacron

```

### Turn off daemons 👿

```bash
# C2S/CIS: CCE-80230-6 (Medium)
# From C2S/CIS: All of these daemons (nfslock, rpcgssd, and rpcidmapd)
# run with elevated privileges, and many listen for network connections.
$ systemctl disable rpcbind.service

# C2S/CIS: CCE-80237-1 (Unknown)
# From C2S/CIS: Unnecessary services should be disabled to decrease the attack surface of the system.
$ systemctl disable nfs.service

# Disable Secure RPC Client Service
$ systemctl disable rpcsvcgssd

# Disable Secure RPC Server Service
$ systemctl disable rpcidmapd

# Disable distribute hardware interrupts across processors on a multiprocessor system
$ systemctl disable irqbalance

# Kdump is a kernel feature which is used to capture crash dumps when the system or kernel crash
$ systemctl disable irqbalance

# Kdump is a kernel feature which is used to capture crash dumps when the system or kernel crash
$ systemctl disable nfslock

$ systemctl enable irqbalance
$ systemctl enable psacct
$ systemctl enable crond


```

### Turn on some daemons 😈

[chrony](https://chrony.tuxfamily.org/comparison.html):

> chrony is a versatile implementation of the Network Time Protocol (NTP). It can synchronise the system clock with NTP servers, reference clocks (e.g. GPS receiver), and manual input using wristwatch and keyboard. It can also operate as an NTPv4 (RFC 5905) server and peer to provide a time service to other computers in the network.

> It is designed to perform well in a wide range of conditions, including intermittent network connections, heavily congested networks, changing temperatures (ordinary computer clocks are sensitive to temperature), and systems that do not run continuosly, or run on a virtual machine.

> Typical accuracy between two machines synchronised over the Internet is within a few milliseconds; on a LAN, accuracy is typically in tens of microseconds. With hardware timestamping, or a hardware reference clock, sub-microsecond accuracy may be possible.

> Two programs are included in chrony, chronyd is a daemon that can be started at boot time and chronyc is a command-line interface program which can be used to monitor chronyd’s performance and to change various operating parameters whilst it is running.

```bash
# C2S/CIS: CCE-27323-5 (Medium)
# From C2S/CIS: Due to its usage for maintenance and
# security-supporting tasks,enabling the cron daemon is essential.
$ systemctl enable crond.service

# C2S/CIS: CCE-27361-5 (Medium)
# From C2S/CIS: Access control methods provide the ability to enhance system security
# posture by restricting services and known good IP addresses and address ranges.
$ yum install tcp_wrappers

# C2S/CIS: CCE-27444-9 (Medium)
# From C2S/CIS: Synchronizing time is essential for authentication services such as Kerberos,
# but it is also important for maintaining accurate logs and auditing possible security breaches.
$ systemctl enable chronyd
```

## Accounts and Access

### Set sensible umask values

A misconfigured `umask` value could result in files with excessive permissions that can be read or written to by unauthorized users.

C2S/CIS: CCE-80202-5 (unknown); C2S/CIS: CCE-80204-1 (unknown)

```bash
# switch to root
$ sudo su

$ vim /etc/profile

# append this to bottom of file:
umask 027

$ source !$

$ vim /etc/bashrc

# append this to bottom of file:
umask 027

$ source !$
```

## Login Banner

C2S/CIS: CCE-27303-7 (Medium)

```bash
$ vim /etc/motd
$ service ssd restart
```

COPY 🍝

```shell
########################################################################################################################
#                                UNAUTHORIZED ACCESS TO THIS DEVICE IS PROHIBITED                                      #
#                  You must have explicit, authorized permission to access or configure this device.                   #
#   Unauthorized attempts and actions to access or use this system may result in civil and/or criminal penalties.      #
#                        All activities performed on this device are logged and monitored.                             #
#                            Disconnect IMMEDIATELY if you are not an authorized user!                                 #
########################################################################################################################
```

### Set password expiration

C2S/CIS: CCE-26486-1 (unknown); C2S/CIS: CCE-27002-5 (Medium); C2S/CIS: CCE-27051-2 (Medium)

```bash
# switch to root user
$ sudo su

# edit file
$ vim /etc/login.defs

# Set below:
PASS_WARN_AGE 7
PASS_MIN_DAYS 7
PASS_MAX_DAYS 90
```

### Set account expiration

C2S/CIS: CCE-27355-7 (Medium)

```bash
$ sudo su

$ vim /etc/default/useradd

# Set below:
INACTIVE=30
```

### Restrict root

C2S/CIS: CCE-27175-9 (High)

Verify only root has UID 0

```bash
# Multiple accounts with a UID of 0 afford more opportunity for potential intruders to guess a password for a privileged account.
$ awk -F: '$3 == 0 && $1 != "root" { print $1 }' /etc/passwd | xargs passwd -l root
```

### Protect direct root logins

C2S/CIS: CCE-27294-8 (Medium)

Disabling direct root logins ensures proper accountability and multifactor authentication to privileged accounts.

```bash
$ echo > /etc/securetty
```

### Verify that all world-writable directories have sticky bits set

C2S/CIS: CCE-80130-8 (Unknown)

Failing to set the sticky bit on public directories allows unauthorized users to delete files in the directory structure.

```bash
# To verify no world writable directories exist without the sticky bit set:
$ df --local -P | awk {'if (NR!=1) print $6'} | xargs -I '{}' find '{}' -xdev -type d \( -perm -0002 -a ! -perm -1000 \) 2>/dev/null
```

### Disable core dumps

C2S/CIS: CCE-26900-1 (Unknown); C2S/CIS: CCE-80169-6 (Unknown)

The core dump files may also contain sensitive information, or unnecessarily occupy large amounts of disk space.

You can restrict access to core dumps to certain users or groups, as described in the limits.conf(5) manual page.

```bash
$ sudo su

$ touch /etc/sysctl.d/hardening.conf
$ echo 'fs.suid_dumpable = 0' > !$

$ vim /etc/security/limits.conf

*     hard   core    0
```

## OpenSSH

```bash
$ sudo su

$ vim /etc/ssh/sshd_config
```

```vim
" C2S/CIS: CCE-27471-2 (High)
" Explicitly disallow SSH login from accounts with empty passwords
PermitEmptyPasswords no

" C2S/CIS: CCE-27082-7 (Medium)
" Sets the number of client alive messages
ClientAliveCountMax 0

" C2S/CIS: CCE-27433-2 (Medium)
" Set short time period
ClientAliveInterval 300

" C2S/CIS: CCE-27363-1 (Medium)
" Override environment options
PermitUserEnvironment no

" C2S/CIS: CCE-27320-1 (High)
" Set correct protocol version
Protocol 2

" C2S/CIS: CCE-27377-1 (Medium)
" Support for .rhosts
IgnoreRhosts yes

" C2S/CIS: CCE-80645-5 (Medium)
" Set specific log level
LogLevel INFO

" C2S/CIS: CCE-27295-5 (High)
" Set algorithms which are FIPS-approved
LogLevel INFO

" C2S/CIS: CCE-27413-4 (Medium)
" Disable host-based authentication
HostbasedAuthentication no

" C2S/CIS: No-CCE (Medium)
" Set authentication attempt limit
MaxAuthTries 4

" C2S/CIS: CCE-27445-6 (Medium)
" Disable root login via SSH
PermitRootLogin no
```

## Network Stack

Hardens the network layer

```bash
$ sudo su

$ vim /etc/sysctl.d/network-stack.conf
```

COPY 🍝:

```conf
net.ipv6.conf.all.accept_redirects = 0
net.ipv6.conf.default.accept_redirects = 0

net.ipv6.conf.default.accept_ra = 0
net.ipv6.conf.all.accept_ra = 0

net.ipv6.conf.all.disable_ipv6 = 1

net.ipv4.conf.default.accept_source_route = 0

net.ipv4.conf.all.accept_source_route = 0

net.ipv4.icmp_ignore_bogus_error_responses = 1

net.ipv4.conf.default.accept_redirects = 0

net.ipv4.conf.all.accept_redirects = 0

net.ipv4.conf.default.rp_filter = 1

net.ipv4.conf.all.rp_filter = 1

net.ipv4.conf.default.secure_redirects = 0

net.ipv4.conf.all.secure_redirects = 0

net.ipv4.conf.all.accept_source_route = 0

net.ipv4.conf.default.log_martians = 1
net.ipv4.conf.all.log_martians = 1

net.ipv4.icmp_echo_ignore_broadcasts = 1

net.ipv4.ip_forward = 0

net.ipv4.conf.default.send_redirects = 0

net.ipv4.conf.all.send_redirects = 0
```

```bash
# relaod config
$ sysctl -p
```

### Disable Zeroconf Networking

Zeroconf network typically occours when you fail to get an address via DHCP, the interface will be assigned a 169.254.0.0 address.

```bash
$ echo "NOZEROCONF=yes" >> /etc/sysconfig/network
```

### Deny All TCP Wrappers

TCP wrappers can provide a quick and easy method for controlling access to applications linked to them. Examples of TCP Wrapper aware applications are sshd, and portmap.

Below commands block all but SSH:

```bash
$ sudo su
$ echo "ALL:ALL" >> /etc/hosts.deny
$ echo "sshd:ALL" >> /etc/hosts.allow
```

## Kernel modules

### Prevent kernel modules being loaded

Although security vulnerabilities in kernel networking code are not frequently discovered, the consequences can be dramatic.

```bash
# Switch to root user
$ sudo su

# Create & edit modules.conf
$ vim /etc/modprobe.d/modules.conf
```

COPY 🍝:

```conf
install dccp /bin/true
install sctp /bin/true
```

## Prompt OS update installation

```bash
$ yum -y install yum-cron
$ chkconfig yum-cron on
```
