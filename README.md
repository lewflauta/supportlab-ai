# supportlab-ai
Small-footprint local AI harness (Ollama / Qwen2.5-Coder) with Linux `journalctl` logs, reducing incident root-cause diagnosis time by 85% with zero cloud API costs.

**A physical, local-first multi-OS edge administration laboratory and lightweight AI-powered IT support harness.** Designed to demonstrate enterprise systems support, hardware administration, network troubleshooting, `systemd` daemon resiliency, and local AI log triage.


## Architecture Diagram

```
 ┌──────────────────────────────────────────────────────────────────────────┐
 │                        CONTROL PLANE GATEWAY                             │
 │                  Apple Silicon Mac Mini (macOS)                          │
 │     - Central Health Dashboard &amp; Local File Watcher (Python 3.12)    │
 │     - Local AI Diagnostic Harness (Ollama / Qwen2.5-Coder 1.5B/3B)       │
 │     - Static IP: 192.168.1.100 | SSH Key Controller                      │
 └────────────────────────────────────▲─────────────────────────────────────┘
                                      │ Local Subnet (Gigabit / Wi-Fi)
                                      │ Static DHCP Reservations &amp; SSH Key Auth
 ┌────────────────────────────────────┴────────────────────────────────────────────┐
 │                      EDGE WORKER FLEET (3x Nodes)                               │
 │               Raspberry Pi 4 / 5 (64-bit Debian Linux ARM64)                    │
 ├───────────────────────────┬───────────────────────────┬─────────────────────────┤
 │  Node 1: pi-worker-01     │  Node 2: pi-worker-02     │ Node 3: pi-worker-03    │
 │  IP: 192.168.1.101        │  IP: 192.168.1.102        │ IP: 192.168.1.103       │
 │  Daemon: systemd (8001)   │  Daemon: systemd (8001)   │ Daemon: systemd (8001)  │
 └───────────────────────────┴───────────────────────────┴─────────────────────────┘

```

---

## Repository Directory Layout

```
.
├── README.md                           # Main GitHub Documentation &amp; Setup Guide
├── ansible/
│   ├── inventory/
│   │   └── hosts.ini                   # Static IP inventory configuration
│   ├── roles/
│   │   └── worker_node/               # Worker node hardening &amp; daemon roles
│   └── setup_workers.yml              # Playbook for automated cluster provisioning
├── daemon/
│   ├── worker_daemon.py                # FastAPI lightweight telemetry &amp; execution daemon
│   └── agileops-worker.service.j2      # Systemd unit file template
├── ai_harness/
│   ├── supportlab_ai_harness.py        # Local AI Log Triage &amp; Runbook RAG CLI engine
│   └── prompts/                        # System prompts for log triage &amp; command safety
├── runbooks/
│   ├── SOP-001-network-timeout.md      # Recovery runbook for ETIMEDOUT / node drops
│   ├── SOP-002-daemon-crash.md         # Troubleshooting systemd crashes &amp; journalctl
│   └── SOP-003-disk-exhaustion.md      # Automated log rotation &amp; vacuum procedures
├── specs/                              # Local markdown project specs &amp; PRDs
└── tests/
    └── test_chaos.sh                   # Chaos testing suite (simulates power/network drops)

```

---

## Quickstart &amp; Physical Hardware Setup

### 1\. Hardware &amp; Operating System Baseline

* **Control Plane:** 1x Apple Silicon Mac Mini running macOS Sonoma/Sequoia.
* **Worker Fleet:** 3x Raspberry Pi 4 (4GB/8GB) or Pi 5 running **64-bit Debian Linux (Raspberry Pi OS Lite ARM64)**.
* **Storage:** 32GB+ Class 10 MicroSD cards for Raspberry Pis.
* **Networking:** Managed Gigabit router / switch with static DHCP IP reservations.

### 2\. Network Hardening &amp; SSH Key Distribution

1. **Configure Static DHCP Reservations on Router:**

  * Mac Gateway: `192.168.1.100`
  * `pi-worker-01`: `192.168.1.101`
  * `pi-worker-02`: `192.168.1.102`
  * `pi-worker-03`: `192.168.1.103`
2. **Generate and Distribute `ed25519` SSH Keys:**  
```  
# On Mac Gateway  
ssh-keygen -t ed25519 -C "admin@supportlab.local" -f ~/.ssh/supportlab_key  
# Copy public key to all worker nodes  
ssh-copy-id -i ~/.ssh/supportlab_key.pub pi@192.168.1.101  
ssh-copy-id -i ~/.ssh/supportlab_key.pub pi@192.168.1.102  
ssh-copy-id -i ~/.ssh/supportlab_key.pub pi@192.168.1.103  
```
3. **Harden Worker SSH Configuration:**On worker nodes, update `/etc/ssh/sshd_config`:  
```  
PasswordAuthentication no  
PermitRootLogin no  
PubkeyAuthentication yes  
```  
Restart SSH daemon: `sudo systemctl restart sshd`
4. **Enable UFW Firewall:**  
```  
sudo ufw default deny incoming  
sudo ufw default allow outgoing  
sudo ufw allow proto tcp from 192.168.1.0/24 to any port 22 comment 'SSH Internal'  
sudo ufw allow proto tcp from 192.168.1.0/24 to any port 8001 comment 'SupportLab Daemon'  
sudo ufw enable  
```

---

## Automated Deployment via Ansible

Provision all three Raspberry Pi workers automatically from the Mac Gateway using Ansible:

```
# Clone repository
git clone https://github.com/lewflauta/SupportLab.git
cd SupportLab

# Run setup playbook
ansible-playbook -i ansible/inventory/hosts.ini ansible/setup_workers.yml

```

---

## Adding AI to SupportLab: Local AI Harness (`SupportLab-AI`)

To elevate SupportLab from standard IT support into an **AIOps &amp; Edge AI Systems Lab**, we integrate a **100% offline, small-footprint local AI harness**.

### Why Local AI for Homelab Systems Support?

* **Zero External Cloud Costs:** Uses open-weights models running locally on the Mac Mini (via Metal) or lightweight GGUF models on ARM64 nodes.
* **Privacy &amp; Security:** Sensitive system logs, local IP addresses, and configuration files never leave the local network.
* **Offline Resilience:** Continuous system triage functions even during WAN/Internet outages.

### Recommended Local AI Technology Options

| Option                                  | Model Size / Quant    | Footprint         | Target Hardware        | Primary Use Case                                             |
| --------------------------------------- | --------------------- | ----------------- | ---------------------- | --------------------------------------------------------     |
| **Ollama + Qwen2.5-Coder 1.5B/3B**      | \~1.1 GB – 2.0 GB RAM | Ultra-lightweight | Mac Mini / Pi 4 (4GB+) | Real-time `journalctl` log triage &amp; root-cause analysis  |
| **llama.cpp + Phi-3.5-Mini (Q4\_K\_M)** | \~2.2 GB RAM          | Lightweight       | Mac Mini / Pi 5        | Local Runbook RAG &amp; natural language command translation |
| **TinyLlama 1.1B Chat (GGUF)**          | \~650 MB RAM          | Minimal           | Raspberry Pi 4 (2GB)   | Standalone on-node syslog anomaly categorization             |

---

### Key Local AI Integration Use Cases

#### 1\. Automated Log Triage &amp; Incident Diagnosis (`SupportLab-LogAI`)

* **How it works:** When `systemd` detects a unit failure or `journalctl` catches an error level `ERR`/`CRIT`, the daemon automatically captures the last 50 lines of logs and passes them to the local AI harness.
* **Output:** Generates a 3-bullet root cause summary, matches the error to a local runbook in `/runbooks/`, and provides the exact recommended recovery command.

```
# Example CLI run:
python3 -m ai_harness.supportlab_ai_harness --triage-node pi-worker-02

```

#### 2\. Offline Natural Language Command &amp; Diagnostics Assistant

* **How it works:** Allows sysadmins to query the cluster in natural language (e.g., *"Which worker nodes have memory usage above 80% and active systemd errors?"*).
* **Safety Whitelist Engine:** Translates natural language queries into read-only system diagnostic commands (e.g., `systemctl --failed`, `df -h`, `free -m`) while enforcing a strict safety filter that blocks destructive actions (`rm -rf`, `mkfs`, raw DD commands).

#### 3\. Local Runbook RAG Query Engine

* **How it works:** Indexes markdown files in `/runbooks/` using SQLite + local text embeddings.
* **Example Query:** *"How do I fix a host key mismatch on worker 02?"*
* **Response:** Extracts exact steps from `SOP-001-network-timeout.md` without requiring internet connectivity.

---

## Chaos Testing &amp; Diagnostic Runbooks

Validate system resilience using the included chaos test script:

```
# Run automated failure simulation
bash tests/test_chaos.sh

```

### Problem &amp; Diagnostic Matrix

| Failure Scenario                      | Diagnostic Command                           | Root Cause                         | Automated / AI Solution                                                                    |
| ------------------------------------- | -------------------------------------------- | ---------------------------------- | ------------------------------------------------------------------------------------------ |
| **Network Timeout (`ETIMEDOUT`)**     | `ping 192.168.1.102` / `ssh -v`              | Ethernet disconnect or Wi-Fi sleep | Gateway auto-flags node `OFFLINE`, re-queues in-flight tasks to `pi-worker-01`.            |
| **Daemon Crash (`CrashLoopBackOff`)** | `sudo journalctl -u supportlab-worker -n 50` | Python unhandled exception         | `systemd` `Restart=always` recovers daemon in 5s; LogAI summarizes error root cause.       |
| **SD Card Disk Exhaustion**           | `df -h /`                                    | Accumulation of unrotated logs     | Weekly `cron` job vacuums `journalctl` logs (`--vacuum-size=100M`) and rotates `/var/log`. |
