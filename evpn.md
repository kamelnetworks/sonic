TODO: Clean this up

This shows an example how to set up an EVPN L2VPN between SONiC 202012 with a Trident 3 switch
and Arista EOS running on a Trident 2 switch.

We are using the new improved VRF-aware FRR managed configuration by enabling
`frr_mgmt_framework_config` in the device metadata.

## SONiC configuration

```json
{
    "BGP_GLOBALS": {
        "default": {
            "local_asn": "65001",
            "log_nbr_state_changes": "true",
            "router_id": "10.0.0.11"
        }
    },
    "BGP_GLOBALS_AF": {
        "l2vpn_evpn": {
            "advertise-all-vni": "true"
        },
        "default|l2vpn_evpn": {
            "advertise-all-vni": "true"
        }
    },
    "BGP_GLOBALS_EVPN_VNI": {
        "default|l2vpn_evpn|1991": {
            "export-rts": [
                "10.0.0.255:1991"
            ],
            "import-rts": [
                "10.0.0.255:1991"
            ],
            "router-distinguisher": "10.0.0.11:1991"
        }
    },
    "BGP_NEIGHBOR": {
        "default|10.0.0.12": {
            "asn": "65002",
            "ebgp_multihop": "true",
            "local_addr": "10.0.0.11",
            "name": "overlay to arista"
        },
        "default|10.99.0.2": {
            "asn": "65002",
            "local_addr": "10.99.0.3",
            "name": "underlay to arista"
        }
    },
    "BGP_NEIGHBOR_AF": {
        "default|10.0.0.12|l2vpn_evpn": {
            "admin_status": "true",
            "advertise-all-vni": "true",
            "route_map_in": [
                "ALLOW"
            ],
            "route_map_out": [
                "ALLOW"
            ],
            "unchanged_nexthop": "true"
        },
        "default|10.99.0.2|ipv4_unicast": {
            "admin_status": "true",
            "route_map_in": [
                "ALLOW"
            ],
            "route_map_out": [
                "ALLOW"
            ]
        }
    },
    "DEVICE_METADATA": {
        "localhost": {
            "frr_mgmt_framework_config": "true"
        }
    },
    "INTERFACE": {
        "Ethernet128": {},
        "Ethernet128|10.99.0.3/31": {}
    },
    "LOOPBACK_INTERFACE": {
        "Loopback0": {},
        "Loopback0|10.0.0.11/32": {}
    },
    "ROUTE_MAP": {
        "ALLOW|1": {
            "route_operation": "permit"
        }
    },
    "ROUTE_REDISTRIBUTE": {
        "default|connected|bgp|ipv4": {}
    },
    "VLAN": {
        "Vlan1991": {
            "vlanid": "1991"
        }
    },
    "VLAN_MEMBER": {
        "Vlan1991|Ethernet112": {
            "tagging_mode": "untagged"
        },
        "Vlan1991|Ethernet124": {
            "tagging_mode": "untagged"
        }
    },
    "VXLAN_EVPN_NVO": {
        "nvo1": {
            "source_vtep": "nve1"
        }
    },
    "VXLAN_TUNNEL": {
        "nve1": {
            "src_ip": "10.0.0.11"
        }
    },
    "VXLAN_TUNNEL_MAP": {
        "nve1|map_1991_Vlan1991": {
            "vlan": "Vlan1991",
            "vni": "1991"
        }
    }
}
```

### BCM modification

You need to make sure your platform's Broadcom configuration file (`.bcm`) has enabled VXLAN.
For Trident 3 these are the settings needed:

```
use_all_splithorizon_groups=1
riot_enable=1
sai_tunnel_support=1
riot_overlay_l3_intf_mem_size=4096
riot_overlay_l3_egress_mem_size=32768
riot_overlay_ecmp_resilient_hash_size=16384
flow_init_mode=1
```

If you do not configure this correctly you will end up having error logs like these:
```
syncd#syncd: [none] SAI_API_TUNNEL:_brcm_sai_vxlan_create_vpn:872 create tunnel initiator setup for net port failed with error Invalid parameter (0xfffffffc).
[..]
syncd#syncd: [none] SAI_API_TUNNEL:_brcm_sai_vxlan_create_vxlan_vpn:77 vxlan vpn create failed with error Entry exists (0xfffffff8).
```

## Arista EOS configuration

```
service routing protocols model multi-agent
!
vlan 1991
!
interface Recirc-Channel627
   no switchport
   switchport recirculation features vxlan
!
interface Ethernet1/1
   mtu 9100
   no switchport
   ip address 10.99.0.2/31
!
interface Ethernet24/1
   switchport access vlan 1991
   switchport
!
interface Ethernet24/2
   switchport access vlan 1991
   switchport
!
interface Ethernet24/3
   switchport access vlan 1991
   switchport
!
interface Ethernet24/4
   switchport access vlan 1991
   switchport
!
interface Ethernet32
   traffic-loopback source system device mac
   switchport
   channel-group recirculation 627
!
interface Loopback0
   ip address 10.0.0.12/32
!
interface Management1
   vrf management
   ip address 172.18.0.6/24
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 1991 vni 1991
!
router bgp 65002
   router-id 10.0.0.12
   no bgp default ipv4-unicast
   neighbor 10.0.0.11 remote-as 65001
   neighbor 10.0.0.11 next-hop-unchanged
   neighbor 10.0.0.11 update-source Loopback0
   neighbor 10.0.0.11 ebgp-multihop 4
   neighbor 10.0.0.11 send-community extended
   neighbor 10.0.0.11 maximum-routes 12000
   neighbor 10.99.0.3 remote-as 65001
   neighbor 10.99.0.3 maximum-routes 12000
   redistribute connected
   !
   vlan 1991
      rd 10.0.0.12:1991
      route-target import 10.0.0.255:1991
      route-target export 10.0.0.255:1991
      redistribute learned
   !
   address-family evpn
      neighbor 10.0.0.11 activate
      neighbor 10.0.0.11 next-hop-unchanged
   !
   address-family ipv4
      neighbor 10.99.0.3 activate
!
```

