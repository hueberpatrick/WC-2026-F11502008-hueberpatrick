
# Lab 3: Deploying a 5G Standalone Network Using OpenAirInterface

> **Scope:** Technical analysis of Lab 3 using the assignment and measurements from a reference implementation. The local VMware installation was not completed, so the experimental tables describe reference results rather than independently collected measurements. Screenshots are not included.

## 1. Goal

In the previous labs, you analyzed 5G signaling messages and network performance using provided captures and measurements.

In this lab, you will deploy your own 5G Standalone (SA) network using OpenAirInterface (OAI), verify the complete connection, investigate RFsimulator communication, perform active network measurements, and troubleshoot a controlled connection failure.

Unlike the previous labs, you will collect measurements directly from your own running 5G network.

**Learning Objectives**

After completing this lab, you should be able to:

1. Deploy an OAI gNB, nrUE, and 5G Core Network.
2. Identify the main stages of 5G SA connection establishment.
3. Verify communication between gNB and nrUE through RFsimulator.
4. Measure user-plane connectivity under different packet sizes.
5. Diagnose connection failures using OAI and 5GC logs.
6. Explain the limitations of a software-simulated 5G network.

## 2. Experimental Environment

Use the *OAI v2.4.0 Build Guide* to prepare the environment before class.

| Component | Configuration |
|---|---|
| Ubuntu | 24.04 (reference setup); 26.04 (local VMware environment) |
| OAI RAN | v2.4.0 |
| 5G Core | OAI CN5G |
| gNB | nr-softmodem |
| UE | nr-uesoftmodem |
| Radio interface | RFsimulator |
| Band | n78 |
| Subcarrier spacing | 30 kHz |
| Resource blocks | 106 |
| UE IMSI | 001010000000001 |
| External DN | 192.168.70.135 |

The reference experiment uses one Ubuntu machine without physical RF hardware. The local Ubuntu 26.04 VMware setup did not reach the OAI deployment stage.

## 3. Task 1 — Deploy and Verify the Complete 5G SA Network

Start the OAI CN5G:

```bash
cd ~/oai-cn5g
docker compose up -d
docker compose ps
```

Start the gNB and nrUE using the commands in the Build Guide.

Observe their terminal outputs and locate evidence of the following stages.

| Stage | Successfull? | Evidence from Logs |
|---|---|---|
| CN5G initialization | Yes | All 10 containers reported `Up` in `docker compose ps` |
| gNB NG Setup | Yes | `Received NGSetupResponse from AMF` |
| UE synchronization | Yes | `Initial sync successful, PCI: 0` |
| RRC connection establishment | Yes | `State = NR_RRC_CONNECTED` |
| UE Registration | Yes | Registration Accept / `5GMM-REGISTERED` |
| PDU Session Establishment | Yes | PDU Session Establishment Accept, UE IPv4 `10.0.0.2` |
| UE tunnel creation | Yes | Interface `oaitun_ue1` configured with `10.0.0.2/24` |

The reference logs indicate that the CN5G functions, gNB, and nrUE completed the necessary connection procedures. NG Setup established the connection between the gNB and AMF, while UE synchronization and random access enabled RRC connection establishment.

After authentication and registration, a PDU Session was established and an IPv4 address was assigned to the UE.

Take screenshots showing the gNB and UE successfully connected.

### Questions

**1. Does a successful NG Setup guarantee that a UE can register? Explain.**

No. NG Setup only confirms that the control-plane connection between the gNB and AMF is established. UE registration is a separate procedure requiring subscriber authentication, security verification, and valid network parameters. Even with successful NG Setup, registration can fail because of an unknown IMSI, incorrect authentication credentials, or unsupported network configuration.

**2. Why can the UE establish an RRC connection but still fail Registration?**

RRC connection establishment provides the control connection between the UE and gNB. However, it does not verify the subscriber identity. Registration is performed through NAS signaling involving the AMF, AUSF, and UDM, and can fail if the IMSI is unknown or authentication credentials are incorrect.

**3. What additional procedure is required after Registration before user data can be exchanged?**

After registration, PDU Session Establishment is required. This procedure assigns an IP address to the UE and establishes the user-plane connection between the gNB and UPF. It also configures the GTP-U tunnel and enables the `oaitun_ue1` interface for user-plane data transmission.

---

## 4. Task 2 — Investigate RFsimulator Communication

OAI RFsimulator replaces the physical RF interface with software-based communication.

The gNB and nrUE exchange IQ samples through a TCP connection.

Check the running processes:

```bash
pgrep -a nr-softmodem
pgrep -a nr-uesoftmodem
```

Inspect the TCP connections:

```bash
sudo ss -tnp | grep 4043
```

If necessary, inspect the listener:

```bash
sudo ss -ltnp | grep 4043
```

The default RFsimulator port in this experiment is 4043.

### Results

The process IDs are taken from one reference implementation and are specific to that run.

| Item | Observed Value |
|---|---|
| gNB process ID | 2963296 |
| nrUE process ID | 2963995 |
| RFsimulator transport protocol | TCP |
| RFsimulator port | 4043 |
| TCP connection state | ESTAB (ESTABLISHED) |
| Connected processes | `nr-softmodem` (PID 2963296), `nr-uesoftmodem` (PID 2963995) |

The RFsimulator uses a local TCP connection on port 4043. The gNB acts as the server while the nrUE connects as a client. This communication method replaces the physical radio channel with digital IQ sample exchange.

### Questions

**1. Why does RFsimulator use a TCP connection instead of a physical antenna?**

RFsimulator allows the gNB and nrUE to communicate without physical RF hardware. Instead of transmitting radio signals through antennas, the two processes exchange digital IQ samples through a local TCP connection. TCP provides reliable and ordered communication, making it suitable for testing the 5G protocol stack in a controlled software environment.

**2. Are the TCP packets exchanged by RFsimulator the same as the UE's application-layer IP packets?**

No. RFsimulator TCP packets contain raw IQ samples representing the simulated radio signal. Application-layer IP packets, such as ICMP packets generated by ping, are processed by the 5G protocol stack and transmitted through the established PDU Session. Therefore, RFsimulator TCP traffic and user-plane IP traffic serve different purposes.

**3. Does using RFsimulator mean that the UE and gNB are performing no physical-layer processing? Explain.**

No. Both `nr-softmodem` and `nr-uesoftmodem` still perform physical-layer processing in software. This includes channel coding, OFDM modulation and demodulation, synchronization, and signal decoding. RFsimulator only replaces the physical radio hardware and wireless propagation channel.

---

## 5. Task 3 — Verify End-to-End Connectivity

Check the UE tunnel interface:

```bash
ip -4 addr show oaitun_ue1
```

Test connectivity to the External DN:

```bash
ping -I oaitun_ue1 -c 10 192.168.70.135
```

### Results

The following measurements are from the same reference implementation as Task 2.

| Measurement | Observed Value |
|---|---|
| UE IP address | 10.0.0.2 |
| Destination IP | 192.168.70.135 |
| Packets transmitted | 10 |
| Packets received | 10 |
| Packet loss (%) | 0% |
| Minimum RTT (ms) | 13.733 |
| Average RTT (ms) | 20.365 |
| Maximum RTT (ms) | 25.283 |

The reference measurement shows that all 10 ICMP packets were received successfully, resulting in 0% packet loss. The average round-trip time was 20.365 ms.

### Questions

**1. Why must Ping use the `oaitun_ue1` interface in this experiment?**

The `oaitun_ue1` interface represents the UE's 5G data connection. Specifying this interface ensures that ICMP packets travel through the simulated 5G user plane rather than through the host computer's regular network connection. Without this parameter, the ping could bypass the gNB and UPF and would not properly verify end-to-end 5G connectivity.

**2. Does successful Registration guarantee successful user-plane connectivity?**

No. Registration only confirms that the UE has been authenticated and accepted by the 5G Core. User-plane connectivity additionally requires PDU Session Establishment, correct GTP-U tunnel configuration, and IP routing through the UPF. Therefore, a UE can successfully register but still be unable to exchange application data.

**3. What does a successful Ping prove, and what does it not prove about the simulated 5G system?**

A successful ping demonstrates that the user-plane connection between the UE and external data network is functioning. It indicates that the gNB, UPF, and associated network interfaces are correctly forwarding packets. However, it does not prove realistic over-the-air performance because RFsimulator does not reproduce all wireless propagation effects, interference, or mobility conditions.

---

## 6. Task 4 — Investigate the Effect of Packet Size

You will now investigate whether changing the ICMP packet size affects the observed RTT.

Perform three experiments using the same established 5G connection.

### Scenario A — Small Packets

```bash
ping -I oaitun_ue1 -s 56 -c 30 -i 0.2 192.168.70.135
```

### Scenario B — Medium Packets

```bash
ping -I oaitun_ue1 -s 512 -c 30 -i 0.2 192.168.70.135
```

### Scenario C — Large Packets

```bash
ping -I oaitun_ue1 -s 1200 -c 30 -i 0.2 192.168.70.135
```

Note: `-s` specifies ICMP payload size, not the complete IP packet size.

### Results

The following table contains reference measurements for 30 ICMP packets per scenario.

| Scenario | ICMP Payload | Avg. RTT (ms) | Max. RTT (ms) | Packet Loss (%) |
|---|---:|---:|---:|---:|
| A | 56 bytes | 21.299 | 24.540 | 0% |
| B | 512 bytes | 23.104 | 26.592 | 0% |
| C | 1200 bytes | 24.807 | 30.321 | 0% |

### RTT Difference

```text
RTT Difference = Average RTT(C) - Average RTT(A)

RTT Difference = 24.807 - 21.299

RTT Difference = 3.508 ms
```

### Questions

**1. Which scenario has the highest average RTT?**

Scenario C, using an ICMP payload size of 1200 bytes, has the highest average RTT of 24.807 ms.

**2. Does increasing packet size always increase RTT in your measurements?**

In the reference measurements, the average RTT increased as packet size increased. The average RTT was 21.299 ms for 56-byte packets, 23.104 ms for 512-byte packets, and 24.807 ms for 1200-byte packets. However, this does not mean that larger packets always produce higher RTT values. Individual measurements can fluctuate because of CPU load, scheduling delays, and software processing.

**3. Why might a larger packet require more transmission or processing resources?**

Larger packets contain more data and therefore require additional processing resources. More bits must be encoded, modulated, transmitted, received, and decoded by the protocol stack. Depending on the radio configuration, larger packets can also require additional resource blocks or scheduling opportunities.

**4. What factors other than packet size could affect RTT in a software-based 5G network?**

RTT can be affected by CPU utilization, operating system scheduling, background processes, Docker networking overhead, and TDD slot timing. The software-based RFsimulator connection and virtual network interfaces can also introduce delays. Because the components run on the same machine, host system performance significantly influences the results.

**5. Can these results be interpreted as real over-the-air 5G latency? Explain.**

No. RFsimulator replaces physical radio transmission with a software-based TCP connection. It does not reproduce all real wireless effects, such as fading, interference, and hardware impairments. Therefore, the measured RTT primarily reflects software processing and scheduling rather than actual over-the-air 5G latency.

Do not assume that RTT must increase with packet size. Use your measured values to support your explanation.

---

## 7. Task 5 — Diagnose a Controlled UE Registration Failure

In this task, you will deliberately introduce one configuration error and investigate why the UE cannot complete registration.

**Important:** Save the original successful UE command before starting. Do not modify the CN5G subscriber database.

### Step 1 — Stop the UE

Press `Ctrl+C` in the nrUE terminal. Keep the CN5G and gNB running.

### Step 2 — Launch the UE with an Incorrect IMSI

Change only the IMSI:

```bash
sudo ./nr-uesoftmodem \
  -r 106 \
  --numerology 1 \
  --band 78 \
  -C 3619200000 \
  --uicc0.imsi 001010000000099 \
  --rfsim \
  --rfsimulator.serveraddr 127.0.0.1
```

The IMSI `001010000000099` is not provisioned in the standard classroom subscriber configuration. Verify this assumption against the instructor's database before class.

Observe the UE and AMF logs:

```bash
cd ~/oai-cn5g
docker compose logs --since=2m oai-amf
```

Allow time for registration attempts, then stop the UE.

### Reference Results

| Item | Observation |
|---|---|
| Modified parameter | IMSI changed from `001010000000001` to `001010000000099` |
| UE synchronized with gNB? | Yes |
| RRC connection established? | Yes |
| Registration successful? | No |
| Error or rejection evidence | `FGS_REGISTRATION_REJECT` received by nrUE |
| Likely cause | IMSI `001010000000099` was not provisioned in the subscriber database |

### Reference Log Evidence

The reference UE log contained the following relevant messages:

```text
[SIM] UICC simulation: IMSI=001010000000099
[HW] Connection to 127.0.0.1:4043 established
[PHY] Initial sync successful, PCI: 0
[NR_RRC] SIB1 decoded
[NR_RRC] State = NR_RRC_CONNECTED
[NAS] Generate Initial NAS Message: Registration Request
[NAS] Received NAS_DOWNLINK_DATA_IND type FGS_REGISTRATION_REJECT
```

The reference AMF log reported:

```text
Received IMSI 001010000000099
Get response with HTTP code (404)
CONTEXT_NOT_FOUND
Request Authentication Vectors failure
5GMM-DEREGISTERED
Registration Reject
```

These messages indicate that the UE could establish the simulated radio connection but was unable to complete subscriber authentication.

### Step 3 — Restore the Original IMSI

Restart the nrUE using:

```text
--uicc0.imsi 001010000000001
```

Confirm that the UE successfully registers again and that Ping works.

### Reference Recovery Results

| Check | Incorrect IMSI | Correct IMSI |
|---|---|---|
| IMSI | 001010000000099 | 001010000000001 |
| Authentication | Failed | Successful |
| Registration state | 5GMM-DEREGISTERED | 5GMM-REGISTERED |
| PDU Session | Not established | Established |
| UE IP address | Not assigned | 10.0.0.2 |
| User-plane connectivity | Unavailable | Available |
| Recovery ping | Not possible | 10/10 received, 0% packet loss |
| Recovery average RTT | — | 20.441 ms |

### Questions

**1. Why can the UE potentially synchronize with the gNB even when the IMSI is incorrect?**

UE synchronization occurs at the physical layer and depends on synchronization signals transmitted by the gNB. These signals, including PSS and SSS, do not require subscriber authentication. Therefore, the UE can synchronize with the cell even when its IMSI is not accepted by the 5G Core.

**2. Which network function is involved in UE authentication?**

The Authentication Server Function (AUSF) supports subscriber authentication in the 5G Core. It works with the Unified Data Management (UDM), which provides subscriber authentication information. The AMF coordinates the registration procedure and exchanges authentication messages with these functions.

**3. Why does changing an IMSI affect Registration but not the RFsimulator TCP connection itself?**

The RFsimulator TCP connection transports IQ samples between the simulated gNB and UE independently of subscriber identity. The IMSI is evaluated at the NAS layer during authentication by the core network. As a result, changing the IMSI can prevent registration while leaving the RFsimulator TCP connection operational.

**4. What evidence proves that restoring the correct configuration solved the problem?**

The reference implementation showed successful NAS registration after restoring IMSI `001010000000001`. The UE received IP address `10.0.0.2` through the established PDU Session, and the `oaitun_ue1` interface became available. A subsequent ping to `192.168.70.135` succeeded with 0% packet loss, confirming that user-plane connectivity had been restored.

Submit evidence from both the failed and recovered connections.

---

## 8. Task 7 — Explain the Relationship Between Simulation and Real 5G

Your OAI gNB and nrUE perform 5G NR protocol processing even though they do not use physical antennas.

### Comparison

| Feature | OAI RFsimulator | Physical 5G System |
|---|---|---|
| gNB and UE protocol stack | Full software implementation of 5G NR | Full protocol stack using dedicated hardware and software |
| OFDM signal processing | Performed in software | Performed using specialized baseband hardware |
| Physical antennas | Replaced by TCP-based IQ sample exchange | Required for wireless transmission and reception |
| RF hardware impairments | Generally absent by default | Phase noise, IQ imbalance, amplifier nonlinearities |
| Wireless propagation effects | Absent by default unless channel modeling is enabled | Path loss, fading, interference, Doppler effects |
| 5G Core connectivity | OAI CN5G through NGAP and GTP-U | Production 5G Core through NGAP and GTP-U |

### Questions

**1. Which parts of the 5G system remain functional when physical RF hardware is replaced with RFsimulator?**

The complete 5G protocol stack remains functional, including PHY, MAC, RLC, PDCP, RRC, and NAS processing. The 5G Core functions, such as AMF, SMF, UPF, AUSF, and UDM, also remain operational. Only the physical radio transmission and reception are replaced by software-based IQ sample exchange.

**2. Why is RFsimulator useful for testing the 5G protocol stack?**

RFsimulator provides a controlled and reproducible environment for testing 5G protocol behavior. It eliminates the need for expensive SDR equipment, physical antennas, and licensed radio spectrum for basic software experiments. This simplifies debugging and makes it easier to analyze signaling, registration, and user-plane communication.

**3. Why might latency and throughput differ between RFsimulator and a real 5G deployment?**

RFsimulator performance depends on CPU speed, operating system scheduling, and software implementation. Real 5G networks also experience wireless interference, fading, mobility, and RF hardware processing delays. These additional factors can influence retransmissions, packet loss, throughput, and latency.

**4. What would need to change if you wanted to test a physical UE instead of OAI nrUE?**

A physical UE would require a real radio connection to the gNB. RFsimulator would need to be replaced by supported SDR hardware, such as a USRP, together with appropriate antennas and frequency configuration. Additionally, the UE would require a compatible SIM or USIM containing subscriber credentials matching those configured in the 5G Core. Radio operation would also need to comply with applicable spectrum regulations.

---

## 9. Final Analysis

Answer each question in three to five sentences.

### 1. Describe the complete process from starting the OAI 5GC to successful user-plane connectivity.

The process begins by starting the OAI CN5G containers using Docker Compose, which initializes the required core network functions. The gNB then establishes an NGAP connection with the AMF through NG Setup. Next, the nrUE connects through RFsimulator, synchronizes with the gNB, performs random access, and establishes an RRC connection. After successful authentication and registration, the UE establishes a PDU Session and receives IP address `10.0.0.2`. Finally, end-to-end connectivity can be verified by sending ICMP packets through the `oaitun_ue1` interface to the external data network.

### 2. Explain the difference between an RFsimulator TCP connection and a 5G PDU Session.

The RFsimulator TCP connection transports digitized IQ samples between the gNB and nrUE, replacing the physical radio interface. It operates independently of subscriber registration and user-plane session management. A PDU Session is a logical 5G connection established by the core network that enables IP data transmission between the UE and an external data network. It includes an assigned UE IP address and user-plane forwarding through the UPF.

### 3. Based on the reference measurements, how did packet size affect RTT and packet loss?

The reference results show that average RTT increased as ICMP payload size increased. Average RTT rose from 21.299 ms for 56-byte packets to 23.104 ms for 512-byte packets and 24.807 ms for 1200-byte packets. The overall difference between the smallest and largest payload sizes was 3.508 ms, while packet loss remained at 0% across all scenarios. These results are consistent with larger packets requiring more processing resources, although individual RTT values can also be influenced by system scheduling.

### 4. Describe the reference registration failure and explain how it was diagnosed and recovered from.

The controlled failure was introduced by changing the UE IMSI from `001010000000001` to `001010000000099`. The UE could still synchronize with the gNB and establish an RRC connection, but the core network rejected registration because the subscriber was not provisioned. The failure was identified through NAS Registration Reject messages and authentication errors in the AMF logs. After restoring the original IMSI, registration and PDU Session Establishment succeeded, and user-plane connectivity was confirmed through a successful ping.

### 5. Identify two limitations of this simulated environment and explain how they might affect experimental results.

One limitation is the absence of realistic wireless propagation effects such as fading, path loss, and interference unless additional channel modeling is enabled. This can produce more stable radio conditions than those encountered in real deployments. Another limitation is that the complete network operates in software on a general-purpose computer, making its performance sensitive to CPU utilization and operating system scheduling. Consequently, measured RTT and throughput may differ significantly from those of a physical 5G system.

---

## 10. Submission Checklist

The laboratory submission requirements are:

- Completed 5G SA connection verification table.
- gNB and nrUE connection evidence.
- RFsimulator TCP connection evidence.
- Baseline ping measurement.
- Three packet-size comparison results.
- RTT difference calculation.
- Controlled failure and recovery analysis.
- RFsimulator versus physical 5G comparison.
- Answers to the final analysis questions.

The numerical experimental tables in this report are based on a supplied reference implementation. Independent measurements and screenshots from the local VMware environment were not obtained.

**Local environment limitation:** Ubuntu 26.04 was launched in VMware, but the OAI deployment could not be completed with the available VM resources and setup time.

## 11. Grading Rubric

| Item | Points |
|---|---:|
| Complete 5G SA deployment | 15 |
| RFsimulator TCP connection analysis | 15 |
| End-to-end user-plane verification | 15 |
| Packet-size experiment and analysis | 20 |
| Controlled failure and recovery | 20 |
| Simulation limitations and final analysis | 15 |
| **Total** | **100** |
