Random good-to-know things about SONiC that we wish were better documented

## Terms

| Term       | What is it? |
|------------|-----------------------------|
| Click CLI  | The `sudo config ....` CLI  |
| KLISH CLI  | The `sonic-cli` CLI         |
| CONFIG_DB  | The Redis database number 4 |


## Random commands

| Description       | Command                     |
|-------------------|-----------------------------|
| Enter KLISH       | `sonic-cli` |
| Set interface description | `sonic-db-cli CONFIG_DB hset "PORT|Ethernet94" description "some fancy description"`
| Enable REST API authentication | `sonic-db-cli CONFIG_DB hset "REST_SERVER|default" client_auth password` |

## Changing the eth0 MAC address

If you are running `eth0` in the management VRF you might want to change the MAC address to avoid
traffic bouncing around and messing with your management connectivity.

SONiC configuration does not currently include a way to change the MAC address on this special interface,
but you can still do it using Debian's interface files:

```
# /etc/network/interfaces.d/eth0-mac 
auto eth0
iface eth0 inet manual
  pre-up /sbin/ip link set dev eth0 addr ee:6e:08:ef:99:39
```

## `frr_mgmt_framework_config`

SONiC has [two](https://github.com/Azure/sonic-buildimage/blob/202012/dockers/docker-fpm-frr/frr/bgpd/gen_bgpd.conf.j2)
major modes of configuring the BGP daemons:

 * The default, older, way that lacks good VRF support
 * The opt-in, newer, way that has fun things like VRFs and EVPN.

You have to set `frr_mgmt_framework_config` in order for
SONiC to switch to using the new templates.
You can set it e.g. by doing:
```
$ sonic-db-cli CONFIG_DB hmset "DEVICE_METADATA|localhost" frr_mgmt_framework_config true
```

When you have set that setting you need to restart BGP (`systemctl restart bgp`).

As of this writing the schema is documented basically at all, so you will have to look at
these two places:

 * The [templates](https://github.com/Azure/sonic-buildimage/tree/202012/src/sonic-frr-mgmt-framework/templates/bgpd). 
   These are used for cold-starting FRR (like with `systemctl restart bgp`).
 * The [hot code-path](https://github.com/Azure/sonic-buildimage/blob/202012/src/sonic-frr-mgmt-framework/frrcfgd/frrcfgd.py).
   This code is used to reconfigure FRR when running by detecting modifications in the `CONFIG_DB`.
   
An example of two BGP speakers on the same switch using two VRFs:

```json
{
	"BGP_GLOBALS": {
		"VrfKamel": {
			"local_asn": 213113
		},
		"VrfMainframe": {
			"local_asn": 206858
		}
	},
	"BGP_NEIGHBOR": {
		"VrfKamel|2a0a:d984::20:6858:1": {
			"asn": "206858",
			"local_addr": "2a0a:d984::21:3113:1",
			"name": "MAINFRAME-NET"
		},
		"VrfMainframe|2a0a:d984::21:3113:1": {
			"asn": "213113",
			"local_addr": "2a0a:d984::20:6858:1",
			"name": "KAMEL-NETWORKS"
		}
	},
	"BGP_NEIGHBOR_AF": {
		"VrfKamel|2a0a:d984::20:6858:1|ipv6_unicast": {
			"admin_status": "true",
			"route_map_in": ["ALLOW"],
			"route_map_out": ["ALLOW"]
		},
		"VrfMainframe|2a0a:d984::21:3113:1|ipv6_unicast": {
			"admin_status": "true",
			"route_map_in": ["ALLOW"],
			"route_map_out": ["ALLOW"]
		}
	},
	"ROUTE_MAP": {
		"ALLOW|1": {
			"route_operation": "permit"
		}
	}
}
```

## Redis databases

The main data store in SONiC is Redis. It uses multiple databases to partition it logically.

The numbers of these are as of this writing:

```
#define APPL_DB             0
#define ASIC_DB             1
#define COUNTERS_DB         2
#define LOGLEVEL_DB         3
#define CONFIG_DB           4
#define PFC_WD_DB           5
#define FLEX_COUNTER_DB     5
#define STATE_DB            6
#define SNMP_OVERLAY_DB     7
#define RESTAPI_DB          8
#define GB_ASIC_DB          9
#define GB_COUNTERS_DB      10
#define GB_FLEX_COUNTER_DB  11
#define CHASSIS_APP_DB      12
#define CHASSIS_STATE_DB    13
```
