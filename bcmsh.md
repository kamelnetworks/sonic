You can enter the Broadcom diagnostic shell by entering `bcmsh`.

# Useful commands

 * `help`
 * `show unit`
   * Shows what ASIC you have
 * `show param`
   * A whole dump of useful information, specifically useful for figuring
     out what ports are available if you need to manual breakout for example.
 * `ps`
   * Shows all configured ports and their status
 * `phy diag xe0 dsc`
   * Shows low-level lane SerDes status, useful for debugging lane mapping
 * `pbmp xe0-xe5`
   * Create a **port bitmap** for given ports. This is used for other commands.
 * `port ${pbmp} enable=true`
   * Configure a port as enabled directly (instead of using SONiC commands)
 * `show counters changed same nz cpu0`
   * Shows all counters that are non-zero for cpu0
 * `bsv`
   * Show SDK and SAI version
 * `pciephy fw version`
   * Shows PCIe firmware version
 * `crm show resources all`
   * Show resource utilization for all critical resources 

You can tell SONiC to use debug level logging on things related to SAI
with the command `swssloglevel -l SAI_LOG_LEVEL_DEBUG -s -a`.
Make sure to restore it back to `SAI_LOG_LEVEL_NOTICE` afterwards.
Debug logging enables an unbelievable amount of logs, so you will likely
run into syslog rate limiting after more than a second of logs.

Terms:

 * ESW - **E**nterprise **SW**itching
 * DEFIP - IPv4 routing table

## BCM Knet

BCM Knet is used to provide interfaces to Linux for all the ports on the switch ASIC.
Since the ports are physically on the switch ASIC and not on the Linux CPU, it
does some fancy things to make it seem like we are operating on the actual ports.
It is important to note that we are not, some things like `tcpdump` will not work
out of the box since not all packets are sent up to the CPU by default.

To figure out what driver is managing an interface you can do the following:

```
$ ethtool --driver Ethernet92 
driver: bcm-knet
[..]
bluecmd@adele:/proc/bcm/knet$ cat debug 
Configuration:
  debug:          0x5020
  mac_addr:       02:10:18:e7:ab:57
  rx_buffer_size: 9238 (0x2416)
  rcpu_mode:      0
  rcpu_dmac:      00:00:00:00:00:00
  rcpu_smac:      00:00:00:00:00:00
  rcpu_ethertype: 0xde08
  rcpu_signature: 0x0
  rcpu_vlan:      1
  use_rx_skb:     1
  num_rx_prio:    1
  check_rcpu_sig: 0
  default_mtu:    9100
  rx_sync_retry:  1000
  use_napi:       0
[..]
```

You can enable quite heavy debugging. Here is an example how to enable
debugging logging with a full packet dump of all TX/RX traffic.
```
/proc/bcm/knet$ echo 'debug=0x10304' > debug 
```

You can see all levels [here](https://github.com/Broadcom-Switch/OpenNSL/blob/a3f85f5567f3142755265d32efb22cfe5afdb22e/sdk-6.5.12-gpl-modules/systems/linux/kernel/modules/bcm-knet/bcm-knet.c#L170-L190).

### MAC address on knet interfaces
The MAC used on the interface is whatever the MAC on the netlink interface is, it is fully controlled by the Linux kernel as usual. You can change it by e.g. "sudo ip link set addr 3c:2c:30:78:5b:80 dev Ethernet92". However this does not mean that the ASIC will route the new MAC to the CPU.

One dirty way of making the traffic reach the CPU is to put an ingress mirror to cpu0 on the port:
```
drivshell>dmirror xe92 mode=ingress destport=cpu0
```

This will of course not work for anything that should route traffic at high speed. What we want is the ip2me trap to match the new MAC. This can be done by changing the L3 interfaces like this:

```
drivshell>l3 intf show
Free L3INTF entries: 16378
Unit  Intf  VRF Group VLAN    Source Mac     MTU TTL Tunnel InnerVlan  NATRealm
------------------------------------------------------------------
0     1     0     0     1    3c:2c:30:78:5b:80  9100 0    0     0     0    
0     2     2     0     4095 3c:2c:30:78:5b:80  9100 0    0     0     0    
0     3     0     0     110  3c:2c:30:78:5b:80  9100 0    0     0     0    
0     4     1     0     4095 3c:2c:30:78:5b:80  9100 0    0     0     0    
drivshell>l3 intf destroy intf=2
drivshell>l3 intf add Vlan=4095 Mac=76:c0:f8:4b:c8:04 Intf=2 vrf=2 mtu=9100
drivshell>l3 intf show
Free L3INTF entries: 16378
Unit  Intf  VRF Group VLAN    Source Mac     MTU TTL Tunnel InnerVlan  NATRealm
------------------------------------------------------------------
0     1     0     0     1    3c:2c:30:78:5b:80  9100 0    0     0     0    
0     2     2     0     4095 76:c0:f8:4b:c8:04  9100 0    0     0     0    
0     3     0     0     110  3c:2c:30:78:5b:80  9100 0    0     0     0    
0     4     1     0     4095 3c:2c:30:78:5b:80  9100 0    0     0     0    

# It looks like bcmsh adds an L2 static entry, but it doesn't really work - so add another one to make things start moving
drivshell> l2 add mac=76:c0:f8:4b:c8:04 port=cpu0 Static=true ReplacePriority=false vlan=4095
```

This is /almost/ fully implemented by https://github.com/Azure/sonic-swss/pull/814/files.
You can set the MAC address and L3 interface table using the following stanza in /etc/sonic/config_db.json:

```json
    "INTERFACE": {
        "Ethernet92": {
            "vrf_name": "VrfKamel",
            "mac_addr": "76:c0:f8:4b:c8:04"
        },
        "Ethernet93": {
            "vrf_name": "VrfMainframe",
            "mac_addr": "76:c0:f8:4b:c8:05"
        }
    }
```

However the L2 table is still missing the static entries to allow ARPs.

Related issues:
 * [[VRF] Router interface SRC mac is not derived from VRF. #1833](https://github.com/Azure/sonic-swss/issues/1833)
 * [When setting MAC address on L3 interfaces, the L2 table is not updated #1847](https://github.com/Azure/sonic-swss/issues/1847)

For now I use the above stanza and this script:

```bash
#!/bin/bash

set -euo pipefail

for mac in $(bcmcmd 'l3 intf show' | awk '/:[0-9a-f][0-9a-f]:[0-9a-f][0-9a-f]:/ {print $6}' | sort | uniq)
do
	if ! bcmcmd 'l2 show' | grep CPU | grep -q "${mac}"; then
		echo "Adding missing MAC: ${mac}"
		bcmcmd "l2 add mac=${mac} port=cpu0 Static=true ReplacePriority=false"
	fi
done
```

## Monitor port using tcpdump

We have to tell the switch chip to send all incoming traffic to the CPU

```
drivshell> dmirror xe0 mode=ingress destport=cpu0
```

Now we can use `tcpdump` as usual on the interface. Beware, since we
are now sending the packets to the CPU Linux will of course start processing
them. So by enabling this hack you might change the behavior of the thing
you are trying to debug.

## L3 routing behavior

The routing tables for IPv4 and IPv6 can be examined using
`l3 defip show` or `l3 ip6route show` respectively.

```
drivshell> l3 defip show
Unit 0, Total Number of DEFIP entries: 393216
#     VRF     Net addr             Next Hop Mac        INTF MODID PORT PRIO CLASS HIT VLAN
16    0        172.18.0.0/24        00:00:00:00:00:00 100003    0     0     0    2 n
48    1        192.168.216.0/24     00:00:00:00:00:00 100003    0     0     0    2 n
80    2        192.168.216.0/24     00:00:00:00:00:00 100003    0     0     0    0 n
112   3        192.168.216.0/24     00:00:00:00:00:00 100003    0     0     0    2 n
112   3        193.228.143.236/31   00:00:00:00:00:00 100003    0     0     0    2 n
1     0        0.0.0.0/0            00:00:00:00:00:00 100002    0     0     0    0 n
3     1        0.0.0.0/0            00:00:00:00:00:00 100002    0     0     0    0 n
5     2        0.0.0.0/0            00:00:00:00:00:00 100002    0     0     0    0 n
7     3        0.0.0.0/0            00:00:00:00:00:00 100005    0     0     0    0 n
```

The columns should be pretty self-explanatory except the INTerFace column.
It referens an egress interface. To list those use `l3 egress show`.

```
drivshell> l3 egress show
Entry  Mac                 Vlan INTF PORT MOD MPLS_LABEL ToCpu Drop RefCount L3MC
100002  3c:2c:30:78:5b:80    1    1     0    0        -1   no  yes    7   no
100003  3c:2c:30:78:5b:80    0 16383    0    0        -1   no   no   21   no
100004  ff:ff:ff:ff:ff:ff    0    6   130    0        -1   no   no    0  yes
100005  00:09:0f:09:bc:07  511    7   130    0        -1   no   no    2   no
100006  00:50:56:97:f3:6d 4095    4    95    0        -1   no   no    1   no
100007  00:50:56:97:f3:6d 4095    5    96    0        -1   no   no    1   no
100008  00:09:0f:09:d4:01 4095    4    95    0        -1   no   no    1   no
100009  00:09:0f:09:d4:01 4095    5    96    0        -1   no   no    1   no
100010  00:09:0f:09:bc:07  511    7   130    0        -1   no   no    1   no
100011  76:c0:f8:4b:c8:5d 4095    4    95    0        -1   no   no    1   no
100012  76:c0:f8:4b:c8:5c 4095    5    96    0        -1   no   no    1   no
```

An important note here is that `100003` with its INTF of `16383` means that the
traffic will be sent to the CPU. That is likely not what you want unless you
want to route some network running onboard the management CPU on the switch.
If you are doing this you have probably made some questionable design choices.

To manually add an IPv6 route you can use `l3 ip6route add` like this:

```
drivshell> l3 ip6route add vrf=3 IP=2a10:11c0:: masklen=32 mac=00:09:0f:09:bc:07 intf=100008
```

## Live changing configuration

It is possible to mess around with the chip configuration live, as long as one is prepared
to have a really mad SONiC (until reboot).

Example 1:

```
drivshell>config portmap_69=77:40
drivshell>init all
Port 69 bandwidth 40 Gb and port 70 can not be both configured
Port 69 bandwidth 40 Gb and port 71 can not be both configured
Port 69 bandwidth 40 Gb and port 72 can not be both configured
PGW_CL4 and PGW_CL5 total line rate bandwidth (230 Gb) exceeds 200 Gb
0:soc_info_config: Port config error !!
0:system_init: system_init: Device reset failed: Invalid configuration
```

Example 2:

```
drivshell>config portmap_70=78:10:i
drivshell>config portmap_71=79:10:i
drivshell>config portmap_72=80:10:i
drivshell>init all
init all
0:soc_egress_drain_cells: MacDrainTimeOut:port 0,xe46, timeout draining packets
0:soc_egress_drain_cells: MacDrainTimeOut:port 0,xe47, timeout draining packets
0:soc_egress_drain_cells: MacDrainTimeOut:port 0,xe46, timeout draining packets
0:soc_egress_drain_cells: MacDrainTimeOut:port 0,xe47, timeout draining packets
0:bcmi_xgs5_bfd_init: uKernel BFD application not available
```

Coupled with online lane changing, it should be possible to do a lot of debugging
regarding breakout and similar issues:

```
port xe60-xe63 en=0
port xe60 lanes 4
port xe60 speed=40000
port xe60-xe63 en=1
```
