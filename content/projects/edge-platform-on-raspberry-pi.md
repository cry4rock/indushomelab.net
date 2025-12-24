---
title: "Edge Platform on Raspberry Pi 3 Model B"
date: 2025-12-24
draft: false
tags:
  - homelab
  - edge-computing
  - platform-engineering
  - observability
categories:
  - Projects
---

## 🚀 Project Overview

This project documents the design and implementation of a **lightweight edge platform** built on a Raspberry Pi, inspired by modern Platform Engineering and SRE practices.

The goal is not just to run containers — but to build:
- A **reliable edge platform**
- With **observability baked in**
- Fully **documented**
- And easy to reproduce

---

## 🎯 Objectives

- Build a production-inspired edge platform on Raspberry Pi
- Run containerized workloads using Docker
- Implement monitoring and observability
- Expose services securely
- Maintain Git-driven documentation

---

## 🧠 Architecture Overview

**Core components:**
- Raspberry Pi (Edge Node)
- Docker & Docker Compose
- Reverse Proxy (Traefik)
- Monitoring (Prometheus + Grafana)
- Logging (future)
- GitHub + Hugo for documentation

> Diagram will be added once the platform is stable.

---

## 🧰 Hardware & Software

### Hardware
- ✅ Raspberry Pi 3 Model B
- ✅ ARMv8 / 64-bit
- ✅ Debian 12 (Bookworm)
- ⚠️ 1 GB RAM
- ⚠️ 16 GB SD card
- ⚠️ 100 Mbps Ethernet

### Software
- Raspberry Pi OS (64-bit)
- Docker
- Docker Compose
- Traefik
- Prometheus
- Grafana

---

## 🛠️ Build Phases

### Phase 0 - Critical Fixes (Before Containers)
- Increase swap from 512 MB to 2 GB (mandetory on Pi 3)
- Reduce GPU memory (headless optimization)

### Phase 1 – Base OS & Access
- Flash Raspberry Pi OS
- Enable SSH
- Secure access

### Phase 2 – Container Platform
- Install Docker
```shell
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
newgrp docker
```
- Validate container runtime
```shell
docker run hello-world
```
- Install Docker Compose plugin
```shell
sudo apt install docker-compose -y
docker-compose version
```
- Define compose structure
```shell
mkdir -p ~/edge-platform/{infra,apps,observability,docs}
```

```text
edge-platform/
├── infra/              # reverse proxy, networking
├── observability/      # prometheus, grafana
├── apps/               # workloads (later)
├── docs/               # local docs / diagrams
```

### Phase 3 – Networking & Security
- Reverse proxy
- TLS (later)
- Service exposure

### Phase 4 – Observability
- Create observability directory
```shell
cd ~/edge-platform
mkdir -p observability/node-exporter
cd observability/node-exporter
```
- Metrics collection
- Dashboards
- Alerts (future)

### Phase 5 – Documentation
- Architecture diagrams
- Runbooks
- Lessons learned

---

## 📸 Gallery

Images and diagrams will be added as the project progresses.

---

## 📘 Lessons Learned (living section)

###📈 Increasing Swap Space on Raspberry Pi (Edge Platform Prerequisite)

**Why Swap Matters on Edge Devices**

The Raspberry Pi 3 is constrained by 1 GB of RAM, which is insufficient for:

- Docker daemon
- Multiple containers
- Observability tooling (Prometheus, Grafana)

Swap acts as a pressure relief valve, preventing:

- OOM (Out-Of-Memory) kills
- Random container crashes
- System lockups under load

⚠️ Swap is not a performance booster — it is a stability mechanism.

For this edge platform, increasing swap is mandatory.

**Current System State**
```shell
free -h
```

Typical output before change:
```text
              total        used        free
Mem:           982Mi        380Mi        120Mi
Swap:          512Mi        138Mi        374Mi
```

The default 512 MB swap is insufficient for containerized workloads.

**Target Configuration**
Setting		Value
Swap size	2 GB
Swap file type	File-based
Use case	Docker + observability

###Step-by-Step: Increase Swap to 2 GB

**1️⃣ Disable current swap**
```shell
sudo dphys-swapfile swapoff
```
**2️⃣ Edit swap configuration**
```shell
sudo nano /etc/dphys-swapfile
```
Locate:
```yaml
CONF_SWAPSIZE=512
```

Change to:
```yaml
CONF_SWAPSIZE=2048
```

Save and exit (CTRL+O, ENTER, CTRL+X).

**3️⃣ Recreate the swap file**
```shell
sudo dphys-swapfile setup
```

This recreates the swap file with the new size.

**4️⃣ Re-enable swap**
```shell
sudo dphys-swapfile swapon
```
**5️⃣ Verify swap is active**
```shell
free -h
```

Expected output:
```text
              total        used        free
Mem:           982Mi        380Mi        120Mi
Swap:          2.0Gi        0Mi          2.0Gi
```
**Validation Checks**
```shell
swapon --show
```

You should see:
```text
NAME       TYPE SIZE USED PRIO
/var/swap  file 2G   0B   -2
```
###Operational Notes (Important)

**⚠️ SD Card Wear**

- Swap increases write activity
- Acceptable for homelab / edge experiments
- Not recommended for heavy production writes

**🧠 Memory Behavior**

- Linux will prefer RAM first
- Swap is used only under pressure
- Prevents Docker from crashing silently

We deliberately increased swap to 2 GB to prioritize platform stability over raw performance, acknowledging edge constraints while enabling realistic container workloads.

---

## 🧪 Platform Constraints

This platform intentionally runs on a Raspberry Pi 3 with limited CPU, memory, and storage to simulate real-world edge constraints.

Design decisions prioritize:
- Low memory footprint
- Minimal container count
- Clear service ownership
- Observability without over-provisioning

---

## 🔗 References

- Docker documentation
- Prometheus
- Grafana
- Platform Engineering concepts

