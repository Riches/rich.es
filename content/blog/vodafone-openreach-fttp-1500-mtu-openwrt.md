+++
title = "Getting a full 1500-byte MTU on Vodafone FTTP on Openreach Network"
date = "2026-10-03"
draft = false
description = "Testing RFC 4638 mini jumbo frames on Vodafone FTTP over Openreach: a working 1500-byte PPPoE MTU with OpenWrt, including IPv6 and the configuration pitfalls."
tags = ["Networking", "OpenWrt", "Vodafone", "Openreach"]
+++

I run Vodafone FTTP over Openreach with a MikroTik RB5009 and OpenWrt. I'd left the MTU at Vodafone's recommended 1492, but wondered whether I could get the full 1500. I hadn't seen Vodafone advertise support for it, so I wasn't sure what to expect.

PPPoE normally uses eight bytes of a 1500-byte Ethernet payload, leaving 1492 for the IP packet. [RFC 4638](https://www.rfc-editor.org/rfc/rfc4638.html) allows a larger payload if both ends and the Ethernet link support it. Set the Ethernet MTU to 1508, and there's room for a full 1500-byte IP packet plus the PPPoE and PPP headers. These slightly larger frames are often called mini jumbo frames, or baby jumbo frames. My router supported them; Vodafone's side was the unknown.

The first attempt took the internet down. I rolled back, waited for it to return, and looked at the negotiation. Vodafone had actually accepted the 1500-byte payload. The failure was down to the old PPPoE connection not closing cleanly: new connection requests went unanswered until Vodafone cleared the old session.

For the retry, I brought PPPoE down cleanly before changing the Ethernet settings, with a backup and automatic rollback ready. It connected in about six seconds. OpenWrt showed 1508 on Ethernet and 1500 on PPPoE, and the router could send and receive full-sized IPv4 and IPv6 packets.

Then came the odd bit: large IPv6 pings from my PC failed, while smaller pings and web traffic worked. The LAN IPv6 MTU was still 1492, and Windows had learned that value too. It was fragmenting the large packets before they even reached the router.

The automatic rollback kicked in while I was figuring that out. I reapplied the WAN settings, set the LAN bridge's IPv6 MTU to 1500, and refreshed the router advertisements. Windows picked up the new value and the large pings worked. With the checks passing, I cancelled the rollback.

So my Vodafone line now carries **1500-byte IP packets over PPPoE for both IPv4 and IPv6**, including traffic from the LAN. I've kept it that way. Any efficiency gain is small: the headers are still there, the Ethernet frames are larger, and I haven't measured a speed improvement. Mostly, I wanted to see whether 1492 was really the limit. On my connection, it isn't.

## The configuration I ended up with

I tested this on 3 October 2026, using OpenWrt 25.12.5 on the RB5009. PPPoE runs directly on `p1`, the Ethernet port connected to the ONT.

| Layer | Device on my router | MTU |
| --- | --- | ---: |
| Ethernet connection to the ONT | `p1` | 1508 |
| PPPoE IP interface | `pppoe-wan` | 1500 |
| LAN IPv6 | `br-lan` | 1500 |

In `/etc/config/network`, I added a device section for the WAN Ethernet port:

```uci
config device 'wan_mini_jumbo'
        option name 'p1'
        option mtu '1508'
```

Use your own WAN port name. If it already has a device section, add the MTU there.

In the existing `wan` interface section:

```uci
        option mtu '1500'
```

OpenWrt's [PPPoE setup script](https://github.com/openwrt/openwrt/blob/openwrt-25.12/package/network/services/ppp/files/ppp.sh) passes this value to `pppd` as both the MTU and MRU.

For LAN IPv6, in the existing **device section named `br-lan`**:

```uci
        option mtu6 '1500'
```

After applying the settings, I refreshed the advertisements:

```sh
/etc/init.d/odhcpd restart
```

By default, [odhcpd](https://github.com/openwrt/odhcpd/blob/5d7be43f/src/config.c) takes the advertised MTU from the interface's IPv6 configuration.

The RB5009's internal DSA conduit, `eth0`, automatically increased to 1512.

Merge these changes into your existing configuration. The key for me was running `ifdown wan` before changing Ethernet settings, then explicitly bringing WAN back up afterwards. Rolling back also needed that last step; restoring the file alone didn't reconnect it.

## How I tested it

I checked negotiation, traffic from the router, and traffic forwarded from the PC. As a baseline, 1492-byte IPv4 packets passed before the changes. At 1500 with fragmentation prohibited, the router reported `Message too large`; tests from the PC got a fragmentation-required response.

On the successful connection, Vodafone's PADO and PADS replies contained `PPP-Max-Payload 0x05DC`. That's 1500 in decimal, confirming acceptance of the larger payload.

I checked the live interface values on OpenWrt with:

```sh
cat /sys/class/net/p1/mtu
cat /sys/class/net/pppoe-wan/mtu
cat /proc/sys/net/ipv6/conf/br-lan/mtu
```

These returned 1508, 1500 and 1500. On Windows, I also checked the IPv6 MTU:

```powershell
Get-NetIPInterface -InterfaceAlias 'Ethernet' -AddressFamily IPv6 |
    Select-Object InterfaceAlias, NlMtu
```

I temporarily installed `iputils-ping` and `tcpdump`. The bundled BusyBox ping lacked `-M do`, needed to prohibit IPv4 fragmentation.

For IPv4, 1472 bytes of data plus 28 bytes of IP and ICMP headers makes a 1500-byte packet. From the router:

```sh
ping -4 -I pppoe-wan -M do -s 1472 -c 100 -i 0.05 1.1.1.1
```

IPv6 needs 1452 bytes of data plus 48 bytes of headers. Use a current global IPv6 address assigned to the router from the ISP's delegated prefix:

```sh
ping -6 -I YOUR_ROUTER_GLOBAL_IPV6_ADDRESS -M do -s 1452 -c 100 -i 0.05 2001:4860:4860::8888
```

Then from Windows, replacing the placeholders with the PC's home Ethernet addresses:

```powershell
ping.exe -4 -S YOUR_LAN_IPV4_ADDRESS -f -l 1472 -n 5 1.1.1.1
ping.exe -6 -S YOUR_CURRENT_LAN_IPV6_ADDRESS -l 1452 -n 5 2001:4860:4860::8844
```

Binding the tests to the home connection and capturing traffic on the WAN confirmed it crossed the Vodafone link. The capture also caught an easy trap: Windows can fragment IPv6 pings itself, so a reply alone doesn't prove a full 1500-byte packet made it through. The final capture showed unfragmented requests and replies for both IP versions.

| Final verification | Result |
| --- | --- |
| 100 router-originated 1500-byte IPv4 ping exchanges | 100 replies, no loss |
| 100 router-originated 1500-byte IPv6 ping exchanges | 100 replies, no loss |
| Five PC-originated 1500-byte exchanges per IP version | All ten passed through the router |
| WAN capture of those 210 exchanges | 420 packets, all carrying unfragmented 1500-byte IP packets |
| HTTPS downloads from the PC over the home connection | 1,000,000 bytes for each IP version |
| DNS through AdGuard Home | Passed |

This worked on my line and equipment. Other Vodafone connections may behave differently, and some internet paths will still have a smaller MTU. If you try it, the checks above should help establish what your own line can do.
