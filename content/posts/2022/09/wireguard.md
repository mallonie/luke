---
title: WireGuard and EdgeOS
description: Install and setup WireGuard on EdgeOS
date: 2022-09-08
tags:
- wireguard
- vpn
- router
- ubiquity
- ui
- unifi
- edgeos
- edge router 12p
- edge router
---

# Background

I spent some time over the last few weeks getting a start on my home network and
removing the ISP provided Router from the network (putting it into modem mode),
so it only passes the internet to my router and does nothing else.

I previously had bought a [UISP EdgeRouter 12P](https://eu.store.ui.com/collections/operator-isp-infrastructure/products/edgerouter-12p)
from Ubiquity, it has been a good Router so far. The documentation around EdgeOS
could be better, it gives commands to run to make changes to the configuration
but doesn't always describe what each of the fields are or how they relate. Maybe
this is something I should come to EdgeOS already knowing, as this seems like a
bit of hardward that is intended for network specialists perhaps? I'm not sure.

So let's get to the meat of this. I've been setting up the network with this router
and a [PiHole](https://pi-hole.net) in order to have all devices on the network
use this self hosted DNS so that I can block domains. I will probably document
this in another post. With that setup I wanted to expand this and protect any devices
that I bring out of the home e.g. phone, laptop, etc...

# EdgeOS

The EdgeOS system has a web interface that allows you to configure all the things,
it is a decent interface and I've been able to work my way around it for the most
part. It does have built in support for PPTP Remote Access and IPsec Site-to-Site,
but I don't know the details of these and they seem like a bit much for setting
up on a phone and laptop.

## Enter WireGuard

So I was aware of WireGuard as a VPN from a friend, he had done a similar setup
with his network and used WireGuard. So I had a look around to see what was available
if anything. Thankfully the community has provided for this: [WireGuard/wireguard-vyatta-ubnt](https://github.com/WireGuard/wireguard-vyatta-ubnt).

This project supplies the required `.deb` file to [install wireguard on the ER12P](https://github.com/WireGuard/wireguard-vyatta-ubnt/wiki/EdgeOS-and-Unifi-Gateway)
using the `E300` release. This has support for the ER4, ER6P and ER12. I took the
chance that the ER12P and the ER12 were not so disimilar as to break the package.
So far this has held true.

## Installing WireGuard

In order to install WireGuard you will need to be able to ssh onto the Router.
You should be able to use the default login details or, if you created your own
user and deleted the default, you should be able to use your user details.

```bash
❯ ssh user@router
  _____    _
 | ____|__| | __ _  ___          (c) 2010-2020
 |  _| / _  |/ _  |/ _ \         Ubiquiti Networks, Inc.
 | |__| (_| | (_| |  __/
 |_____\__._|\__. |\___|         https://www.ubnt.com
             |___/

Welcome to EdgeOS

By logging in, accessing, or using the Ubiquiti product, you
acknowledge that you have read and understood the Ubiquiti
License Agreement (available in the Web UI at, by default,
http://192.168.1.1) and agree to be bound by its terms.

user@router password: 
Linux er12p 4.9.79-UBNT #1 SMP Thu Jun 30 07:03:20 UTC 2022 mips64
Welcome to EdgeOS
user@router:~$
```

The previously linked [install guide](https://github.com/WireGuard/wireguard-vyatta-ubnt/wiki/EdgeOS-and-Unifi-Gateway)
describes what is needed in order to install and setup WireGuard on EdgeOS. I
will outline some of that here. First thing we need to do is download and install
the `.deb` package, replacing the envvar values as required:

```bash
user@router:~$ export RELEASE=1.0.20220627 BOARD=e300-v2 MODULE=1.0.20210914
user@router:~$ curl -OL https://github.com/WireGuard/wireguard-vyatta-ubnt/releases/download/${RELEASE}-1/${BOARD}-v${RELEASE}-v${MODULE}.deb
user@router:~$ sudo dpkg -i ${BOARD}-v${RELEASE}-v${MODULE}.deb
```

## Configuring WireGuard and EdgeOS

We are now installed and can configure WireGuard to allow connections from clients.
These clients are identified by a private and public key pair, the public key is
used as part of the configuration, which we will go through now. First let's setup
the key pair for our Router:

```bash
user@router:~$ wg genkey | tee /config/auth/wg.key | wg pubkey > /config/auth/wg.public
```

With this keypair we can now setup the config for WireGuard in EdgeOS. We'll need
to enter the `configure` mode, run a few `set` commands and finally `commit`, `save`
and `exit`:

```bash
user@router:~$ configure
# Let's set the address subent for the wireguard interface wg0
# The IP in the subnet is where the router will live for this subnet, in this
# example that is 192.168.2.1
user@router:~$ set interfaces wireguard wg0 address 192.168.2.1/24
# This is the port wireguard is listening on
user@router:~$ set interfaces wireguard wg0 listen-port 51820
user@router:~$ set interfaces wireguard wg0 route-allowed-ips true
# We also need to set the private key we generated as part of this interface
user@router:~$ set interfaces wireguard wg0 private-key /config/auth/wg.key
# We need to allow access to the WireGuard Interface port in our Firewall, make
# sure to set $RULE_NUMBER to something that does not conflict with an existing
# rule or it will be over written
user@router:~$ set firewall name WAN_LOCAL rule $RULE_NUMBER action accept
user@router:~$ set firewall name WAN_LOCAL rule $RULE_NUMBER protocol udp
user@router:~$ set firewall name WAN_LOCAL rule $RULE_NUMBER description "WireGuard VPN"
user@router:~$ set firewall name WAN_LOCAL rule $RULE_NUMBER destination port 51820
# Finally let's save all these changes
user@router:~$ commit
user@router:~$ save
user@router:~$ exit
```

Now that WireGuard is setup on the Router we should add our first client, to do
this we'll need to get the public key from a client. If you don't already have
one we'll need to create one. I'll work with the Android app here, we'll create
a new WireGuard tunnel:

![Create WireGuard Tunnel](/images/wireguard-android.jpg "Create WireGuard Tunnel - Android App")

So we're creating a new Tunnel with the name `home-network`, creating a Private
and Public key, setting the IP Address that this client will have `192.168.2.100/32`
and if you have your own DNS Server, such as a [pi-hole](https://pi-hole.net) you
can set the IP address for that in here. Next you will want to copy the Public key
and create an entry for this in your Router config, this can be done as follows:

```bash
user@router:~$ configure
# Now we're going to add our first client device. This device will be able
# to connet to the internal network from outside by establishing a VPN connection
# to the Public IP with the above listen-port. Replace $PUBLIC_KEY_ID with the
# public key for your client device e.g. kaxZYUOQZVPpu/rhiUK0Rc2L3DWGld7omTu6ZhOGk3Y=
user@router:~$ set interfaces wireguard wg0 peer $PUBLIC_KEY_ID description "My Phone when outside"
user@router:~$ set interfaces wireguard wg0 peer $PUBLIC_KEY_ID allowed-ips 192.168.2.100/32
# And save changes
user@router:~$ commit
user@router:~$ save
user@router:~$ exit
```

Finally on our client device we need to setup a Peer which will be the Router.
For this we'll need the Public IP (or a domain if you have one associated with
your public ip) and the Port we've setup WireGuard to use.

![WireGuard Tunner Peer](/images/wireguard-android-peer.jpg "WireGuard Tunnel Peer - Android App")

So enter the public key you generated earlier when setting up the router: `/config/auth/wg.public`
Then enter the Public IP (or domain) and Port in the `Endpoint` field and finally
set the `Allowed IPs` to the value you would like, if you want all traffic to
run through your home network set this value to `0.0.0.0/0`. You can also have IPv6
address run through this by adding `, ::/0` to the field. Now all traffic will
go through this connection. However you could limit this, if you instead only
want traffic local to your home network to go through you can set this to a range
that encompases this e.g. `192.168.0.0/16`, now everything else will use your
devices public connection for anything outside of this range.

You can get pretty complex with `AllowedIPs` and [this website](https://www.procustodibus.com/blog/2021/03/wireguard-allowedips-calculator/)
will help you to calculate the setting if you need more than a simple range.

# Conclusion

This is a great setup if you want to have access to your home-network while out
and about. I've been using it now for few weeks and having this enabled and my
DNS queries running thought the [pi-hole](https://pi-hole.net), reducing the ads
and tracking that generally follow you around the internet has been great. It's
also really useful to be able to access devices at home if needed, so far I have
this on my phone, but I will be setting it up on my Laptop. So while I generally
do not go out and use my laptop, with this setup I can see myself being a bit
more likely to do that.

I hope this has been of use, it's good to have it documented here for myself.

