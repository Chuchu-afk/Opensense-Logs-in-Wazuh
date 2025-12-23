# OPNsense → Wazuh Syslog Integration (Intentional Build Log)
![Wazuh](https://github.com/Chuchu-afk/Opensense-Logs-in-Wazuh/blob/main/wazuh%20opnsense.png?raw=true)
## Purpose

This project was intentionally designed to simulate an **enterprise-grade network security logging pipeline** using open-source tools.  
The goal was not simply to “get logs into a SIEM,” but to **understand, validate, and document** how firewall telemetry flows from a perimeter device into a centralized security platform — including the failure modes that commonly break these integrations in real environments.

This lab reflects the type of problems encountered by SOC analysts, security engineers, and infrastructure teams when onboarding network devices into SIEM platforms such as Wazuh, Splunk, or Elastic.

---

## Table of Contents
1. Purpose  
2. Goals  
3. Lab Topology (Simplified & Detailed)  
4. Design Decisions  
5. Implementation Steps (Working Configuration)  
6. Validation & Verification  
7. Troubleshooting & Failure Analysis (Chronological, Intentional)  
8. Lessons Learned  

---

## Goals

- Forward firewall and system logs from OPNsense to Wazuh
- Use **syslog over UDP** to emulate real firewall telemetry
- Ensure logs are **parsed, indexed, and visible** in Wazuh Dashboard
- Understand **why logs fail** when they do not appear
- Document all decisions, assumptions, failures, and fixes

---

## Lab Topology (Simplified)

```
[ Internal Network ]
        |
   [ OPNsense ]
        |
   (Syslog UDP 514)
        |
   [ Wazuh Manager ]
        |
   [ Wazuh Indexer + Dashboard ]
```

> All IP addresses in this documentation are intentionally **censored** and represented as:
- `192.168.x.x` (internal)
- `<WAZUH_SERVER_IP>`

---

## Lab Topology (Detailed)

- **OPNsense Firewall**
  - Acts as perimeter security device
  - Generates firewall, system, and service logs
  - Forwards logs via syslog (UDP/514)

- **Wazuh Server (All-in-One)**
  - Wazuh Manager
  - Wazuh Indexer
  - Wazuh Dashboard
  - Receives and processes syslog events

---

## Design Decisions

### Why Syslog (UDP/514)?
- Matches how most enterprise firewalls export logs
- Stateless and lightweight
- Failure scenarios are realistic and common

### Why Not Install a Wazuh Agent on OPNsense?
- OPNsense is FreeBSD-based
- Wazuh agent support is limited and unstable
- Syslog ingestion is the **recommended and supported method**

---

## Implementation Steps (Working Configuration)

### Step 1 — Configure Wazuh to Accept Syslog

On the **Wazuh server**, edit:

```
/var/ossec/etc/ossec.conf
```

Ensure the following block exists **inside `<ossec_config>`**:

```xml
<!-- Syslog ingestion (OPNsense) -->
<remote>
  <connection>syslog</connection>
  <port>514</port>
  <protocol>udp</protocol>
  <allowed-ips>0.0.0.0/0</allowed-ips>
</remote>
```

Restart Wazuh Manager:

```bash
sudo systemctl restart wazuh-manager
```

Verify listener:

```bash
sudo ss -lunp | grep 514
```

Expected:
- `wazuh-remoted` listening on UDP 514

---

### Step 2 — Configure OPNsense Syslog Forwarding

Navigate to:

```
System → Settings → Logging / targets
```

Create a new destination:

| Field | Value |
|-----|------|
| Enabled | ✔ |
| Transport | UDP (IPv4) |
| Hostname | `<WAZUH_SERVER_IP>` |
| Port | `514` |
| Applications | Select All |
| Levels | Select All |
| Facilities | Select All |
| RFC5424 | ❌ (disabled) |

Save and apply.

---

### Step 3 — Firewall Rule (OPNsense)

On **LAN interface**, create a rule:

- **Action:** Pass
- **Protocol:** UDP
- **Source:** LAN network (`192.168.x.0/24`)
- **Destination:** Wazuh server (`192.168.x.x`)
- **Destination Port:** 514
- **Description:** Allow syslog to Wazuh

> This rule ensures logs are not silently dropped before reaching Wazuh.

---

## Validation & Verification

### Verify Logs Are Arriving (Server Side)

```bash
sudo tcpdump -ni any udp port 514
```

Expected:
- Continuous traffic from OPNsense

### Verify Wazuh Is Processing Logs

```bash
sudo tail -n 50 /var/ossec/logs/archives/archives.log
```

Expected:
- `filterlog`, `opnsense`, and firewall entries

---

### Verify in Wazuh Dashboard

- Navigate to **Discover**
- Select index:
  ```
  wazuh-alerts-*
  ```
- Search:
  ```
  opnsense OR filterlog
  ```

---

## Troubleshooting & Failure Analysis (Intentional)

This section documents **real failures encountered**, the reasoning behind each investigation, and the resolution.

---

### Issue 1 — Logs Not Appearing in Dashboard

**Observation**
- `tcpdump` showed traffic
- Dashboard was empty

**Analysis**
- Logs arriving ≠ logs indexed
- Required confirmation at archive layer

**Fix**
- Verified `archives.log`
- Confirmed correct index (`wazuh-alerts-*`)
- Adjusted time range in Dashboard

---

### Issue 2 — UDP 514 Listening but No Data

**Observation**
- Wazuh listening
- No logs arriving

**Analysis**
- Firewall rules likely blocking traffic
- OPNsense blocks silently by default

**Fix**
- Added explicit LAN rule allowing UDP 514
- Re-tested with `tcpdump`

---

### Issue 3 — Permissions Denied on archives.log

**Observation**
- `ls` returned permission denied

**Analysis**
- Wazuh logs are restricted by design

**Fix**
```bash
sudo tail /var/ossec/logs/archives/archives.log
```

---

### Issue 4 — Dashboard Shows No Index Data (Last)

**Why This Was Last**
- UI symptoms often mislead
- Infrastructure must be verified first

**Fix**
- Confirmed ingestion
- Verified index pattern
- Confirmed timestamps

---

## Lessons Learned

- Syslog ingestion is deceptively simple but fragile
- Always validate **network → service → log → index → UI**
- Wazuh Dashboard visibility ≠ data absence
- Troubleshooting must follow a layered approach

---

## Final Outcome

- OPNsense logs successfully ingested
- Logs visible in Wazuh Dashboard
- Enterprise-style logging pipeline validated
- Full troubleshooting knowledge documented

---

## Status

✅ Complete  
📘 Documented  
🔒 Sanitized for public GitHub  
💼 Portfolio-ready
