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
| Set interface description | `sonic-db-cli CONFIG_DB hset "PORT|Ethernet94" description "some fancy description"` |
| Reset configuration | `sonic-cfggen -H -k "$(sonic-cfggen -d -v 'DEVICE_METADATA.localhost.hwsku')" --print-data > /etc/sonic/config_db.json` |
| Reset configuration w/ new SKU | `sonic-cfggen -H -k "MySKU" --print-data > /etc/sonic/config_db.json; sed -i 's/^HWSKU=.*/HWSKU=MySKU/' /etc/sonic/sonic-environment` |
| Hardware reset SFP module | `sudo sfputil reset Ethernet0`
| Save configuration before rebooting to new image | `sudo config save -y /host/old_config/config_db.json` 

## Enable REST and gNMI API authentication

By default the REST and gNMI API is unauthenticated (!!) so you want to enable some sort of authentication.

Today, due to some weirdness in handling `nil` values you can load the following configuration blob:
```json
{
    "REST_SERVER": {
        "default": {
            "client_auth": "user",
            "server_crt": "",
            "server_key": "",
            "ca_crt": "",
            "port": "",
            "log_level": ""
        }
    },
    "TELEMETRY": {
        "gnmi": {
            "client_auth": "True",
            "port": "8080",
            "log_level": "2"
        }
    }
}
```

This loads a configuration that enables user authentication that seems to be
PAM based - but it has not been verified.

Hopefully in the future this can be a `config` command...

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

## Reading DOM / DDM data for transceivers

The easiest way to do this is by using e.g. `sfputil show eeprom -d -p Ethernet12`.

Output looks like this:

```
Ethernet12: SFP EEPROM detected
        Connector: LC
        Encoding: 64B66B
        Extended Identifier: Power Class 4(3.5W max), CDR present in Rx Tx
        Extended RateSelect Compliance: Unknown
        Identifier: QSFP28 or later
        Length Cable Assembly(m): 0
        Length OM1(m): 0
        Length OM2(m): 0
        Length OM3(2m): 0
        Length(km): 2
        Nominal Bit Rate(100Mbs): 255
        Specification compliance:
        Vendor Date Code(YYYY-MM-DD Lot): 2020-08-22 
        Vendor Name: AOI
        Vendor OUI: 00-29-26
        Vendor PN: AQPLBCQ4EDMA1072
        Vendor Rev: A
        Vendor SN: 75520H10771
        ChannelMonitorValues:
                RX1Power: -17.6195dBm
                RX2Power: -17.1897dBm
                RX3Power: -16.8825dBm
                RX4Power: -infdBm
                TX1Bias: 38.7840mA
                TX2Bias: 38.7840mA
                TX3Bias: 38.7840mA
                TX4Bias: 38.7840mA
        ModuleMonitorValues:
                Temperature: 34.3438C
                Vcc: 3.3938Volts
```

## Manually changing `hwsku`

If you do change this in `config_db.json`, make sure to look at `/etc/sonic/sonic-environment` and ensure it updates or
the syncd process might pick up the old configuration.

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

## Recovering SONiC bootloader from ONIE Rescue

If you for example do a BIOS reset and the UEFI boot entries are dropped, you can restore it like this:

```
$ efibootmgr -c -L SONiC-OS -l '\EFI\SONiC-OS\grubx64.efi'
```

## SONiC and LLDP

LLDP in SONiC is transmitted on the knet interfaces and thus completely untagged, regardless of VLAN configuration.
This is working as intended, LLDP is commonly transmitted either on VLAN 1 or untagged.

## ERSPAN a port

You can mirror a port (in this case `Ethernet96`) for debugging purposes like this:
```
# This has to be in the default VRF, switch is 172.18.0.250 and ERSPAN receiver is at 172.18.0.251 
sudo config mirror_session erspan add test-erspan 172.18.0.250 172.18.0.251 10 10 35006 0 Ethernet96 both
```

See `sudo config mirror_session erspan add --help` for details what the options are.
Also see the [ACL](acl.md) page for some details about ERSPAN.

**Note:** The current CLI does not support v6 addresses as the source/destination, and SWSS [rejects IPv6 addresses](https://github.com/sonic-net/sonic-swss/blob/1221eae41788176b5de0db3e869006ff22d2d4ff/orchagent/mirrororch.cpp#L396-L413). Unknown reason why, SAI seems to support it from reading the docs.

This is an example of a mirror session:

```json
{
    "MIRROR_SESSION": {
        "blackbox.sonix.int": {
            "direction": "RX",
            "dscp": "10",
            "dst_ip": "100.100.99.1",
            "gre_type": "35006",
            "queue": "0",
            "src_ip": "100.100.99.4",
            "src_port": "Ethernet192",
            "ttl": "10",
            "type": "ERSPAN"
        }
    }
}
```

## LACP / PortChannel

You can find a example in [SONiC Command Line Interface Guide](https://github.com/sonic-net/sonic-utilities/blob/master/doc/Command-Reference.md#portchannels). If you're using this to connect to other switches and routers, make sure to set fallback to false (so that you don't make a L2 loop).

If the PortChannel isn't showing up correcty, check if teamd feature is enabled and running:
```
(vrf:mgmt)root@sonic:~# show feature status teamd
Feature    State    AutoRestart
---------  -------  -------------
teamd      enabled  enabled
```
