# Open5GS EPC & srsRAN 4G with ZeroMQ UE / RAN Sample Configuration - UPG-VPP(DPDK/VPP UPF(PGW-U))
This describes a simple configuration for working Open5GS EPC and UPG-VPP.
In particular, see [here](https://github.com/s5uishida/install_vpp_upf_dpdk) for UPG-VPP configuration.

---

### [Sample Configurations and Miscellaneous for Mobile Network](https://github.com/s5uishida/sample_config_misc_for_mobile_network)

---

<a id="toc"></a>

## Table of Contents

- [Overview of Open5GS CUPS-enabled EPC Simulation Mobile Network](#overview)
- [Changes in configuration files of Open5GS EPC, UPG-VPP and srsRAN 4G ZMQ UE / RAN](#changes)
  - [Changes in configuration files of Open5GS EPC C-Plane](#changes_cp)
  - [Changes in configuration files of Open5GS EPC U-Plane](#changes_up)
  - [Changes in configuration files of UPG-VPP](#changes_vpp)
  - [Changes in configuration files of srsRAN 4G ZMQ UE / RAN](#changes_srs)
    - [Changes in configuration files of RAN](#changes_ran)
    - [Changes in configuration files of UE](#changes_ue)
- [Network settings of Open5GS EPC, UPG-VPP and srsRAN 4G ZMQ UE / RAN](#network_settings)
  - [Network settings of UPG-VPP and Data Network Gateway](#network_settings_up)
- [Build Open5GS, UPG-VPP and srsRAN 4G ZMQ UE / RAN](#build)
- [Run Open5GS EPC, UPG-VPP and srsRAN 4G ZMQ UE / RAN](#run)
  - [Run UPG-VPP](#run_vpp)
  - [Run Open5GS EPC U-Plane](#run_up)
  - [Run Open5GS EPC C-Plane](#run_cp)
  - [Run srsRAN 4G ZMQ RAN](#run_ran)
  - [Run srsRAN 4G ZMQ UE](#run_ue)
- [Ping google.com](#ping)
  - [Case for going through PDN 10.45.0.0/16](#ping_1)
- [Changelog (summary)](#changelog)
---

<a id="overview"></a>

## Overview of Open5GS CUPS-enabled EPC Simulation Mobile Network

This describes a simple configuration of C-Plane, UPG-VPP and Data Network Gateway for Open5GS EPC.
**Note that this configuration is implemented with Proxmox VE VMs.**

The following minimum configuration was set as a condition.
- One SGW-U/UPF(PGW-U) and Data Network Gateway
- One UE and one APN

The built simulation environment is as follows.

<img src="./images/network-overview.png" title="./images/network-overview.png" width=1000px></img>

The EPC / DPDK/VPP PGW-U / UE / RAN used are as follows.
- EPC - Open5GS v2.7.5 (2025.04.25) - https://github.com/open5gs/open5gs
- DPDK/VPP PGW-U - UPG-VPP v1.13.0 (2024.03.25) - https://github.com/travelping/upg-vpp
- UE / RAN - srsRAN 4G (2024.02.01) - https://github.com/srsran/srsRAN_4G

Each VMs are as follows.  
| VM | SW & Role | IP address | OS | CPU<br>(Min) | Mem<br>(Min) | HDD<br>(Min) |
| --- | --- | --- | --- | --- | --- | --- |
| VM1 | Open5GS EPC C-Plane | 192.168.0.111/24 | Ubuntu 24.04 | 1 | 2GB | 20GB |
| VM2 | Open5GS EPC U-Plane (SGW-U) | 192.168.0.112/24 | Ubuntu 24.04 | 1 | 1GB | 20GB |
| VM-UP | UPG-VPP U-Plane (PGW-U) | 192.168.0.151/24 | Ubuntu **22.04** | 2 | 8GB | 20GB |
| VM-DN | Data Network Gateway  | 192.168.0.152/24 | Ubuntu 24.04 | 1 | 1GB | 10GB |
| VM3 | srsRAN 4G ZMQ RAN (eNodeB) | 192.168.0.121/24 | Ubuntu 22.04 | 1 | 2GB | 10GB |
| VM4 | srsRAN 4G ZMQ UE | 192.168.0.122/24 | Ubuntu 22.04 | 1 | 2GB | 10GB |

The network interfaces of each VM are as follows.
**Note. Do not enable(up) any devices that will be under the control of DPDK.
These devices will be enabled and set IP addresses in the `init.conf` file of UPG-VPP.**
| VM | Device | Model | Linux Bridge | IP address | Interface | Under DPDK |
| --- | --- | --- | --- | --- | --- | --- |
| VM1 | ens18 | VirtIO | vmbr1 | 10.0.0.111/24 | (NAPT NW) | -- |
| | ens19 | VirtIO | mgbr0 | 192.168.0.111/24 | (Mgmt NW) | -- |
| | ens20 | VirtIO | vmbr4 | 192.168.14.111/24 | Sxb (N4 for 5GC) | -- |
| VM2 | ens18 | VirtIO | vmbr1 | 10.0.0.112/24 | (NAPT NW) | -- |
| | ens19 | VirtIO | mgbr0 | 192.168.0.112/24 | (Mgmt NW) | -- |
| | ens20 | VirtIO | vmbr3 | 192.168.13.112/24 | S1-U,S5u (N3 for 5GC) | -- |
| VM-UP | ~~ens18~~ | ~~VirtIO~~ | ~~vmbr1~~ | ~~10.0.0.151/24~~ | ~~(NAPT NW)~~ ***down*** | -- |
| | ens19 | VirtIO | mgbr0 | 192.168.0.151/24 | (Mgmt NW) | -- |
| | ens20 | VirtIO | vmbr3 | 192.168.13.151/24 | S5u (N3 for 5GC) | x |
| | ens21 | VirtIO | vmbr4 | 192.168.14.151/24 | Sxb (N4 for 5GC) | x |
| | ens22 | VirtIO | vmbr6 | 192.168.16.151/24 | SGi (N6 for 5GC) | x |
| VM-DN | ens18 | VirtIO | vmbr1 | 10.0.0.152/24 | (NAPT NW) | -- |
| | ens19 | VirtIO | mgbr0 | 192.168.0.152/24 | (Mgmt NW) | -- |
| | ens20 | VirtIO | vmbr6 | 192.168.16.152/24 | SGi (N6 for 5GC) | -- |
| VM3 | ens18 | VirtIO | vmbr1 | 10.0.0.121/24 | (NAPT NW) | -- |
| | ens19 | VirtIO | mgbr0 | 192.168.0.121/24 | (Mgmt NW) | -- |
| | ens20 | VirtIO | vmbr3 | 192.168.13.121/24 | S1-U (N3 for 5GC) | -- |
| VM4 | ens18 | VirtIO | vmbr1 | 10.0.0.122/24 | (NAPT NW) | -- |
| | ens19 | VirtIO | mgbr0 | 192.168.0.122/24 | (Mgmt NW) | -- |

Linux Bridges of Proxmox VE are as follows.
| Linux Bridge | Network CIDR | Interface |
| --- | --- | --- |
| vmbr1 | 10.0.0.0/24 | NAPT NW |
| mgbr0 | 192.168.0.0/24 | Mgmt NW |
| vmbr3 | 192.168.13.0/24 | S1-U,S5u for EPC |
| vmbr4 | 192.168.14.0/24 | Sxb for EPC |
| vmbr6 | 192.168.16.0/24 | SGi for EPC |

Set network instance to `internet`.
| Network Instance |
| --- |
| internet |

Subscriber Information (other information is the same) is as follows.  
| UE | IMSI | APN | OP/OPc |
| --- | --- | --- | --- |
| UE | 001010000000100 | internet | OPc |

I registered these information with the Open5GS WebUI.
In addition, [3GPP TS 35.208](https://www.3gpp.org/DynaReport/35208.htm) "4.3 Test Sets" is published by 3GPP as test data for the 3GPP authentication and key generation functions (MILENAGE).

The PDN is as follows.
| PDN | APN | TUNnel interface of UE |
| --- | --- | --- |
| 10.45.0.0/16 | internet | tun_srsue |

The main information of eNodeB is as follows.
| MCC | MNC | TAC | eNodeB ID | Cell ID | E-UTRAN Cell ID |
| --- | --- | --- | --- | --- | --- |
| 001 | 01 | 1 | 0x19b | 0x01 | 0x19b01 |

<a id="changes"></a>

## Changes in configuration files of Open5GS EPC, UPG-VPP and srsRAN 4G ZMQ UE / RAN

Please refer to the following for building Open5GS, UPG-VPP and srsRAN 4G ZMQ respectively.
- Open5GS v2.7.5 (2025.04.25) - https://open5gs.org/open5gs/docs/guide/02-building-open5gs-from-sources/
- UPG-VPP v1.13.0 (2024.03.25) - https://github.com/s5uishida/install_vpp_upf_dpdk
- srsRAN 4G (2024.02.01) - https://github.com/s5uishida/build_srsran_4g_zmq_disable_rf_plugins

<a id="changes_cp"></a>

### Changes in configuration files of Open5GS EPC C-Plane

The following parameters can be used in the logic that selects SGW-U and UPF(PGW-U) as the connection destination by PFCP.

- APN
- TAC (Tracking Area Code)
- e_CellID

For the sake of simplicity, I used only APN this time.

- `open5gs/install/etc/open5gs/mme.yaml`
```diff
--- mme.yaml.orig       2025-04-27 11:38:05.000000000 +0900
+++ mme.yaml    2025-05-04 08:17:19.608382075 +0900
@@ -12,7 +12,7 @@
   freeDiameter: /root/open5gs/install/etc/freeDiameter/mme.conf
   s1ap:
     server:
-      - address: 127.0.0.2
+      - address: 192.168.0.111
   gtpc:
     server:
       - address: 127.0.0.2
@@ -27,14 +27,14 @@
         port: 9090
   gummei:
     - plmn_id:
-        mcc: 999
-        mnc: 70
+        mcc: 001
+        mnc: 01
       mme_gid: 2
       mme_code: 1
   tai:
     - plmn_id:
-        mcc: 999
-        mnc: 70
+        mcc: 001
+        mnc: 01
       tac: 1
   security:
     integrity_order : [ EIA2, EIA1, EIA0 ]
```
- `open5gs/install/etc/open5gs/sgwc.yaml`
```diff
--- sgwc.yaml.orig      2024-05-02 19:52:00.000000000 +0900
+++ sgwc.yaml   2025-05-04 08:26:48.859933645 +0900
@@ -14,10 +14,11 @@
       - address: 127.0.0.3
   pfcp:
     server:
-      - address: 127.0.0.3
+      - address: 192.168.0.111
     client:
       sgwu:
-        - address: 127.0.0.6
+        - address: 192.168.0.112
+          apn: internet
 
 ################################################################################
 # GTP-C Server
```
- `open5gs/install/etc/open5gs/smf.yaml`
```diff
--- smf.yaml.orig       2025-04-27 11:38:05.000000000 +0900
+++ smf.yaml    2025-05-04 10:05:14.916099877 +0900
@@ -7,29 +7,23 @@
   max:
     ue: 1024  # The number of UE can be increased depending on memory size.
 #    peer: 64
+  parameter:
+    use_upg_vpp: true
 
 smf:
-  sbi:
-    server:
-      - address: 127.0.0.4
-        port: 7777
-    client:
-#      nrf:
-#        - uri: http://127.0.0.10:7777
-      scp:
-        - uri: http://127.0.0.200:7777
   pfcp:
     server:
-      - address: 127.0.0.4
+      - address: 192.168.14.111
     client:
       upf:
-        - address: 127.0.0.7
+        - address: 192.168.14.151
+          dnn: internet
   gtpc:
     server:
       - address: 127.0.0.4
   gtpu:
     server:
-      - address: 127.0.0.4
+      - address: 192.168.14.111
   metrics:
     server:
       - address: 127.0.0.4
@@ -37,13 +31,10 @@
   session:
     - subnet: 10.45.0.0/16
       gateway: 10.45.0.1
-    - subnet: 2001:db8:cafe::/48
-      gateway: 2001:db8:cafe::1
+      dnn: internet
   dns:
     - 8.8.8.8
     - 8.8.4.4
-    - 2001:4860:4860::8888
-    - 2001:4860:4860::8844
   mtu: 1400
 #  p-cscf:
 #    - 127.0.0.1
```

<a id="changes_up"></a>

### Changes in configuration files of Open5GS EPC U-Plane

- `open5gs/install/etc/open5gs/sgwu.yaml`
```diff
--- sgwu.yaml.orig      2024-05-02 19:52:00.000000000 +0900
+++ sgwu.yaml   2024-10-27 09:32:03.556465579 +0900
@@ -11,13 +11,13 @@
 sgwu:
   pfcp:
     server:
-      - address: 127.0.0.6
+      - address: 192.168.0.112
     client:
 #      sgwc:    # SGW-U PFCP Client try to associate SGW-C PFCP Server
 #        - address: 127.0.0.3
   gtpu:
     server:
-      - address: 127.0.0.6
+      - address: 192.168.13.112
 
 ################################################################################
 # PFCP Server
```

<a id="changes_vpp"></a>

### Changes in configuration files of UPG-VPP

See [here](https://github.com/s5uishida/install_vpp_upf_dpdk#conf) for the original files.

- `upg-vpp/startup.conf`  
There is no change.

- `upg-vpp/init.conf`  
There is no change.

<a id="changes_srs"></a>

### Changes in configuration files of srsRAN 4G ZMQ UE / RAN

<a id="changes_ran"></a>

#### Changes in configuration files of RAN

- `srsRAN_4G/build/srsenb/enb.conf`
```diff
--- enb.conf.example    2024-02-03 23:26:02.000000000 +0900
+++ enb.conf    2024-03-10 17:00:27.383907337 +0900
@@ -22,9 +22,9 @@
 enb_id = 0x19B
 mcc = 001
 mnc = 01
-mme_addr = 127.0.1.100
-gtp_bind_addr = 127.0.1.1
-s1c_bind_addr = 127.0.1.1
+mme_addr = 192.168.0.111
+gtp_bind_addr = 192.168.13.121
+s1c_bind_addr = 192.168.0.121
 s1c_bind_port = 0
 n_prb = 50
 #tm = 4
@@ -80,8 +80,8 @@
 #time_adv_nsamples = auto
 
 # Example for ZMQ-based operation with TCP transport for I/Q samples
-#device_name = zmq
-#device_args = fail_on_disconnect=true,tx_port=tcp://*:2000,rx_port=tcp://localhost:2001,id=enb,base_srate=23.04e6
+device_name = zmq
+device_args = fail_on_disconnect=true,tx_port=tcp://192.168.0.121:2000,rx_port=tcp://192.168.0.122:2001,id=enb,base_srate=23.04e6
 
 #####################################################################
 # Packet capture configuration
```
- `srsRAN_4G/build/srsenb/rr.conf`
```diff
--- rr.conf.example     2024-02-03 23:26:02.000000000 +0900
+++ rr.conf     2023-05-02 11:52:54.000000000 +0900
@@ -55,7 +55,7 @@
   {
     // rf_port = 0;
     cell_id = 0x01;
-    tac = 0x0007;
+    tac = 0x0001;
     pci = 1;
     // root_seq_idx = 204;
     dl_earfcn = 3350;
```

<a id="changes_ue"></a>

#### Changes in configuration files of UE

- `srsRAN_4G/build/srsue/ue.conf`
```diff
--- ue.conf.example     2024-02-03 23:26:02.000000000 +0900
+++ ue.conf     2023-05-02 12:01:28.000000000 +0900
@@ -42,8 +42,8 @@
 #continuous_tx     = auto
 
 # Example for ZMQ-based operation with TCP transport for I/Q samples
-#device_name = zmq
-#device_args = tx_port=tcp://*:2001,rx_port=tcp://localhost:2000,id=ue,base_srate=23.04e6
+device_name = zmq
+device_args = tx_port=tcp://192.168.0.122:2001,rx_port=tcp://192.168.0.121:2000,id=ue,base_srate=23.04e6
 
 #####################################################################
 # EUTRA RAT configuration
@@ -139,9 +139,9 @@
 [usim]
 mode = soft
 algo = milenage
-opc  = 63BFA50EE6523365FF14C1F45F88737D
-k    = 00112233445566778899aabbccddeeff
-imsi = 001010123456780
+opc  = E8ED289DEBA952E4283B54E88E6183CA
+k    = 465B5CE8B199B49FAA5F0A2EE238A6BC
+imsi = 001010000000100
 imei = 353490069873319
 #reader =
 #pin  = 1234
@@ -180,8 +180,8 @@
 #                      Supported: 0 - NULL, 1 - Snow3G, 2 - AES, 3 - ZUC
 #####################################################################
 [nas]
-#apn = internetinternet
-#apn_protocol = ipv4
+apn = internet
+apn_protocol = ipv4
 #user = srsuser
 #pass = srspass
 #force_imsi_attach = false
```

<a id="network_settings"></a>

## Network settings of Open5GS EPC, UPG-VPP and srsRAN 4G ZMQ UE / RAN

<a id="network_settings_up"></a>

### Network settings of UPG-VPP and Data Network Gateway

See [this1](https://github.com/s5uishida/install_vpp_upf_dpdk#setup_up) and [this2](https://github.com/s5uishida/install_vpp_upf_dpdk#setup_dn).

<a id="build"></a>

## Build Open5GS, UPG-VPP and srsRAN 4G ZMQ UE / RAN

Please refer to the following for building Open5GS, UPG-VPP and srsRAN 4G ZMQ UE / RAN respectively.
- Open5GS v2.7.5 (2025.04.25) - https://open5gs.org/open5gs/docs/guide/02-building-open5gs-from-sources/
- UPG-VPP v1.13.0 (2024.03.25) - https://github.com/s5uishida/install_vpp_upf_dpdk
- srsRAN 4G (2024.02.01) - https://github.com/s5uishida/build_srsran_4g_zmq_disable_rf_plugins

Install MongoDB on Open5GS EPC C-Plane machine.
[MongoDB Compass](https://www.mongodb.com/products/compass) is a convenient tool to look at the MongoDB database.

<a id="run"></a>

## Run Open5GS EPC, UPG-VPP and srsRAN 4G ZMQ UE / RAN

First run UPG-VPP and EPC U-Plane(SGW-U), then EPC C-Plane, the RAN, and the UE.

<a id="run_vpp"></a>

### Run UPG-VPP

See [this](https://github.com/s5uishida/install_vpp_upf_dpdk#run).

<a id="run_up"></a>

### Run Open5GS EPC U-Plane

```
./install/bin/open5gs-sgwud &
```

<a id="run_cp"></a>

### Run Open5GS EPC C-Plane

```
./install/bin/open5gs-hssd &
./install/bin/open5gs-pcrfd &
sleep 1
./install/bin/open5gs-mmed &
./install/bin/open5gs-sgwcd &
./install/bin/open5gs-smfd &
```
The status of PFCP association between UPG-VPP and Open5GS SMF(PGW-C) is as follows.
```
vpp# show upf association 
Node: 192.168.14.111
  Recovery Time Stamp: 2025/05/04 12:31:01:000
  Sessions: 0
vpp#
```

<a id="run_ran"></a>

### Run srsRAN 4G ZMQ RAN

Run srsRAN 4G ZMQ RAN and connect to Open5GS EPC.
```
# cd srsRAN_4G/build/srsenb
# ./src/srsenb enb.conf
---  Software Radio Systems LTE eNodeB  ---

Reading configuration file enb.conf...

Built in Release mode using commit ec29b0c1f on branch master.

Opening 1 channels in RF device=zmq with args=fail_on_disconnect=true,tx_port=tcp://192.168.0.121:2000,rx_port=tcp://192.168.0.122:2001,id=enb,base_srate=23.04e6
Supported RF device list: zmq file
CHx base_srate=23.04e6
CHx id=enb
Current sample rate is 1.92 MHz with a base rate of 23.04 MHz (x12 decimation)
CH0 rx_port=tcp://192.168.0.122:2001
CH0 tx_port=tcp://192.168.0.121:2000
CH0 fail_on_disconnect=true
Current sample rate is 11.52 MHz with a base rate of 23.04 MHz (x2 decimation)
Current sample rate is 11.52 MHz with a base rate of 23.04 MHz (x2 decimation)
Setting frequency: DL=2680.0 Mhz, UL=2560.0 MHz for cc_idx=0 nof_prb=50

==== eNodeB started ===
Type <t> to view trace
```
The Open5GS C-Plane log when executed is as follows.
```
05/04 12:31:38.660: [mme] INFO: eNB-S1 accepted[192.168.0.121]:56695 in s1_path module (../src/mme/s1ap-sctp.c:114)
05/04 12:31:38.660: [mme] INFO: eNB-S1 accepted[192.168.0.121] in master_sm module (../src/mme/mme-sm.c:108)
05/04 12:31:38.660: [mme] INFO: [Added] Number of eNBs is now 1 (../src/mme/mme-context.c:3031)
05/04 12:31:38.661: [mme] INFO: eNB-S1[192.168.0.121] max_num_of_ostreams : 30 (../src/mme/mme-sm.c:157)
```

<a id="run_ue"></a>

### Run srsRAN 4G ZMQ UE

Run srsRAN 4G ZMQ UE and connect to Open5GS EPC.
```
# cd srsRAN_4G/build/srsue
# ./src/srsue ue.conf
Reading configuration file ue.conf...

Built in Release mode using commit ec29b0c1f on branch master.

Opening 1 channels in RF device=zmq with args=tx_port=tcp://192.168.0.122:2001,rx_port=tcp://192.168.0.121:2000,id=ue,base_srate=23.04e6
Supported RF device list: zmq file
CHx base_srate=23.04e6
CHx id=ue
Current sample rate is 1.92 MHz with a base rate of 23.04 MHz (x12 decimation)
CH0 rx_port=tcp://192.168.0.121:2000
CH0 tx_port=tcp://192.168.0.122:2001
Waiting PHY to initialize ... done!
Attaching UE...
Current sample rate is 1.92 MHz with a base rate of 23.04 MHz (x12 decimation)
Current sample rate is 1.92 MHz with a base rate of 23.04 MHz (x12 decimation)
.
Found Cell:  Mode=FDD, PCI=1, PRB=50, Ports=1, CP=Normal, CFO=-0.2 KHz
Current sample rate is 11.52 MHz with a base rate of 23.04 MHz (x2 decimation)
Current sample rate is 11.52 MHz with a base rate of 23.04 MHz (x2 decimation)
Found PLMN:  Id=00101, TAC=1
Random Access Transmission: seq=9, tti=181, ra-rnti=0x2
RRC Connected
Random Access Complete.     c-rnti=0x46, ta=0
Network attach successful. IP: 10.45.0.2
 nTp) ((t) 4/5/2025 3:32:17 TZ:99
```
The Open5GS C-Plane log when executed is as follows.
```
05/04 12:32:17.529: [mme] INFO: InitialUEMessage (../src/mme/s1ap-handler.c:431)
05/04 12:32:17.529: [mme] INFO: [Added] Number of eNB-UEs is now 1 (../src/mme/mme-context.c:5148)
05/04 12:32:17.529: [mme] INFO: Unknown UE by S_TMSI[G:2,C:1,M_TMSI:0xc00006c1] (../src/mme/s1ap-handler.c:510)
05/04 12:32:17.529: [mme] INFO:     ENB_UE_S1AP_ID[1] MME_UE_S1AP_ID[1] TAC[1] CellID[0x19b01] (../src/mme/s1ap-handler.c:607)
05/04 12:32:17.529: [mme] INFO: Unknown UE by GUTI[G:2,C:1,M_TMSI:0xc00006c1] (../src/mme/mme-context.c:3840)
05/04 12:32:17.529: [mme] INFO: [Added] Number of MME-UEs is now 1 (../src/mme/mme-context.c:3631)
05/04 12:32:17.529: [emm] INFO: [] Attach request (../src/mme/emm-sm.c:474)
05/04 12:32:17.529: [emm] INFO:     GUTI[G:2,C:1,M_TMSI:0xc00006c1] IMSI[Unknown IMSI] (../src/mme/emm-handler.c:246)
05/04 12:32:17.550: [emm] INFO: Identity response (../src/mme/emm-sm.c:444)
05/04 12:32:17.550: [emm] INFO:     IMSI[001010000000100] (../src/mme/emm-handler.c:482)
05/04 12:32:17.617: [mme] INFO: [Added] Number of MME-Sessions is now 1 (../src/mme/mme-context.c:5162)
05/04 12:32:17.657: [sgwc] INFO: [Added] Number of SGWC-UEs is now 1 (../src/sgwc/context.c:239)
05/04 12:32:17.657: [sgwc] INFO: [Added] Number of SGWC-Sessions is now 1 (../src/sgwc/context.c:924)
05/04 12:32:17.657: [sgwc] INFO: UE IMSI[001010000000100] APN[internet] (../src/sgwc/s11-handler.c:268)
05/04 12:32:17.658: [gtp] INFO: gtp_connect() [127.0.0.4]:2123 (../lib/gtp/path.c:60)
05/04 12:32:17.658: [smf] INFO: [Added] Number of SMF-UEs is now 1 (../src/smf/context.c:1033)
05/04 12:32:17.658: [smf] INFO: [Added] Number of SMF-Sessions is now 1 (../src/smf/context.c:3190)
05/04 12:32:17.658: [smf] INFO: UE IMSI[001010000000100] APN[internet] IPv4[10.45.0.2] IPv6[] (../src/smf/s5c-handler.c:303)
05/04 12:32:17.665: [gtp] INFO: gtp_connect() [192.168.13.151]:2152 (../lib/gtp/path.c:60)
05/04 12:32:17.922: [emm] INFO: [001010000000100] Attach complete (../src/mme/emm-sm.c:1548)
05/04 12:32:17.922: [emm] INFO:     IMSI[001010000000100] (../src/mme/emm-handler.c:286)
05/04 12:32:17.922: [emm] INFO:     UTC [2025-05-04T03:32:17] Timezone[0]/DST[0] (../src/mme/emm-handler.c:292)
05/04 12:32:17.922: [emm] INFO:     LOCAL [2025-05-04T12:32:17] Timezone[32400]/DST[0] (../src/mme/emm-handler.c:296)
```
The Open5GS U-Plane log when executed is as follows.
```
05/04 12:32:17.681: [sgwu] INFO: UE F-SEID[UP:0x170 CP:0xe8e] (../src/sgwu/context.c:170)
05/04 12:32:17.681: [sgwu] INFO: [Added] Number of SGWU-Sessions is now 1 (../src/sgwu/context.c:175)
05/04 12:32:17.689: [gtp] INFO: gtp_connect() [192.168.13.151]:2152 (../lib/gtp/path.c:60)
05/04 12:32:17.946: [gtp] INFO: gtp_connect() [192.168.13.121]:2152 (../lib/gtp/path.c:60)
```
The PDU session establishment status of UPG-VPP is as follows.
```
vpp# show upf session 
CP F-SEID: 0x0000000000000055 (85) @ 192.168.14.111
UP F-SEID: 0x0000000000000055 (85) @ 192.168.14.151 (192.168.14.111 ::)
User ID: IMEI:3534900698733153
  PFCP Association: 0
  TEID assignment per choose ID
PDR: 1 @ 0x7fe44053b658
  Precedence: 65535
  PDI:
    Fields: 0000000c
    Source Interface: Core
    Network Instance: internet
    UE IP address (destination):
      IPv4 address: 10.45.0.2
    SDF Filter [1]:
      permit out ip from any to assigned 
  Outer Header Removal: no
  FAR Id: 1
  URR Ids: [] @ 0x0
  QER Ids: [1] @ 0x7fe440540128
PDR: 2 @ 0x7fe44053b6d8
  Precedence: 65535
  PDI:
    Fields: 0000000d
    Source Interface: Access
    Network Instance: internet
    Local F-TEID: 622494450 (0x251a82f2)
            IPv4: 192.168.13.151
    UE IP address (source):
      IPv4 address: 10.45.0.2
    SDF Filter [1]:
      permit out ip from any to assigned 
  Outer Header Removal: GTP-U/UDP/IPv4
  FAR Id: 2
  URR Ids: [] @ 0x0
  QER Ids: [1] @ 0x7fe496550398
PDR: 3 @ 0x7fe44053b758
  Precedence: 255
  PDI:
    Fields: 00000001
    Source Interface: CP-function
    Network Instance: internet
    Local F-TEID: 949355945 (0x389605a9)
            IPv4: 192.168.13.151
  Outer Header Removal: GTP-U/UDP/IPv4
  FAR Id: 1
  URR Ids: [] @ 0x0
  QER Ids: [] @ 0x0
PDR: 4 @ 0x7fe44053b7d8
  Precedence: 255
  PDI:
    Fields: 00000009
    Source Interface: Access
    Network Instance: internet
    Local F-TEID: 622494450 (0x251a82f2)
            IPv4: 192.168.13.151
    SDF Filter [1]:
      permit out 58 from ff02::2 to assigned 
  Outer Header Removal: GTP-U/UDP/IPv4
  FAR Id: 3
  URR Ids: [] @ 0x0
  QER Ids: [] @ 0x0
FAR: 1
  Apply Action: 00000002 == [FORWARD]
  Forward:
    Network Instance: internet
    Destination Interface: 0
    Outer Header Creation: [GTP-U/UDP/IPv4],TEID:000095e4,IP:192.168.13.112
FAR: 2
  Apply Action: 00000002 == [FORWARD]
  Forward:
    Network Instance: internet
    Destination Interface: 1
FAR: 3
  Apply Action: 00000002 == [FORWARD]
  Forward:
    Network Instance: internet
    Destination Interface: 3
    Outer Header Creation: [GTP-U/UDP/IPv4],TEID:00000001,IP:192.168.14.111
vpp# 
```
The result of `ip addr show` on VM4 (UE) is as follows.
```
6: tun_srsue: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UNKNOWN group default qlen 500
    link/none 
    inet 10.45.0.2/24 scope global tun_srsue
       valid_lft forever preferred_lft forever
```

<a id="ping"></a>

## Ping google.com

Specify the UE's TUNnel interface and try ping.

<a id="ping_1"></a>

### Case for going through PDN 10.45.0.0/16

Run `tcpdump` on VM-DN and check that the packet goes through N6 (ens20).
- `ping google.com` on VM4 (UE)
```
# ping google.com -I tun_srsue -n
PING google.com (172.217.174.110) from 10.45.0.2 tun_srsue: 56(84) bytes of data.
64 bytes from 172.217.174.110: icmp_seq=1 ttl=110 time=40.2 ms
64 bytes from 172.217.174.110: icmp_seq=2 ttl=110 time=42.7 ms
64 bytes from 172.217.174.110: icmp_seq=3 ttl=110 time=43.7 ms
```
- Run `tcpdump` on VM-DN
```
# tcpdump -i ens20 -n
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on ens20, link-type EN10MB (Ethernet), snapshot length 262144 bytes
12:34:55.268854 IP 10.45.0.2 > 172.217.174.110: ICMP echo request, id 2, seq 1, length 64
12:34:55.284465 IP 172.217.174.110 > 10.45.0.2: ICMP echo reply, id 2, seq 1, length 64
12:34:56.273340 IP 10.45.0.2 > 172.217.174.110: ICMP echo request, id 2, seq 2, length 64
12:34:56.288515 IP 172.217.174.110 > 10.45.0.2: ICMP echo reply, id 2, seq 2, length 64
12:34:57.275953 IP 10.45.0.2 > 172.217.174.110: ICMP echo request, id 2, seq 3, length 64
12:34:57.291252 IP 172.217.174.110 > 10.45.0.2: ICMP echo reply, id 2, seq 3, length 64
```
In addition to `ping`, you may try to access the web by specifying the TUNnel interface with `curl` as follows.
- `curl google.com` on VM4 (UE)
```
# curl --interface tun_srsue google.com
<HTML><HEAD><meta http-equiv="content-type" content="text/html;charset=utf-8">
<TITLE>301 Moved</TITLE></HEAD><BODY>
<H1>301 Moved</H1>
The document has moved
<A HREF="http://www.google.com/">here</A>.
</BODY></HTML>
```
- Run `tcpdump` on VM-DN
```
12:38:19.872418 IP 10.45.0.2.60984 > 172.217.174.110.80: Flags [S], seq 1787652631, win 64240, options [mss 1460,sackOK,TS val 269799255 ecr 0,nop,wscale 7], length 0
12:38:19.889287 IP 172.217.174.110.80 > 10.45.0.2.60984: Flags [S.], seq 3357340548, ack 1787652632, win 65535, options [mss 1412,sackOK,TS val 1711326121 ecr 269799255,nop,wscale 8], length 0
12:38:19.915976 IP 10.45.0.2.60984 > 172.217.174.110.80: Flags [.], ack 1, win 502, options [nop,nop,TS val 269799297 ecr 1711326121], length 0
12:38:19.915976 IP 10.45.0.2.60984 > 172.217.174.110.80: Flags [P.], seq 1:75, ack 1, win 502, options [nop,nop,TS val 269799297 ecr 1711326121], length 74: HTTP: GET / HTTP/1.1
12:38:19.934781 IP 172.217.174.110.80 > 10.45.0.2.60984: Flags [.], ack 75, win 1050, options [nop,nop,TS val 1711326167 ecr 269799297], length 0
12:38:19.977455 IP 172.217.174.110.80 > 10.45.0.2.60984: Flags [P.], seq 1:774, ack 75, win 1050, options [nop,nop,TS val 1711326209 ecr 269799297], length 773: HTTP: HTTP/1.1 301 Moved Permanently
12:38:20.004802 IP 10.45.0.2.60984 > 172.217.174.110.80: Flags [.], ack 774, win 501, options [nop,nop,TS val 269799385 ecr 1711326209], length 0
12:38:20.004802 IP 10.45.0.2.60984 > 172.217.174.110.80: Flags [F.], seq 75, ack 774, win 501, options [nop,nop,TS val 269799385 ecr 1711326209], length 0
12:38:20.021835 IP 172.217.174.110.80 > 10.45.0.2.60984: Flags [F.], seq 774, ack 76, win 1050, options [nop,nop,TS val 1711326253 ecr 269799385], length 0
12:38:20.047688 IP 10.45.0.2.60984 > 172.217.174.110.80: Flags [.], ack 775, win 501, options [nop,nop,TS val 269799428 ecr 1711326253], length 0
```
Also, when trying iperf3 client on VM4 (UE), first change the default GW interface to `tun_srsue`. Below is an example of my environment (VM4).
```
# ip link set dev ens18 down
# ip route add default dev tun_srsue
```
Next, to avoid IP fragmentation, set as follows according to the instructions in [here](https://github.com/s5uishida/simple_confirmed_info_for_mobile_network#footnotes) [7].
```
# ip link set tun_srsue mtu 1464
```
Then, bind the assigned IP address `10.45.0.2` and run iperf3 client. The following is an example of connecting to iperf3 server running on VM-DN `192.168.16.152`.
```
# iperf3 -B 10.45.0.2 -c 192.168.16.152
```
You could now connect to the PDN and send any packets on the network using UPG-VPP.

---

Now you could work Open5GS EPC with UPG-VPP.
I would like to thank the excellent developers and all the contributors of Open5GS, UPG-VPP, VPP, DPDK and srsRAN 4G.

<a id="changelog"></a>

## Changelog (summary)

- [2025.05.04] Updated to UPG-VPP `v1.13.0` and Open5GS `v2.7.5 (2025.04.25)`. Changed the VM environment from Virtualbox to Proxmox VE.
- [2024.03.24] Updated to UPG-VPP `v1.12.0`.
- [2023.11.10] Changed from [oai-cn5g-upf-vpp](https://gitlab.eurecom.fr/oai/cn5g/oai-cn5g-upf-vpp) to its base [travelping/upg-vpp](https://github.com/travelping/upg-vpp).
- [2023.07.23] Initial release.
