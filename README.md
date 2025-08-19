# Splunk Add-on for Unix and Linux (TA-unix)

> A concise, practical README to help you install, configure, and get value fast from the **Splunk Add-on for Unix and Linux** on Splunk Enterprise.

![status](https://img.shields.io/badge/status-active-brightgreen) ![splunk](https://img.shields.io/badge/Splunk-Enterprise%2010%2B-blue) ![linux](https://img.shields.io/badge/Linux-amd64-lightgrey)

---

## Table of Contents

* [Overview](#overview)
* [Prerequisites](#prerequisites)
* [Quick Start](#quick-start)
* [Installation Options](#installation-options)

  * [A) Local Splunk Enterprise (CLI)](#a-local-splunk-enterprise-cli)
  * [B) Heavy/Universal Forwarder](#b-heavyuniversal-forwarder)
  * [C) Search Head / Indexer](#c-search-head--indexer)
* [Configuration](#configuration)

  * [Inputs](#inputs)
  * [Indexes & Sourcetypes](#indexes--sourcetypes)
  * [CIM Mappings](#cim-mappings)
* [Validation & Sample Searches](#validation--sample-searches)
* [Dashboards You Can Build Quickly](#dashboards-you-can-build-quickly)
* [Performance & Tuning](#performance--tuning)
* [Security Notes](#security-notes)
* [Troubleshooting](#troubleshooting)
* [Dev & Testing Tips](#dev--testing-tips)
* [Versioning](#versioning)
* [License](#license)

---

## Overview

The **Splunk Add-on for Unix and Linux (TA-unix)** collects host metrics and system inventory from Linux/Unix systems and normalizes data into Splunk-friendly **sourcetypes**. It ships scripted and modular inputs for CPU, memory, disk, process, network, and security-relevant telemetry. Use it standalone for admin visibility, or pair it with Splunk App for \*NIX (visualizations) and Splunk CIM for cross-domain correlation.

**What you get, fast:**

* Host performance: CPU, memory, IO, filesystem, processes.
* Network: interfaces, sockets, connections, ports.
* Platform inventory: uname, packages (optional), users/groups (optional).
* CIM-ready fields for Infrastructure Monitoring.

> **Note:** This README targets Splunk Enterprise **10.x on Linux amd64**.

---

## Prerequisites

* Linux host with shell access (amd64).
* Splunk Enterprise 10.x (or a compatible Heavy/Universal Forwarder).
* User with `sudo` to install packages and manage Splunk.
* Network egress for downloading packages.

---

## Quick Start

Below installs Splunk Enterprise locally (Ubuntu/Debian-style host) and gets TA-unix running quickly.

### 1) Install Splunk Enterprise

* First we need to download the Splunk Enterprise Debian package:
```bash
wget -O splunk-10.0.0-e8eb0c4654f8-linux-amd64.deb "https://download.splunk.com/products/splunk/releases/10.0.0/linux/splunk-10.0.0-e8eb0c4654f8-linux-amd64.deb"
```

* Next is to install the package into `/opt/splunk`, running Debian post‑install scripts; if Splunk is already present, it performs an in‑place upgrade:
```bash
sudo dpkg -i splunk-10.0.0-e8eb0c4654f8-linux-amd64.deb
```
<img width="1181" height="562" alt="image" src="https://github.com/user-attachments/assets/0ca2b59b-418d-4b72-b4c8-fc48dab0b206" />
🧾 What the installer output means:

> If you see `Setting up splunk (10.0.0) ...` followed by `complete`, the install succeeded.

| What you see | What it means |
|---|---|
| `(Reading database …)` | `dpkg` is scanning existing packages. |
| `Preparing to unpack … / preflight` | Pre‑install checks; environment looks OK. |
| `This looks like an upgrade …` | You already had Splunk; this is an in‑place reinstall/upgrade. |
| `extracting splunk_preinstall_base64 … / Adding execution bit` | Installer places helper scripts and sets executable permissions. |
| `KVStore path="/opt/splunk/var/lib/splunk/kvstore"` | Location of Splunk’s built‑in key‑value database used by apps/lookups. |
| `CPU Vendor/Family/Model … AVX/SSE4.2/AES‑NI` | Hardware capabilities detected for diagnostics/perf hints. |
| `Active KVStore version upgrade precheck PASSED (result: 0)` | KVStore data is compatible — safe to proceed. |
| `Attempting to stop … / splunkd is not running` | Service was already stopped; that’s fine. |
| `Unpacking splunk (10.0.0) over (10.0.0)` | Files are being replaced in place. |
| `Setting up splunk (10.0.0)` | Post‑install scripts run (create/update dirs, services). |
| `find: '/opt/splunk/lib/python3.7/site-packages': No such file or directory` | Harmless warning: Splunk 10 bundles a newer embedded Python under `/opt/splunk/lib/python3.x/`; the script checks an old 3.7 path. It **does not affect** installation. |
| `complete` | ✅ `dpkg` finished successfully.

🗄️ KVStore — concise notes
* What it is: Splunk’s embedded document store for small structured data. Records are JSON docs in collections with an automatic primary key _key. Data lives under $SPLUNK_DB/kvstore/ and is managed by splunkd.
* How you use it:
  * SPL: | inputlookup / | lookup / | outputlookup when the lookup definition points to a KV Store collection.
  * REST: CRUD and queries via _/servicesNS/{owner}/{app}/storage/collections/data/{collection}_ (supports where=, limit=, sort=).
* Good for: Enrichment/reference tables, app state, user prefs, asset/inventory lists—data that changes often but isn’t large enough to index as events.
* Not for: Large analytic datasets, heavy write throughput, or relational-style joins—use indexes or an external DB instead.
* Why mentioned during install/upgrade: The installer validates KVStore path/permissions and checks version/schema compatibility; may stage migrations. PASSED (result: 0) means you’re safe to proceed. In SHC, this is coordinated across members.
* Ops tips: Watch kvstore.log and splunkd.log for health, include $SPLUNK_DB/kvstore/ in backups, use TTL (_expireAt) to auto-expire stale docs, and ensure disk/ulimits are adequate on search heads.



```bash
sudo /opt/splunk/bin/splunk start --accept-license
```
<img width="1118" height="473" alt="image" src="https://github.com/user-attachments/assets/1e53fd13-1bdb-4606-b0d7-f29b66815292" />
<img width="1514" height="682" alt="image" src="https://github.com/user-attachments/assets/b04a75f8-d96b-4568-ab40-06f8995c4dce" />
<img width="1517" height="456" alt="image" src="https://github.com/user-attachments/assets/8354c5ef-5700-4af0-b2c9-47068d001000" />



### 2) Install the Splunk Add-on for Unix and Linux

**Option A — via CLI (local instance):**

```bash
# Download latest TA-unix .tgz from Splunkbase (or your artifacts repo)
# Example path once downloaded: /tmp/splunk-add-on-for-unix-and-linux_x.y.z.tgz

sudo /opt/splunk/bin/splunk install app /tmp/splunk-add-on-for-unix-and-linux_*.tgz -auth admin:Chang3Me
```

**Option B — copy to apps directory:**

```bash
sudo tar -xzf /tmp/splunk-add-on-for-unix-and-linux_*.tgz -C /opt/splunk/etc/apps/
```

Restart Splunk to load the add-on:

```bash
sudo /opt/splunk/bin/splunk restart
```

### 3) Enable common inputs

The TA includes default input stanzas under `default/inputs.conf`. Copy to `local/inputs.conf` and enable what you need.

```bash
sudo mkdir -p /opt/splunk/etc/apps/Splunk_TA_nix/local
sudo tee /opt/splunk/etc/apps/Splunk_TA_nix/local/inputs.conf >/dev/null <<'EOF'
[script://./bin/cpu.sh]
interval = 60
enabled = 1
index = os

[script://./bin/memory.sh]
interval = 60
enabled = 1
index = os

[script://./bin/iostat.sh]
interval = 60
enabled = 1
index = os

[script://./bin/df.sh]
interval = 300
enabled = 1
index = os

[script://./bin/processes.sh]
interval = 300
enabled = 1
index = os
EOF
```

Restart Splunk after changes:

```bash
sudo /opt/splunk/bin/splunk restart
```

---

## Installation Options

### A) Local Splunk Enterprise (CLI)

* Good for single-node testing, POCs, and lab work.
* Use `splunk install app` for add-ons and apps.
* Persist changes in `local/*.conf` (never edit `default/`).

### B) Heavy/Universal Forwarder

* Install TA-unix on forwarders to collect data *at the edge*.
* Send to indexers via `outputs.conf`.
* Keep any search-time props/transforms on search heads (or deploy via SHC Deployer).

### C) Search Head / Indexer

* Install TA-unix if you need search-time knowledge (props/transforms, field aliases, CIM extractions) available at search time.
* For index-time transformations, ensure parity across parsing tier.

---

## Configuration

### Inputs

Common scripted inputs (enable as needed):

* `cpu.sh`, `memory.sh`, `iostat.sh`, `df.sh`, `swap.sh`
* `interfaces.sh`, `netstat.sh`, `lsof.sh` (optional, may require elevated privileges)
* `processes.sh`, `uptime.sh`, `vmstat.sh`
* `users.sh`, `groups.sh` (inventory)

> Tip: Start with 60s intervals for CPU/memory and 300s for disk/process to balance fidelity and licensing.

### Indexes & Sourcetypes

Recommend a dedicated index (e.g., `os`). Key sourcetypes you’ll see:

* `unix:cpu`, `unix:memory`, `unix:df`, `unix:process`, `unix:iostat`, `unix:netstat`, `unix:interfaces`, `unix:vmstat`

Example `indexes.conf` (indexer):

```ini
[os]
homePath = $SPLUNK_DB/os/db
coldPath = $SPLUNK_DB/os/colddb
thawedPath = $SPLUNK_DB/os/thaweddb
maxTotalDataSizeMB = 500000
repFactor = auto
```

### CIM Mappings

* Maps primarily to **Compute Inventory** and **Performance** data models.
* Ensure the Splunk Common Information Model (CIM) add-on is installed on search heads.

---

## Validation & Sample Searches

Sanity checks once data is flowing:

```spl
index=os sourcetype=unix:cpu | timechart avg(pctIdle) as pctIdle avg(pctUser) as pctUser
```

```spl
index=os sourcetype=unix:memory | stats latest(used) as used latest(free) as free by host
```

```spl
index=os sourcetype=unix:df | stats latest(usepct) as usepct by host filesystem | where usepct>80
```

```spl
index=os sourcetype=unix:process | stats count by host process
```

```spl
index=os sourcetype=unix:netstat state=LISTEN | stats values(local_address) as listen_addrs by host
```

**Field health:**

```spl
index=os | head 1000 | fieldsummary | search field IN(host, sourcetype, source)
```

---

## Dashboards You Can Build Quickly

* **Host Overview:** CPU, memory, load, disk usage by host with drilldowns.
* **Storage Heatmap:** Filesystem `usepct` treemap to spot capacity issues.
* **Network Ports:** Listening ports by host, anomalous changes over time.
* **Top Talkers (Processes):** Processes by CPU or RSS growth.

> Export these as SimpleXML or Dashboards Studio. Keep panels using the `os` index and `unix:*` sourcetypes for portability.

---

## Performance & Tuning

* Right-size input **intervals** to manage license consumption.
* Use **whitelists/blacklists** in inputs to exclude noisy filesystems or interfaces.
* Consider **compressed forwarding** and ACKs for WAN links.
* Keep **TA-unix search-time extractions** on search heads only for efficiency.

---

## Security Notes

* Some scripts (e.g., `lsof.sh`, `netstat.sh`) may reveal sensitive info; restrict role access to sourcetypes if needed.
* Validate scripts under `bin/` are executable and reviewed. Avoid custom edits in `default/`.
* Rotate Splunk admin credentials; use role-based access control and secure management port 8000.

---

## Troubleshooting

### ℹ️ Installer message you can ignore

During `dpkg -i ...` you may see:

```
find: '/opt/splunk/lib/python3.7/site-packages': No such file or directory
```

**What it means**

* Splunk Enterprise 10.x bundles a newer embedded Python 3 runtime (not 3.7), so that legacy path doesn’t exist.
* The package’s post-install script briefly probes that path; `find` logs a warning when it’s missing.
* This is **non-fatal** and does not indicate a failed install. If you also see `Setting up splunk (10.0.0) ...` followed by `complete`, installation succeeded.

**Verify the install**

```bash
/opt/splunk/bin/splunk version
/opt/splunk/bin/splunk status
/opt/splunk/bin/python3 -V
ls /opt/splunk/lib | grep -E 'python3\.[0-9]+'
```

You should see Splunk’s version and a `python3.x` directory (other than `python3.7`). If `status` shows `splunkd is running`, you’re good.

* **No data?** Check `splunkd.log` and `introspection` index.
* **Script failures?** Run scripts manually: `sudo -u splunk /opt/splunk/etc/apps/Splunk_TA_nix/bin/cpu.sh`.
* **Permissions:** Ensure `splunk` user can execute shell scripts and read system utilities (e.g., `iostat` requires `sysstat`).
* **Sourcetype mismatches:** Inspect `props.conf` and `transforms.conf`.
* **Excessive volume:** Increase intervals; filter with `blacklist` and `whitelist` in inputs.

Useful commands:

```bash
sudo /opt/splunk/bin/splunk btool inputs list --debug | less
sudo /opt/splunk/bin/splunk btool props list --debug | less
sudo tail -f /opt/splunk/var/log/splunk/splunkd.log
```

---

## Dev & Testing Tips

* Keep your `local/` directory under version control (or generate from templates).
* Use **Deployment Server** or your CD pipeline to fan out `local/` configs to forwarders.
* For reproducible labs, spin up ephemeral VMs/containers and load a sample data pack.

---

## Versioning

This repo targets **Splunk Enterprise 10.0.0+** and the contemporary **Splunk Add-on for Unix and Linux**. Pin your TA version in `manifest` or your pipeline.

---

## License

MIT (or your project’s license). Include third-party notices if you ship TA artifacts.

---

## Appendix: Minimal `inputs.conf` for a small lab

```ini
[script://$SPLUNK_HOME/etc/apps/Splunk_TA_nix/bin/cpu.sh]
interval = 60
enabled = 1
index = os

[script://$SPLUNK_HOME/etc/apps/Splunk_TA_nix/bin/memory.sh]
interval = 60
enabled = 1
index = os

[script://$SPLUNK_HOME/etc/apps/Splunk_TA_nix/bin/df.sh]
interval = 300
enabled = 1
index = os
```
