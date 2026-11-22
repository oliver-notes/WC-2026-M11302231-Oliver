---
title: Lab 4 — 5G CU/DU Split Deployment and F1-C/F1-U Analysis

---

# Lab 4 — 5G CU/DU Split Deployment and F1-C/F1-U Analysis

:::spoiler 目錄 : 
 [toc] 
:::

## 2. Network architecture and experimental parameters

```text
                         OAI 5G Core
                     AMF / SMF / UPF
                  (AMF 192.168.70.132)
                           |
                   N2 (NGAP) / N3 (GTP-U)
                           |
                      +----------+
                      |  gNB-CU  |  F1-C IP: 127.0.0.3
                      | RRC/PDCP |  NG IP: 192.168.70.129
                      +----------+
                         |     |
        F1-C: F1AP/SCTP |     | F1-U: GTP-U/UDP 2152
                         |     |
                      +-----------+
                      |  gNB-DU   |  F1 IP: 127.0.0.4
                      |RLC/MAC/PHY|  gNB-DU ID 3584 (du-rfsim)
                      +-----------+
                           |
                    RFsimulator (IQ/TCP)
                           |
                         nrUE
                           |
                  oaitun\_ue1 (10.0.0.2)
                           |
                       ICMP Ping
```

|Parameter|My configuration (observed)|
|-|-|
|OAI version|`v2.4.0`|
|OAI path|`\~/openairinterface5g`|
|RAN|One gNB-CU + one gNB-DU + one nrUE|
|Radio|RFsimulator, no physical SDR|
|NR band|n78 (TDD)|
|Bandwidth|106 PRBs|
|Numerology|1 (30 kHz SCS)|
|CU F1 address (observed)|`127.0.0.3`|
|DU F1 address (observed)|`127.0.0.4`|
|DU SSB NR-ARFCN (observed)|`630048` (3,450.72 MHz)|
|PCI / TAC|0 / 1|
|PLMN|MCC 001 / MNC 01, SST 1|
|UE frequency argument|`-C 3450720000`|
|UE IMSI|`001010000000001`|
|External DN|`192.168.70.135`|


## 3. Task 1 — Inspect the split configurations

### Step 1. Confirm the configuration files exist

```bash
cd \~/openairinterface5g
git describe --tags        # v2.4.0
ls targets/PROJECTS/GENERIC-NR-5GC/CONF | grep -iE "cu|du"
```

Both `gnb-cu.sa.f1.conf` and `gnb-du.sa.band78.106prb.rfsim.pci0.conf` exist.

### Step 2. Check frequency and F1 network settings

```bash
cd targets/PROJECTS/GENERIC-NR-5GC/CONF
grep -nE "amf\_ip|GNB\_IPV4|local\_s\_address|remote\_s\_address|mcc|mnc" gnb-cu.sa.f1.conf
grep -nE "local\_n\_address|remote\_n\_address|absoluteFrequencySSB" gnb-du.sa.band78.106prb.rfsim.pci0.conf
```

![CU 設定檔 grep](https://hackmd.io/_uploads/BJ1m-Qg1mx.png)

![DU 設定檔 grep](https://hackmd.io/_uploads/SyjmWXgkQx.png)

**Record:**

|Item|Handout value|Observed|
|-|-|-|
|CU F1 address|127.0.0.3|127.0.0.3 (`local\_s\_address`)|
|DU F1 address|127.0.0.4|127.0.0.4 (`local\_n\_address`; `remote\_n\_address` = 127.0.0.3)|
|DU SSB NR-ARFCN|630048|630048|
|PRB count|106|106|
|AMF IP (CU)|—|192.168.70.132|

CU `remote\_s\_address = "0.0.0.0"` means the CU accepts F1 connections from any DU.

**Questions**

**Questions**

1. **Which protocol layers mainly reside in the CU, and which reside in the DU?**
   CU: RRC, SDAP, PDCP (non-real-time). DU: RLC, MAC, PHY (real-time scheduling and radio processing).

2. **What is the purpose of the F1 interface?**
   It connects CU and DU so they act as one gNB: F1-C (F1AP/SCTP) carries control signalling, F1-U (GTP-U/UDP) carries user data.

3. **Why must the UE radio frequency be compatible with the DU rather than taken blindly from the previous lab?**
   The UE syncs to the SSB sent by the DU (630048, 3450.72 MHz), not Lab 3's 3619.20 MHz, so I used `-C 3450720000`.

## 4. Task 2 — Deploy the split 5G SA network

Each RAN process's output was saved with `tee` so the logs can be checked later. Start order: CN5G → CU → tcpdump (Task 3) → DU → nrUE.

### Terminal 1 — Start CN5G

```bash
cd \~/oai-cn5g
docker compose up -d
docker compose ps
```


### Terminal 2 — Start gNB-CU

```bash
cd \~/openairinterface5g/cmake\_targets/ran\_build/build

sudo ./nr-softmodem \
  -O ../../../targets/PROJECTS/GENERIC-NR-5GC/CONF/gnb-cu.sa.f1.conf 2>\&1 | tee \~/cu.log
```

### Terminal 3 — Start gNB-DU

```bash
cd \~/openairinterface5g/cmake\_targets/ran\_build/build

sudo ./nr-softmodem \
  -O ../../../targets/PROJECTS/GENERIC-NR-5GC/CONF/gnb-du.sa.band78.106prb.rfsim.pci0.conf \
  --rfsim 2>\&1 | tee \~/du.log
```

### Terminal 4 — Start nrUE

```bash
cd \~/openairinterface5g/cmake\_targets/ran\_build/build

sudo ./nr-uesoftmodem \
  -r 106 \
  --numerology 1 \
  --band 78 \
  -C 3450720000 \
  --uicc0.imsi 001010000000001 \
  --rfsim \
  --rfsimulator.serveraddr 127.0.0.1 2>\&1 | tee \~/ue.log
```
![CN5G containers](https://hackmd.io/_uploads/BJJmFQxyQx.png)

![CUDU_running_evidence](https://hackmd.io/_uploads/rktSQQly7e.png)


```text
cu.log: \[NR\_RRC] Received F1 Setup Request from gNB\_DU 3584 (du-rfsim) on assoc\_id 71
cu.log: \[NR\_RRC] Accepting DU 3584 (du-rfsim), sending F1 Setup Response
du.log: \[MAC]    received F1 Setup Response from CU cu-rfsim
```

### Check the UE connection

```bash
ip -4 addr show oaitun\_ue1
ping -I oaitun\_ue1 -c 30 -i 0.2 192.168.70.135
```

![oaitun_ue1](https://hackmd.io/_uploads/rknCbQe17e.png)

![ping_30_time](https://hackmd.io/_uploads/r1KkGmxJ7x.png)

**Evidence:**

|Item|Observed|
|-|-|
|UE IP (`oaitun\_ue1`)|10.0.0.2/24 |
|Transmitted / Received|30 / 30 |
|Packet loss|0% |
|Average RTT|**7.063 ms**（min 5.793 / max 8.469）
|


## 5. Task 3 — Capture F1-C and F1-U in a single PCAP

### Step 1. Start packet capture

Started after CN5G and CU, **before** the DU, so the SCTP association and F1 Setup are captured:

```bash
sudo tcpdump -i lo -s 0 -nn \
  -w ~/lab4-f1-complete2.pcap \
  'host 127.0.0.3 and host 127.0.0.4 and (sctp or udp port 2152)'
```

### Step 2. Trigger control-plane and user-plane traffic

1. Started the DU → SCTP association and F1 Setup.
2. Started the nrUE → RRC and UE Context procedures.
3. After `oaitun\_ue1` appeared, ran `ping -I oaitun\_ue1 -c 30 -i 0.2 192.168.70.135`.
4. Pressed `Ctrl+C` in the tcpdump terminal.
5. Verified the file:

```bash
ls -lh \~/lab4-f1-complete2.pcap    # 19K, 123 packets
```

## 6. Task 4 — Analyze F1-C (F1AP / SCTP)

Display filters: `sctp`, `f1ap`

![f1ap_overview](https://hackmd.io/_uploads/HkmziXx1Ql.png)

![F1AP_sequence](https://hackmd.io/_uploads/S1rqsQlJXg.png)

|Message|Direction|Packet No. (my capture)|Meaning|
|-|-|-|-|
|SCTP INIT / INIT\_ACK / COOKIE\_ECHO / COOKIE\_ACK|DU ↔ CU|1–4 |Establish SCTP association|
|F1 Setup Request|DU → CU (127.0.0.4 → 127.0.0.3)|5|DU initiates F1 connection and provides served-cell information|
|F1 Setup Response|CU → DU|7|CU accepts F1 Setup|
|GNBDUConfigurationUpdate / Ack|DU → CU / CU → DU|9 / 10|DU informs CU of configuration changes|
|Initial UL RRC Message Transfer (RRC Setup Request)|DU → CU|12|UE initial RRC request forwarded to CU|
|DL RRC Message Transfer (RRC Setup)|CU → DU|13|CU delivers RRC Setup toward UE|
|UL RRC Message Transfer (RRC Setup Complete)|DU → CU|14|Carries NAS Registration Request|
|UE Context Setup Request / Response|CU → DU / DU → CU|27 / 28|Establish UE-specific resources, DRB and F1-U tunnels|
|DL RRC Message Transfer (RRC Reconfiguration)|CU → DU|29|CU configures the DRB on the UE|
|UL RRC Message Transfer (RRC Reconfiguration Complete)|DU → CU|30|UE confirms reconfiguration|
|UE Context Modification Request / Response|CU → DU / DU → CU|31 / 32|Further UE context update|

**Observed sequence:** Between packets 14 and 27 the CU relays NAS Authentication (15–16) and NAS Security Mode (17–18) between the UE and the AMF, then runs the RRC (AS) Security Mode procedure (19–20) and UE Capability Enquiry (21–22). Packets 17 and 19 are both called "Security Mode", but 17 is NAS (UE ↔ AMF) and 19 is RRC (UE ↔ CU).

### F1 Setup Request (packet 5)

![F1SetupRequest IEs](https://hackmd.io/_uploads/S1e7hQe17g.png)

![DU served cell information](https://hackmd.io/_uploads/HkAH37e1Xe.png)


gNB-DU ID = 3584, gNB-DU Name = `du-rfsim`, one served cell with NR PCI 0, 5GS TAC 1 and TDD mode, plus MIB and SIB1. The SSB frequency 630048 is encoded inside `measurementTimingConfiguration` (RRC-encoded, shown as hex). These values match `cu.log` and the DU config file.

### UE Context Setup (packets 27 / 28)

![UE Context Setup Request](https://hackmd.io/_uploads/SJZu27l17x.png)

![UE Context Setup Response](https://hackmd.io/_uploads/HJZnhml17e.png)


||Packet|Tunnel address|TEID|Other|
|-|-|-|-|-|
|UL (allocated by CU)|27, CU → DU|127.0.0.3|`0xb26e9fc9`|QFI 1, 5QI 9, SST 1|
|DL (allocated by DU)|28, DU → CU|127.0.0.4|`0xb47b290d`|DRB ID 1, LCID 4|

**Questions**

1. **Which node initiates F1 Setup, and which acknowledges it?**
   The DU sends F1 Setup Request (pkt 5, 127.0.0.4 → 127.0.0.3); the CU replies with F1 Setup Response (pkt 7).

2. **Explain the difference between SCTP association establishment and F1 Setup.**
   SCTP (pkts 1–4) only opens the transport connection; F1 Setup is the F1AP procedure on top of it where the DU registers its ID (3584) and cell with the CU.

3. **How can a NAS Registration Request be transported over F1-C?**
   NAS is nested in RRC Setup Complete, which is carried in an F1AP UL RRC Message Transfer (pkt 14); the DU just forwards it to the CU.

4. **What is the role of UE Context Setup?**
   The CU asks the DU to set up resources for the UE, including the DRB and F1-U tunnels (UL TEID 0xb26e9fc9, DL TEID 0xb47b290d in pkts 27/28).

5. **Does F1 Setup alone prove successful UE Registration? Explain.**
   No. F1 Setup happens before any UE attaches; registration is proven by Registration Accept, the UE IP (10.0.0.2) and a successful ping.

## 7. Task 5 — Analyze F1-U (GTP-U / UDP)

Display filters:

```text
gtp
gtp \&\& ip.src == 127.0.0.4 \&\& ip.dst == 127.0.0.3
gtp \&\& ip.src == 127.0.0.3 \&\& ip.dst == 127.0.0.4
```

![Uplink F1-U T-PDU](https://hackmd.io/_uploads/HJil6meyQe.png)

![Downlink F1-U T-PDU](https://hackmd.io/_uploads/B1uVa7eJQe.png)


|Field|Uplink (packet 33)|Downlink (packet 38)|
|-|-|-|
|Outer IP|127.0.0.4 → 127.0.0.3 (DU → CU)|127.0.0.3 → 127.0.0.4 (CU → DU)|
|UDP ports|2152 → 2152|2152 → 2152|
|Message type|T-PDU (0xff)|T-PDU (0xff)|
|TEID|`0xb26e9fc9`|`0xb47b290d`|
|Flags / ext. header|0x30, none|0x34, NR RAN Container (0x84)|
|Encapsulated payload|PDCP header `80 00 00` + IPv6 Router Solicitation|PDCP header `80 00 00` + IPv4 ICMP Echo Reply 192.168.70.135 → 10.0.0.2|

|Direction|Filter|Packet count|
|-|-|-|
|UL (DU → CU)|`gtp \&\& ip.src == 127.0.0.4`|36|
|DL (CU → DU)|`gtp \&\& ip.src == 127.0.0.3`|30|

The 30 DL packets are the 30 Echo Replies. The 36 UL packets are the 30 Echo Requests plus 6 UE-originated packets with no reply, such as the IPv6 Router Solicitation. **The TEIDs used on F1-U exactly match the values negotiated over F1-C in packets 27 and 28.**

### Distinguish F1-U from N3

* **F1-U:** GTP-U between gNB-DU (127.0.0.4) and gNB-CU (127.0.0.3).
* **N3:** GTP-U between gNB-CU (192.168.70.129) and the UPF on the Docker network, which is not in this loopback capture.

**Questions**

1. **Which transport protocol carries F1-U?**
   UDP (port 2152), with GTP-U on top.

2. **What does a GTP-U TEID identify?**
   The tunnel endpoint, i.e. which UE bearer a packet belongs to (here UL 0xb26e9fc9, DL 0xb47b290d).

3. **Which direction is `127.0.0.4 → 127.0.0.3` in this setup?**
   Uplink, DU → CU.

4. **How would you distinguish F1-U from N3 in a packet capture?**
   By the outer IPs: F1-U is DU ↔ CU (127.0.0.4 ↔ .3), N3 is CU ↔ UPF (192.168.70.x). F1-U also carries PDCP PDUs, while N3 carries plain IP packets.

5. **Why do F1-C and F1-U use different transport protocols?**
   Signalling needs reliable, ordered delivery (SCTP); user data needs low overhead and speed (UDP).


## 8. Task 6 — Compare monolithic and split gNB

|Feature|Lab 3: Monolithic gNB|Lab 4: CU/DU Split|
|-|-|-|
|Number of RAN processes|1 (`nr-softmodem`)|2 (CU + DU `nr-softmodem`)|
|Location of RRC/PDCP|Inside the single gNB|gNB-CU|
|Location of RLC/MAC/PHY|Inside the single gNB|gNB-DU|
|F1-C interface present?|No|Yes (F1AP/SCTP, 127.0.0.4 ↔ 127.0.0.3)|
|F1-U interface present?|No|Yes (GTP-U/UDP 2152)|
|UE connected to the same 5GC?|Yes (OAI CN5G)|Yes (same OAI CN5G)|
|End-to-end Ping works?|Yes|Yes (30/30, 0% loss)|

The split only adds the F1 interface inside the RAN; the UE and 5GC still see one gNB. So the UE still has to sync to the SSB, register with the AMF and set up a PDU session, just as in Lab 3.
