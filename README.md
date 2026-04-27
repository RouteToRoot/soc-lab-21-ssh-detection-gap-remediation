# Lab 21 — SSH Detection Gap Remediation & SIEM Validation

## Executive Summary

A persistent SSH authentication log detection gap was identified in Lab 13 and confirmed across Labs 14–20. SSH failed authentication events were present in the Kali Linux systemd journal but were not being forwarded to Elastic SIEM's `logs-system.auth-*` data stream, preventing alert generation. This lab documents the root cause analysis, remediation steps, and end-to-end validation confirming the gap is resolved.

---

## Incident Ticket (ServiceNow Simulation)

| Field | Details |
|---|---|
| **Incident ID** | INC-0021 |
| **Date/Time Detected** | April 13, 2026 (Lab 13) |
| **Date/Time Remediated** | April 26, 2026 @ 19:56 PST |
| **Detected By** | Manual log review — Elastic SIEM Discover |
| **Severity** | High |
| **Category** | Detection Engineering |
| **Subcategory** | Log Forwarding / SIEM Ingestion |
| **Short Description** | SSH auth logs not forwarding to Elastic SIEM |
| **Detailed Description** | Kali Linux uses systemd-journald as its logging backend. SSH authentication failures were captured in the journal but rsyslog was not installed, so no `/var/log/auth.log` file existed. Elastic Agent's System integration requires a file-based log source and could not ingest journal-only logs. This caused all SSH brute force simulation events to go undetected by Elastic SIEM across Labs 13–20. |
| **IOCs** | `invalid_user@127.0.0.1`, repeated failed SSH authentication attempts |
| **Analysis** | Root cause: rsyslog not installed on Kali Linux. Secondary cause: `ForwardToSyslog` not enabled in journald.conf. Combined effect: no auth.log file, no Elastic ingestion, no alerts. |
| **Impact Assessment** | SSH brute force attacks against this host would not generate SIEM alerts. Detection capability was effectively zero for this attack vector across the entire lab environment. |
| **Response Actions Taken** | Enabled ForwardToSyslog in journald.conf, installed rsyslog, restarted systemd-journald, confirmed auth.log creation, validated SSH events writing to file, confirmed ingestion into logs-system.auth-* in Elastic, ran 10-attempt brute force simulation, confirmed 31 alerts fired. |
| **Recommended Actions** | Verify log forwarding on all Linux hosts before deploying detection rules. Treat missing auth.log as a critical gap during any SOC onboarding or audit. |
| **Status** | Resolved |

---

## Lab Objectives

- Identify the root cause of the SSH authentication log detection gap
- Remediate the gap by installing rsyslog and enabling syslog forwarding
- Confirm SSH auth events are writing to `/var/log/auth.log`
- Validate Elastic SIEM is ingesting events into `logs-system.auth-*`
- Simulate a brute force attack and confirm alerts fire end-to-end

---

## Environment Overview

| Component | Details |
|---|---|
| Host OS | Windows |
| VM | Kali Linux (VMware Workstation) |
| SIEM | Elastic Cloud Serverless (GCP Iowa) |
| Agent | Elastic Agent v9.3.3 |
| Integration | System v2.16.2 |
| Log Target | `/var/log/auth.log` |

---

## Root Cause Analysis

Kali Linux does not install rsyslog by default. Without rsyslog, systemd-journald has no syslog consumer to forward logs to, and no `/var/log/auth.log` file is created. Elastic Agent's System integration monitors file-based log sources — without the file, SSH authentication events were never ingested into Elastic SIEM.

Additionally, `ForwardToSyslog` in `/etc/systemd/journald.conf` was set to `no` by default, meaning even if rsyslog had been installed, forwarding would not have occurred without this configuration change.

---

## Remediation Workflow

### Step 1 — Enable syslog forwarding in journald

```bash
sudo nano /etc/systemd/journald.conf
# Changed: ForwardToSyslog=yes
sudo systemctl restart systemd-journald
```

### Step 2 — Install rsyslog

```bash
sudo apt install rsyslog -y
```

### Step 3 — Confirm auth.log exists

```bash
ls /var/log/auth.log
# Output: /var/log/auth.log
```

### Step 4 — Simulate SSH brute force

```bash
for i in {1..10}; do ssh -o ConnectTimeout=5 invalid_user@127.0.0.1; done
```

### Step 5 — Confirm events in auth.log

```bash
sudo grep "invalid_user" /var/log/auth.log
```

### Step 6 — Confirm ingestion in Elastic

KQL query in Discover:

Result: 61 documents returned

### Step 7 — Confirm alerts in Kibana

- Rule triggered: SSH Authentication Failure Detected
- Alert count: 31
- Host: kali (100%)
- Severity: Medium
- Risk Score: 47

---

## Evidence

| File | Description |
|---|---|
| `elastic-auth-logs-confirmed.png` | Kibana Discover showing 61 system.auth documents from kali host |
| `elastic-ssh-alerts-remediated.png` | Kibana Alerts page showing 31 SSH Authentication Failure alerts |

---

## Detection Engineering Insights

- Always verify file-based log sources exist before deploying detection rules
- Kali Linux requires rsyslog installation and journald forwarding configuration — this is not default
- A detection rule with no log source is silent — it will not error, it simply will not fire
- This gap was present across 8 labs without generating a single false negative indicator, making it difficult to detect without explicit log source verification

---

## Conclusions

The SSH authentication log detection gap has been fully remediated. SSH brute force events now flow from the Kali Linux systemd journal → rsyslog → `/var/log/auth.log` → Elastic Agent → `logs-system.auth-*` → Kibana alert. End-to-end detection is confirmed with live evidence.

---

## Next Steps

- Deploy Splunk SIEM and replicate detection coverage (Labs 22–25)
- Build portfolio landing page at routetoroot.github.io
- Begin job application process
