---
title: "Compute Blade: PXE Boot"
description: Setup Netboot.xyz for PXE Booting Compute Blade with RPi CM4
date: 2025-05-25
tags:
- pxe
- network
- netboot.xyz
- raspberry pi
- cm4
- compute blade
---

I recently received my Compute Blade kickstarter backer rewards, enough for a 4 node cluster. This document is to outline
what I needed to do in order to have them PXE boot.

## netboot.xyz

To start with I have deployed an instance of [netboot.xyz](https://netboot.xyz) to my network, in my case it's running
on a Raspberry Pi 3b that I already have on my network.

It was a pretty simple job to get an instance running as it already has container images that can be run with your chosen
runtime, in this case docker. I use the following script to run it, this is fairly simple and can be used to update the image
by running the script again:

```bash
#!/usr/bin/env bash

docker stop netbootxyz
docker rm netbootxyz
docker run -d \
  --name=netbootxyz \
  -p 3000:3000 \           # sets webapp port
  -p 69:69/udp \           # sets tftp port
  -p 8082:80 \             # optional, exposes the contents of /assets
  -v /mnt/config:/config \ # optional
  -v /mnt/assets:/assets \ # optional
  --restart unless-stopped \
  ghcr.io/netbootxyz/netbootxyz:latest # set a specific version here if you don't want to update when rerunning the script
```

That's your PXE server setup, next we need to tell devices on the network about this server.

## dnsmasq dhcp configuration

In order to boot the Compute Blade with the CM4 module we need to set some dnsmasq dhcp options in our DHCP server. I'm
making use of the DHCP Server built into the Ubiquity EdgeRouter 12P, these should work generally though I have seen a lot
of confusing information around the web on this.

The following settings are what I have that enables my Compute Blades to PXE boot, change the `${SERVER_DOMAIN}` and
`${SERVER_IP}` to match where you have deployed netboot.xyz:
- `pxe-service=0,Raspberry Pi Boot`
- `dhcp-match=set:arm64,option:client-arch,b`
- `dhcp-boot=tag:arm64,netboot.xyz-arm64.efi,${SERVER_DOMAIN},${SERVER_IP}`
- `dhcp-match=set:amd32,option:client-arch,2`
- `dhcp-match=set:amd32,option:client-arch,6`
- `dhcp-boot=tag:amd32,netboot.xyz.efi,${SERVER_DOMAIN},${SERVER_IP}`
- `dhcp-match=set:amd64,option:client-arch,7`
- `dhcp-match=set:amd64,option:client-arch,8`
- `dhcp-match=set:amd64,option:client-arch,9`
- `dhcp-boot=tag:amd64,netboot.xyz.efi,${SERVER_DOMAIN},${SERVER_IP}`

I believe `${SERVER_DOMAIN}` is not required, in which case you should keep the `,` so it would look something like this:
`dhcp-boot=tag:arm64,netboot.xyz-arm64.efi,,${SERVER_IP}`

Through messing around with these I have found that the CM4 requires that `pxe-service` is set as above or it will not
work at all. Then we need to `dhcp-match` the correct architecture for the CM4 which is `option:client-arch,b`, the
`set:arm64` is setting up a tag that is referenced in the `dhcp-boot` config. I have listed the other boot and match
examples just to show how you can setup `x86` and `x86_64` architectures.

## Booting the Compute Blade

At this point if you boot the Compute Blade you should see that it tries to hit the PXE server as configured but will
be looking for and not finding `start.elf` and `fixup.dat` or `start4.elf` and `fixup4.dat`, so we need to provide these
files. Here is an example of what this failing might look like:

```
  board: rev ...
   boot: mode NETWORK 2 order ... retry 0/0 restart 26/-1
     SD: card not detected
   part: 0 mbr ...
     fw: start.elf fixup.dat
    net: up ip: 192.168.0.20 sn: 255.255.255.0 gw: 0.0.0.0
   tftp: 192.168.0.10 XX:XX:XX:XX:XX:XX
display: DISP0: HDMI HPD=1 EDID=ok #2 DISP1: HPD=0 EDID=none #0

ARP 192.168.0.10 XX:XX:XX:XX:XX:XX
NET 192.168.0.20 255.255.255.0 gw 0.0.0.0 tftp 192.168.0.10
TFTP 1: File not found
TFTP 1: File not found
TFTP 1: File not found
TFTP 1: File not found
TFTP 1: File not found
TFTP 1: File not found
TFTP 1: File not found
DHCP src: XX:XX:XX:XX:XX:XX 192.168.0.1
YI_ADDR 192.168.0.20
SI_ADDR 192.168.0.1
TFTP 1: File not found
Firmware not found
```

You can get a copy of these files from the [official firmware](https://github.com/raspberrypi/firmware) or Uptime Industries
provide a [custom build](https://github.com/uptime-industries/compute-blade-cm4-uefi), which I have used.

Once you have downloaded the firmware you should add the files to the config directory of the netboot.xyz instance. There
are two ways that you can do this:
- directly into `/mnt/config/menus`, this will mean any raspberry pi system will pick up these files
- into a sub directory, `/mnt/config/menus/abc123`, where `abc123` is the serial number of the device you want to grab these specific files

Now that you have these in place when you boot the Compute Blade it will grab these files, then grab the configured `efi`
file in `dhcp-boot`. You will eventually see something like the following on the Compute Blade:

![Netboot.xyz Menu](/images/menu.jpg "Netboot.xyz Menu")

There we go, now successfully booted and able to install your chosen OS. If you have changes you would like to make to
the boot config, that can be done in the `config.txt` file downloaded in the firmware, see the docs [here](https://www.raspberrypi.com/documentation/computers/config_txt.html).
