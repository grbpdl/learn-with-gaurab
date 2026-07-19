---
title: "Server Overlaod Bot on port 22"
description: "Changing the default port to prevent the bot atttack"
date: "2026-05-22"
tags:
  - blog
  - first
author: "Your Name"
cover: "https://example.com/image.jpg"
---

# Infrastructure Update: Hardening SSH & Fixing CI/CD Timeouts

This document details a recent infrastructure issue encountered during the **Infocare CI/CD Deploy** workflow, where the GitHub Actions runner failed to connect to the production server. It outlines the root causes, the architecture changes implemented, and standard operating procedures to maintain or replicate this setup.

---

## ⚙️ The Problem

### 1. The Port 22 Traffic Jam (`i/o timeout`)
Because the production server was listening for SSH connections on the standard port `22`, it was constantly targeted by automated internet bots executing brute-force attacks. 
* **The Impact:** The massive volume of concurrent junk connection handshakes exhausted the server's network stack sockets and CPU resources. 
* **The Symptom:** When the legitimate GitHub Actions runner tried to connect, it got stuck in the queue, resulting in a pipeline failure: `Error: Process completed with exit code 1 (i/o timeout)`.

### 2. The Dual-Stack Misconfiguration (`Connection Refused`)
When moving SSH to a custom port (`2222`), modern `systemd` network managers on Ubuntu default to binding exclusively to the IPv6 loopback interface (`[::]:2222`).
* **The Impact:** External GitHub runners and internal loopback tools trying to connect via IPv4 were instantly dropped with a `Connection Refused` error. 
* **The Symptom:** The server was listening, but only on half the network spectrum.

---

## 🛠️ The Solution

We permanently bypassed the bot traffic and fixed the pipeline by executing three main operations:
1. **Port Obscurity:** Moved the SSH gateway to port `2222` to instantly drop background bot scanner noise to zero.
2. **Dual-Stack Binding:** Force-configured `systemd` to explicitly bind to both IPv4 (`0.0.0.0`) and IPv6 (`[::]`).
3. **Daemon Flushing:** Restarted the Docker daemon to rebuild its virtual routing tables (`iptables`) after the host network shifted.

---

## 🚀 Step-by-Step Implementation Guide

Follow these exact steps to apply or verify this configuration on the server:

### Step 1: Override the Systemd SSH Socket
Instead of editing core system configuration files directly, use a systemd override block to safely update the listening stream.

```bash
sudo systemctl edit ssh.socket
