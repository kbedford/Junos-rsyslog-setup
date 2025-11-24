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
````

Explanation:

* `any any` → send all facilities and all severities
* `match ".*"` → match all messages (simple regex)

You can narrow this later if you want to send only specific logs.

---

## Step 2 – Ensure rsyslog Is Installed on Ubuntu

On the Ubuntu server:

```bash
sudo apt-get update
sudo apt-get install rsyslog
```

Confirm rsyslog is running:

```bash
sudo systemctl status rsyslog
```

You should see `active (running)`.

---

## Step 3 – Enable Remote Syslog Reception

Edit the main rsyslog config:

```bash
sudo vi /etc/rsyslog.conf
```

Uncomment or add the following lines (typically near the top or in the input section):

```conf
# Enable UDP syslog reception
module(load="imudp")
input(type="imudp" port="514")

# Enable TCP syslog reception (optional, used later in hardening section)
#module(load="imtcp")
#input(type="imtcp" port="514")
```

> **Note:** For now, we will keep using UDP. The TCP/TLS part comes in the hardening section.

---

## Step 4 – Create rsyslog Template and Rules

We want a custom storage structure:

* Directory: `/var/log/junos-messages`
* One file per sending host per day, e.g.
  `10.49.102.254-2025-11-24.log`

Create a dedicated config file:

```bash
sudo vi /etc/rsyslog.d/junos.conf
```

Add the following:

```conf
# Template for Junos messages:
# One file per sending host per day:
#   /var/log/junos-messages/<IP>-YYYY-MM-DD.log
template(name="JunosMsgs"
         type="string"
         string="/var/log/junos-messages/%FROMHOST-IP%-%$YEAR%-%$MONTH%-%$DAY%.log")

# Apply the template for messages coming from the Junos device
if $fromhost-ip == '10.49.102.254' then -?JunosMsgs
& stop
```

Key points:

* `FROMHOST-IP` → IP of the sending host
* `$YEAR`, `$MONTH`, `$DAY` → current date
* `-?JunosMsgs` → use the `JunosMsgs` template and avoid fsync on each write (for performance)
* `& stop` → prevents double-logging into `/var/log/syslog` etc.

> If you send logs from multiple Junos devices, you can:
>
> * Either match each IP explicitly, or
> * Use a broader condition (e.g. by subnet) based on your environment.

---

## Step 5 – Create Log Directory and Set Permissions

Create the directory:

```bash
sudo mkdir -p /var/log/junos-messages
```

Set ownership and permissions:

```bash
sudo chown syslog:syslog /var/log/junos-messages
sudo chmod 755 /var/log/junos-messages
```

---

## Step 6 – Open Firewall for Syslog Traffic

If you use UFW or another firewall, allow syslog traffic.

With UFW:

```bash
sudo ufw allow 514/udp
# If using TCP later:
# sudo ufw allow 514/tcp
```

Verify:

```bash
sudo ufw status
```

---

## Step 7 – Restart rsyslog

Apply configuration changes:

```bash
sudo systemctl restart rsyslog
```

Check status:

```bash
sudo systemctl status rsyslog
```

If there’s a syntax issue (e.g. missing `)` in template), rsyslog will complain here or in `/var/log/syslog`.

---

## Step 8 – Verification

### 8.1 Verify from Junos

On the SRX:

```bash
show system syslog
```

You should see a host entry pointing to `10.49.101.34`.

Trigger some logs (e.g. commit a config or generate a test event).

### 8.2 Verify on Ubuntu

List the log directory:

```bash
ls -l /var/log/junos-messages
```

You should see something like:

```text
10.49.102.254-2025-11-24.log
```

Tail the file:

```bash
sudo tail -f /var/log/junos-messages/10.49.102.254-$(date +%Y-%m-%d).log
```

You should see live syslog messages from the SRX.

---

## Step 9 – Log Rotation with logrotate

Without rotation, the log files under `/var/log/junos-messages` will grow indefinitely.

We can use **logrotate** to:

* Rotate logs daily or weekly
* Keep a fixed number of historical copies
* Compress old logs
* Signal rsyslog to reopen file handles (optional but recommended)

### 9.1 Create logrotate Configuration

Create a new file:

```bash
sudo vi /etc/logrotate.d/junos-messages
```

Add the following:

```conf
/var/log/junos-messages/*.log {
    daily
    rotate 14
    compress
    delaycompress
    missingok
    notifempty
    create 0640 syslog syslog
    sharedscripts
    postrotate
        # Tell rsyslog to reopen log files
        /usr/bin/systemctl reload rsyslog.service > /dev/null 2>&1 || true
    endscript
}
```

Explanation:

* `daily` → rotate once per day
* `rotate 14` → keep 14 old log files (per file)
* `compress` / `delaycompress` → compress older logs to save space
* `create 0640 syslog syslog` → ensure new files have correct permissions/ownership
* `postrotate ... reload rsyslog` → makes rsyslog reopen file handles cleanly after rotation

> You can adjust `rotate`, `daily` (e.g. `weekly`), and permissions to suit your retention policies.

### 9.2 Test logrotate

Run logrotate in debug mode:

```bash
sudo logrotate -d /etc/logrotate.d/junos-messages
```

If everything looks good, force a rotation test:

```bash
sudo logrotate -f /etc/logrotate.d/junos-messages
```

Then check `/var/log/junos-messages` for rotated (and possibly compressed) log files.

---

## Step 10 – Security Hardening (TCP and TLS)

UDP is simple and lightweight but provides **no reliability, no encryption, and no integrity**. For production, consider:

1. Using **TCP** instead of UDP
2. Using **TLS** to encrypt syslog traffic over untrusted networks

### 10.1 Enable TCP Reception in rsyslog

In `/etc/rsyslog.conf`, ensure these lines are **uncommented**:

```conf
module(load="imtcp")
input(type="imtcp" port="514")
```

Restart rsyslog:

```bash
sudo systemctl restart rsyslog
```

### 10.2 Configure Junos to Send Syslog over TCP

On the Junos device, change your syslog configuration to use TCP:

```bash
configure
delete system syslog host 10.49.101.34
set system syslog host 10.49.101.34 any any
set system syslog host 10.49.101.34 match ".*"
set system syslog host 10.49.101.34 transport tcp
commit
exit
```

> **Note:** Syntax varies slightly between Junos versions; use `?` in CLI if needed:
> `set system syslog host 10.49.101.34 transport ?`

### 10.3 Enable TLS for Syslog (High-Level Overview)

To enable **TLS** between Junos and rsyslog, you typically need to:

1. **On Ubuntu (rsyslog):**

   * Install the needed TLS modules (often already included).
   * Generate or obtain a server certificate (and CA if needed).
   * Configure rsyslog to:

     * load `gtls` module
     * configure TLS certificates
     * listen on a secure port (e.g. `6514/tcp`) using TLS

   Example rsyslog snippet (high-level, not exhaustive):

   ```conf
   module(load="imtcp")
   module(load="gtls")

   # TLS config (paths depend on your environment)
   global(
     DefaultNetstreamDriver="gtls"
     DefaultNetstreamDriverCAFile="/etc/ssl/certs/ca-certificates.crt"
     DefaultNetstreamDriverCertFile="/etc/rsyslog.d/certs/server-cert.pem"
     DefaultNetstreamDriverKeyFile="/etc/rsyslog.d/certs/server-key.pem"
   )

   # Secure TCP listener (IANA-recommended port 6514)
   input(
     type="imtcp"
     port="6514"
     StreamDriver="gtls"
     StreamDriverMode="1"   # run driver in TLS-only mode
     StreamDriverAuthMode="x509/name"
   )
   ```

2. **On Junos:**

   * Install the CA certificate (and client certificate if using mutual auth).
   * Define a **TLS profile** in `services ssl`.
   * Reference that profile under the syslog host configuration.

   Example (conceptual):

   ```bash
   configure
   set services ssl certificate junos-syslog-cert certificate ...
   set services ssl profile tls-syslog protocol-version all
   set services ssl profile tls-syslog ca-profile <your-ca-profile>

   set system syslog host 10.49.101.34 any any
   set system syslog host 10.49.101.34 transport tls-profile tls-syslog
   set system syslog host 10.49.101.34 port 6514
   commit
   exit
   ```

   > Exact TLS commands and hierarchy can vary by Junos release — always refer to the version-specific Junos documentation.

3. **Open Firewall for Secure Port:**

   ```bash
   sudo ufw allow 6514/tcp
   ```

### 10.4 When Should You Use TLS?

You should strongly consider TLS if:

* Syslog traffic crosses untrusted networks (e.g. WAN, internet, shared infrastructure).
* Logs contain sensitive or customer-identifying information.
* You have compliance or internal security requirements mandating encryption in transit.

---

## Troubleshooting

If logs are not appearing on the Ubuntu server, check the following:

1. **Is rsyslog listening?**

   ```bash
   sudo ss -lunp | grep 514
   # For TCP:
   sudo ss -lntp | grep 514
   ```

2. **Any rsyslog syntax errors?**

   ```bash
   sudo systemctl status rsyslog
   sudo tail -n 50 /var/log/syslog
   ```

3. **Network connectivity from Junos to Ubuntu:**

   ```bash
   ping 10.49.101.34
   ```

4. **Firewall rules:**

   Ensure UDP/TCP port 514 (and 6514 if using TLS) is allowed.

5. **`fromhost-ip` match:**

   Confirm the IP in:

   ```conf
   if $fromhost-ip == '10.49.102.254' then -?JunosMsgs
   ```

   actually matches the **source IP** the SRX uses to send syslog.
   If the SRX uses a loopback or different interface, update the IP accordingly.

---

## Summary

In this guide we:

* Configured a Junos SRX to send syslog to an Ubuntu server
* Enabled rsyslog to receive remote syslog over UDP
* Created a custom template and rules to store Junos logs under `/var/log/junos-messages/` with one file per host per day
* Integrated **logrotate** to manage log growth
* Outlined how to harden the setup using **TCP** and **TLS**

This provides a solid, production-ready foundation for collecting and managing Junos syslog on Ubuntu, and can be extended easily to integrate with Elastic Stack, Loki, Splunk, or other observability platforms.

```
```
