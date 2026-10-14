# 🛡️ SDN-Based DDoS Detection and Automatic Mitigation

![Python](https://img.shields.io/badge/Python-3.9+-blue?logo=python&logoColor=white)
![Ryu](https://img.shields.io/badge/Controller-Ryu-orange)
![OpenFlow](https://img.shields.io/badge/OpenFlow-1.3-green)
![Mininet](https://img.shields.io/badge/Emulator-Mininet-lightgrey)
![scikit-learn](https://img.shields.io/badge/ML-scikit--learn-F7931E?logo=scikitlearn&logoColor=white)
![Flask](https://img.shields.io/badge/Dashboard-Flask-black?logo=flask)
![License](https://img.shields.io/badge/License-MIT-yellow)

A Software Defined Networking (SDN) lab that **detects DDoS attacks in real time** using a machine-learning classifier and **automatically blocks attackers** by pushing OpenFlow drop rules from a Ryu controller. A live web dashboard shows traffic, status, blocked hosts, and alerts.

---

## 📑 Table of Contents

1. [Overview](#-overview)
2. [Key Features](#-key-features)
3. [Architecture](#-architecture)
4. [Network Topology](#-network-topology)
5. [How It Works](#-how-it-works)
6. [Project Structure](#-project-structure)
7. [Prerequisites](#-prerequisites)
8. [Installation](#-installation)
9. [Running the Project](#-running-the-project)
10. [Demo Walkthrough](#-demo-walkthrough)
11. [Machine Learning Module](#-machine-learning-module)
12. [Results](#-results)
13. [Configuration](#-configuration)
14. [Testing](#-testing)
15. [Troubleshooting](#-troubleshooting)
16. [Future Work](#-future-work)
17. [License](#-license)

---

## 📖 Overview

Traditional networks tie the control logic to each device, so reacting to an attack means manually reconfiguring routers and switches. **SDN separates the control plane from the data plane**, which gives a central controller a global view of traffic and the power to reprogram the network instantly.

This project uses that idea to build a closed-loop security system:

> **Monitor → Extract features → Classify (ML) → Mitigate → Report**

| Problem | How this project solves it |
|---|---|
| DDoS floods saturate links and servers | Detects abnormal flow statistics within seconds |
| Manual response is slow | Controller installs DROP flows automatically |
| Static thresholds miss attack variants | Random Forest model learns attack patterns |
| No visibility | Live dashboard with graphs and event log |

---

## ✨ Key Features

- 🔍 **Real-time monitoring**: flow and port stats polled every 3–5 seconds
- 🤖 **ML-based detection**: Random Forest (compared with Decision Tree and SVM)
- 🚫 **Automatic mitigation**: high-priority DROP rules with hard timeout
- ⏱️ **Auto-unblock**: blocked hosts are released after the timeout expires
- ✅ **Whitelist support**: protect legitimate hosts from false positives
- 📊 **Live dashboard**: packets/sec, flows, blocked IPs, alert log
- 🧯 **Fallback mode**: threshold-based detection if `model.pkl` is missing
- 🌊 **Multiple attack types**: SYN flood, UDP flood, ICMP flood

---

## 🏗️ Architecture

![Architecture](docs/images/architecture.svg)

The system has three layers:

| Layer | Components | Role |
|---|---|---|
| **Application plane** | Flask dashboard, ML classifier, mitigation engine | Decision-making and visualization |
| **Control plane** | Ryu controller (OpenFlow 1.3) | Collects stats, runs detection, pushes flow rules |
| **Data plane** | Open vSwitch switches and Mininet hosts | Forwards traffic and reports statistics |

---

## 🌐 Network Topology

![Topology](docs/images/topology.svg)

| Host | IP | Role |
|---|---|---|
| h1, h4, h7 | 10.0.0.1, .4, .7 | Benign clients |
| h5, h6 | 10.0.0.5, .6 | Victim servers |
| h2, h3, h8 | 10.0.0.2, .3, .8 | Attackers |

> The exact hosts may differ slightly depending on `topology/custom_topo.py`. Adjust the table to match your code.

---

## ⚙️ How It Works

![Workflow](docs/images/workflow.svg)

1. **Poll**: the controller sends `OFPFlowStatsRequest` and `OFPPortStatsRequest` to every switch.
2. **Extract features**: stats are converted into a feature vector (see below).
3. **Classify**: the ML model labels each interval `NORMAL` or `ATTACK`.
4. **Identify**: source IPs and ports contributing most to the flood are found.
5. **Mitigate**: a high-priority DROP flow with a hard timeout is installed.
6. **Report**: the event is logged and pushed to the dashboard.
7. **Recover**: when the timeout expires the block is removed automatically.

### Features used for detection

| Feature | Meaning |
|---|---|
| Packet rate | Packets per second in the interval |
| Byte rate | Bytes per second in the interval |
| Flow duration | Average duration of active flows |
| Packet count | Total packets in the interval |
| Source IP entropy | Randomness of source addresses (high in spoofed floods) |
| Unique sources per destination | Many sources to one victim indicates DDoS |
| Protocol ratio | Share of TCP / UDP / ICMP traffic |

---

## 📁 Project Structure

```
sdn-ddos-detection-mitigation/
├── controller/
│   ├── ddos_controller.py      # Ryu app: switching, stats polling, detection, mitigation
│   └── feature_extractor.py    # Flow stats to features
├── topology/
│   └── custom_topo.py          # Mininet topology (switches + hosts)
├── traffic/
│   ├── normal_traffic.sh       # Benign traffic generator
│   └── attack_traffic.sh       # SYN / UDP / ICMP flood scripts
├── ml/
│   ├── collect_dataset.py      # Collects labeled flow data to CSV
│   ├── train_model.py          # Trains and compares models, saves model.pkl
│   └── evaluate.py             # Metrics and confusion matrix plots
├── dashboard/
│   ├── app.py                  # Flask backend (REST API)
│   └── templates/index.html    # Live charts (Chart.js)
├── docs/
│   ├── architecture.md
│   ├── report_outline.md
│   └── images/                 # Diagrams and screenshots used in this README
├── tests/                      # Unit tests
├── config.yaml                 # All settings
├── run_all.sh                  # One-command startup
├── requirements.txt
└── README.md
```

---

## 🧰 Prerequisites

| Requirement | Details |
|---|---|
| OS | **Linux** (Ubuntu 22.04 recommended). On Windows use **WSL2** or a **VirtualBox VM** |
| RAM / Disk | 8 GB RAM, 20 GB free disk |
| Python | 3.9+ |
| Tools | Mininet, Open vSwitch, hping3, iperf, git |

---

## 📦 Installation

```bash
# 1. System packages
sudo apt update
sudo apt install -y mininet openvswitch-switch hping3 iperf \
                    python3-pip python3-venv git

# 2. Clone the repo
git clone https://github.com/<your-username>/sdn-ddos-detection-mitigation.git
cd sdn-ddos-detection-mitigation

# 3. Python environment
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt

# 4. Start Open vSwitch
sudo service openvswitch-switch start
```

---

## ▶️ Running the Project

### Option A: One command

```bash
chmod +x run_all.sh
sudo ./run_all.sh
```

### Option B: Manual (4 terminals)

| Terminal | Command | Purpose |
|---|---|---|
| 1 | `ryu-manager controller/ddos_controller.py` | Start the controller **first** |
| 2 | `sudo python3 topology/custom_topo.py` | Start Mininet (opens `mininet>` CLI) |
| 3 | `python3 dashboard/app.py` | Start the dashboard |
| 4 | *(optional)* logs, dataset collection | |

Open the dashboard at **http://localhost:5000**.

---

## 🎬 Demo Walkthrough

**Step 1: Verify connectivity**
```
mininet> pingall
```

**Step 2: Normal traffic (no blocking expected)**
```
mininet> h1 ping -c 5 h5
mininet> h4 iperf -c 10.0.0.5 -t 10
```
Dashboard shows status 🟢 **Normal**.

![Dashboard normal](docs/images/dashboard_normal.png)

**Step 3: Launch an attack**
```
mininet> h2 hping3 -S --flood -p 80 10.0.0.5
```
Within a few seconds the traffic graph spikes and the status changes to 🔴 **Under Attack**.

![Dashboard attack](docs/images/dashboard_attack.png)

**Step 4: Mitigation**

The attacker (`10.0.0.2`) appears in the **Blocked Hosts** table and the flood stops.

![Blocked hosts](docs/images/dashboard_blocked.png)

**Step 5: Verify the block**
```
mininet> h2 ping -c 3 h5      # should fail (100% packet loss)
mininet> h1 ping -c 3 h5      # benign host still works
```

**Step 6: Auto-recovery**: after the timeout (default 60 s) the rule is removed and `h2` can communicate again.

> 📸 **Add your screenshots:** save them as `docs/images/dashboard_normal.png`, `dashboard_attack.png`, `dashboard_blocked.png` (Mininet terminal and Ryu logs too, if you like).

---

## 🧠 Machine Learning Module

```bash
python3 ml/collect_dataset.py     # generates labeled normal + attack data (CSV)
python3 ml/train_model.py         # trains models, saves best as model.pkl
python3 ml/evaluate.py            # metrics + confusion matrix
```

| Model | Notes |
|---|---|
| **Random Forest** | Main model, best accuracy / speed trade-off |
| Decision Tree | Lightweight baseline |
| SVM | Comparison model |

If `model.pkl` is not found, the controller automatically switches to **threshold-based detection**.

---

## 📈 Results

> Replace the values below with your own results after running `evaluate.py`.

| Model | Accuracy | Precision | Recall | F1-score |
|---|---|---|---|---|
| Random Forest | – | – | – | – |
| Decision Tree | – | – | – | – |
| SVM | – | – | – | – |

| Metric | Value |
|---|---|
| Detection latency | – seconds |
| Mitigation time | – seconds |
| False positive rate | – % |

**Confusion matrix**

![Confusion matrix](docs/images/confusion_matrix.png)

**Traffic before vs. after mitigation**

![Mitigation graph](docs/images/before_after_mitigation.png)

---

## 🔧 Configuration

All settings live in `config.yaml`:

```yaml
controller:
  poll_interval: 5          # seconds between stats requests
detection:
  model_path: ml/model.pkl
  fallback_packet_rate: 1000  # pkts/sec threshold if no model
mitigation:
  block_timeout: 60         # seconds
  whitelist: ["10.0.0.1", "10.0.0.4"]
dashboard:
  host: 0.0.0.0
  port: 5000
```

---

## ✅ Testing

```bash
pytest tests/ -v
```

Unit tests cover feature extraction and detection logic.

---

## 🩹 Troubleshooting

| Problem | Fix |
|---|---|
| Switch not connecting to controller | Start Ryu **before** Mininet |
| Mininet stuck / old state | `sudo mn -c` |
| Port 6653 already in use | `sudo pkill -f ryu-manager` |
| Ryu `eventlet` error | Use the pinned versions in `requirements.txt` (`eventlet==0.30.2`) |
| Open vSwitch not running | `sudo service openvswitch-switch start` |
| `ovsk` fails on WSL2 | Use `--switch user` or switch to a VirtualBox Ubuntu VM |
| Dashboard empty | Check the controller is running and the API port matches `config.yaml` |

---

## 🚀 Future Work

- Multi-controller setup (ONOS / OpenDaylight) for high availability
- P4-programmable switches for in-network detection
- Deep learning models (LSTM / autoencoders) for zero-day attack detection
- Rate limiting and traffic redirection to a scrubbing center instead of hard drop
- Integration with real hardware OpenFlow switches
- Support for application-layer (HTTP) DDoS detection

---

## 📜 License

Released under the [MIT License](LICENSE).

## 🙏 Acknowledgements

[Ryu SDN Framework](https://ryu-sdn.org/) · [Mininet](http://mininet.org/) · [Open vSwitch](https://www.openvswitch.org/) · [scikit-learn](https://scikit-learn.org/)
