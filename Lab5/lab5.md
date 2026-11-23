
# Lab 5: From Initial Access to RRC Connection Setup (Student Guide)

## 1. Lab Goal

In Lab 1, you observed UE registration with the 5GC and the establishment of data service. This lab moves the observation point earlier: before registering with the Core network, how does a UE find a gNB, obtain cell system information, perform Random Access, and finally establish an RRC connection?

The main procedure in this lab is:

```text
Cell search / initial synchronization
→ MIB / SIB1
→ Msg1 PRACH
→ Msg2 RAR
→ Msg3 RRCSetupRequest
→ Msg4 RRCSetup
→ RRCSetupComplete
→ RRC_CONNECTED
```

This lab ends at `RRC_CONNECTED`. NAS Registration, Authentication, and PDU Session establishment belong to the Core network scope covered in Lab 1.

## 2. Evidence Used in This Lab

You will receive three files produced by the same successful OAI run:

| File | Purpose |
|---|---|
| `Lab5-ue.log` | Observe UE cell search, synchronization, PRACH, and RRC state. |
| `Lab5-gnb.log` | Cross-check gNB-side behavior when needed. |
| `Lab5.pcap` | Use Wireshark to observe MIB, SIB1, RAR, and RRC messages. |

You are not required to start OAI for this lab. Your main task is to reconstruct and explain the access procedure using the supplied log and PCAP files. Section 7 provides optional steps for reproducing the files yourself.

## 3. Learning Objectives

After completing this lab, you should be able to:

1. Explain the UE procedure from Cell Search to `RRC_CONNECTED` in the correct order.
2. Distinguish the roles of broadcast system information, Random Access, and RRC Connection Setup.
3. Obtain complementary evidence from a UE log and a Wireshark MAC-NR trace.
4. Explain the meaning of `RRC_CONNECTED` and how it differs from 5GC Registration.

## 4. Open the Supplied Files

Perform all analysis from the `openairinterface5g` repository root. Put the three supplied files in `lab5-artifacts/`, or in another location of your choice. If you use another location, adjust the paths in the commands accordingly.

```text
lab5-artifacts/Lab5-ue.log
lab5-artifacts/Lab5-gnb.log
lab5-artifacts/Lab5.pcap
```

Open `lab5-artifacts/Lab5.pcap` in Wireshark.

If Wireshark shows only UDP packets or the `nr-rrc` filter does not work, complete the following setup once:

1. Open **Analyze → Enabled Protocols...**, search for, and enable `MAC-NR-FRAMED`.
2. Open **Preferences → Protocols → MAC-NR** and enable NR RRC decoding for BCCH/CCCH data.
3. Open **Preferences → Protocols → RLC-NR** and enable RRC decoding for CCCH data and AM reassembly.
4. Close and reopen the PCAP.

## 5. Submission and Grading

Add a brief explanation below every screenshot. You do not need to draw boxes or arrows on screenshots.

### Task 1: From Cell Search to Msg2 RAR — 35 points

Complete the following four steps in order. For each step, provide the requested values or screenshots and briefly explain what the evidence shows and why it matters in the access procedure.

#### Step 1: Find Cell Search and Initial Synchronization — 8 points

Run the following command on `Lab5-ue.log`:

```bash
grep -nE 'Starting cell search|Initial sync successful' lab5-artifacts/Lab5-ue.log
```

Record whether UE synchronization succeeded and the PCI value. Explain: after completing initial synchronization, what is the UE able to do?

**Student result — Task 1 / Step 1**

```text
56: Starting cell search with center freq: 3619200000, bandwidth: 106.
63: Initial sync successful, PCI: 0
```

**Synchronization:** Successful.

**PCI:** `0`

After initial synchronization, the UE has established the cell's timing and frequency reference. It can decode the PBCH/MIB and obtain additional broadcast system information, including SIB1.

Synchronization alone does not mean that the UE has established an RRC connection or registered with the 5G Core.

The UE log also confirms `SIB1 decoded` at line 71.

#### Step 2: Find MIB and SIB1 — 8 points

In Wireshark, enter:

```text
nr-rrc
```

Find one `MIB` packet and one `SIB1` packet. Record the frame number of each and provide screenshots.

MIB and SIB1 are repeatedly broadcast system-information messages. The UE must obtain the common configuration before it can continue the access procedure. Briefly explain the purpose of MIB and SIB1 for UE access.

**Student result — Task 1 / Step 2**

| Message | Wireshark frame | Observed transport |
|---|---:|---|
| MIB | **2** | BCH (BCCH-BCH-Message) |
| SIB1 | **1** | DL-SCH (BCCH-DL-SCH-Message) |

![Wireshark MIB and SIB1 overview](images/task1-mib-sib1-overview.png)

*Figure 1: Wireshark with the `nr-rrc` display filter showing repeated MIB and SIB1 packets.*

![Decoded MIB](images/task1-mib.png)

*Figure 2: Decoded MIB in frame 2, showing common subcarrier spacing, SSB subcarrier offset, and cell access information.*

![Decoded SIB1](images/task1-sib1.png)

*Figure 3: Decoded SIB1 in frame 1, including cellSelectionInfo, cellAccessRelatedInfo, servingCellConfigCommon and ue-TimersAndConstants.*

**MIB explanation:**

The Master Information Block (MIB) provides fundamental cell information needed by the UE to continue the initial access procedure.

It includes the common subcarrier spacing and information needed to locate and decode SIB1.

In the selected MIB packet, the following parameters were visible:

| MIB parameter | Observed value |
|---|---|
| Frame | 2 |
| `subCarrierSpacingCommon` | `scs30or120` |
| `ssb-SubcarrierOffset` | `0` |
| `dmrs-TypeA-Position` | `pos2` |
| `cellBarred` | `notBarred` |
| `intraFreqReselection` | `notAllowed` |

**SIB1 explanation:**

System Information Block 1 (SIB1) provides cell-access and common radio configuration information.

It includes parameters required for cell selection and the Random Access procedure.

The selected SIB1 packet contained:

- `cellSelectionInfo`
- `cellAccessRelatedInfo`
- `servingCellConfigCommon`
- `ue-TimersAndConstants`

The MIB and SIB1 are repeatedly broadcast. Therefore, their order in the recorded PCAP does not necessarily represent the order in which the UE first decoded them.

#### Step 3: Find Msg1 PRACH — 8 points

Run the following command on the UE log:

```bash
grep -nE 'PRACH' lab5-artifacts/Lab5-ue.log
```

Record the Msg1 log line number. From the line that shows the actual PRACH transmission, find:

```text
frame.slot = ______
preambleIndex = ______
```

Next, return to Wireshark. Clear the display filter, press `Cmd/Ctrl + F`, choose **Packet details** as the search scope, and search for:

```text
RACH Preamble chosen
```

After locating the PRACH event recorded by the gNB tracer, record:

```text
RAPID = ______
```

This packet is displayed as `MAC-LTE RACH Preamble chosen`; it is the OAI tracer MAC-event format.

**Student result — Task 1 / Step 3**

| Measurement | Observed value |
|---|---|
| UE log line for PRACH transmission | **81** |
| Frame.Slot | **553.19** |
| Preamble Index | **16** |
| Wireshark frame | **489** |
| Wireshark RAPID | **16** |
| Random Access attempt | **0** |

![PRACH transmission in UE log](images/task1-prach-log.png)

*Figure 4: UE log showing PRACH transmission at frame.slot 553.19 with preambleIndex 16.*

![PRACH preamble chosen in Wireshark](images/task1-prach-wireshark.png)

*Figure 5: Wireshark MAC-LTE tracer event showing RAPID 16 in frame 489.*

Submit one UE-log screenshot and one Wireshark screenshot. Briefly answer:

1. Which Random Access message is this PRACH transmission?
2. At which `frame.slot` and with which `preambleIndex` does the UE log show transmission?
3. Does the Wireshark `RAPID` match the UE-log `preambleIndex`? What does this show?

**Answers — Task 1 / Step 3**

**1. Which Random Access message is this PRACH transmission?**

The PRACH transmission is **Msg1**, the initial Random Access preamble of the four-step contention-based Random Access procedure.

The UE transmits a selected preamble to initiate access to the gNB.

**2. At which frame.slot and with which preambleIndex does the UE log show transmission?**

The UE log shows:

```text
frame.slot = 553.19
preambleIndex = 16
```

These values were recorded in log line 81.

**3. Does the Wireshark RAPID match the UE-log preambleIndex? What does this show?**

Yes.

The UE log reports `preambleIndex = 16`, and the Wireshark tracer reports `RAPID = 16`.

This confirms that the preamble recorded by the gNB tracer matches the preamble selected by the UE.

#### Step 4: Find Msg2 RAR in Wireshark — 11 points

Return to Wireshark and clear the display filter. Press `Cmd/Ctrl + F`, choose **Packet details** as the search scope, and search for `RAR` or `Random Access Response`. After locating the packet, record its frame number and find the following values in `MAC-NR RAR`:

```text
RAPID = ______
Temporary C-RNTI = ______
```

**Student result — Task 1 / Step 4**

| Field | Observed value |
|---|---|
| Wireshark frame | **490** |
| Message | Msg2 — MAC-NR RAR |
| RAPID | **16** |
| Temporary C-RNTI | **61982 (`0xF21E`)** |
| RA-RNTI | 271 |
| Timing Advance | 0 |

![Random Access Response](images/task1-rar.png)

*Figure 6: MAC-NR Random Access Response in frame 490, showing RAPID 16 and Temporary C-RNTI 61982.*

**Explanation:**

Msg2 is the Random Access Response (RAR) transmitted by the gNB.

It responds to the selected PRACH preamble and provides:

- A Random Access Preamble Identifier (RAPID).
- Timing Advance information.
- An uplink grant for Msg3.
- A Temporary C-RNTI.

The RAPID of 16 matches the PRACH preambleIndex identified in Msg1.

The Temporary C-RNTI `61982` is used in the following RRC connection establishment messages.

---

### Task 2: Complete Random Access and RRC Connection Setup — 40 points

Complete the following steps in order. For each message, provide a screenshot and record the requested frame number, fields, and brief explanation.

#### Step 1: Find RRCSetupRequest — 10 points

In Wireshark, use:

```text
nr-rrc.rrcSetupRequest_element
```

Expand `UL-CCCH-Message → rrcSetupRequest` and find:

```text
Frame number = ______
Context RNTI = ______
```

Briefly answer:

1. Does the `Context RNTI` match the `Temporary C-RNTI` in the Task 1 / Step 4 RAR? What does this show?
2. Can you claim that the RRC connection has been established after seeing only `RRCSetupRequest`? Why or why not?

**Student result — Task 2 / Step 1**

| Field | Observed value |
|---|---|
| Frame Number | **491** |
| Message | RRCSetupRequest (Msg3) |
| Context RNTI | **61982 (`0xF21E`)** |
| Logical channel | UL-CCCH |
| Establishment Cause | `mo-Signalling` |
| UE Identity | `randomValue` |

![RRCSetupRequest frame number](images/task2-rrc-setup-request-frame.png)

*Figure 7: Wireshark frame 491 containing RRCSetupRequest.*

![RRCSetupRequest details](images/task2-rrc-setup-request.png)

*Figure 8: RRCSetupRequest showing Context RNTI 61982 and establishmentCause mo-Signalling.*

**Answers:**

**1. Does the Context RNTI match the Temporary C-RNTI in the Task 1 / Step 4 RAR? What does this show?**

Yes.

The Context RNTI in the RRCSetupRequest is `61982`, and the Temporary C-RNTI assigned in the RAR is also `61982`.

This demonstrates that the UE uses the temporary identifier assigned during Random Access for its subsequent uplink transmission.

**2. Can you claim that the RRC connection has been established after seeing only RRCSetupRequest? Why or why not?**

No.

RRCSetupRequest is a request from the UE to the gNB.

The gNB must still send RRCSetup, and the UE must subsequently send RRCSetupComplete.

Therefore, observing only RRCSetupRequest does not prove that RRC connection establishment has been completed.

#### Step 2: Find RRCSetup — 10 points

Clear the previous filter and enter:

```text
nr-rrc.rrcSetup_element
```

Expand `DL-CCCH-Message → rrcSetup` and find:

```text
Frame number = ______
Context RNTI = ______
rrc-TransactionIdentifier = ______
```

Check whether `Contention Resolution` also appears in the packet.

Below the screenshot, briefly answer:

1. Which message is this in four-step Random Access?
2. What does it mean that `Contention Resolution` and `RRCSetup` appear in the same downlink message?

**Student result — Task 2 / Step 2**

| Field | Observed value |
|---|---|
| Frame Number | **492** |
| Message | RRCSetup (Msg4) |
| Context RNTI | **61982 (`0xF21E`)** |
| rrc-TransactionIdentifier | **1** |
| Contention Resolution | **Present** |

![RRCSetup frame number](images/task2-rrc-setup-frame.png)

*Figure 9: Wireshark frame 492 containing RRCSetup.*

![RRCSetup message details](images/task2-rrc-setup.png)

*Figure 10: RRCSetup with Transaction Identifier 1 and MAC Contention Resolution in the same downlink packet.*

**Answers:**

**1. Which message is this in four-step Random Access?**

This is **Msg4**, the fourth message of the four-step contention-based Random Access procedure.

It carries the RRCSetup message from the gNB to the UE.

**2. What does it mean that Contention Resolution and RRCSetup appear in the same downlink message?**

The gNB combines the contention-resolution information and RRCSetup in the same downlink transmission.

This allows the UE to determine whether contention resolution was successful while receiving the configuration required for RRC connection establishment.

The MAC layer handles contention resolution, while the RRC layer handles RRCSetup.

#### Step 3: Find RRCSetupComplete — 12 points

Clear the previous filter and enter:

```text
nr-rrc.rrcSetupComplete_element
```

Select the packet and first verify that `MAC-NR UL-SCH` appears at the top. Expand:

```text
RLC-NR → PDCP-NR → NR Radio Resource Control (RRC) protocol
  → UL-DCCH-Message → message: c1
```

Expand `message: c1` further and confirm that it contains `rrcSetupComplete`. Record:

```text
Frame number = ______
Context RNTI = ______
rrc-TransactionIdentifier = ______
```

Below the screenshot, briefly answer:

1. Does the `Context RNTI` still match the `Temporary C-RNTI` in the RAR?
2. Does the `rrc-TransactionIdentifier` match the `RRCSetup` in Step 2? What is the purpose of this comparison?

**Student result — Task 2 / Step 3**

| Field | Observed value |
|---|---|
| Frame Number | **498** |
| Message | RRCSetupComplete |
| Context RNTI | **61982 (`0xF21E`)** |
| rrc-TransactionIdentifier | **1** |
| Transport | MAC-NR UL-SCH |
| Logical channel | UL-DCCH |
| RLC | RLC-NR |
| PDCP | PDCP-NR |

![RRCSetupComplete message details](images/task2-rrc-setup-complete.png)

*Figure 11: RRCSetupComplete in frame 498, showing the UL-SCH transport, RLC-NR and PDCP-NR layers, Context RNTI 61982, and Transaction Identifier 1.*

**Answers:**

**1. Does the Context RNTI still match the Temporary C-RNTI in the RAR?**

Yes.

The Context RNTI in RRCSetupComplete is `61982`, matching the Temporary C-RNTI assigned in the Random Access Response.

This confirms that the RRCSetupComplete message belongs to the same UE context.

**2. Does the rrc-TransactionIdentifier match the RRCSetup in Step 2? What is the purpose of this comparison?**

Yes.

RRCSetup in frame 492 uses Transaction Identifier `1`.

RRCSetupComplete in frame 498 also uses Transaction Identifier `1`.

The matching identifier confirms that RRCSetupComplete corresponds to the correct RRCSetup transaction.

#### Step 4: Confirm the Final Random Access and RRC Connection Result — 8 points

Run the following command on the UE log:

```bash
grep -nE '4-Step RA procedure succeeded|Received NR_RRCSetup|RRCSetupComplete|State = NR_RRC_CONNECTED' \
  lab5-artifacts/Lab5-ue.log
```

From the output, record:

```text
4-Step RA procedure succeeded: line ______
Received NR_RRCSetup: line ______
State = NR_RRC_CONNECTED: line ______
Generating RRCSetupComplete: line ______
```

Submit a screenshot of this log output and briefly answer:

1. In log order, what are the three events from successful Random Access to generating `RRCSetupComplete`?
2. What do `4-Step RA procedure succeeded` and `State = NR_RRC_CONNECTED` each prove?

**Student result — Task 2 / Step 4**

The UE log contains the following events:

```text
87: 4-Step RA procedure succeeded. CBRA: Contention Resolution is successful.
88: Received NR_RRCSetup
90: State = NR_RRC_CONNECTED
93: Generating RRCSetupComplete (bytes33)
```

![Successful Random Access and RRC connection](images/task2-rrc-connected-log.png)

*Figure 12: UE log showing successful contention resolution, RRCSetup reception, transition to NR_RRC_CONNECTED, and RRCSetupComplete generation.*

**Answers:**

**1. In log order, what are the three events from successful Random Access to generating RRCSetupComplete?**

The three main stages are:

1. The four-step Random Access procedure succeeds, including contention resolution.
2. The UE receives RRCSetup and transitions to `NR_RRC_CONNECTED`.
3. The UE generates RRCSetupComplete to confirm the RRC setup transaction.

The relevant events appear in log lines 87, 88, 90 and 93.

**2. What do 4-Step RA procedure succeeded and State = NR_RRC_CONNECTED each prove?**

`4-Step RA procedure succeeded` confirms that the contention-based Random Access procedure has been completed successfully.

`State = NR_RRC_CONNECTED` confirms that the UE's RRC state machine has entered the connected state.

However, this does not by itself demonstrate successful NAS registration or PDU Session Establishment in the 5G Core.

---

### Task 3: Draw an Initial Access MSC Diagram — 25 points

Using the log and Wireshark evidence found in Tasks 1 and 2, draw your own Message Sequence Chart (MSC). It must include two vertical lifelines, `UE` and `gNB`, and the following six messages: MIB/SIB1, Msg1, Msg2, Msg3, Msg4, and `RRCSetupComplete`.

Determine and draw the correct order and direction of the messages yourself. Add a one-sentence purpose next to each arrow.

**Student result — Task 3 / Initial Access MSC**

![Initial Access Message Sequence Chart](images/task3-initial-access-msc.png)

*Figure 13: Initial Access Message Sequence Chart with UE and gNB lifelines, showing the order, direction and purpose of the messages from system information acquisition to RRC_CONNECTED.*

### Message sequence and purpose

| Order | Message | Direction | Purpose |
|---:|---|---|---|
| 1 | MIB / SIB1 | gNB → UE | Broadcast system information and common access configuration. |
| 2 | Msg1 PRACH | UE → gNB | Initiate Random Access by transmitting a selected preamble. |
| 3 | Msg2 RAR | gNB → UE | Provide timing advance, uplink grant and Temporary C-RNTI. |
| 4 | Msg3 RRCSetupRequest | UE → gNB | Request establishment of an RRC connection. |
| 5 | Msg4 RRCSetup + Contention Resolution | gNB → UE | Resolve contention and provide RRC connection configuration. |
| 6 | RRCSetupComplete | UE → gNB | Confirm completion of the RRC setup transaction. |

### Experimental evidence

| Message | Evidence |
|---|---|
| MIB | Wireshark frame 2 |
| SIB1 | Wireshark frame 1 |
| Msg1 PRACH | UE log line 81; Wireshark frame 489 |
| Msg2 RAR | Wireshark frame 490 |
| Msg3 RRCSetupRequest | Wireshark frame 491 |
| Msg4 RRCSetup | Wireshark frame 492 |
| RRCSetupComplete | Wireshark frame 498 |

### UE state transition

The UE starts in `RRC_IDLE` during initial access.

It first synchronizes with the gNB and obtains broadcast system information.

After successfully completing Random Access and receiving RRCSetup, the UE enters `RRC_CONNECTED`.

The UE then generates RRCSetupComplete to confirm the configuration transaction.

The UE log explicitly reports `State = NR_RRC_CONNECTED` at line 90.

RRC connection establishment must be distinguished from 5GC Registration, which is a separate NAS procedure.

| Item | Points |
|---|---:|
| UE and gNB lifelines, with the two UE states identified | 5 |
| Complete message order: MIB/SIB1, Msg1, Msg2, Msg3, Msg4, RRCSetupComplete | 12 |
| Correct direction for every message arrow | 5 |
| Reasonable brief purpose for every major stage | 3 |
| **Task 3 subtotal** | **25** |

| Item | Points |
|---|---:|
| Task 1: Initial Access evidence timeline | 35 |
| Task 2: Key-message analysis | 40 |
| Task 3: Initial Access MSC diagram | 25 |
| **Total** | **100** |

## 7. Optional: Reproduce the Log and PCAP Files

This section is not required for the assignment. If you want to generate the same kind of evidence using your own OAI run, follow these steps in an OAI/5GC environment that was already set up in an earlier class.

The commands below assume that you are currently in the `openairinterface5g` repository root. Use three terminals and first create a directory for the artifacts:

```bash
mkdir -p lab5-artifacts
```

### Terminal A: Start the gNB and Wait for the Tracer

```bash
cd cmake_targets/ran_build/build

sudo -E env RFSIMULATOR=server ./nr-softmodem \
  -O ../../../ci-scripts/conf_files/gnb.sa.band78.106prb.rfsim.conf \
  --rfsim --T_stdout 2 \
  2>&1 | tee ../../../lab5-artifacts/Lab5-gnb.log
```

### Terminal B: Record the gNB MAC/RRC Trace

```bash
cd cmake_targets/ran_build/build

./common/utils/T/tracer/record \
  -d ../../../common/utils/T/T_messages.txt \
  -o ../../../lab5-artifacts/Lab5-gnb-wireshark.raw \
  -on WIRESHARK
```

The gNB in Terminal A continues after the tracer connects.

### Terminal C: Start the UE and Save the UE Log

```bash
cd cmake_targets/ran_build/build

sudo -E RFSIMULATOR=127.0.0.1 ./nr-uesoftmodem \
  -O ../../../targets/PROJECTS/GENERIC-NR-5GC/CONF/ue.conf \
  --rfsim -r 106 --numerology 1 --band 78 -C 3619200000 \
  2>&1 | tee ../../../lab5-artifacts/Lab5-ue.log
```

After `State = NR_RRC_CONNECTED` appears, wait a few seconds, then stop the terminals in this order: recorder in Terminal B, UE in Terminal C, and gNB in Terminal A.

### Convert the Trace to a Wireshark PCAP

```bash
cd cmake_targets/ran_build/build

./common/utils/T/tracer/macpdu2wireshark \
  -d ../../../common/utils/T/T_messages.txt \
  -i ../../../lab5-artifacts/Lab5-gnb-wireshark.raw \
  -to-file ../../../lab5-artifacts/Lab5.pcap
```

You should obtain:

```text
lab5-artifacts/Lab5-ue.log
lab5-artifacts/Lab5-gnb.log
lab5-artifacts/Lab5.pcap
```
