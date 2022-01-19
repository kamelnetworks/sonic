
# ACLs and Mirroring


## ERSPAN with ACLs

First note: **It is important that the switch has the tunnel destination as an ARP entry.**

If it does not, the tunnel will not become active in `show mirror_session`. No error message
will be given, but things will simply not work.

ERSPAN can be configured like this:

```shell
#                                                     - SRC -       - DST -   (1)(2)  (3)
# (1) DSCP value
# (2) TTL value
# (3) GRE type, here we use Type-II 0x88BE = 35006
$ sudo config mirror_session erspan add MyERSPAN 172.18.0.250 172.18.0.251 10 10 35006
```

You can then assign ACLs to be sent to the ERSPAN destination like this:

```json
{
    "ACL_TABLE": {
        "ACL-IPV4-ERSPAN": {
            "policy_desc": "Monitor selected traffic",
            "ports": [
                "Ethernet96"
            ],
            "type": "MIRROR"
        },
        "ACL-IPV6-ERSPAN": {
            "policy_desc": "Monitor selected traffic",
            "ports": [
                "Ethernet96"
            ],
            "type": "MIRRORV6"
        }
    }, 
    "ACL_RULE": {
        "ACL-IPV4-ERSPAN|MIRROR-DST-BGP": {
            "MIRROR_ACTION": "MyERSPAN",
            "SRC_IP": "185.1.215.0/24",
            "DST_IP": "185.1.215.0/24",
            "L4_DST_PORT": "179"
        },
        "ACL-IPV4-ERSPAN|MIRROR-SRC-BGP": {
            "MIRROR_ACTION": "MyERSPAN",
            "SRC_IP": "185.1.215.0/24",
            "DST_IP": "185.1.215.0/24",
            "L4_SRC_PORT": "179"
        },
        "ACL-IPV6-ERSPAN|MIRRORV6-DST-BGP": {
            "MIRROR_ACTION": "MyERSPAN",
            "SRC_IPV6": "2001:7f8:117::/64",
            "DST_IPV6": "2001:7f8:117::/64",
            "L4_DST_PORT": "179"
        },
        "ACL-IPV6-ERSPAN|MIRRORV6-SRC-BGP": {
            "MIRROR_ACTION": "MyERSPAN",
            "SRC_IPV6": "2001:7f8:117::/64",
            "DST_IPV6": "2001:7f8:117::/64",
            "L4_SRC_PORT": "179"
        }
    }
}
```

**Note:** ACL tables seem to count as a critical resource but on some platforms (like Broadcom, not Mellanox)
both IPv6 and IPv4 mirroring tables are combined. You can see this by a log message like this:

```
addAclTable: Created ACL table ACL-IPV6-ERSPAN-BGP as a sibling of ACL-IPV4-ERSPAN-BGP
```

**Important:** Since the v6 and v4 tables may be merged you have to ensure that the name of the rules
are unique between the two IP versions - e.g. by prefixing "MIRRORV6" on the v6 rules. Otherwise the
v6 rules will silently overwrite the v4 rules as of this writing (SONiC 202012).

Mirroring on egress seems to not be supported on Broadcom and will fail with a SAI error in the logs.
