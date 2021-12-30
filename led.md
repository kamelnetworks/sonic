# LEDs in SONiC

For the interface LEDs SONiC has two main approaches that we have seen:

 1) Use [ledd](https://github.com/Azure/sonic-platform-daemons/tree/master/sonic-ledd) to drive LEDs using platform plugins
 2) Use a LED processor inside the switching ASIC

For the platforms we have looked at, Trident 2 and Trident 3, all devices have been implemented using (2) and `ledd` (1) has been disabled.

## Broadcom LED driving

Broadcom LED driving is started by
[start_led.sh](https://github.com/Azure/sonic-buildimage/blob/master/platform/broadcom/docker-syncd-brcm/start_led.sh)
in the `syncd` container.

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

This generation uses a Cortex-M0 ARM CPU to drive a LED/linkscan program.
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
