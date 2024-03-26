# Open5GS EPC & srsRAN 4G with ZeroMQ UE / RAN Sample Configuration - VPP-UPF(PGW-U) with DPDK
This describes a simple configuration for working Open5GS EPC and VPP-UPF with DPDK.
In particular, see [here](https://github.com/s5uishida/install_vpp_upf_dpdk#annex_1) for VPP-UPF with DPDK configuration.

**If UPG-VPP built with [this instruction](https://github.com/s5uishida/install_vpp_upf_dpdk#annex_1) does not work well, please try OAI-CN5G-UPF-VPP built with [this instruction](https://github.com/s5uishida/install_vpp_upf_dpdk#build).**

---

### [Sample Configurations and Miscellaneous for Mobile Network](https://github.com/s5uishida/sample_config_misc_for_mobile_network)

---

<a id="toc"></a>

## Table of Contents

- [Overview of Open5GS CUPS-enabled EPC Simulation Mobile Network](#overview)
- [Changes in configuration files of Open5GS EPC, VPP-UPF and srsRAN 4G ZMQ UE / RAN](#changes)
  - [Changes in configuration files of Open5GS EPC C-Plane](#changes_cp)
  - [Changes in configuration files of Open5GS EPC U-Plane](#changes_up)
  - [Changes in configuration files of VPP-UPF](#changes_vpp)
  - [Changes in configuration files of srsRAN 4G ZMQ UE / RAN](#changes_srs)
    - [Changes in configuration files of RAN](#changes_ran)
    - [Changes in configuration files of UE](#changes_ue)
- [Network settings of Open5GS EPC, VPP-UPF and srsRAN 4G ZMQ UE / RAN](#network_settings)
  - [Network settings of VPP-UPF and Data Network Gateway](#network_settings_up)
- [Build Open5GS, VPP-UPF and srsRAN 4G ZMQ UE / RAN](#build)
- [Run Open5GS EPC, VPP-UPF and srsRAN 4G ZMQ UE / RAN](#run)
  - [Run VPP-UPF](#run_vpp)
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

This describes a simple configuration of C-Plane, VPP-UPF and Data Network Gateway for Open5GS EPC.
**Note that this configuration is implemented with Virtualbox VMs.**

The following minimum configuration was set as a condition.
- One SGW-U/UPF(PGW-U) and Data Network Gateway
- One UE and one APN

The built simulation environment is as follows.

<img src="./images/network-overview.png" title="./images/network-overview.png" width=1000px></img>

The EPC / VPP-UPF / UE / RAN used are as follows.
- EPC - Open5GS v2.7.0 (2024.03.24) - https://github.com/open5gs/open5gs
- VPP-UPF - UPG-VPP v1.12.0 (2024.01.25) - https://github.com/travelping/upg-vpp
- UE / RAN - srsRAN 4G (2024.02.01) - https://github.com/srsran/srsRAN_4G

Each VMs are as follows.  
| VM | SW & Role | IP address | OS | CPU<br>(Min) | Memory<br>(Min) | HDD<br>(Min) |
| --- | --- | --- | --- | --- | --- | --- |
| VM1 | Open5GS EPC C-Plane | 192.168.0.111/24 | Ubuntu 22.04 | 1 | 2GB | 20GB |
| VM2 | Open5GS EPC U-Plane(SGW-U) | 192.168.0.112/24 | Ubuntu 22.04 | 1 | 1GB | 20GB |
| VM-UP | UPG-VPP U-Plane(PGW-U) | 192.168.0.151/24 | Ubuntu **20.04** | 2 | 8GB | 20GB |
| VM-DN | Data Network Gateway  | 192.168.0.152/24 | Ubuntu 22.04 | 1 | 1GB | 10GB |
| VM3 | srsRAN 4G ZMQ RAN (eNodeB) | 192.168.0.121/24 | Ubuntu 22.04 | 1 | 2GB | 10GB |
| VM4 | srsRAN 4G ZMQ UE | 192.168.0.122/24 | Ubuntu 22.04 | 1 | 2GB | 10GB |

The network interfaces of each VM are as follows.
**Note. Do not enable(up) any devices that will be under the control of DPDK.
These devices will be enabled and set IP addresses in the `init.conf` file of VPP-UPF.**
| VM | Device | Network Adapter | IP address | Interface | Under DPDK |
| --- | --- | --- | --- | --- | --- |
| VM1 | enp0s3 | NAT(default) | 10.0.2.15/24 | (VM default NW) | -- |
| | enp0s8 | Bridged Adapter | 192.168.0.111/24 | (Mgmt NW) | -- |
| | enp0s9 | NAT Network | 192.168.14.111/24 | Sxb (N4 for 5GC) | -- |
| VM2 | enp0s3 | NAT(default) | 10.0.2.15/24 | (VM default NW) | -- |
| | enp0s8 | Bridged Adapter | 192.168.0.112/24 | (Mgmt NW) | -- |
| | enp0s9 | NAT Network | 192.168.13.112/24 | S1-U,S5u (N3 for 5GC) | -- |
| VM-UP | enp0s3 | NAT(default) | 10.0.2.15/24 | (VM default NW) | -- |
| | enp0s8 | Bridged Adapter | 192.168.0.151/24 | (Mgmt NW) | -- |
| | enp0s9 | NAT Network | 192.168.13.151/24 | S5u (N3 for 5GC) | x |
| | enp0s10 | NAT Network | 192.168.14.151/24 | Sxb (N4 for 5GC) | x |
| | enp0s16 | NAT Network | 192.168.16.151/24 | SGi (N6 for 5GC) | x |
| VM-DN | enp0s3 | NAT(default) | 10.0.2.15/24 | (VM default NW) | -- |
| | enp0s8 | Bridged Adapter | 192.168.0.152/24 | (Mgmt NW) | -- |
| | enp0s9 | NAT Network | 192.168.16.152/24 | SGi (N6 for 5GC) | -- |
| VM3 | enp0s3 | NAT(default) | 10.0.2.15/24 | (VM default NW) | -- |
| | enp0s8 | Bridged Adapter | 192.168.0.121/24 | (Mgmt NW) | -- |
| | enp0s9 | NAT Network | 192.168.13.121/24 | S1-U (N3 for 5GC) | -- |
| VM4 | enp0s3 | NAT(default) | 10.0.2.15/24 | (VM default NW) | -- |
| | enp0s8 | Bridged Adapter | 192.168.0.122/24 | (Mgmt NW) | -- |

NAT networks of Virtualbox  are as follows.
| Network Name | Network CIDR | Note |
| --- | --- | --- |
| N3 | 192.168.13.0/24 | S1-U,S5u for EPC |
| N4 | 192.168.14.0/24 | Sxb for EPC |
| N6 | 192.168.16.0/24 | SGi for EPC |

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

## Changes in configuration files of Open5GS EPC, VPP-UPF and srsRAN 4G ZMQ UE / RAN

Please refer to the following for building Open5GS, VPP-UPF and srsRAN 4G ZMQ respectively.
- Open5GS v2.7.0 (2024.03.24) - https://open5gs.org/open5gs/docs/guide/02-building-open5gs-from-sources/
- UPG-VPP v1.12.0 (2024.01.25) - https://github.com/s5uishida/install_vpp_upf_dpdk#annex_1
- srsRAN 4G (2024.02.01) - https://github.com/s5uishida/build_srsran_4g_zmq_disable_rf_plugins

<a id="changes_cp"></a>

### Changes in configuration files of Open5GS EPC C-Plane

The following parameters including APN can be used in the logic that selects SGW-U as the connection destination by PFCP.

- APN
- TAC (Tracking Area Code)
- e_CellID

For the sake of simplicity, I used only APN this time. Please refer to [here](https://github.com/open5gs/open5gs/pull/560#issue-483001043) for the logic to select SGW-U.

- `open5gs/install/etc/open5gs/mme.yaml`
```diff
--- mme.yaml.orig       2024-03-24 15:36:48.000000000 +0900
+++ mme.yaml    2024-03-24 22:45:30.234769981 +0900
@@ -11,7 +11,7 @@
   freeDiameter: /root/open5gs/install/etc/freeDiameter/mme.conf
   s1ap:
     server:
-      - address: 127.0.0.2
+      - address: 192.168.0.111
   gtpc:
     server:
       - address: 127.0.0.2
@@ -26,14 +26,14 @@
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
--- sgwc.yaml.orig      2024-03-24 15:36:48.000000000 +0900
+++ sgwc.yaml   2024-03-24 22:45:54.437378657 +0900
@@ -13,10 +13,11 @@
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
--- smf.yaml.orig       2024-03-24 15:36:48.000000000 +0900
+++ smf.yaml    2024-03-25 00:10:24.030693017 +0900
@@ -8,39 +8,29 @@
 #    peer: 64
 
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
         port: 9090
   session:
     - subnet: 10.45.0.1/16
-    - subnet: 2001:db8:cafe::1/48
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
--- sgwu.yaml.orig      2024-03-24 15:36:48.000000000 +0900
+++ sgwu.yaml   2024-03-24 22:41:40.582500287 +0900
@@ -10,13 +10,13 @@
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

### Changes in configuration files of VPP-UPF

See [here](https://github.com/s5uishida/install_vpp_upf_dpdk#changes_up) for the original files.

- `openair-upf/startup.conf`  
There is no change.

- `openair-upf/init.conf`  
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

## Network settings of Open5GS EPC, VPP-UPF and srsRAN 4G ZMQ UE / RAN

<a id="network_settings_up"></a>

### Network settings of VPP-UPF and Data Network Gateway

See [this1](https://github.com/s5uishida/install_vpp_upf_dpdk#setup_up) and [this2](https://github.com/s5uishida/install_vpp_upf_dpdk#setup_dn).

<a id="build"></a>

## Build Open5GS, VPP-UPF and srsRAN 4G ZMQ UE / RAN

Please refer to the following for building Open5GS, VPP-UPF and srsRAN 4G ZMQ UE / RAN respectively.
- Open5GS v2.7.0 (2024.03.24) - https://open5gs.org/open5gs/docs/guide/02-building-open5gs-from-sources/
- UPG-VPP v1.12.0 (2024.01.25) - https://github.com/s5uishida/install_vpp_upf_dpdk#annex_1
- srsRAN 4G (2024.02.01) - https://github.com/s5uishida/build_srsran_4g_zmq_disable_rf_plugins

Install MongoDB on Open5GS EPC C-Plane machine.
[MongoDB Compass](https://www.mongodb.com/products/compass) is a convenient tool to look at the MongoDB database.

<a id="run"></a>

## Run Open5GS EPC, VPP-UPF and srsRAN 4G ZMQ UE / RAN

First run VPP-UPF and EPC U-Plane(SGW-U), then EPC C-Plane, the RAN, and the UE.

<a id="run_vpp"></a>

### Run VPP-UPF

See [this](https://github.com/s5uishida/install_vpp_upf_dpdk#run_upg_vpp).

<a id="run_up"></a>

### Run Open5GS EPC U-Plane

```
./install/bin/open5gs-sgwud &
```

<a id="run_cp"></a>

### Run Open5GS EPC C-Plane

```
./install/bin/open5gs-mmed &
./install/bin/open5gs-sgwcd &
./install/bin/open5gs-smfd &
./install/bin/open5gs-hssd &
./install/bin/open5gs-pcrfd &
```
The status of PFCP association between VPP-UPF and Open5GS SMF(PGW-C) is as follows.
```
vpp# show upf association 
Node: 192.168.14.111
  Recovery Time Stamp: 2024/03/24 23:13:29:000
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

==== eNodeB started ===
Type <t> to view trace
Current sample rate is 11.52 MHz with a base rate of 23.04 MHz (x2 decimation)
Current sample rate is 11.52 MHz with a base rate of 23.04 MHz (x2 decimation)
Setting frequency: DL=2680.0 Mhz, UL=2560.0 MHz for cc_idx=0 nof_prb=50
```
The Open5GS C-Plane log when executed is as follows.
```
03/24 23:14:10.284: [mme] INFO: eNB-S1 accepted[192.168.0.121]:33283 in s1_path module (../src/mme/s1ap-sctp.c:114)
03/24 23:14:10.284: [mme] INFO: eNB-S1 accepted[192.168.0.121] in master_sm module (../src/mme/mme-sm.c:108)
03/24 23:14:10.284: [mme] INFO: [Added] Number of eNBs is now 1 (../src/mme/mme-context.c:2829)
03/24 23:14:10.284: [mme] INFO: eNB-S1[192.168.0.121] max_num_of_ostreams : 30 (../src/mme/mme-sm.c:150)
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
Random Access Transmission: seq=40, tti=181, ra-rnti=0x2
RRC Connected
Random Access Complete.     c-rnti=0x46, ta=0
Network attach successful. IP: 10.45.0.2
 nTp) ((t) 24/3/2024 14:14:50 TZ:99
```
The Open5GS C-Plane log when executed is as follows.
```
03/24 23:14:49.530: [mme] INFO: InitialUEMessage (../src/mme/s1ap-handler.c:406)
03/24 23:14:49.530: [mme] INFO: [Added] Number of eNB-UEs is now 1 (../src/mme/mme-context.c:4735)
03/24 23:14:49.530: [mme] INFO: Unknown UE by S_TMSI[G:2,C:1,M_TMSI:0xc000053b] (../src/mme/s1ap-handler.c:485)
03/24 23:14:49.530: [mme] INFO:     ENB_UE_S1AP_ID[1] MME_UE_S1AP_ID[1] TAC[1] CellID[0x19b01] (../src/mme/s1ap-handler.c:585)
03/24 23:14:49.530: [mme] INFO: Unknown UE by GUTI[G:2,C:1,M_TMSI:0xc000053b] (../src/mme/mme-context.c:3586)
03/24 23:14:49.530: [mme] INFO: [Added] Number of MME-UEs is now 1 (../src/mme/mme-context.c:3379)
03/24 23:14:49.530: [emm] INFO: [] Attach request (../src/mme/emm-sm.c:423)
03/24 23:14:49.530: [emm] INFO:     GUTI[G:2,C:1,M_TMSI:0xc000053b] IMSI[Unknown IMSI] (../src/mme/emm-handler.c:236)
03/24 23:14:49.576: [emm] INFO: Identity response (../src/mme/emm-sm.c:393)
03/24 23:14:49.576: [emm] INFO:     IMSI[001010000000100] (../src/mme/emm-handler.c:428)
03/24 23:14:49.717: [mme] INFO: [Added] Number of MME-Sessions is now 1 (../src/mme/mme-context.c:4749)
03/24 23:14:49.802: [sgwc] INFO: [Added] Number of SGWC-UEs is now 1 (../src/sgwc/context.c:239)
03/24 23:14:49.802: [sgwc] INFO: [Added] Number of SGWC-Sessions is now 1 (../src/sgwc/context.c:882)
03/24 23:14:49.802: [sgwc] INFO: UE IMSI[001010000000100] APN[internet] (../src/sgwc/s11-handler.c:239)
03/24 23:14:49.803: [gtp] INFO: gtp_connect() [127.0.0.4]:2123 (../lib/gtp/path.c:60)
03/24 23:14:49.803: [smf] INFO: [Added] Number of SMF-UEs is now 1 (../src/smf/context.c:1019)
03/24 23:14:49.803: [smf] INFO: [Added] Number of SMF-Sessions is now 1 (../src/smf/context.c:3090)
03/24 23:14:49.804: [smf] INFO: UE IMSI[001010000000100] APN[internet] IPv4[10.45.0.2] IPv6[] (../src/smf/s5c-handler.c:275)
03/24 23:14:49.828: [gtp] INFO: gtp_connect() [192.168.13.151]:2152 (../lib/gtp/path.c:60)
03/24 23:14:50.146: [emm] INFO: [001010000000100] Attach complete (../src/mme/emm-sm.c:1384)
03/24 23:14:50.146: [emm] INFO:     IMSI[001010000000100] (../src/mme/emm-handler.c:275)
03/24 23:14:50.147: [emm] INFO:     UTC [2024-03-24T14:14:50] Timezone[0]/DST[0] (../src/mme/emm-handler.c:281)
03/24 23:14:50.147: [emm] INFO:     LOCAL [2024-03-24T23:14:50] Timezone[32400]/DST[0] (../src/mme/emm-handler.c:285)
```
The Open5GS U-Plane log when executed is as follows.
```
03/24 23:14:49.804: [sgwu] INFO: UE F-SEID[UP:0x57c CP:0xb3f] (../src/sgwu/context.c:171)
03/24 23:14:49.804: [sgwu] INFO: [Added] Number of SGWU-Sessions is now 1 (../src/sgwu/context.c:176)
03/24 23:14:49.830: [gtp] INFO: gtp_connect() [192.168.13.151]:2152 (../lib/gtp/path.c:60)
03/24 23:14:50.149: [gtp] INFO: gtp_connect() [192.168.13.121]:2152 (../lib/gtp/path.c:60)
```
The PDU session establishment status of VPP-UPF is as follows.
```
vpp# show upf session 
CP F-SEID: 0x000000000000021f (543) @ 192.168.14.111
UP F-SEID: 0x000000000000021f (543) @ 192.168.14.151 (192.168.14.111 ::)
User ID: IMEI:3534900698733153
  PFCP Association: 0
  TEID assignment per choose ID
PDR: 1 @ 0x7f75155b50a8
  Precedence: 255
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
  QER Ids: [1] @ 0x7f75155c8eb8
PDR: 2 @ 0x7f75155b5128
  Precedence: 255
  PDI:
    Fields: 0000000d
    Source Interface: Access
    Network Instance: internet
    Local F-TEID: 773966225 (0x2e21c991)
            IPv4: 192.168.13.151
    UE IP address (source):
      IPv4 address: 10.45.0.2
    SDF Filter [1]:
      permit out ip from any to assigned 
  Outer Header Removal: GTP-U/UDP/IPv4
  FAR Id: 2
  URR Ids: [] @ 0x0
  QER Ids: [1] @ 0x7f75155c8ed8
PDR: 3 @ 0x7f75155b51a8
  Precedence: 1000
  PDI:
    Fields: 00000001
    Source Interface: CP-function
    Network Instance: internet
    Local F-TEID: 966718908 (0x399ef5bc)
            IPv4: 192.168.13.151
  Outer Header Removal: GTP-U/UDP/IPv4
  FAR Id: 1
  URR Ids: [] @ 0x0
  QER Ids: [] @ 0x0
PDR: 4 @ 0x7f75155b5228
  Precedence: 1
  PDI:
    Fields: 00000009
    Source Interface: Access
    Network Instance: internet
    Local F-TEID: 773966225 (0x2e21c991)
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
    Outer Header Creation: [GTP-U/UDP/IPv4],TEID:0000b30f,IP:192.168.13.112
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
5: tun_srsue: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UNKNOWN group default qlen 500
    link/none 
    inet 10.45.0.2/24 scope global tun_srsue
       valid_lft forever preferred_lft forever
```

<a id="ping"></a>

## Ping google.com

Specify the UE's TUNnel interface and try ping.

<a id="ping_1"></a>

### Case for going through PDN 10.45.0.0/16

Run `tcpdump` on VM-DN and check that the packet goes through N6 (enp0s9).
- `ping google.com` on VM4 (UE)
```
# ping google.com -I tun_srsue -n
PING google.com (142.250.207.46) from 10.45.0.2 tun_srsue: 56(84) bytes of data.
64 bytes from 142.250.207.46: icmp_seq=2 ttl=59 time=75.0 ms
64 bytes from 142.250.207.46: icmp_seq=3 ttl=59 time=67.8 ms
64 bytes from 142.250.207.46: icmp_seq=4 ttl=59 time=83.0 ms
```
- Run `tcpdump` on VM-DN
```
# tcpdump -i enp0s9 -n
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on enp0s9, link-type EN10MB (Ethernet), snapshot length 262144 bytes
23:19:54.615768 IP 10.45.0.2 > 142.250.207.46: ICMP echo request, id 2, seq 2, length 64
23:19:54.634720 IP 142.250.207.46 > 10.45.0.2: ICMP echo reply, id 2, seq 2, length 64
23:19:55.625361 IP 10.45.0.2 > 142.250.207.46: ICMP echo request, id 2, seq 3, length 64
23:19:55.642523 IP 142.250.207.46 > 10.45.0.2: ICMP echo reply, id 2, seq 3, length 64
23:19:56.651818 IP 10.45.0.2 > 142.250.207.46: ICMP echo request, id 2, seq 4, length 64
23:19:56.669141 IP 142.250.207.46 > 10.45.0.2: ICMP echo reply, id 2, seq 4, length 64
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
23:20:43.687750 IP 10.45.0.2.34970 > 142.250.207.46.80: Flags [S], seq 2260815696, win 64240, options [mss 1460,sackOK,TS val 1749478147 ecr 0,nop,wscale 7], length 0
23:20:43.704196 IP 142.250.207.46.80 > 10.45.0.2.34970: Flags [S.], seq 14912001, ack 2260815697, win 65535, options [mss 1460], length 0
23:20:43.758458 IP 10.45.0.2.34970 > 142.250.207.46.80: Flags [.], ack 1, win 64240, length 0
23:20:43.758459 IP 10.45.0.2.34970 > 142.250.207.46.80: Flags [P.], seq 1:75, ack 1, win 64240, length 74: HTTP: GET / HTTP/1.1
23:20:43.758634 IP 142.250.207.46.80 > 10.45.0.2.34970: Flags [.], ack 75, win 65535, length 0
23:20:43.816815 IP 142.250.207.46.80 > 10.45.0.2.34970: Flags [P.], seq 1:774, ack 75, win 65535, length 773: HTTP: HTTP/1.1 301 Moved Permanently
23:20:43.866667 IP 10.45.0.2.34970 > 142.250.207.46.80: Flags [.], ack 774, win 63467, length 0
23:20:43.866725 IP 10.45.0.2.34970 > 142.250.207.46.80: Flags [F.], seq 75, ack 774, win 63467, length 0
23:20:43.866842 IP 142.250.207.46.80 > 10.45.0.2.34970: Flags [.], ack 76, win 65535, length 0
23:20:43.882416 IP 142.250.207.46.80 > 10.45.0.2.34970: Flags [F.], seq 774, ack 76, win 65535, length 0
23:20:43.939567 IP 10.45.0.2.34970 > 142.250.207.46.80: Flags [.], ack 775, win 63467, length 0
```
Also, when trying iperf3 client on VM4 (UE), first change the default GW interface to `tun_srsue`. Below is an example of my VirtualBox VM (VM4).
```
# ip link set dev enp0s3 down
# ip route add default dev tun_srsue
```
Next, bind the assigned IP address `10.45.0.2` and run iperf3 client. The following is an example of connecting to iperf3 server running on VM-DN `192.168.16.152`.
```
# iperf3 -B 10.45.0.2 -c 192.168.16.152
```
You could now connect to the PDN and send any packets on the network using VPP-UPF with DPDK.

---

Now you could work Open5GS EPC with VPP-UPF.
I would like to thank the excellent developers and all the contributors of Open5GS, OpenAir CN 5G for UPF, UPG-VPP, VPP, DPDK and srsRAN 4G.

<a id="changelog"></a>

## Changelog (summary)

- [2024.03.24] Updated to UPG-VPP v1.12.0.
- [2023.11.10] Changed VPP-UPF from [oai-cn5g-upf-vpp](https://gitlab.eurecom.fr/oai/cn5g/oai-cn5g-upf-vpp) to its base [travelping/upg-vpp](https://github.com/travelping/upg-vpp).
- [2023.07.23] Initial release.
