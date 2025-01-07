---
layout: post
title: PPP over serial, no dial-up
date: 2025-01-07
permalink: /ppp-over-direct-serial.html
---

Suppose you have a null-modem serial line between 2 devices,
one of them Internet-connected.
The existing guides make no sense
because you're trying to set up `mgetty` without a modem.

The following post is based on these three sources:

* https://dogemicrosystems.ca/wiki/Dial-up_pool#The_Dial-in_Server

* https://superuser.com/questions/457052/communication-between-two-systems-using-pppd

* https://tldp.org/HOWTO/PPP-HOWTO/x1552.html

It could be a single-board computer and your 'big' computer,
or in my case,
it could be your machine and a faulty virtual machine
that can't do networking over Ethernet (and Slirp),
but does have a serial passthrough to the host.

This usecase appears to be undocumented online.

## Prerequisites

I'm going to assume you're running Linux on both machines,
and that they both have `pppd` and `ip`
(either `ip`, or you could use `ifconfig`/`route`,
but you'll have to translate).

These instructions may or may not be applicable to other operating systems.

## Server setup

(The IP addresses given here are examples, but working examples. Adjust to taste.)

### Enable IPv4 packet forwarding

This is simple enough.
Run `sysctl net.ipv4.ip_forward=1`,
or if you'd like to make it persistent across reboots,
create `/etc/sysctl.d/ipv4-forwarding.conf` with the following line:

```
net.ipv4.ip_forward=1
```

### Enable masquerading

If you have iptables, run

```console
# iptables -t nat -A POSTROUTING -s 192.168.32.0/24 -o $IFACE -j MASQUERADE
```

I believe the nftables equivalent would look like

```console
# nft add rule nat postrouting ip saddr 192.168.32.0/24 oifname $IFACE masquerade
```

(in both cases, replace `$IFACE` with the interface
you're connected to the Internet with;
typically eth0, but it could be wlan0, too)

Making these changes persistent is left as an exercise for the reader.

## Run pppd on both client and server

On the server, run:

```
# pppd noauth local lock nocrtscts netmask 255.255.255.0 \
  proxyarp lcp-echo-failure 60 \
  $SERVER_TTY $SPEED 192.168.32.1:192.168.32.2
```

On the client, run:

```
# pppd lock local noauth $CLIENT_TTY $SPEED
```

The IP address argument can be left out on the client end.


Replace `$SERVER_TTY` with the server's `/dev/tty*` device
that's connected to the client (e.g. `/dev/ttyUSB0`)
and vice-versa for `$CLIENT_TTY` (e.g. `/dev/ttyACM0`).

Replace `$SPEED` with the speed of the serial link.
115200 is a safe bet if you're unsure.

I'll try to explain, but no promises.
Better explanations can be found in `pppd show-options` and its man page.

* `lock`: "Lock serial device with UUCP-style lock file"

* `local`: this is not a modem,
  so don't emit control characters related to those.

* `noauth`: disable authentication.

* `nocrtcts`: disable hardware flow control.

* `netmask`: not sure why this is here?

* `proxyarp`: "sets up a proxy ARP entry in the PPP server's ARP table that
  basically says 'send all packets destined for the PPP client to me'"
  ([source](https://tldp.org/HOWTO/PPP-HOWTO/x1552.html))

* `lcp-echo-failure 60`: lengthens the number of pings before declaring a
  failed connection to 60 pings. Makes timing more forgiving.

If you're unsure if anything happened, try adding the arguments

* `debug 9`: maximum debug level. Very noisy.

* `nodetach`: makes pppd not detach from your terminal.

A successful connection would have you see the lines

```
local  IP address 192.168.32.1
remote IP address 192.168.32.2
```

(or in reverse, if you applied it to the client instead)

## Gateway and DNS

You can successfully ping 192.168.32.1, but nothing else works!
That's very easy to fix. On the client, run:

```console
# ip route add default via 192.168.32.1 dev ppp0
```

Then ping your favorite IP.

Set a DNS server.
If the client uses `resolv.conf`, add a line like

```
nameserver 9.9.9.9
```

Then ping your favorite domain.

Congrats, you now have a very slow connection to the Internet!
If your serial link is running at 115200 baud,
this would mean your connection should be about 14.4 kilobytes per second.
