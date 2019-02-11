This guide demonstrates how to build a wired/wireless router using the PC Engines [APU platform](https://www.pcengines.ch/apu.htm) and a free operating system like [OpenBSD](https://www.openbsd.org/) or [Debian](https://www.debian.org/releases/jessie/amd64/ch01s03.html.en) to be used for [network address translation](https://computer.howstuffworks.com/nat.htm), as a stateful firewall, to filter Web traffic, and much more.

I am **not** responsible for anything you do by following any part of this guide!

See also [drduh/Debian-Privacy-Server-Guide](https://github.com/drduh/Debian-Privacy-Server-Guide).

# Overview

The completed router configuration will enable:

* An egress Ethernet interface for Internet routing - can be connected to WAN or a cable modem
* A local wireless interface on `192.168.1.0/24`
* A local Ethernet interface on `10.8.1.0/24`
* A local Ethernet interface on `172.16.1.0/24`
* An additional (4th) Ethernet interface is available on APU4+

## Hardware

Order directly from [PC Engines](https://www.pcengines.ch/order.htm) or through a reseller.

This guide should work on any PC Engines APU model. Here is a suggested parts list:

| Part | Description | Cost
|------|-------------|------
| [apu4c4](https://pcengines.ch/apu4c4.htm) | apu4c4 system board | $138.00
| [case1d2bluu](https://pcengines.ch/case1d2bluu.htm) | Enclosure 3 LAN, blue | $10.00
| [ac12vus2](https://pcengines.ch/ac12vus2.htm) | AC adapter 12V 2A US plug | $4.40
| [msata16g](https://pcengines.ch/msata16g.htm) | SSD M-Sata 16GB MLC, Phison S11 | $15.50
| [wle200nx](https://pcengines.ch/wle200nx.htm) | Compex WLE200NX miniPCI express card | $19.00
| 2 x [pigsma](https://pcengines.ch/pigsma.htm) | Cable I-PEX -> reverse SMA | $3.00
| 2 x [antsmadb](https://pcengines.ch/antsmadb.htm) | Antenna reverse SMA dual band | $4.10

To connect over serial, you will need a [USB to Serial (9-Pin) Converter Cable](https://www.amazon.com/gp/product/B00IDSM6BW) and [Modem Serial RS232 Cable](https://www.amazon.com/gp/product/B000067SCH), also available from [PC Engines](https://www.pcengines.ch/usbcom1a.htm).

## Assembly

Before starting, read over the relevant APU series manual:

* [APU2](https://www.pcengines.ch/pdf/apu2.pdf)
* [APU3](https://www.pcengines.ch/pdf/apu3.pdf)
* [APU4](https://www.pcengines.ch/pdf/apu4.pdf)

Clear an area to work. Unpack all the materials. Follow [apu cooling assembly instructions](https://www.pcengines.ch/apucool.htm) to install the heat conduction plate.

Attach the mSATA disk and miniPCI wireless adapter in their respective slots.

**Note** Wireless radio cards are ESD sensitive, especially the RF switch and the power amplifier. To avoid damage by electrostatic discharge, the following installation procedure is [recommended](https://www.pcengines.ch/wle200nx.htm):

1. Touch your hands and the bag containing the radio card to a ground point on the router board (for example one of the mounting holes). This will equalize the potential of radio card and router board.
1. Install the radio card in the miniPCI express socket.
1. Install the pigtail cable in the cut-out of the enclosure. This will ground the pigtail to the enclosure.
1. Touch the I-PEX connector of the pigtail to the mounting hole (discharge), then plug onto the radio card (this is where the pre-requisite patience comes in.

Power on the board. To avoid arcing, plug in the DC jack first, then plug the adapter into mains.

By default, a memory test should run, and you should hear the board make a loud "beep" noise. If not, reboot and press `F10` to select `Payload [memtest]` and complete at least one pass.

# Installer preparation

Use another computer to prepare an installer for either OpenBSD or Debian.

Insert a USB disk. Run `dmesg` to identify its label.

## OpenBSD

Download the latest installation image release - [`amd64/install64.fs`](https://cdn.openbsd.org/pub/OpenBSD/6.4/amd64/install64.fs) - and copy the file to the USB disk:

On OpenBSD:

```console
$ doas dd if=install64.fs of=/dev/rsd2c bs=1m
```

On Linux:

```console
$ sudo dd if=install64.fs of=/dev/sdd bs=1M
```

## Debian

Download the latest network installer image - [`netinst.iso`](https://cdimage.debian.org/debian-cd/current/amd64/iso-cd/) - and copy the file to the USB disk:

On OpenBSD:

```console
$ doas dd if=debian-9.7.0-amd64-netinst.iso of=/dev/rsd2c bs=1m
```

On Linux:

```console
$ sudo dd if=debian-9.7.0-amd64-netinst.iso of=/dev/sdd bs=1M
```

Unplug the USB disk and plug it into the APU.

# Connect over serial

The APU uses 115200 baud rate, 8N1 (8 data bits, no parity, 1 stop bit).

On OpenBSD, use [cu](https://man.openbsd.org/cu):

```console
$ doas cu -r -s 115200 -l cuaU0
```

On Linux, use [screen](https://www.gnu.org/software/screen/manual/screen.html):

```console
$ screen /dev/ttyUSB0 115200 8N1
```

Or use [minicom](https://linux.die.net/man/1/minicom):

```console
$ sudo minicom -D /dev/ttyUSB0
```

Power up the APU board, DC jack first.

Make note of the BIOS version displayed briefly during boot.

# Updating BIOS

If the BIOS version is [out of date](https://pcengines.github.io/), download and extract [TinyCore Linux](https://pcengines.ch/file/apu2-tinycore6.4.img.gz) and the latest BIOS release.

**Note** Check the release notes as some PCIe cards are not detected on certain OSes with newer/mainline versions.

## Prepare

Mount a USB disk and write the TinyCore image, then copy the `.rom` file:

```console
$ sudo dd if=apu2-tinycore6.4.img of=/dev/sdd bs=1M
511+0 records in
511+0 records out
535822336 bytes (536 MB, 511 MiB) copied, 22.9026 s, 23.4 MB/s

$ sudo mount /dev/sdd1 /mnt/usb

$ tar zxvf apu4_v4.8.0.7.rom.tar.gz

$ sudo cp -v apu4_v4.8.0.7.rom /mnt/usb

$ sudo umount /mnt/usb
```

## Boot

Connect the USB disk to the APU, press `F10` at boot and select the USB disk:

```console
SeaBIOS (version rel-1.11.0.7-0-gdc51f90)

Press F10 key now for boot menu

Select boot device:

1. USB MSC Drive Samsung Flash Drive DUO 1100
2. AHCI/0: SATA SSD ATA-11 Hard-Disk (111 GiBytes)
3. Payload [setup]
4. Payload [memtest]
```

## Flash

Check the current BIOS versions:

```console
root@pcengines:~# dmesg | grep apu
[    0.000000] DMI: PC Engines apu4/apu4, BIOS v4.8.0.7 12/03/2018
```

Save the existing BIOS image and write the new one, then reboot:

```console
root@pcengines:~# cd /media/SYSLINUX

root@pcengiens:~# flashrom -p internal -r apu4.rom.$(date +%F)

root@pcengines:~# flashrom -p internal -w apu4_v4.8.0.7.rom
flashrom v0.9.8-r1888 on Linux 3.16.6-tinycore (i686)
flashrom is free software, get the source code at http://www.flashrom.org

Calibrating delay loop... OK.
coreboot table found at 0xcfed1000.
Found chipset "AMD FCH".
Enabling flash write... OK.
Found Winbond flash chip "W25Q64.V" (8192 kB, SPI) mapped at physical address 0xff800000.
Reading old flash chip contents... done.
Erasing and writing flash chip... Erase/write done.
Verifying flash... VERIFIED.

root@pcengines:~# reboot
```

## Verify

Verify the BIOS version by checking serial output during boot, or from the operating system:

```
PC Engines apu4
coreboot build 20180312
BIOS version v4.8.0.7
```

OpenBSD:

```console
$ dmesg | grep bios
bios0 at mainbus0: SMBIOS rev. 2.7 @ 0xcfe9a020 (7 entries)
bios0: vendor coreboot version "v4.8.0.7" date 12/03/2018
bios0: PC Engines apu4
acpi0 at bios0: rev 2
```

Debian:

```console
$ sudo dmesg | grep apu
[    0.000000] DMI: PC Engines apu4/apu4, BIOS v4.8.0.7 12/03/2018
```

# Installing the OS

Press `F10` at boot and select the USB disk.

## OpenBSD

First set the serial console parameters:

```console
Booting from Hard Disk...
Using drive 0, partition 3.
Loading......
probing: pc0 com0 com1 com2 com3 mem[639K 3580M 496M a20=on]
disk: hd0+ hd1+*
>> OpenBSD/amd64 BOOT 3.41
boot> stty com0 115200
boot> set tty com0
switching console to com>> OpenBSD/amd64 BOOT 3.34
boot> [Press Enter]
```

Select the Install option:

```console
Welcome to the OpenBSD/amd64 6.4 installation program.
(I)nstall, (U)pgrade, (A)utoinstall or (S)hell? I
```

When presented with a list of network interfaces, `em0` or `re0` is the Ethernet port closest to the serial port:

```console
Available network interfaces are: em0 em1 em2 em3 vlan0.

Available network interfaces are: re0 re1 re2 athn0 vlan0.
```

Use DHCP or configure a static route:

```console
Which network interface do you wish to configure? (or 'done') [re0]
IPv4 address for re0? (or 'dhcp' or 'none') [dhcp] 192.168.1.2
Netmask for re0? [255.255.255.0]
IPv6 address for re0? (or 'autoconf' or 'none') [none]
Available network interfaces are: re0 re1 re2 athn0 vlan0.
Which network interface do you wish to configure? (or 'done') [done]
Default IPv4 route? (IPv4 address or none) 192.168.1.1
add net default: gateway 192.168.1.1
DNS domain name? (e.g. 'example.com') [my.domain] local
DNS nameservers? (IP address list or 'none') [none] 192.168.1.1
```

Configure the root password and set up a user account:

```console
Password for root account? (will not echo)
Password for root account? (again)
Start sshd(8) by default? [yes]
Change the default console to com0? [yes]
Available speeds are: 9600 19200 38400 57600 115200.
Which speed should com0 use? (or 'done') [115200]
Setup a user? (enter a lower-case loginname, or 'no') [no] sysadm
Full name for user sysadm? [sysadm]
Password for user sysadm? (will not echo)
Password for user sysadm? (again)
WARNING: root is targeted by password guessing attacks, pubkeys are safer.
Allow root ssh login? (yes, no, prohibit-password) [no]
```

Select the internal mSATA disk and configure the partions:

```console
Available disks are: sd0 sd1 sd2.
Which disk is the root disk? ('?' for details) [sd0] ?
sd0: ATA, SB2, SBFM naa.0000000000000000 (111.8G)
sd1: Samsung, Flash Drive DUO, 1100 serial.0000000000000000 (29.9G)
sd2: Multiple, Card Reader, 1.00 serial.0000000000000000
Available disks are: sd0 sd1 sd2.
Which disk is the root disk? ('?' for details) [sd0]
No valid MBR or GPT.
Use (W)hole disk MBR, whole disk (G)PT or (E)dit? [whole] W
Setting OpenBSD MBR partition to whole sd0...done.
The auto-allocated layout for sd0 is:
                 size           offset  fstype [fsize bsize   cpg]
  a:             1.0G               64  4.2BSD   2048 16384     1 # /
  b:             4.2G          2097216    swap
  c:           111.8G                0  unused
  d:             4.0G         10939072  4.2BSD   2048 16384     1 # /tmp
  e:            11.9G         19327648  4.2BSD   2048 16384     1 # /var
  f:             2.0G         44351360  4.2BSD   2048 16384     1 # /usr
  g:             1.0G         48545664  4.2BSD   2048 16384     1 # /usr/X11R6
  h:            16.3G         50642816  4.2BSD   2048 16384     1 # /usr/local
  i:             2.0G         84777504  4.2BSD   2048 16384     1 # /usr/src
  j:             6.0G         88971808  4.2BSD   2048 16384     1 # /usr/obj
  k:            63.4G        101554720  4.2BSD   2048 16384     1 # /home
```

**Note** The 111.8G "unused" space partition (/dev/sd0c) is actually the entire disk.

Select "Auto layout" to continue:

```console
Use (A)uto layout, (E)dit auto layout, or create (C)ustom layout? [a] A
/dev/rsd0a: 1024.0MB in 2097152 sectors of 512 bytes
6 cylinder groups of 202.47MB, 12958 blocks, 25984 inodes each
/dev/rsd0k: 64883.7MB in 132881824 sectors of 512 bytes
321 cylinder groups of 202.47MB, 12958 blocks, 25984 inodes each
/dev/rsd0d: 4096.0MB in 8388576 sectors of 512 bytes
21 cylinder groups of 202.47MB, 12958 blocks, 25984 inodes each
/dev/rsd0f: 2048.0MB in 4194304 sectors of 512 bytes
11 cylinder groups of 202.47MB, 12958 blocks, 25984 inodes each
/dev/rsd0g: 1024.0MB in 2097152 sectors of 512 bytes
6 cylinder groups of 202.47MB, 12958 blocks, 25984 inodes each
/dev/rsd0h: 16667.3MB in 34134688 sectors of 512 bytes
83 cylinder groups of 202.47MB, 12958 blocks, 25984 inodes each
/dev/rsd0j: 6144.0MB in 12582912 sectors of 512 bytes
31 cylinder groups of 202.47MB, 12958 blocks, 25984 inodes each
/dev/rsd0i: 2048.0MB in 4194304 sectors of 512 bytes
11 cylinder groups of 202.47MB, 12958 blocks, 25984 inodes each
/dev/rsd0e: 12218.6MB in 25023712 sectors of 512 bytes
61 cylinder groups of 202.47MB, 12958 blocks, 25984 inodes each
Available disks are: sd1 sd2.
Which disk do you wish to initialize? (or 'done') [done]
```

Select a [mirror](https://www.openbsd.org/ftp.html) and complete the setup, then unplug the USB disk and reboot.

See the OpenBSD [FAQ](https://www.openbsd.org/faq/faq4.html#Install) for more information.

## Debian

At the install menu, press `Tab` to edit boot options and replace `quiet` with:

```
console=ttyS0,115200n8
```

Proceed through the installer, selecting *Guided - use entire disk* as the partioning method. Be sure to select the SSD disk rather than the USB drive as the installation target.

Select the option to store `/var`, `/tmp` and `/home` on separate partitions [if possible](https://www.debian.org/releases/stable/armel/apcs03.html.en).

Be sure to select `/dev/sda` or equivalent SSD disk at the GRUB loader installation prompt.

# First boot

## OpenBSD

The following boot parameters have been appended to `/etc/boot.conf` by the installer and everything should just work:

```
stty com0 115200
set tty com0
```

## Debian

After the GRUB menu, output will get stuck at:

```
Loading Linux 4.9.0-6-amd64 ...
Loading initial ramdisk ...
```

Reboot and select `e` at the GRUB menu to enter edit mode, scroll down and replace the word `quiet` with:

```
console=ttyS0,115200n8
```
    
**Note** If arrow keys do not work in GRUB, try using emacs key bindings to navigate the text field:

* `Control-B` to move left
* `Control-F` to move right
* `Control-P` to move up
* `Control-N` to move down

Press `Control-X` to continue booting and you should see console output.

**Note** If you get an error like, `Alert! /dev/sdX1 does not exist dropping to shell` and are dropped to an initramfs prompt, reboot and edit the `quiet` line to point to `/dev/sda1` or correct partition.

# First login

## OpenBSD

Log in as `root` and install any [pending updates](https://man.openbsd.org/syspatch) (unless already following -current):

```console
$ syspatch -v
```

Install any pending [firmware updates](https://man.openbsd.org/fw_update):

```console
$ fw_update
```

Edit `/etc/doas.conf` to allow the regular user to run [privileged commands](https://man.openbsd.org/doas.conf) without a password:

```
permit nopass keepenv :wheel
permit nopass keepenv root
```

Install any needed software:

```console
$ pkg_add vim zsh curl pftop vnstat
```

Log out as `root` when finished.

## Debian

Log in as `root` to get started.

Update GRUB by editing `/etc/default/grub` and replacing `quiet` with `console=ttyS0,115200n8`, then update the configuration:

```console
root@pcengines:~# update-grub2
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-4.9.0-8-amd64
Found initrd image: /boot/initrd.img-4.9.0-8-amd64
```

Install any pending updates or install software:

```console
root@pcengines:~# apt-get update && apt-get -y upgrade

root@pcengines:~# apt-get -y install \
    sudo ssh tmux lsof vim zsh git \
    dnsmasq privoxy hostapd \
    iptables iptables-persistent \
    curl dnsutils ntp net-tools tcpdump \
    make autoconf gcc gnupg ca-certificates apt-transport-https \
    man-db jmtpfs file htop lshw less
```

**Optional** Change the default login shell to Zsh:

```
# chsh -s /usr/bin/zsh sysadm
```

# Configure network interfaces

Ethernet or wireless network interfaces can now be configured.

## OpenBSD

On the APU, set a local network interface address and make it permanent:

```console
$ doas ifconfig em1 10.8.1.1 255.255.255.0

$ echo "inet 10.8.1.1 255.255.255.0" | doas tee /etc/hostname.em1
```

Configure an OpenBSD client with DHCP by following the [Networking FAQ](https://www.openbsd.org/faq/faq6.html) or using a static address:

```console
$ doas ifconfig em1 10.8.1.4 255.255.255.0

$ ping -c 1 10.8.1.1
PING 10.8.1.1 (10.8.1.1): 56 data bytes
64 bytes from 10.8.1.1: icmp_seq=0 ttl=255 time=0.845 ms
```

**Optional** Randomize MAC addresses on boot:

```console
$ echo "lladdr random" | doas tee -a /etc/hostname.em0 /etc/hostname.em1 /etc/hostname.em2
```

## Debian

On the APU and on another computer, determine the interface names available:

```console
root@pcengines:~# lshw -C network | grep "logical name"
       logical name: enp1s0
       logical name: enp2s0
       logical name: enp3s0
       logical name: wlp4s0

user@localhost$ sudo lshw -C network | grep "logical name"
       logical name: eno1
       logical name: enp3s0
```

On the APU, edit `/etc/network/interfaces` to append:

```
allow-hotplug enp2s0
iface enp2s0 inet static
address 10.8.1.1
netmask 255.255.255.0
gateway 10.8.1.1
```

Where `enp1s0` is the Ethernet port nearest to the serial port.

Restart networking and bring up the interface:

```console
root@pcengines:~# service networking restart

root@pcengines:~# ifup enp2s0
```

On another Linux computer, edit `/etc/network/interfaces` to append:

```
allow-hotplug eno1
iface eno1 inet static
address 10.8.1.2
netmask 255.255.255.0
gateway 10.8.1.1
```

Then also restart networking and bring up the interface:

```console
$ sudo service networking restart

$ sudo ifup eno1
```

Or on another OpenBSD computer, edit `/etc/hostname.em0` to append:

```
inet 10.8.1.4 255.255.255.0
```

It should now be possible to ping the router:

```console
$ ping -c 1 10.8.1.1
PING 10.8.1.1 (10.8.1.1): 56 data bytes
64 bytes from 10.8.1.1: icmp_seq=0 ttl=64 time=0.519 ms
```

To configure the wireless interface, edit `/etc/network/interfaces` on the APU to include:

```
auto wlp4s0
iface wlp4s0 inet static
address 192.168.1.1
netmask 255.255.255.0
hostapd /etc/hostapd.conf
```

Log out as `root` or reboot to continue.

# Configure SSH

From a client, an SSH connection to the APU should be possible, but not yet authorized:

```console
$ ssh sysadm@10.8.1.1
The authenticity of host '10.8.1.1 (10.8.1.1)' can't be established.
ECDSA key fingerprint is SHA256:AAAAA.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '10.8.1.1' (ECDSA) to the list of known hosts.
Permission denied (publickey,password).
```

If using a [YubiKey](https://github.com/drduh/YubiKey-Guide), copy its public key to clipboard:

```console
$ ssh-add -L | awk '{print $1" "$2}' | xclip
```

Or generate new key on the client and copy it to clipboard:

```console
$ ssh-keygen -f ~/.ssh/pcengines -C 'sysadm'
Generating public/private rsa key pair.
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in .ssh/pcengines.
Your public key has been saved in .ssh/pcengines.pub.

$ cat ~/.ssh/pcengines.pub | xclip
```

On the APU, over the serial connection, as the primary user (e.g., `sysadm` - *not* `root`), configure SSH to accept that key by pasting it into `~/.ssh/authorized_keys`:

```console
$ mkdir ~/.ssh ; cat > ~/.ssh/authorized_keys
[Paste clipboard contents using the middle mouse button]
[Then press Control-D to save]
```

SSH from a client will now work:

```console
$ ssh sysadm@10.8.1.1 -i ~/.ssh/pcengines
Host key fingerprint is SHA256:AAAAA

Linux pcengines 4.9.0-8-amd64 #1 SMP Debian 4.9.130-2 (2018-10-27) x86_64
sysadm@pcengines~ %
```

Configure the connection on a client by editing `~/.ssh/config`:

```
Host pcengines
  HostName 10.8.1.1
  IdentityFile ~/.ssh/pcengines
  User sysadm
  Port 22
  ControlMaster auto
  ControlPath ~/.ssh/master-%r@%h:%p
  ControlPersist 1m
```

Connect using the new alias:

```console
$ ssh pcengines
Host key fingerprint is SHA256:AAAAA
Last login: Mon Jan 1 12:00:00 2018 from 10.8.1.2
sysadm@pcengines~ %
```

**Optional** Clone my configuration repository for the rest of the setup:

```console
$ git clone https://github.com/drduh/config
```

# DHCP and DNS

[Dnsmasq](http://www.thekelleys.org.uk/dnsmasq/doc.html) can be used to provide [DHCP](https://en.wikipedia.org/wiki/Dynamic_Host_Configuration_Protocol) and DNS.

Use [drduh/config/dnsmasq.conf](https://github.com/drduh/config/blob/master/dnsmasq.conf) with blocklists and configure to your needs:

```console
$ sudo cp config/dnsmasq.conf /etc/dnsmasq.conf

$ cat config/domains/* | sudo tee -a /etc/dnsmasq.conf

$ sudo vim /etc/dnsmasq.conf

$ git clone https://github.com/StevenBlack/hosts

$ sudo cp hosts/hosts /etc/dns-blocklist
```

## OpenBSD

Install dnsmasq, start the service and enable it on boot:

```console
$ doas pkg_add dnsmasq

$ doas rcctl start dnsmasq
dnsmasq(ok)

$ doas rcctl enable dnsmasq
```

## Debian

Restart the service and enable it on boot:

```console
$ sudo service dnsmasq restart

$ sudo update-rc.d dnsmasq enable
```

# Wireless

## OpenBSD

**Note** Wireless performance is currently significantly worse on OpenBSD than Debian.

Edit `/etc/hostname.athn0` to include:

```shell
inet 192.168.1.1 255.255.255.0
media autoselect mode 11n mediaopt hostap chan 11
nwid NAME wpakey PASSWORD
```

Restart networking:

```console
$ doas sh /etc/netstart
```

## Debian

Install the default hostapd configuration:

```console
$ zcat /usr/share/doc/hostapd/examples/hostapd.conf.gz | sudo tee -a /etc/hostapd.conf
```

Or use [drduh/config/hostapd.conf](https://github.com/drduh/config/blob/master/hostapd.conf):

```console
$ sudo cp config/hostapd.conf /etc/hostapd.conf

$ sudo vim /etc/hostapd.conf
```

Ensure hostapd starts:

```console
$ sudo hostapd -dd /etc/hostapd.conf
```

If there are any syntax errors, you may need a newer version of hostapd:

```console
$ curl -O https://w1.fi/releases/hostapd-2.7.tar.gz

$ sha256sum hostapd-2.7.tar.gz
21b0dda3cc3abe75849437f6b9746da461f88f0ea49dd621216936f87440a141  hostapd-2.7.tar.gz

$ tar xf hostapd-2.7.tar.gz

$ cd hostapd-2.7/hostapd

$ sudo apt-get install pkg-config libnl-3-dev libssl-dev libnl-genl-3-dev

$ cp defconfig .config

$ make -sj8

$ sudo make install

$ hostapd -v
hostapd v2.7

$ sudo hostapd -d /etc/hostapd.conf
Configuration file: /etc/hostapd.conf
wlp4s0: interface state UNINITIALIZED->COUNTRY_UPDATE
ACS: Automatic channel selection started, this may take a bit
wlp4s0: interface state COUNTRY_UPDATE->ACS
wlp4s0: ACS-STARTED
wlp4s0: ACS-COMPLETED freq=5220 channel=44
Using interface wlp4s0 with hwaddr 00:20:91:f8:bb:fb and ssid "foo"
wlp4s0: interface state ACS->ENABLED
wlp4s0: AP-ENABLED
wlp4s0: STA 00:20:91:d1:c3:8a IEEE 802.11: authenticated
wlp4s0: STA 00:20:91:d1:c3:8a IEEE 802.11: associated (aid 1)
wlp4s0: AP-STA-CONNECTED 00:20:91:d1:c3:8a
```

**Note** You may need to manually assign the interface an address:

```console
$ sudo ifconfig wlp4s0 192.168.1.1
```

# IP forwarding

In order to be a router, [IP forwarding](https://www.kernel.org/doc/Documentation/networking/ip-sysctl.txt) must be enabled.

## OpenBSD

Enable now and on boot:

```console
$ doas sysctl net.inet.ip.forwarding=1
net.inet.ip.forwarding: 0 -> 1

$ echo "net.inet.ip.forwarding=1" | doas tee -a /etc/sysctl.conf
```

## Debian

Enable now and on boot:

```console
$ sudo sysctl -w net.ipv4.ip_forward=1

$ echo "net.ipv4.ip_forward=1" | sudo tee --append /etc/sysctl.conf
```

# Configure firewall

## OpenBSD

See [PF - Building a Router](https://www.openbsd.org/faq/pf/example1.html), or use [drduh/config/pf/](https://github.com/drduh/config/blob/master/pf/) files:

```console
$ doas mkdir /etc/pf
$ doas curl -Lfvo /etc/pf.conf https://raw.githubusercontent.com/drduh/config/master/pf/pf.conf
$ doas curl -Lfvo /etc/pf/blocklist https://raw.githubusercontent.com/drduh/config/master/pf/blocklist
$ doas curl -Lfvo /etc/pf/martians https://raw.githubusercontent.com/drduh/config/master/pf/martians
$ doas curl -Lfvo /etc/pf/private https://raw.githubusercontent.com/drduh/config/master/pf/private
```

Turn PF off and back on again:

```console
$ doas pfctl -d ; doas pfctl -e -f /etc/pf.conf
pf disabled
pf enabled 
```

**Optional** Use [drduh/config/scripts/pf-blocklist.sh](https://github.com/drduh/config/blob/master/scripts/pf-blocklist.sh) to find and block unwanted networks.

To inspect blocked traffic:

```console
$ doas tcpdump -qni pflog0
```

## Debian

Use [Iptables](https://en.wikipedia.org/wiki/Iptables) to manage a stateful firewall.

Use [drduh/config/scripts/iptables.sh](https://github.com/drduh/config/blob/master/scripts/iptables.sh) and edit it to your needs:

```console
$ cp config/scripts/iptables.sh firewall.sh

$ vim firewall.sh

$ chmod +x firewall.sh

$ sudo ./firewall.sh
```

Apply the firewall rules on boot:

```console
$ sudo iptables-save | sudo tee /etc/iptables/rules.v4
```

# Web filtering

[Privoxy](https://www.privoxy.org/) is a powerful Web proxy capable of filtering and rewriting URLs.

It can be used in combination with [Lighttpd](https://www.lighttpd.net/) with [mod_magnet](https://redmine.lighttpd.net/projects/1/wiki/Docs_ModMagnet) to block or replace ads/content with custom pages and images.

## Debian

Install Privoxy and Lighttpd with magnet mod:

```console
$ sudo apt-get -y install privoxy lighttpd lighttpd-mod-magnet
```

Use my configurations and edit them to your needs:

* [drduh/config/privoxy/config](https://github.com/drduh/config/blob/master/privoxy/config)
* [drduh/config/privoxy/user.action](https://github.com/drduh/config/blob/master/privoxy/user.action)
* [drduh/config/lighttpd/lighttpd.conf](https://github.com/drduh/config/blob/master/lighttpd/lighttpd.conf)
* [drduh/config/lighttpd/magnet.luau](https://github.com/drduh/config/blob/master/lighttpd/magnet.luau)

```console
$ sudo cp config/privoxy/config /etc/privoxy
$ sudo cp config/privoxy/user.action /etc/privoxy/

$ sudo cp config/lighttpd/lighttpd.conf /etc/lighttpd
$ sudo cp config/lighttpd/magnet.luau /etc/lighttpd
```

Restart both services and check their logs:

```console
$ sudo service privoxy restart

$ sudo tail -f /var/log/privoxy/logfile
Info: Program name: /usr/sbin/privoxy
Info: Loading filter file: /etc/privoxy/default.filter
Info: Loading filter file: /etc/privoxy/user.filter
Info: Loading actions file: /etc/privoxy/default.action
Info: Loading actions file: /etc/privoxy/match-all.action
Info: Loading actions file: /etc/privoxy/user.action
Info: Listening on port 8118 on IP address 127.0.0.1
Info: Listening on port 8118 on IP address 10.8.1.1
Info: Listening on port 8118 on IP address 172.16.1.1
Request: example.com/
Crunch: Redirected: http://bbc.com/

$ sudo service lighttpd restart

$ sudo cat /var/log/lighttpd/error.log
2019-01-01 12:00:00: (log.c.217) server started
```

# Security and maintenance

To confirm the firewall is configured correctly, run a port scan from an external host:

```console
$ nmap -v -A -T4 1.2.3.4 -Pn
```

So long as no services are exposed to the Internet, the risk of *remote* compromise is minimal.

Install a USB camera and configure [Motion](https://motion-project.github.io/) to monitor and detect physical access.

It may also be possible to increase system entropy with an external source like [OneRNG](http://onerng.info/).

## OpenBSD

Check open ports and listening programs with `doas fstat | grep net` or `doas netstat -a -n -p udp -p tcp`.

Check running processes and logged-in users with `ps -A` and `last`.

Pay attention to [OpenBSD errata](https://www.openbsd.org/errata.html) and apply security fixes periodically with `doas syspatch`.

OpenBSD releases occur approximately every six months - [follow current snapshots](https://www.openbsd.org/faq/current.html) for faster updates.

Mount `/tmp` as `tmpfs` (in memory only) by replacing the tmp partition line in `/etc/fstab` with the following line. Be sure to increase the size >1G for system upgrades!

```
swap /tmp mfs rw,noexec,nosuid,nodev,-s=256M 0 0
```

Check temperatures with `sysctl hw.sensors` or configure [sensorsd](https://man.openbsd.org/OpenBSD-current/man8/sensorsd.8).

## Debian

Check open ports and listening programs with `sudo lsof -Pni` or `sudo netstat -npl`.

Check running processes and logged-in users with `ps -eax` and `last -F`.

Pay attention to [Debian security advisories](https://lists.debian.org/debian-security-announce/recent).

Run `sudo apt-get update && sudo apt-get upgrade` periodically or configure [unattended upgrades](https://wiki.debian.org/UnattendedUpgrades).

See also [Debian SSD Optimizations](https://wiki.debian.org/SSDOptimization).

# Similar work

* [elad/openbsd-apu2](https://github.com/elad/openbsd-apu2)
* [martinbaillie/homebrew-openbsd-pcengines-router](https://github.com/martinbaillie/homebrew-openbsd-pcengines-router)
* [northox/openbsd-apu2](https://github.com/northox/openbsd-apu2)
* [vedetta-com/vedetta](https://github.com/vedetta-com/vedetta)

