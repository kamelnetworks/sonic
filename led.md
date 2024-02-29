# LEDs in SONiC

For the interface LEDs SONiC has two main approaches that we have seen:

 1) Use [ledd](https://github.com/Azure/sonic-platform-daemons/tree/master/sonic-ledd) to drive LEDs using platform plugins
 2) Use a LED processor inside the switching ASIC

For the platforms we have looked at, Trident 2 and Trident 3, all devices have been implemented using (2) and `ledd` (1) has been disabled.

## Broadcom LED driving

Broadcom LED driving is started by
[start_led.sh](https://github.com/Azure/sonic-buildimage/blob/master/platform/broadcom/docker-syncd-brcm/start_led.sh)
in the `syncd` container.

bcmsh commands for debugging LEDs:

  * `led status`

When the `led auto on` command is execute the `ledproc_linkscan_cb` built-in to the Broadcom SDK will take charge of
pushing new LED data to the LED processors. Depending on the custom LED code this may or may not be used, but can
be used to base LEDs on things like if the interface speed is 100G or not.

Something potentially interesting is that at starting at BRCM SAI 6.0.0.10 the `led auto on` registers `ledproc_linkscan_cb`
but calls it the "new" version "without port speed", while doing `led auto off` refers to the older version as having port
speed. This change is probably why issues like [this](https://github.com/sonic-net/sonic-buildimage/issues/10103) came to be.

```
drivshell>bsv
BRCM SAI ver: [8.4.0.2], OCP SAI ver: [1.11.0], SDK ver: [sdk-6.5.27] CANCUN ver: [06.04.01]
drivshell>led auto off
- Using old linkscan_cb with port speed
drivshell>led auto on
- Using new linkscan_cb without port speed
```

### Dumping current `led_control_data`

You can use the `cint` mode in `bcmsh` to verify the contents of the `led_control_data` bank.

```c
drivshell>cint
Entering C Interpreter. Type 'exit;' to quit.

cint> void dump_led_data() {
    uint8 data[1024];
    if (bcm_switch_led_control_data_read(0, 0, 0, data, sizeof(data)) != 0) {
        print "Failed to read!";
        return;
    }
    uint16 i;
    for (i = 0; i < sizeof(data); i++) {
        if (data[i] != 0) {
            printf("Offset %4d: 0x%02x\n", i, data[i]);
        }
    }
}
cint> dump_led_data();
Offset   32: 0x05
Offset   68: 0x05
Offset   98: 0x03
Offset   99: 0x03
```

In the above output we can see that the physical port 33, 69, 99, and 100 are up.

### Trident 2 / Tomahawk 2

This generation uses an unknown micro-controller architecture. Trident 2 has 2 controllers, Tomahawk 2 seems to have at least 4.

A typical LED programming looks like this:
```
modreg CMIC_LEDUP0_PORT_ORDER_REMAP_0_3    REMAP_PORT_0=0x3f  REMAP_PORT_1=0x3e  REMAP_PORT_2=0x3d  REMAP_PORT_3=0x3c   
modreg CMIC_LEDUP0_PORT_ORDER_REMAP_4_7    REMAP_PORT_4=0x3b  REMAP_PORT_5=0x3a  REMAP_PORT_6=0x39  REMAP_PORT_7=0x38   
[...]
led 0 stop
led 0 prog 86 FF 06 FF C2 7F 60 [..]
led 0 auto on
led 0 start
[..]
```

Example: On AS5712-54X it seems that two LED controllers are in use, LED 0 is in charge of the left-most port LEDs while LED 1 is in charge of the right-most.

### Trident 3 / Tomahawk 3

This generation uses a Cortex-M0 ARM CPU to drive a LED/linkscan program. This CPU is referred to as CMICX.
You can see how it is loaded by looking at `led_proc_init.soc` or sometimes `sai_preinit_cmd.soc`.

``` 
led auto off
led stop
m0 load 0 0x0 /usr/share/sonic/platform/linkscan_led_fw.bin
m0 load 0 0x3800 /usr/share/sonic/platform/custom_led.bin
led auto on
led start
```

These programs, `linkscan_led_fw.bin` and `custom_led.bin`, are checked in as
binary blobs so modifying them or porting a new platform is extremely time consuming.

The address `0x3800` is called `CUSTOM_HANDLER_ADDR`. You can find the source code for making your own custom LED code in the [OpenBCM](https://github.com/Broadcom-Network-Switching-Software/OpenBCM/tree/master/sdk-6.5.27/tools/led/cmicx) ([backup](https://github.com/bluecmd/OpenBCM/tree/master/sdk-6.5.27/tools/led/cmicx)) repository.

[SONIX](https://sonix.network/) publishes their custom LED code [at github](https://github.com/sonix-network/broadcom-leds).

## Innovium Teralynx 7 LED driving

The IVM77700 communicates directly with two CPLDs in order to drive LED status. What type of protocol it uses and how it determines the port split between them is unknown.
