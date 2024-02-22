You can enter the Innovium diagnostic shell by entering `ivmshell`.

# Useful commands

 * `ifcs show version`
   * Show SDK and library versions.
 * `ifcs show devport`
   * Shows all configured physical ports, alternative command `port info`.
 * `ifcs set devport 217 admin_state 0|1`
   * Set a port's admin state to down/up respectively.
 * `ifcs show l2_entry`
   * Shows learned L2 forwarding information.
 * `ifcs show route info`
   * Show L3 routing table.
 * `ifcs show nexthop`
   * Show possible nexthops, things like flooding groups and unicast hosts are here.
 * `ifcs show rate devport filter nz`
   * Show how many Gbps are currently running over physical ports.
 * `diagtest serdes aapl 89 4 "aapl serdes -display"`
   * Show the SerDes statistics on devport 89, lane 4 (i.e. lane 93). See below for details how to read this.

# SerDes information

Example output:

```
########## SerDes state dump for SBus address :1a  ##############################
sd16C_txcpam4_rxcm4_ns_09 LSB rev 14. Firmware: 0x10A5_208D_003.

   Addr TX Width Pam Inv Gray Pcode Swzl  CF Rate Div  Gbps     Data Out Pre3     Pre2     Pre1    Atten     Post Vert Amp Slew T2
-------- -- ----- --- --- ---- ----- ---- --- ---- --- ----- -------- --- ---- -------- -------- -------- -------- ---- --- ---- --
    :1a  1    40   2  ON  OFF   OFF  OFF  40  2.0  66 10.29     CORE   1    0        0        0        0        0                   

   Addr RX Width Pam Inv Gray Pcode Swzl  CF Rate Div  Gbps      Data     Qual     Cmp_Mode  EI OK LK LB o_core  Term Phase E     Errors        BER
-------- -- ----- --- --- ---- ----- ---- --- ---- --- ----- --------- -------- ------------ --- -- -- -- ------ ----- ----- - ---------- ----------
    :1a  1    40   2 OFF  OFF   OFF  OFF  42  2.0  66 11.19       OFF   unqual          XOR Dis 11  1  0 0x0030 FLOAT     0 1          -          -  

TX PLL gains: bbGAIN=25, intGAIN= 7
RX PLL gains: bbGAIN= 2, intGAIN= 7
RX: EI threshold = 6

   Addr         SPICO             TX PLL PCS FIFO Data Edge Test  DFE  CDC
-------- ------------- ------------------ -------- ---- ---- ---- ---- -----
    :1a        REFCLK             REFCLK F66      iclk qclk iclk iclk  DATA

             CTLE         |       RxFFE         |                  VOS                  |            VERNIER             |                      DFE                      |           Eye Height          | 1 /    |State/
 Addr|DC LF HF BW G1 G2 SC|  1  2  3  4  5  6  7|         DATA          | TEST  |  CDR  |UO UE MO ME LO LE TO TE EO EE TP|G1 G2     1  2  3  4  5  6  7  8  9  A  B  C  D|                               | dwell  |Status
-----+--------------------+---------------------+-----------------------+-------+-------+--------------------------------+-----------------------------------------------+-------------------------------+--------+------
  :1a|00 0a 04 04 00 00 00|  0  0  0  0  0  0  0|  0   0   0   0   0   0| ff  9f|  0   0| 0  0  0  0  0  0  0  0  0  0  0|ff ff        3  3  2 -1  1  0  0  0  0  0      |   fc   fc   fc   fc   fc   fc |   1e-06|08 e2
```
