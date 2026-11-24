# TCP Congestion Control under Network Impairments: Reno vs CUBIC vs BBR

**CS3001 Computer Networks — Final Project**
**Team:** Bismah Sheikh, Adeel Intekhab, Fatima Haider

We compare three TCP congestion-control algorithms (Reno, CUBIC and BBR) on a small virtual network built with Mininet. Traffic is sent with iperf3 under five network conditions (normal, 1% loss, +50 ms delay, a 5 Mbit/s bandwidth limit, and 1% loss + 50 ms delay). Every test runs 3 times, and we measure throughput, retransmissions, congestion window, queueing delay and fairness. Packet captures are analysed in Wireshark.

This README lets another group rebuild the setup, run the live proof of concept (POC) and repeat every experiment.

---

## Contents

1. [Deliverables checklist](#1-deliverables-checklist)
2. [Architecture and addressing plan](#2-architecture-and-addressing-plan)
3. [Requirements and installation](#3-requirements-and-installation)
4. [Project files](#4-project-files)
5. [Run the POC (live demo)](#5-run-the-poc-live-demo)
6. [Repeat the experiments](#6-repeat-the-experiments)
7. [Packet captures](#7-packet-captures)
8. [Results summary](#8-results-summary)
9. [Known issues and tips](#9-known-issues-and-tips)
10. [Team](#10-team)
11. [AI-use disclosure](#11-ai-use-disclosure)

---

## 1. Deliverables checklist

| # | Deliverable | Where to find it |
| --- | --- | --- |
| 1 | Project proposal | Submitted separately in week 1 |
| 2 | Architecture diagram with addressing plan | `report/architecture_diagram.png`, [Section 2](#2-architecture-and-addressing-plan) of this README, and Figure 1 / Section 3 of the final report |
| 3 | Working POC, demonstrated live | [Section 5](#5-run-the-poc-live-demo) |
| 4 | Packet captures (`.pcapng`) and annotated screenshots | `pcaps/*.pcapng`, `report/annotated_*.png`, [Section 7](#7-packet-captures), and Section 7 of the final report |
| 5 | Experimental evaluation (5 conditions, 3 runs each, mean ± standard deviation) | [Section 8](#8-results-summary), `results/summary.csv`, and Section 6 of the final report |
| 6 | Technical report and README | `report/TCP Congestion Control under Network Impairments Reno vs CUBIC vs BBR.docx` (final technical report), `report/TCP Congestion Control under Network Impairments Reno vs CUBIC vs BBR  Extended.docx` (full step-by-step version, supplementary), and this file |
| 7 | AI-use disclosure | [Section 11](#11-ai-use-disclosure), and both reports |

---

## 2. Architecture and addressing plan

Four machines in a straight line, all created by Mininet inside one Ubuntu virtual machine. There are no switches: each pair of machines is joined directly by a virtual cable. The two routers are ordinary Mininet hosts with IP forwarding switched on.

```
            10.0.1.0/24                     10.0.2.0/24  (bottleneck)              10.0.3.0/24
 +----------+                    +----------+                    +----------+                    +----------+
 |    h1    | h1-eth0    r1-eth1 |    r1    | r1-eth2    r2-eth1 |    r2    | r2-eth2    h2-eth0 |    h2    |
 |  sender  |--------------------|  router  |--------------------|  router  |--------------------| receiver |
 |          | 10.0.1.10 10.0.1.1 |          | 10.0.2.1  10.0.2.2 |          | 10.0.3.1 10.0.3.10 |          |
 +----------+                    +----------+                    +----------+                    +----------+
  fq queue on h1-eth0           1% loss and 5 Mbit/s limit      +50 ms delay applied to
  (paces outgoing packets)      applied to data leaving         ACKs leaving r2-eth1
                                r1-eth2 (towards h2)            (towards h1)

  Data:  h1 (port 4xxxx-5xxxx) ---- TCP ---->  h2 (port 5201, iperf3 server)
  ACKs:  h1 <---- TCP ----  h2
```

### Addressing plan

| Node | Role | Interface | IP address | Subnet | Routing |
| --- | --- | --- | --- | --- | --- |
| h1 | Sender (iperf3 client) | h1-eth0 | 10.0.1.10/24 | 10.0.1.0/24 | Default route via 10.0.1.1 |
| r1 | Router 1 | r1-eth1 | 10.0.1.1/24 | 10.0.1.0/24 | Forwarding on |
| r1 | Router 1 | r1-eth2 | 10.0.2.1/24 | 10.0.2.0/24 | Static: 10.0.3.0/24 via 10.0.2.2 |
| r2 | Router 2 | r2-eth1 | 10.0.2.2/24 | 10.0.2.0/24 | Static: 10.0.1.0/24 via 10.0.2.1 |
| r2 | Router 2 | r2-eth2 | 10.0.3.1/24 | 10.0.3.0/24 | Forwarding on |
| h2 | Receiver (iperf3 server) | h2-eth0 | 10.0.3.10/24 | 10.0.3.0/24 | Default route via 10.0.3.1 |

### Protocols, ports and communication flows

| Flow | Protocol | From → To | Ports |
| --- | --- | --- | --- |
| Test data | TCP (Reno, CUBIC or BBR) | h1 → h2 | Client: random high port (for example 58976). Server: 5201 |
| Acknowledgements (ACKs) | TCP | h2 → h1 | Server 5201 → client port |
| iperf3 control channel (test setup and results) | TCP | h1 ↔ h2 | Server 5201 |
| Second flow, fairness tests only | TCP | h1 → h2 | Server 5202 |
| Reachability check (`ping`) | ICMP echo request / reply | h1 ↔ h2 | none |
| Path check (`traceroute`) | UDP probes, ICMP "time exceeded" / "port unreachable" replies | h1 → h2, replies from r1, r2, h2 | UDP 33434 and up |

### Network conditions

| Name | Condition | Applied on | Linux command (inside `set_impairment.sh`) |
| --- | --- | --- | --- |
| `base` | Normal | none | none |
| `loss1` | 1% random packet loss | r1-eth2 | `tc qdisc add dev r1-eth2 root netem loss 1% limit 100000` |
| `delay50` | +50 ms round-trip time | r2-eth1 | `tc qdisc add dev r2-eth1 root netem delay 50ms limit 100000` |
| `bw5` | 5 Mbit/s bandwidth limit | r1-eth2 | `tc qdisc add dev r1-eth2 root tbf rate 5mbit burst 15kb latency 400ms` |
| `lossdelay` | 1% loss + 50 ms delay | r1-eth2 and r2-eth1 | both of the above |

---

## 3. Requirements and installation

### 3.1 Machine

- Ubuntu running on Linux. We used an Ubuntu virtual machine (Python 3.12) in **VirtualBox** on a Windows laptop. Windows alone does not work, because Mininet, `tc` and BBR are Linux kernel features.
- VirtualBox settings we used (VM switched off while changing them):
  - **System → Processor:** 4 to 6 CPUs.
  - **System → Acceleration → Paravirtualization Interface: KVM** (gives Linux a more precise clock; this matters for BBR).
- Ubuntu installed directly on a computer (not in a VM) is even better; see [Section 9](#9-known-issues-and-tips).

### 3.2 Install the tools

```bash
sudo apt update
sudo apt install mininet iperf3 tcpdump wireshark iproute2 ethtool traceroute python3-matplotlib -y
```

### 3.3 Check that BBR is available

```bash
sudo modprobe tcp_bbr
sysctl net.ipv4.tcp_available_congestion_control
```

The output must include `reno cubic bbr`. If `bbr` is missing, the project cannot run on that kernel.

### 3.4 Get the project

Download or clone this repository and put it in your home folder so the path is `~/CN_Project` (rename the folder to `CN_Project` if GitHub gave it the repository name). **Run every command from inside `~/CN_Project`**, not from inside `scripts/`; otherwise result files are saved in the wrong place.

---

## 4. Project files

```
CN_Project/
├── README.md                    this file
├── scripts/
│   ├── topology.py              builds the network (h1, r1, r2, h2)
│   ├── set_impairment.sh        applies one network condition
│   ├── run_experiments.py       runs all tests automatically
│   └── plots.py                 draws the graphs and the summary table
├── results/
│   ├── <algorithm>_<condition>_run<n>.json      45 test results (iperf3 JSON)
│   ├── fair_<algorithm>_run<n>_f1.json / _f2    18 fairness results (2 flows per test)
│   ├── experiment_log.csv       one line per test
│   └── summary.csv              mean and standard deviation of every test
├── graphs/                      6 graphs (PNG)
├── pcaps/                       6 packet captures (.pcapng)
└── report/
    ├── architecture_diagram.png         network diagram with addressing plan
    ├── annotated_retransmissions.png    annotated Wireshark screenshot (loss recovery)
    ├── annotated_duplicate_acks.png     annotated Wireshark screenshot (loss detection)
    ├── annotated_handshake.png          annotated Wireshark screenshot (TCP handshake, SYN)
    ├── TCP Congestion Control under Network Impairments Reno vs CUBIC vs BBR.docx             technical report
    └── TCP Congestion Control under Network Impairments Reno vs CUBIC vs BBR  Extended.docx   full step-by-step report with every screenshot (supplementary)
```

What each script does:

| Script | What it does |
| --- | --- |
| `topology.py` | Creates the 4 machines and links, sets IP addresses and static routes, turns IP forwarding on in the routers, turns hardware offloads off (so every packet is a real packet and "1% loss" means 1% of packets), stops Linux from reusing old connection data between tests, and puts the `fq` queue on h1. On its own it opens the `mininet>` prompt after a ping test. |
| `set_impairment.sh` | Takes one word (`base`, `loss1`, `delay50`, `bw5`, `lossdelay`). Removes old rules, then applies the new ones. Run it on **both** routers: r1 handles loss and the speed limit, r2 handles delay. |
| `run_experiments.py` | Builds the network, starts the iperf3 servers, then for every algorithm, condition and run: applies the condition, runs iperf3 for 30 s, saves the JSON file, removes the condition. Then runs the fairness tests. Retries a failed test once, skips tests that already have a valid file (use `--force` to redo them), and writes `experiment_log.csv`. |
| `plots.py` | Reads all JSON files, computes mean and standard deviation over the 3 runs, draws 6 graphs into `graphs/` and writes `results/summary.csv`. |

---

## 5. Run the POC (live demo)

About 5 minutes. This shows the network working and the main result live: under packet loss with a realistic round trip, BBR keeps sending much faster than CUBIC.

**Step 1: Start the network.**

```bash
cd ~/CN_Project
sudo mn -c                          # clean up any old network
sudo python3 scripts/topology.py
```

Expected: a ping from h1 to h2 with `0% packet loss` and `ttl=62` (the packets crossed both routers), the line `Available congestion control: reno cubic bbr`, then the `mininet>` prompt.

**Step 2: Check the route.**

```
mininet> h1 traceroute -n 10.0.3.10
```

Expected: 3 hops, `10.0.1.1` → `10.0.2.2` → `10.0.3.10`.

**Step 3: Start the iperf3 server on h2.**

```
mininet> h2 iperf3 -s -D
```

**Step 4: Apply 1% loss + 50 ms delay, then compare CUBIC and BBR.**

```
mininet> r1 bash scripts/set_impairment.sh lossdelay
mininet> r2 bash scripts/set_impairment.sh lossdelay
mininet> h1 ping -c 3 10.0.3.10
mininet> h1 iperf3 -c 10.0.3.10 -t 15 -C cubic
mininet> h1 iperf3 -c 10.0.3.10 -t 15 -C bbr
```

Expected: ping times of about 50 ms. On the `receiver` line, CUBIC gets about **2 to 3 Mbit/s**, and BBR gets clearly more. In our runs BBR got anywhere from about 6 to over 50 Mbit/s, because its speed varies a lot inside a VM (see [Section 9](#9-known-issues-and-tips)); it is always well above CUBIC.

**Step 5 (optional): Show queueing delay ("bufferbloat") on the 5 Mbit/s link.**

```
mininet> r1 bash scripts/set_impairment.sh bw5
mininet> r2 bash scripts/set_impairment.sh bw5
mininet> h1 iperf3 -c 10.0.3.10 -t 20 -C cubic > /tmp/cubic.txt & sleep 5; ping -c 10 10.0.3.10
mininet> h1 iperf3 -c 10.0.3.10 -t 20 -C bbr > /tmp/bbr.txt & sleep 5; ping -c 10 10.0.3.10
```

Expected: while CUBIC is sending, ping times are around **300 ms or more** (the router queue is full). While BBR is sending, they stay around **40 to 60 ms**.

**Step 6: Reset and exit.**

```
mininet> r1 bash scripts/set_impairment.sh base
mininet> r2 bash scripts/set_impairment.sh base
mininet> exit
```

---

## 6. Repeat the experiments

### 6.1 Run everything (about 30 minutes)

```bash
cd ~/CN_Project
sudo mn -c
sudo python3 scripts/run_experiments.py --force
sudo chown -R $USER:$USER ~/CN_Project     # files made with sudo belong to root; take them back
python3 scripts/plots.py
```

This runs:

- **45 main tests:** 3 algorithms × 5 conditions × 3 runs, 30 seconds each.
- **9 fairness tests:** two flows of the same algorithm at the same time on the 5 Mbit/s link, 3 runs per algorithm.

Do not use the VM for anything else while it runs. Restarting the VM just before running gives the most stable BBR results (see [Section 9](#9-known-issues-and-tips)).

### 6.2 Run a subset

```bash
# one quick test
sudo python3 scripts/run_experiments.py --algos cubic --conds loss1 --runs 1 --no-fairness --force
# only the delay conditions, all algorithms
sudo python3 scripts/run_experiments.py --conds delay50 lossdelay --no-fairness --force
```

Options: `--algos` (reno, cubic, bbr), `--conds` (base, loss1, delay50, bw5, lossdelay), `--runs N`, `--no-fairness`, `--force`.

### 6.3 What you get

| Output | Contents |
| --- | --- |
| `results/*.json` | Full iperf3 output, with a sample every 0.5 s: throughput, congestion window, smoothed RTT, retransmissions |
| `results/experiment_log.csv` | When each test ran, whether it worked, its throughput and retransmissions |
| `results/summary.csv` | Mean and standard deviation for every algorithm and condition, plus the fairness results |
| `graphs/1_throughput.png` | Throughput, mean ± standard deviation |
| `graphs/2_retransmissions.png` | Retransmissions, mean ± standard deviation |
| `graphs/3_cwnd_over_time.png` | Congestion window over time (run 1) |
| `graphs/4_queueing_delay_bw5.png` | Queueing delay over time on the 5 Mbit/s link |
| `graphs/5_queueing_delay.png` | Mean queueing delay per condition |
| `graphs/6_fairness.png` | Jain's fairness index |

**How the metrics are calculated:**

- **Throughput:** `end.sum_received.bits_per_second` (measured at the receiver).
- **Retransmissions:** `end.sum_sent.retransmits`.
- **Congestion window:** `intervals[].streams[0].snd_cwnd`.
- **Queueing delay:** each sample's smoothed RTT (`intervals[].streams[0].rtt`) minus the lowest RTT seen in the same test.
- **Jain's fairness index** for two flows: (x1 + x2)² / (2 × (x1² + x2²)). 1.0 means a perfectly equal share.

---

## 7. Packet captures

### 7.1 Record a capture

Inside the `mininet>` prompt, after `h2 iperf3 -s -D`, set a condition on both routers, then record one 15-second test per algorithm on h1's interface (headers only, 128 bytes per packet, to keep files small):

```
mininet> r1 bash scripts/set_impairment.sh lossdelay
mininet> r2 bash scripts/set_impairment.sh lossdelay
mininet> h1 for a in reno cubic bbr; do tcpdump -i h1-eth0 -s 128 -Z root -w pcaps/${a}_lossdelay.pcap & P=$!; sleep 1; iperf3 -c 10.0.3.10 -t 15 -C $a; sleep 1; kill $P; sleep 1; done
```

Repeat with `bw5` (and `_bw5` in the file name) for the 5 Mbit/s captures. `-Z root` is needed so tcpdump is allowed to write into the project folder.

### 7.2 Convert to .pcapng

The captures already in `pcaps/` are in `.pcapng` format. If you record new ones, note that tcpdump saves `.pcap`. To get `.pcapng` (the format the course asks for), either open the file in Wireshark and use **File → Save As → pcapng**, or run:

```bash
for f in pcaps/*.pcap; do editcap -F pcapng "$f" "${f%.pcap}.pcapng"; done
```

(`editcap` comes with Wireshark.)

### 7.3 Captures included

| File | Algorithm | Condition | What it shows |
| --- | --- | --- | --- |
| `reno_lossdelay.pcapng` | Reno | 1% loss + 50 ms | Handshake, data and ACKs, duplicate ACKs, fast retransmissions, slow growth after losses |
| `cubic_lossdelay.pcapng` | CUBIC | 1% loss + 50 ms | Same exchanges as Reno |
| `bbr_lossdelay.pcapng` | BBR | 1% loss + 50 ms | Same loss rate as Reno, but about 25 times more data sent |
| `reno_bw5.pcapng` | Reno | 5 Mbit/s limit | Round-trip time stays high (full queue) |
| `cubic_bw5.pcapng` | CUBIC | 5 Mbit/s limit | Throughput close to 5 Mbit/s |
| `bbr_bw5.pcapng` | BBR | 5 Mbit/s limit | Start-up, queue drain, then a low round-trip time |

### 7.4 Finding the key exchanges in Wireshark

| Exchange | Wireshark display filter | What to look at |
| --- | --- | --- |
| TCP three-way handshake | `tcp.flags.syn == 1` | SYN, then SYN+ACK, between h1 (random port) and h2 (port 5201). There are two connections: iperf3's control channel and the data stream |
| Data and acknowledgements | `tcp.port == 5201` | Data packets 10.0.1.10 → 10.0.3.10 (Len = 1448), ACKs back (Len = 0) |
| Loss detection | `tcp.analysis.duplicate_ack` | The receiver repeats the same Ack number ("still waiting for this byte") |
| Loss recovery | `tcp.analysis.retransmission` | Fast retransmissions of the missing sequence number |
| Connection close | `tcp.flags.fin == 1 \|\| tcp.flags.reset == 1` | FIN and RST at the end of the test |
| Summary | Analyze → Expert Information | Counts of retransmissions, duplicate ACKs and other events |

Graphs: select a data packet (filter `tcp.len > 1000`, click the first row), then **Statistics → TCP Stream Graphs** → *Stevens* (data sent over time), *Round Trip Time* or *Throughput*. If a graph is empty, click **Switch Direction**: the title must read `10.0.1.10 → 10.0.3.10`.

---

## 8. Results summary

All values are **mean ± standard deviation over 3 runs** of 30 seconds (from `results/summary.csv`).

### 8.1 Throughput (Mbit/s)

| Condition | Reno | CUBIC | BBR |
| --- | --- | --- | --- |
| Baseline | 4,701 ± 955 | 4,819 ± 986 | 3,717 ± 930 |
| 1% loss | 42.6 ± 3.5 | 28.0 ± 4.7 | 1,888 ± 331 |
| +50 ms delay | 1,164 ± 46 | 190 ± 42 | 178 ± 120 |
| 5 Mbit/s limit | 4.52 ± 0.03 | 4.53 ± 0.03 | 4.62 ± 0.05 |
| 1% loss + 50 ms | 2.35 ± 0.18 | 2.40 ± 0.33 | 6.45 ± 1.29 |

### 8.2 Retransmissions (packets per 30 s test)

| Condition | Reno | CUBIC | BBR |
| --- | --- | --- | --- |
| Baseline | 114 ± 122 | 397 ± 515 | 54,116 ± 2,857 |
| 1% loss | 1,110 ± 87 | 746 ± 110 | 421,729 ± 72,535 |
| +50 ms delay | 18 ± 5 | 17 ± 12 | 17 ± 27 |
| 5 Mbit/s limit | 146 ± 28 | 26 ± 3 | 0 ± 0 |
| 1% loss + 50 ms | 57 ± 1 | 58 ± 3 | 178 ± 43 |

### 8.3 Queueing delay (ms)

| Condition | Reno | CUBIC | BBR |
| --- | --- | --- | --- |
| Baseline | 0.06 ± 0.03 | 0.08 ± 0.05 | 1.67 ± 1.05 |
| 1% loss | 5.89 ± 0.47 | 9.41 ± 6.83 | 2.69 ± 1.51 |
| +50 ms delay | 9.04 ± 1.92 | 12.07 ± 2.07 | 10.89 ± 3.95 |
| 5 Mbit/s limit | 202 ± 27 | 239 ± 29 | 35.6 ± 5.5 |
| 1% loss + 50 ms | 10.77 ± 1.78 | 5.91 ± 3.86 | 13.99 ± 2.77 |

### 8.4 Fairness (Jain's index, 2 flows on the 5 Mbit/s link)

| Reno | CUBIC | BBR |
| --- | --- | --- |
| 0.9997 ± 0.0002 | 0.9963 ± 0.0041 | 0.9965 ± 0.0013 |

### 8.5 Main findings

- **Random loss:** BBR stays far faster than Reno and CUBIC, which slow down after every lost packet. With 1% loss + 50 ms delay, Reno and CUBIC match the Mathis formula's prediction (about 2.3 to 2.6 Mbit/s).
- **Bandwidth limit:** all three use the full 5 Mbit/s, but Reno and CUBIC fill the router queue (200+ ms of extra delay) while BBR keeps it short (about 36 ms) with no retransmissions.
- **Fairness:** two flows of the same algorithm share the link almost exactly equally.
- **Delay only:** BBR's result is held back by the VM (see [Section 9](#9-known-issues-and-tips)), so it should not be read as BBR's real performance.

---

## 9. Known issues and tips

| Problem | Cause | Fix |
| --- | --- | --- |
| BBR is slow or varies a lot when delay is added | BBR spaces out its packets with a timer, and timers in a VM fire late. BBR measures this as a slow network and slows down further | Keep the `fq` queue on h1 (already in `topology.py`), set Paravirtualization to KVM, restart the VM before testing BBR, or use Ubuntu installed directly on a computer |
| `Connection refused` from iperf3 | No server running. Every network restart starts from scratch | Run `h2 iperf3 -s -D` again |
| Result files appear in `scripts/results/` | Commands were run from inside `scripts/` | Always run from `~/CN_Project` |
| `Permission denied` when running `plots.py` | Files created with `sudo` belong to root | `sudo chown -R $USER:$USER ~/CN_Project` |
| Mininet errors such as "File exists" when starting | A previous network was not cleaned up | `sudo mn -c` |
| 5 Mbit/s link only gives about 2.5 Mbit/s | `burst` value too small for a VM | Keep `burst 15kb` in `set_impairment.sh` |
| Empty Wireshark graph | The graph shows the ACK direction | Click **Switch Direction** |

---

## 10. Team

| Member | Responsibilities |
| --- | --- |
| Bismah Sheikh | Environment setup, network topology and routing, baseline tests |
| Adeel Intekhab | Network conditions (impairment script), running the experiments, fairness tests |
| Fatima Haider | Data processing and graphs, Wireshark analysis, report and slides |

*(Roles from our team work plan; update this table if the final division of work was different.)*

---

## 11. AI-use disclosure

*(Draft: check it against the course's AI policy and adjust before submitting.)*

We used **Claude** (an AI assistant made by Anthropic) throughout this project:

- **Planning and learning:** explaining the project, the tools (Mininet, iperf3, tc, Wireshark) and the three algorithms, and breaking the work into steps.
- **Code:** Claude wrote `set_impairment.sh`, `run_experiments.py` and `plots.py`, and improved `topology.py` from the starter script in our team plan. We ran, tested and edited them (for example, adding the fifth condition and the `fq` line).
- **Debugging:** Claude helped diagnose the speed-limiter `burst` problem, BBR's pacing problem in the VM (fixed with `fq` and the KVM setting), and file-permission errors. We ran every test that confirmed or rejected each fix.
- **Analysis and writing:** Claude helped interpret the results (including the Mathis formula check), drafted the technical report and this README, and cropped screenshots for the report.

**Done by the team:** running every command and experiment on our own Ubuntu VM, collecting all results and packet captures, taking the Wireshark screenshots, deciding on changes, and checking the text. All numbers come from our own result files.
