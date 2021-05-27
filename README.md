# A minimal hardened Arch Linux server base installation guide

---

**Disclaimer:**

- This guide is used for a minimal virtual server. Therefore no `EFI` or `LVM` needed.
- I use `curl` to avoid installing wget (just to spare an addtional tool)
- I dont use the Arch Linux AUR because i would need to install `base-devel` with compiler, etc. I don't want build tools on the server.
- Important for server operation: Daily updates  
- If I messed something up, understood something wrong or could do something better please give me a note! :-)

---

## Installation procedure

- The official [Arch Linux installation guide](https://wiki.archlinux.org/title/Installation_guide) is the basis for this document.

### Installation

**The basics:**

- Download the most current .iso file
- Verify the file signature
- Boot the .iso

**The installation:**

- Get network connectivity
- Adjust keyboard, disk, partion layout, etc. to your needs in the following commands to your needs

Let's begin

```shell
loadkeys de-latin1
timedatectl set-ntp true
cfdisk /dev/sda #For me: /dev/sda1 with 512M (/boot), /dev/sda2 for the rest, swapfile will be created later
mkfs.ext4 /dev/sda1
mkfs.ext4 /dev/sda2
mount /dev/sda2 /mnt
mkdir /mnt/boot
mount /dev/sda1 /mnt/boot
vim /etc/pacman.d/mirrorlist #remove all but f4st
pacstrap /mnt base linux-hardened linux-firmware grub vim zsh sudo fzf tmux htop openssh #adjust for your needs
genfstab -U /mnt >> /mnt/etc/fstab
arch-chroot /mnt
```

Now we are in the changeroot of the new system

```shell
ln -sf /usr/share/zoneinfo/Europe/Berlin /etc/localtime
hwclock --systohc
vim /etc/locale.gen #uncomment "en_US.UTF-8 UTF-8"
locale-gen
echo "LANG=en_US.UTF-8" > /etc/locale.conf
echo "KEYMAP=de-latin1" > /etc/vconsole.conf
echo hostname > /etc/hostname #adjust hostname
vim /etc/hosts #adjust with hostname
vim /etc/systemd/network/20-wired.network
```

`20-wired.network` (adjust interface):

```shell
[Match]
Name=enp0s3

[Network]
DHCP=yes
```

```shell
mkinitcpio -P #to be on the safe side
passwd #set the root password
dd if=/dev/zero of=/swapfile bs=1M count=512 status=progress
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
echo "/swapfile none swap defaults 0 0" >> /etc/fstab
grub-install --target=i386-pc /dev/sda
grub-mkconfig -o /boot/grub/grub.cfg
exit
```

Now we are back in the environment of the Live CD

```shell
umount -R /mnt #if the device is busy something is messed up, troubleshoot with 'fuser /mnt'
reboot
```

### Post-install

Now the new minimal Arch installation should have successfully booted

Login as root and execute the following commands:

```shell
systemctl enable systemd-networkd
systemctl start systemd-networkd
systemctl enable systemd-resolved
systemctl start systemd-resolved
systemctl enable sshd
systemctl start sshd
pacman -Suyy
curl -L https://git.grml.org/f/grml-etc-core/etc/zsh/zshrc --output ~/.zshrc
curl -L https://git.grml.org/f/grml-etc-core/etc/skel/.zshrc --output ~/.zshrc.local
useradd -m user -s /usr/bin/zsh #adjust username
password user
usermod -aG wheel user #could maybe be done during useradd
EDITOR=/usr/bin/vim visudo #give group wheel rights to use sudo -> This is a securiy whole because the user can get full root access
exit
```

Logoff as root and login as the newly created user.

```shell
curl -L https://git.grml.org/f/grml-etc-core/etc/zsh/zshrc --output ~/.zshrc
curl -L https://git.grml.org/f/grml-etc-core/etc/skel/.zshrc --output ~/.zshrc.local
exit #logoff and login again for the new .zshrc to load
```

Add this to `~/.vimrc` to prevent visual mode when highlighting in vim with the mouse, improve tabstops and enable syntax highlighting:

```vi
set tabstop=4
set mouse-=a
syntax on
```

## Hardening

The following hardening steps are based on the official [Arch Linux security guide](https://wiki.archlinux.org/title/Security) + some other guides e.g. for the TCP/IP stack

Disable LLMNR in `/etc/systemd/resolved.conf`:

```shell
LLMNR=no
```

Optionally use DNS over TLS in `/etc/systemd/resolved.conf`:

```shell
DNS=9.9.9.9#dns.quad9.net
DNSOverTLS=yes
```

Harden some mountpoints (/proc, /tmp, /var/tmp and /dev/shm)

`hidepid=2` also hides processes from users which do not belong to them

```sh
#hardened /proc, /tmp, /var/tmp and /dev/shm
proc /proc proc nosuid,nodev,noexec,hidepid=2,gid=proc 0 0
tmpfs /tmp tmpfs rw,size=512M,noexec,nodev,nosuid 0 0
/tmp /var/tmp tmpfs noexec,nosuid,nodev,bind 0 0
tmpfs /dev/shm tmpfs defaults,nosuid,noexec,nodev 0 0
```

Change default file permissions of some relevant files

```sh
sudo chmod 700 /boot /etc/{iptables}
```

Change default umask from `022` to `027` so that newly created files are not world readable. In `/etc/profile`:

```sh
umask 027
```

Contents of `/etc/security/faillock.conf`

```sh
dir = /var/run/faillock
audit
silent
deny = 3
fail_interval = 900
unlock_time = 600
even_deny_root
admin_group = wheel
```

Enforce a delay after a failed login attempt. Add the following line to `/etc/pam.d/system-login` to add a delay of at least 4 seconds between failed login attempts:

```sh
auth optional pam_faildelay.so delay=4000000
```

Enforce signed kernel modules and the kernel lockdown mode through adding the parameters to `/etc/default/grub`:

```sh
GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet lockdown=mode module.sig_enforce=1"
```

Disable ptracte debugging completely:

```sh
echo "kernel.yama.ptrace_scope=3" > /etc/sysctl.d/ptrace-scope.conf
```

Harden TCP/IP stack

```sh
echo "net.ipv4.conf.all.rp_filter=1" > /etc/sysctl.d/rp_filter.conf
echo "sysctl net.ipv4.conf.default.rp_filter=1" >> /etc/sysctl.d/rp_filter.conf
echo "net.ipv4.tcp_rfc1337=1" > /etc/sysctl.d/tcp_rfc1337.conf
```

Before changing the `sshd_config` as described below ensure that your ssh public key is in the `.authorized_keys` file

Harden OpenSSH server through `/etc/ssh/sshd_config`:

```sh
Port xy #change to something > 1024 and not in /etc/services so that default nmap scan cannot find ssh
AddressFamily inet
KexAlgorithms curve25519-sha256,curve25519-sha256@libssh.org,diffie-hellman-group-exchange-sha256,diffie-hellman-group16-sha512,diffie-hellman-group18-sha512,diffie-hellman-group14-sha256
HostKeyAlgorithms rsa-sha2-512,rsa-sha2-256,ssh-ed25519
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,aes128-gcm@openssh.com 
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com,umac-128-etm@openssh.com
PermitRootLogin no
AuthorizedKeysFile	.ssh/authorized_keys
PasswordAuthentication no
ChallengeResponseAuthentication no
```

Check that nmap in default config does not find the ssh-service (because the port has been changed):

```sh
sudo nmap -sS -Pn <ip>
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times will be slower.
Starting Nmap 7.91 ( https://nmap.org ) at 2021-05-19 17:11 CEST
Nmap scan report for <ip>
Host is up (0.00042s latency).
All 1000 scanned ports on <ip> are closed
MAC Address: [...]

Nmap done: 1 IP address (1 host up) scanned in 0.33 seconds
```

Check the ssh ciphers with nmap:

```sh
sudo nmap -sV -p<port> --script ssh2-enum-algos <ip>
Starting Nmap 7.91 ( https://nmap.org ) at 2021-05-19 17:09 CEST
Nmap scan report for <ip>
Host is up (0.00022s latency).

PORT      STATE SERVICE VERSION
20489/tcp open  ssh     OpenSSH 8.6 (protocol 2.0)
| ssh2-enum-algos: 
|   kex_algorithms: (6)
|       curve25519-sha256
|       curve25519-sha256@libssh.org
|       diffie-hellman-group-exchange-sha256
|       diffie-hellman-group16-sha512
|       diffie-hellman-group18-sha512
|       diffie-hellman-group14-sha256
|   server_host_key_algorithms: (4)
|       rsa-sha2-512
|       rsa-sha2-256
|       ssh-ed25519
|   encryption_algorithms: (3)
|       chacha20-poly1305@openssh.com
|       aes256-gcm@openssh.com
|       aes128-gcm@openssh.com
|   mac_algorithms: (3)
|       hmac-sha2-512-etm@openssh.com
|       hmac-sha2-256-etm@openssh.com
|       umac-128-etm@openssh.com
|   compression_algorithms: (2)
|       none
|_      zlib@openssh.com
MAC Address: [...]

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 0.48 seconds
```
