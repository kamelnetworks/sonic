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

