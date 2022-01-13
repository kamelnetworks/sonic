# QSFP -> SFP adapters

Also known as **Q**SFP-to-**S**FP **A**dapter (QSA).

One can envision that electrically these adapters can be quite simple -
simply map data lane 1 to the SFP data lane, and map the I2C lanes to the SFP I2C lanes and you are done.
Indeed, this is one type of adapter - e.g. 10GTek's QSA-100A.

There is a problem with this approach though - the I2C protocol is different between QSFP and SFP!
The standard for SFP uses two I2C addresses (0x50 and 0x51) while the QSFP one only uses one (0x50).
Location and format of the basic transceiver information is thankfully the same between the two standards, but critically DOM/DDM differs.
That is why you will commonly find QSA stating that DOM/DDM information will not be available when using an adapter.
Thankfully, if the I2C bus is really passed through (and not routed to e.g. a QSA EEPROM) this becomes a software problem which we can solve.

Relevant standards:

 * SFF-8472 Management Interface for SFP+
 * SFF-8436 QSFP+ 4X 10 Gb/s Pluggable Transceiver

## SONiC

SONiC sort of uses an out-of-tree kernel driver called [`optoe`](https://github.com/opencomputeproject/oom/tree/master/optoe), but not really.
While in SONiC's Porting Guide it states that using optoe is highly recommended, SONiC does not itself use the optoe functionallity as
(of this writing) but rather relies on reading and writing to the relevant I2C bus by itself.

This means that in order to support a QSFP-to-SFP adapter in SONiC today one has to find the platform-dependent `SfpUtil` implementation
and change it to report the port as an SFP port instead of a QSFP port. This can be done by modifying the `qsfp_ports` function
and possibly `get_port_name` to keep the naming stable.

## optoe

There exists [a document](https://github.com/Azure/SONiC/blob/master/doc/sfp-refactor/sfp-refactor.md) introducing optoe
as the common way to read/write to transceivers. As of this writing it has not been scheduled in the SONiC roadmap however.

## EEPROM on SFP

Example EEPROM where 0x100 is mapped to I2C address 0x51.
```
00000000  0b 04 07 80 00 00 00 00  00 00 00 03 67 00 50 ff  |............g.P.|
00000010  00 00 00 00 50 72 6f 20  31 30 20 4f 70 74 69 78  |....Pro 10 Optix|
00000020  20 20 20 20 00 00 00 00  48 55 41 2d 53 46 50 2d  |    ....HUA-SFP-|
00000030  31 30 47 2d 44 57 44 4d  31 41 20 20 06 08 35 cc  |10G-DWDM1A  ..5.|
00000040  06 1a 00 00 49 4e 47 42  4c 30 35 31 30 31 30 32  |....INGBL0510102|
00000050  20 20 20 20 31 37 30 31  32 30 20 20 68 f0 05 2d  |    170120  h..-|
00000060  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00000100  4e 00 f8 00 4b 00 fb 00  90 88 71 48 8c a0 75 30  |N...K.....qH..u0|
00000110  e0 9c 13 88 d6 d8 27 10  9b 83 18 a6 62 1f 27 10  |......'.....b.'.|
00000120  0c 5a 00 19 07 cb 00 28  3a 00 1e 00 37 00 21 00  |.Z.....(:...7.!.|
00000130  ff ff ff ff ff ff ff ff  00 00 00 00 00 00 00 00  |................|
00000140  00 00 00 00 3f 80 00 00  00 00 00 00 01 00 00 00  |....?...........|
00000150  01 00 00 00 01 00 00 00  01 00 00 00 00 00 00 a2  |................|
00000160  1f 27 83 55 aa 2f 3b 78  01 19 00 00 2a b9 30 00  |.'.U./;x....*.0.|
00000170  00 00 00 00 00 00 03 00  00 0f 0c 00 00 00 00 00  |................|
00000180  ff ff ff ff ff ff ff ff  ff ff ff ff ff ff ff ff  |................|
*
00000200
```

## EEPROM on QSFP

Example EEPROM from a 100G QSFP module:

```
00000000  11 07 00 00 00 ff 00 00 00 00 00 00 00 00 00 00  |.....ÿ..........|
00000010  00 00 00 00 00 00 20 64 00 00 83 26 00 00 00 00  |...... d...&....|
00000020  00 00 2f 76 27 88 2c 74 27 42 4e 76 4f b7 47 30  |../v'.,t'BNvO·G0|
00000030  52 8e 1a f4 1a a4 1a 9a 17 48 83 26 81 bd 83 26  |R..ô.¤...H.&.½.&|
00000040  20 64 00 00 00 00 00 00 00 00 00 00 00 00 00 00  | d..............|
00000050  00 00 00 00 00 00 00 aa aa 00 00 00 00 00 00 00  |.......ªª.......|
00000060  00 00 ff 00 00 00 00 00 00 00 00 00 00 00 00 00  |..ÿ.............|
00000070  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  |................|
00000080  11 cc 07 80 00 00 00 00 00 00 00 03 ff 02 02 00  |.Ì..........ÿ...|
00000090  00 00 00 40 43 6f 6c 6f 72 43 68 69 70 20 6c 74  |...@ColorChip lt|
000000a0  64 20 20 20 14 e4 25 e9 43 31 30 30 51 53 46 50  |d   .ä%éC100QSFP|
000000b0  43 57 44 4d 34 30 30 42 30 30 66 58 05 14 37 74  |CWDM400B00fX..7t|
000000c0  06 0f ff fa 31 37 31 35 30 32 33 38 20 20 20 20  |..ÿú17150238    |
000000d0  20 20 20 20 31 37 30 34 30 34 20 20 0c 18 67 a4  |    170404  ..g¤|
000000e0  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  |................|
000000f0  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  |................|
```
