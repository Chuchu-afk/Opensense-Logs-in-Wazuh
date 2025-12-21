# OPNsense → Wazuh Syslog Integration & Troubleshooting Guide

## Overview

This document covers forwarding OPNsense firewall logs to Wazuh,
validating ingestion, and troubleshooting visibility issues.

------------------------------------------------------------------------

## Architecture

-   OPNsense → Syslog (UDP 514)
-   Wazuh Manager → Archives → OpenSearch

------------------------------------------------------------------------

## Step 1: Configure OPNsense Syslog

Path: System → Settings → Logging / Targets

Settings: - Enabled: Yes - Transport: UDP - Port: 514 - Hostname:
WAZUH_SERVER_IP - Applications: All - Facilities: All - Levels: All -
RFC5424: Disabled

------------------------------------------------------------------------

## Step 2: Enable Syslog in Wazuh

Edit:

``` bash
sudo vi /var/ossec/etc/ossec.conf
```

``` xml
<remote>
  <connection>syslog</connection>
  <port>514</port>
  <protocol>udp</protocol>
  <allowed-ips>0.0.0.0/0</allowed-ips>
</remote>
```

Restart:

``` bash
sudo systemctl restart wazuh-manager
```

------------------------------------------------------------------------

## Step 3: Enable Log Archiving

``` xml
<global>
  <logall>yes</logall>
  <logall_json>yes</logall_json>
</global>
```

------------------------------------------------------------------------

## Step 4: Verify Log Arrival

``` bash
sudo tcpdump -ni any udp port 514
sudo tail -n 20 /var/ossec/logs/archives/archives.log
```

------------------------------------------------------------------------

## Step 5: Enable Archive Indexing

Edit Filebeat:

``` bash
sudo vi /etc/filebeat/filebeat.yml
```

``` yaml
filebeat.modules:
  - module: wazuh
    alerts:
      enabled: true
    archives:
      enabled: true
```

Restart:

``` bash
sudo systemctl restart filebeat
```

------------------------------------------------------------------------

## Step 6: Create Index Pattern

Pattern:

    wazuh-archives-*

Time field:

    @timestamp

------------------------------------------------------------------------

## Validation

-   Logs visible in Discover
-   Index `wazuh-archives-*` exists
-   OPNsense logs searchable (`filterlog`, `opnsense`)

------------------------------------------------------------------------

## Common Issues

-   No index → Filebeat archives disabled
-   No logs → logall not enabled
-   Seen in tcpdump only → indexing missing

