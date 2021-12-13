# Raspberry-Pi-OpenWRT-Guide
OpenWRT setup on Raspberry Pi 3+

Grab a copy of the image for the latest Raspberry Pi compatible release. [Use this link](https://firmware-selector.openwrt.org/?version=21.02.1) to select firmware. In this version, we will use 32-bit version for Raspberry Pi 4. Check sha256sum against 

```
mkdir openwrt && cd $_ || exit
wget -O openwrt-raspi.img.gz https://downloads.openwrt.org/releases/21.02.1/targets/bcm27xx/bcm2709/openwrt-21.02.1-bcm27xx-bcm2709-rpi-2-ext4-factory.img.gz

# Replace with sha256sum from download page
echo "07fa423e39faacd5f26bc516f5d62518aee7ff964fd6406809575ed858f58b04" > SHA256SUM

echo $(cat SHA256SUM) openwrt-raspi.img.gz | sha256sum --check
```

Ensure "# openwrt-raspi.img.gz: OK"

```
gunzip -k openwrt-rapi.img.gz
# remove -k flag to delete compressed file
```

Plug in the Micro SD card

```
sudo dmesg | tail
```

Select the drive and format

```
sudo dd if=./openwrt-raspi.img of = /dev/sd[X] bs=2M conv=fsync status=progress
```

Eject the drive and insert into the Pi. Connect the Ethernet of the Pi to the LAN. 

From the client computer

```
nmap -T4 -sn 192.168.1.0/24 | grep "OpenWrt" --after-context=1
```

And SSH to router

```
ssh root@OpenWrt.lan
```

Enter a new password

```
passwd
```

Then, begin configuring network details.
```
uci set network.lan.ipaddr=192.168.1.15
uci set network.lan.gateway=192.168.1.1
uci set network.lan.dns=9.9.9.9
uci commit network
reboot
```

Now, chances are if you're using a Pi 4 with an ethernet dongle, the OpenWRT kernel likely does not have your needed USB drivers compile. If this is the case, will not be able to receive internet through the dongle.

In my case, I switched OpenWRT wireless module to client mode and tethered the connection through my phone's wireless network to install the necessary modules

# Instructions for client mode

Now, update and install packages

```
opkg update && opkg install kmod-usb-net-rtl8152
```

For the following, I recommend using the web inteface. You'll be adding a new firewall zone and configuring the dongle to receive packets on the WAN side. 

Go to **Network -> Firewall** 

Set the following values:

1. lan -> wan accept/accept/accept
2. wan -> reject reject/accept/reject, masquerading, MSS clamping

Next, in **Network -> Interfaces**, select _Add New Interface_.

This process is to make the USB dongle receive internet access through the dongle

Add a new interface "WAN" with the protocol "DHCP Client". Select ethernet adapter eth1. Create/assign firewall zone WAN.

# Cleanup and securing

## Add sudo & user

```
NAME=cuckoo
opkg update
opkg install shadow-useradd sudo
useradd -s /bin/ash -N -m $NAME
passwd $NAME
echo '%users ALL=(ALL) ALL' |EDITOR=/usr/bin/tee visudo
```

Alternatively, if you don’t want the password prompt:
```
echo '%users ALL=(ALL) NOPASSWD: ALL' |EDITOR=/usr/bin/tee visudo
```

Generate an SSH key and share with the user.

```
NAME=cuckoo
ssh-keygen -t ed25519 -b 4096 -f ~/.ssh/id_ed25519_router -C "Pi-OpenWRT"
ssh-copy-id -i ~/.ssh/id_ed25519 "${NAME}"@openwrt  

# Verify ability to connect

```

Finally, securing SSH

```
uci set dropbear.@dropbear[0].PasswordAuth="off"
uci set dropbear.@dropbear[0].RootPasswordAuth="off"
uci set dropbear.@dropbear[0].RootLogin="on"
uci commit dropbear
service dropbear restart
```

And reboot

