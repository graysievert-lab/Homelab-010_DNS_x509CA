# Emulating OpenWRT arm64 router on Proxmox VE 
This guide has translation to [Russian language](https://habr.com/ru/articles/826526/).

You will need root access to an installation of Proxmox Virtual Environment, In my case it was v8.2.2. 
I was setting up OpenWrt v23.05.3.

Here's the plan:
- First we will emulate Openwrt on a x86_64 architecture, just as a backup if the next step fails
- Then we'll set up proxmox to emulate arm version of Openwrt
- And finally, ensure our emulated router resembles real-world network configuration


## OpenWrt x86_64 

Go to OpenWRT [release page](https://downloads.openwrt.org/releases/), select a release, then `targets -> x86 -> 64`. 
On the Proxmox host, download the archive and unpack it:
```bash
$ cd /var/lib/vz/template/iso

$ wget -c https://downloads.openwrt.org/releases/23.05.3/targets/x86/64/openwrt-23.05.3-x86-64-generic-squashfs-combined-efi.img.gz

$ gunzip openwrt-23.05.3-x86-64-generic-squashfs-combined-efi.img.gz
```
launch vm with the next available ID
```bash
$ qm create $(pvesh get /cluster/nextid) \
--name "openwrt-amd64" \
--description "openwrt-amd64" \
--arch x86_64 \
--tags "openwrt" \
--bios ovmf \
--efidisk0 file=local-zfs:4,efitype=4m,pre-enrolled-keys=0 \
--sockets 1 \
--cores 2 \
--memory 1024 \
--vga type=serial0 \
--serial0 socket \
--boot order=scsi0 \
--scsihw virtio-scsi-pci \
--scsi0 file=local-zfs:0,import-from="/var/lib/vz/template/iso/openwrt-23.05.3-x86-64-generic-squashfs-combined-efi.img" \
--net0 model=virtio,bridge=vmbr0,firewall=1,link_down=0,mtu=1
```
Start the machine
```bash
$ qm start 108 ; qm terminal 108
```
Check system info
```bash
$ cat /etc/os-release 
NAME="OpenWrt"
VERSION="23.05.3"
ID="openwrt"
ID_LIKE="lede openwrt"
PRETTY_NAME="OpenWrt 23.05.3"
VERSION_ID="23.05.3"
HOME_URL="https://openwrt.org/"
BUG_URL="https://bugs.openwrt.org/"
SUPPORT_URL="https://forum.openwrt.org/"
BUILD_ID="r23809-234f1a2efa"
OPENWRT_BOARD="x86/64"
OPENWRT_ARCH="x86_64"
OPENWRT_TAINTS=""
OPENWRT_DEVICE_MANUFACTURER="OpenWrt"
OPENWRT_DEVICE_MANUFACTURER_URL="https://openwrt.org/"
OPENWRT_DEVICE_PRODUCT="Generic"
OPENWRT_DEVICE_REVISION="v0"
OPENWRT_RELEASE="OpenWrt 23.05.3 r23809-234f1a2efa"
```


## OpenWrt ARM64

As mentioned in [release notes](https://pve.proxmox.com/wiki/Roadmap#Proxmox_VE_8.1) in order to be able to emulate (U)EFI firmware for ARM64 virtual machines, we would need to manually install `pve-edk2-firmware-aarch64` package.
```bash
$ apt install pve-edk2-firmware-aarch64
```
Now let's prepare the image.
Go to OpenWRT release page, select a release, then `targets -> armsr -> armv8`. 
On the Proxmox host, download the archive and unpack it:
```bash
$ cd /var/lib/vz/template/iso

$ wget -c https://downloads.openwrt.org/releases/23.05.3/targets/armsr/armv8/openwrt-23.05.3-armsr-armv8-generic-squashfs-combined.img.gz

$ gunzip openwrt-23.05.3-armsr-armv8-generic-squashfs-combined.img.gz
```
Create vm with the following command
```bash
$ qm create $(pvesh get /cluster/nextid) \ 
--name "openwrt" \
--description "openwrt" \
--tags "openwrt" \
--arch aarch64 \
--bios ovmf \
--efidisk0 file=local-zfs:4,efitype=4m,pre-enrolled-keys=0 \
--sockets 1 \
--cores 2 \
--memory 1024 \
--vga type=serial0 \
--serial0 socket \
--boot order=scsi0 \
--scsihw  virtio-scsi-pci \
--scsi0 file=local-zfs:0,import-from="/var/lib/vz/template/iso/openwrt-23.05.3-armsr-armv8-generic-squashfs-combined.img" \
--net0 model=virtio,bridge=vmbr0,firewall=1,link_down=0,mtu=1
```
load
```bash
$ qm start 108 ; qm terminal 108
```
Check system info
```bash
$ cat /etc/os-release  
NAME="OpenWrt"
VERSION="23.05.3"
ID="openwrt"
ID_LIKE="lede openwrt"
PRETTY_NAME="OpenWrt 23.05.3"
VERSION_ID="23.05.3"
HOME_URL="https://openwrt.org/"
BUG_URL="https://bugs.openwrt.org/"
SUPPORT_URL="https://forum.openwrt.org/"
BUILD_ID="r23809-234f1a2efa"
OPENWRT_BOARD="armsr/armv8"
OPENWRT_ARCH="aarch64_generic"
OPENWRT_TAINTS=""
OPENWRT_DEVICE_MANUFACTURER="OpenWrt"
OPENWRT_DEVICE_MANUFACTURER_URL="https://openwrt.org/"
OPENWRT_DEVICE_PRODUCT="Generic"
OPENWRT_DEVICE_REVISION="v0"
OPENWRT_RELEASE="OpenWrt 23.05.3 r23809-234f1a2efa"
```

## Additional network intefaces

Ok that was just one network interface which automatically becomes `br-lan` and interferes with our existing dhcp server on our network. To change it into the router mode we need to add second network interface. 
Let's create `vmbr1` bridge device in proxmox:
```bash
$ cp /etc/network/interfaces /etc/network/interfaces.new

$ cat <<EOF>>/etc/network/interfaces.new
auto vmbr1
iface vmbr1 inet static
        address 192.168.1.0/24
        bridge-ports none
        bridge-stp off
        bridge-fd 0
EOF

$ systemctl start pvenetcommit

$ systemctl restart networking
```

When two network interfaces are awailable in Openwrt, it would assign the first one `eth0` as LAN and the second one `eth1` as WAN. 
In order to simulate typical home network setup and avoid interference between dhcp servers we need our new `vmbr1` interface to become `eth0` in openwrt and proxmox main interface `vmbr0` to become `eth1`. 
Let's cleanup
```bash
$ qm stop <VMID> ; qm destroy <VMID>
```
VM creation command becomes
```bash
$ qm create $(pvesh get /cluster/nextid) \
--name "openwrt-aarch64" \
--description "openwrt-aarch64" \
--tags "openwrt" \
--arch aarch64 \
--bios ovmf \
--efidisk0 file=local-zfs:4,efitype=4m,pre-enrolled-keys=0 \
--sockets 1 \
--cores 2 \
--memory 1024 \
--vga type=serial0 \
--serial0 socket \
--boot order=scsi0 \
--scsihw  virtio-scsi-pci \
--scsi0 file=local-zfs:0,import-from="/var/lib/vz/template/iso/openwrt-23.05.3-armsr-armv8-generic-squashfs-combined.img" \
--net1 model=virtio,bridge=vmbr0,firewall=1,link_down=0,mtu=1 \
--net0 model=virtio,bridge=vmbr1,firewall=1,link_down=0,mtu=1
```
Start the machine
```bash
$ qm start <VMID> ; qm terminal <VMID>
```

After it is loaded let's check the network config
```bash
$ ip a 
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel master br-lan state UP qlen 1000
    link/ether bc:24:11:46:1e:eb brd ff:ff:ff:ff:ff:ff
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP qlen 1000
    link/ether bc:24:11:f5:5b:ae brd ff:ff:ff:ff:ff:ff
    inet 10.1.2.239/24 brd 10.1.2.255 scope global eth1
       valid_lft forever preferred_lft forever
    inet6 fe80::be24:11ff:fef5:5bae/64 scope link 
       valid_lft forever preferred_lft forever
4: br-lan: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP qlen 1000
    link/ether bc:24:11:46:1e:eb brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.1/24 brd 192.168.1.255 scope global br-lan
       valid_lft forever preferred_lft forever
    inet6 fd88:4211:f71f::1/60 scope global noprefixroute 
       valid_lft forever preferred_lft forever
    inet6 fe80::be24:11ff:fe46:1eeb/64 scope link 
       valid_lft forever preferred_lft forever


$ cat /etc/config/network 
config interface 'loopback'
        option device 'lo'
        option proto 'static'
        option ipaddr '127.0.0.1'
        option netmask '255.0.0.0'
config globals 'globals'
        option ula_prefix 'fd88:4211:f71f::/48'
config device
        option name 'br-lan'
        option type 'bridge'
        list ports 'eth0'
config interface 'lan'
        option device 'br-lan'
        option proto 'static'
        option ipaddr '192.168.1.1'
        option netmask '255.255.255.0'
        option ip6assign '60'
config interface 'wan'
        option device 'eth1'
        option proto 'dhcp'
config interface 'wan6'
        option device 'eth1'
        option proto 'dhcpv6'
```
As one can see, wan interface is mapped to `eth1` which got an IP `10.1.2.239` from the real router.

Openwrt by default exposes management interfaces on LAN side. Let's change that to simplify our interaction with emulated router. 
Please note that the following steps should not be taken on your real router. In order to open management interfaces from WAN side run the following commands.
```bash
$ uci add firewall rule
$ uci set firewall.@rule[-1].name='Allow-Admin'
$ uci set firewall.@rule[-1].enabled='true'
$ uci set firewall.@rule[-1].src='wan'
$ uci set firewall.@rule[-1].proto='tcp'
$ uci set firewall.@rule[-1].dest_port='22 80 443'
$ uci set firewall.@rule[-1].target='ACCEPT'
$ uci add firewall rule
$ uci commit firewall  
$ service firewall restart
```
check the connection (note there is no request for password or ssh key!!!): 
```bash
$ ssh root@10.1.2.239
BusyBox v1.36.1 (2024-03-22 22:09:42 UTC) built-in shell (ash)
  _______                     ________        __
 |       |.-----.-----.-----.|  |  |  |.----.|  |_
 |   -   ||  _  |  -__|     ||  |  |  ||   _||   _|
 |_______||   __|_____|__|__||________||__|  |____|
          |__| W I R E L E S S   F R E E D O M
 -----------------------------------------------------
 OpenWrt 23.05.3, r23809-234f1a2efa
 -----------------------------------------------------
=== WARNING! =====================================
There is no root password defined on this device!
Use the "passwd" command to set up a new password
in order to prevent unauthorized SSH logins.
--------------------------------------------------
```
Now we are ready for experiments with the openwrt router without chances to break the device.
