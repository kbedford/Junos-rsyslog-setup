# Junos-rsyslog-setup
Junos rsyslog setup

# Collecting Junos SRX Syslog on Ubuntu Using rsyslog

This guide describes how to send syslog messages from a Junos SRX (or any Junos device) to an Ubuntu server running **rsyslog**, and how to:

- Store logs in a dedicated directory with one file per device per day  
- Rotate those logs cleanly using **logrotate**  
- Optionally harden the setup using **TCP** and **TLS**

---

## Table of Contents

1. [Overview](#overview)  
2. [Lab Topology & Assumptions](#lab-topology--assumptions)  
3. [Step 1 – Configure Syslog on Junos](#step-1--configure-syslog-on-junos)  
4. [Step 2 – Ensure rsyslog Is Installed on Ubuntu](#step-2--ensure-rsyslog-is-installed-on-ubuntu)  
5. [Step 3 – Enable Remote Syslog Reception](#step-3--enable-remote-syslog-reception)  
6. [Step 4 – Create rsyslog Template and Rules](#step-4--create-rsyslog-template-and-rules)  
7. [Step 5 – Create Log Directory and Set Permissions](#step-5--create-log-directory-and-set-permissions)  
8. [Step 6 – Open Firewall for Syslog Traffic](#step-6--open-firewall-for-syslog-traffic)  
9. [Step 7 – Restart rsyslog](#step-7--restart-rsyslog)  
10. [Step 8 – Verification](#step-8--verification)  
11. [Step 9 – Log Rotation with logrotate](#step-9--log-rotation-with-logrotate)  
12. [Step 10 – Security Hardening (TCP and TLS)](#step-10--security-hardening-tcp-and-tls)  
13. [Troubleshooting](#troubleshooting)  
14. [Summary](#summary)

---

## Overview

By default, many Junos deployments still send syslog over **UDP/514** to a central collector. On the Linux side, **rsyslog** is a popular and flexible choice for acting as that collector.

In this guide we:

- Configure Junos to send all syslog messages to an Ubuntu host  
- Enable rsyslog to receive remote messages  
- Use a custom template to write Junos logs to a dedicated directory  
- Configure **logrotate** for those logs  
- Show options for **hardening** via TCP and TLS

---

## Lab Topology & Assumptions

- **Junos SRX** (syslog sender)  
  - Example IP: `10.49.102.254`

- **Ubuntu Server** (syslog receiver)  
  - Example IP: `10.49.101.34`  
  - Runs **rsyslog** and will store log files under `/var/log/junos-messages`

Adjust IPs to match your environment.

---

## Step 1 – Configure Syslog on Junos

On the Junos device:

```bash
configure
set system syslog host 10.49.101.34 any any
set system syslog host 10.49.101.34 match ".*"
commit
exit
