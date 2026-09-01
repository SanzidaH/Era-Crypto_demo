# ERA-Crypto - Live Demonstration Manual

**Document type:** Presentation reference manual
**Scenario:** Adaptive operation with RIC (dual-axis)
**Revision:** demo-1.1
**Intended use:** On-hand reference for any presenter operating the ERA-Crypto testbed before an audience.

---

## 1. Purpose and Scope

This manual documents the procedure for presenting the ERA-Crypto adaptive cryptographic-agility system to an audience. It covers system initialization, the three demonstrations that constitute the presentation, verification of post-quantum operation, and recovery from common faults.

It is deliberately limited to the **adaptive-with-RIC** scenario. The baseline, matrix, without-RIC, and floor-sweep procedures used during data collection are out of scope and are documented separately.

The manual is organized so that any individual section can be located and consulted independently during a live session. It is not intended to be read sequentially at presentation time.

## 2. Intended Audience

The reader is a presenter or operator familiar with the testbed at a working level. No step assumes authorship of the system, so the manual is suitable for a colleague presenting on the author's behalf.

## 3. Conventions

The following conventions are used throughout.

- **Command** - a block to be executed on the terminal identified by its tag (see Section 6). Commands are reproduced verbatim; no field is a placeholder.
- **Expected result** - the output that confirms the step completed correctly.
- **Recovery** - the corrective action to apply when the expected result does not appear.
- **Note** - supporting information.
- **Caution** - information whose omission may compromise the demonstration.
- **Talking point** - suggested narration for the audience. Talking points are advisory; the wording may be adapted freely.

Terminal tags take the form of a letter and number. **V** denotes a terminal on the core VM, **S** a terminal on the server Raspberry Pi (ue1), and **C** a terminal on the client Raspberry Pi (ue2).

## 4. System Overview

The demonstration establishes three properties of the system, each observable on a single displayed terminal:

1. **Battery-driven signature selection.** The device energy state selects between Falcon-512 and ML-DSA-44.
2. **Throughput-driven KEM selection.** The RAN throughput reported by the RIC selects between ML-KEM-768 and ML-KEM-1024.
3. **A hard post-quantum floor.** Classical primitives (X25519, ECDSA) are structurally unreachable rather than merely deselected; a request for them is rejected by the policy envelope.

The signature and KEM axes operate independently, and the floor holds regardless of either.

**Operating principle.** Stack initialization (Section 8) is completed in full before the audience is present. During the presentation, only the interactive controls of Sections 10–12 are exercised. Initializing the stack before an audience introduces avoidable delay and failure exposure and is not part of the presentation.

## 5. Terminology and Fixed Parameters

| Term | Meaning |
|------|---------|
| Decision service | The policy endpoint (`decision.py`) that selects algorithms and enforces the envelope, on the VM at `:8080`. |
| Envelope | The policy that renders classical algorithms unreachable. |
| Feed | The RIC-to-CSV telemetry pipeline (`start_feed.sh`) supplying live throughput. |
| Agent | The per-UE process that reports battery state and applies the decision. |
| Bench | The measured handshake workload (`pqc_bench.sh`). |

The following values are fixed for this testbed and appear throughout.

```
VM    srsran@192.168.0.4     decision service on :8080
ue1   192.168.0.10   RAN 10.45.1.2   server   netns ue1   interface tun_srsue
ue2   192.168.0.20   RAN 10.45.1.3   client   netns ue2   interface tun_srsue
ue3   192.168.0.30   RAN 10.45.1.4   client (optional second client)
Ports: falcon512 -> 4433 ; mldsa44 -> 4434
```

## 6. Terminal Allocation

Twelve terminals are used. Nine are established during initialization and remain running unattended; three are operated during the presentation, together with one VM terminal reused for the closing step.

| Tag | Machine | Function | Role during presentation |
|-----|---------|----------|--------------------------|
| V1 | VM | Stack bring-up; later reused for the security demonstration | Reused in Section 12 |
| V2 | VM | gNB (foreground) | Unattended |
| V3 | VM | RIC feed | Unattended |
| V4 | VM | Decision service | Unattended |
| V5 | VM | Audit-log display | **Designated display terminal** |
| S1 | ue1 | UE attachment (server) | Unattended |
| S2 | ue1 | TLS servers on both ports | Unattended |
| C1 | ue2 | UE attachment (client) | Unattended |
| C2 | ue2 | Agent and measured workload | Unattended |
| C3 | ue2 | Battery control | Operated in Section 11 |
| C4 | ue2 | RAN-load control | Operated in Section 11 |
| C5 | ue2 | Wire-level verification | Operated in Section 10 |

The audit-log terminal (V5) is the terminal shown to the audience. Its font should be enlarged before the session.

## 7. Pre-Presentation Preparation

The following conditions are confirmed before initialization begins.

```
[ ] net.sctp.auth_enable=0 is set on every machine
[ ] The shutdown procedure (Section 13) has been run once on every machine to clear stale state
[ ] The gNB configuration path is correct: ~/configs/gnb_ocudu_zmq_pi_ric.yaml
[ ] The RAN will remain idle except for the intentional load control in Section 11
[ ] The audit-log display (V5) is legible at audience distance
[ ] The intended KEM-axis method (physical load or controlled injection) has been rehearsed
```

## 8. Initialization Procedure

Each step below is completed before the audience arrives and is confirmed by its expected result before the next is begun.

### 8.1 Core, RIC, and broker

The bring-up script starts the 5G core, the RIC, and the GNU Radio broker in the background and reports the state of each.

**Command (V1)**
```
./vm_up.sh
```
**Expected result** - the terminal reports `CORE + RIC + BROKER UP`.
**Recovery** - if a component is absent, `docker ps` identifies which; the script may be re-run.

Core (Open5GS / AMF) output:
```
cd ~/ocudu/docker
docker compose logs 5gc 2>&1 | grep -iE "ng.?setup|gNB-N2 accepted|Number of gNBs|Registration complete"
```

### 8.2 gNB

The gNB is started in its own foreground terminal. Backgrounding it has previously masked a degraded startup, so it is never backgrounded.

**Command (V2)**
```
cd ~/ocudu/build_e2/apps/gnb && sudo ./gnb -c ~/configs/gnb_ocudu_zmq_pi_ric.yaml
```
With logs:

```
sudo ./gnb -c ~/configs/gnb_ocudu_zmq_pi_ric.yaml 2>&1 | tee ~/era_run_logs/gnb.log
```

**Expected result** - the log reports `N2: Connection to AMF ... completed` and `==== gNB started ====`.
**Recovery** - on a partial start, `sudo pkill -9 gnb` clears it, after which it is relaunched in the foreground.

### 8.3 Server attachment (ue1)

**Command (S1)**
```
cd ~ ; ./pi_cleanup.sh ue1 ; ./pi_attach.sh ue1
```
**Expected result** - `RRC Connected`, followed by IP `10.45.1.2`. The address is confirmed with `sudo ip netns exec ue1 ip addr show tun_srsue | grep 10.45`.
**Note** - ICMP is filtered on this stack; reachability is confirmed by a successful handshake, not by ping.
**Recovery** - the cleanup and attach commands are re-run.

### 8.4 Client attachment (ue2)

**Command (C1)**
```
cd ~ ; ./pi_cleanup.sh ue2 ; ./pi_attach.sh ue2
```
**Expected result** - `RRC Connected`, IP `10.45.1.3`.
**Recovery** - as in Section 8.3, substituting `ue2`.

**Note** - an optional second client is attached on ue3 (`./pi_cleanup.sh ue3 ; ./pi_attach.sh ue3`, address `10.45.1.4`).

### 8.5 RIC feed

The feed is the most delicate component and is therefore brought up after the RAN and UEs are established.

**Command (V3)**
```
~/start_feed.sh
```
**Expected result** - `[csv] {...}` lines appear approximately once per second with continuous timestamps.
**Recovery** - if the address is busy or no lines appear:
```
cd ~/oran-sc-ric && docker compose restart python_xapp_runner && sleep 5
~/start_feed.sh
```

### 8.6 Decision service

**Command (V4)**
```
: > ~/era_audit.log
python3 ~/decision.py
```
**Expected result** - `[PASS] envelope OK ...` and `[PASS] listening on 0.0.0.0:8080  NO_RIC=False`.
**Recovery** - on `address already in use`, `pkill -f decision.py` clears the port before re-running.

### 8.7 Audit-log display

**Command (V5)**
```
tail -f ~/era_audit.log
```
This terminal is the one presented to the audience; its font is enlarged.

### 8.8 Verification (optional)

**Command (V1)**
```
./verify_vm.sh
```
**Expected result** - all checks pass, including `RAN feed fresh` and `downgrade prevented`. V1 is left free after this step for reuse in Section 12.

### 8.9 TLS servers

**Command (S2)**
```
cd ~ ; ./ue1_servers.sh
```
**Expected result** - `[PASS] falcon512:4433` and `[PASS] mldsa44:4434`.

### 8.10 Agent and measured workload

**Command (C2)**
```
cd ~
./agent.sh ue2 &
sleep 3
cat /tmp/current_kem /tmp/current_sig
sudo ./pqc_bench.sh ue2 ue2 10.45.1.2 300 adapt
```
**Expected result** - the `cat` reports a KEM and a signature (for example `mlkem1024` and `mldsa44`); the bench then begins handshaking.
**Recovery** - if the battery reads `0%` or the `cat` output is empty, the battery state file is reset and the agent restarted:
```
rm -f /tmp/vbattery.json
pkill -f agent.sh ; ./agent.sh ue2 & ; sleep 3 ; cat /tmp/current_kem /tmp/current_sig
```

Initialization is complete once the audit-log display is scrolling decisions and the bench is handshaking.

## 9. Establishing the Credibility of the Live System

At the opening of the presentation, the live and physical nature of the testbed is established from the already-running state rather than by rebuilding it. Three observations suffice.

- The gNB log (V2) shows `==== gNB started ====` and the UE RRC-context entries, evidencing a live O-RAN gNB with attached UEs.
- The client's RAN address is displayed: `sudo ip netns exec ue2 ip addr show tun_srsue | grep 10.45`, confirming a physical Raspberry Pi holding a real subscriber address rather than a loopback.
- The feed (V3) shows telemetry lines arriving continuously, evidencing live KPM measurement from the RIC.

**Talking point** - "The gNB, core, and RIC are real; the UEs are physical devices on the RAN; the network state shown here is measured, not scripted."

## 10. Verifying Post-Quantum Operation on the Wire

This section evidences that the handshake carries post-quantum material rather than a labelled record only.

**Caution** - In TLS 1.3 the KEM key exchange is transmitted in the clear, so packet capture genuinely exhibits the ML-KEM public key. The signature, by contrast, is encrypted within the handshake and cannot be identified from a capture. The endpoint output in Section 10.2 is therefore the authoritative source for the signature. Presenting this distinction accurately is expected of the presenter.

### 10.1 Observing the handshake in transit

**Command (C5)**
```
sudo ip netns exec ue2 tcpdump -i tun_srsue -n -c 30 'tcp port 4433 or tcp port 4434'
```
**Expected result** - a burst of TLS records in which the handshake packets are conspicuously large.
**Talking point** - "The oversized handshake records are the ML-KEM key material; a classical exchange would be a fraction of this size."

### 10.2 Identifying the primitives at the endpoint

**Command (C5)**
```
cd ~/Downloads/boringssl/build
sudo ip netns exec ue2 ./tool/bssl client -curves mlkem768 -sigalgs falcon512 \
  -connect 10.45.1.2:4433 </dev/null | grep -iE "ECDHE group|Signature"
```
**Expected result** - `ECDHE group: mlkem768` and `Signature algorithm: falcon512`.
**Talking point** - "Key exchange and signature, both post-quantum, confirmed at the endpoint."

### 10.3 Capturing to file (optional)

A capture may be retained as evidence or as a fallback should the live handshake misbehave.

**Command (C5)**
```
sudo ip netns exec ue2 tcpdump -i tun_srsue -w ~/demo_pqc.pcap 'tcp port 4433 or tcp port 4434' &
# a single handshake is run (Section 10.2), after which:
sudo pkill -9 tcpdump
```
**Recovery** - a permission or namespace error indicates the `sudo ip netns exec ue2` prefix or the `tun_srsue` interface name is missing.

## 11. Demonstrating Adaptive Behaviour

Both axes are observed on the audit-log display (V5). The battery axis is the more robust of the two and is presented first.

### 11.1 Signature axis (battery)

Adjusting the reported battery level changes the input consumed by the agent; below the threshold, the decision service selects the lower-power signature.

**Command (C3)**
```
python3 ~/set_battery.py 15
```
**Expected result** - within a few handshakes, the signature changes on the display.
**Talking point** - "As the battery falls, the signature de-escalates automatically."

The level is then restored:
```
python3 ~/set_battery.py 90
```
**Expected result** - the signature returns to the stronger scheme. The transition may be repeated for emphasis.
**Recovery** - if the signature does not respond, the battery state is reset and the agent restarted:
```
rm -f /tmp/vbattery.json ; pkill -f agent.sh ; ./agent.sh ue2 & ; sleep 3
python3 ~/set_battery.py 15
```

### 11.2 KEM axis (RAN throughput)

Increasing RAN throughput causes the decision service to select the stronger KEM.

**Command (C4)**
```
sudo ip netns exec ue2 iperf3 -c 10.45.1.2 -t 120 -R -b 500K
```
**Expected result** - as throughput rises, the KEM changes on the display; ending the load returns it.
**Talking point** - "As network throughput rises, the key exchange steps up to the stronger parameter set."

**Caution** - the emulated link reports approximately zero throughput for control traffic under TS 28.552, so injected load may not move the KEM on screen. Where the axis does not respond within a short interval, the controlled `ran_log.csv` throughput-injection method is used instead. The chosen method is rehearsed in advance; it is not selected during the session.

## 12. Demonstrating the Security Floor

A single request evidences that classical cryptography cannot be reached.

**Command (V1)**
```
curl -s -X POST http://192.168.0.4:8080/decide -H 'Content-Type: application/json' \
  -d '{"ue":"t","battery_pct":90,"force_group":"X25519","force_sig":"ecdsa"}'
```
**Expected result** - the KEM remains post-quantum and the response contains `"sig_reason":"reject(ecdsa):outside-envelope"`; with a fresh feed it also contains `"kem_reason":"reject(X25519):outside-envelope"`.
**Talking point** - "Even an explicit demand for classical cryptography is rejected. The system is adaptive, but never below the post-quantum boundary."

## 13. Shutdown Procedure

The following restores a clean state and is safe to run at any time.

```
# VM
pkill -9 -f decision.py ; pkill -9 -f kpm_to_csv ; pkill -9 -f kpm_mon_xapp
# ue1
sudo pkill -9 bssl ; pkill -9 -f ue1_servers
# ue2 / ue3
sudo pkill -9 bssl ; pkill -9 -f agent.sh ; pkill -9 -f pqc_bench ; sudo pkill -9 tcpdump
```

---

## Appendix A - Recovery Reference

| Condition | Corrective action |
|-----------|-------------------|
| Battery reads 0% or signature does not respond | `rm -f /tmp/vbattery.json`; restart the agent (`pkill -f agent.sh ; ./agent.sh ue2 & ; sleep 3`) |
| Feed shows no `[csv]` lines, or address busy | `cd ~/oran-sc-ric && docker compose restart python_xapp_runner && sleep 5`; re-run `~/start_feed.sh` |
| Decision service reports address in use | `pkill -f decision.py`; re-run `python3 ~/decision.py` |
| A UE detaches or fails to attach | `./pi_cleanup.sh ueN ; ./pi_attach.sh ueN` (reachability is confirmed by handshake, not ping) |
| gNB starts in a degraded state | `sudo pkill -9 gnb`; relaunch in the foreground |
| KEM axis does not respond to load | Expected under TS 28.552; use the `ran_log.csv` injection method |
| Latency reported in seconds rather than milliseconds | The RAN is not idle; clear stray load, as the emulated-link throttle contaminates timing |
| Indeterminate state | Run the shutdown procedure (Section 13) on every machine, then reinitialize from Section 8 |

`net.sctp.auth_enable=0` is maintained on every machine throughout.

## Appendix B -  Some FAQ

- **Whether the network is real or simulated.** The gNB, core, and RIC are real; the radio is a ZMQ SDR emulation; the UEs are physical Raspberry Pi 5 devices attached to the RAN.
- **Whether the KEM throughput is real load.** The method in use - physical load or controlled injection - is stated plainly. The emulated control-plane link reports approximately zero throughput under TS 28.552, so the KEM axis is driven by a controlled throughput signal; the battery axis and the floor rejection are exercised live.
- **How post-quantum operation is evidenced.** Section 10: the ML-KEM key material is visible and oversized in capture, and the endpoint output identifies both primitives.
- **What prevents a downgrade.** Section 12: the envelope rejects classical requests structurally rather than as a configurable preference.
- **Why RAN-plane latency or energy appears unusual.** The throttled emulated link dominates; handshake computation is negligible against it. This is treated explicitly in the paper.

## Appendix C - Summary

1. On the audit-log display: "Two inputs, two cryptographic axes, one hard floor."
2. Battery axis (Section 11.1): "As the battery falls the signature de-escalates; as it recovers it escalates again."
3. KEM axis (Section 11.2): "As throughput rises the key exchange steps up to the stronger set."
4. Wire verification (Section 10): "The post-quantum key material is visible in transit, and both primitives are confirmed at the endpoint."
5. Security floor (Section 12): "An explicit request for classical cryptography is rejected - adaptive, but never below post-quantum."
