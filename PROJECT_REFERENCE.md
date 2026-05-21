# Firewall Path Tracer — Project Reference (Living Document)

> **Last Updated:** May 2026  
> **Purpose:** Single source of truth for understanding the entire system without re-scanning files  
> **Audience:** Engineers onboarding, maintaining, or extending this codebase

---

## Table of Contents

1. [Repository Structure](#1-repository-structure)
2. [Webserver Architecture](#2-webserver-architecture)
3. [API Catalog](#3-api-catalog)
4. [Data Storage Model](#4-data-storage-model)
5. [Key User Flows](#5-key-user-flows)
6. [Per-File Reference](#6-per-file-reference)
7. [Naming Conventions & Invariants](#7-naming-conventions--invariants)
8. [Legacy/Duplicate Implementations](#8-legacyduplicate-implementations)
9. [Update Procedure](#9-update-procedure)
10. [Diff Log](#10-diff-log)

---

## 1. Repository Structure

```
project2/
├── server.py              # Primary Python/Flask backend (~9981 lines) (ACTIVE)
├── server.js              # Legacy Node.js/Express backend (DEPRECATED)
├── run_server.py          # Launcher script for Python server
│
├── utils/
│   ├── __init__.py
│   ├── helpers.py           # Formatting/validation helpers
│   └── file_helpers.py      # Device state & cache helpers
├── parsers/
│   ├── __init__.py
│   └── output_parsers.py    # Show route/ip/version/fxos output parsers
├── services/
│   ├── __init__.py
│   └── show_version_cache.py # Show version cache CRUD & extraction
├── ssh/
│   ├── __init__.py
│   └── client.py            # SSH connections, jumpbox pool, command execution
│
├── app/
│   └── web/
│       ├── templates/
│       │   ├── index.html           # Main SPA template
│       │   └── index.direct.html    # Direct mode template (alternative entry)
│       └── static/
│           ├── css/
│           │   └── styles.css       # Global styles
│           ├── js/
│           │   ├── app.js           # Core navigation, credentials, data collection
│           │   ├── firewall_backups.js  # Backup comparison feature
│           │   ├── global_rules.js  # Global ACL rule generation
│           │   ├── show_version.js  # Show version collection UI
│           │   ├── firewall_rules_auto.js  # Firewall Rules Auto (Phase 1: rule preparation)
│           │   └── test_myers_diff.html  # Myers diff visual test page
│           └── vendor/
│               ├── bootstrap/        # Bootstrap 5
│               └── fontawesome/     # Font Awesome icons
│
├── config/
│   ├── asa_inventory.txt       # ASA device inventory (hostname,ip,type,is_gateway)
│   ├── ftd_inventory.txt       # FTD device inventory
│   ├── fxos_inventory.txt      # FXOS device inventory
│   ├── FirewallDeploy_inventory.txt  # Firewall deploy device inventory
│   └── global_rules_inventory.txt # Firewall hostnames for global rules
│
├── data/
│   ├── collected_devices/       # Device cache (hostname.json)
│   ├── device_state/            # Per-device state (failover, preferred_role)
│   ├── session_logs/            # Test execution logs (written by server.py)
│   ├── show_version_cache/      # Show version data cache
│   ├── tmp_diff/                # Scalable compare job storage
│   └── live_test_logs/          # LEGACY: present on disk but unused by Flask server
│
├── docs/
│   ├── PROGRESS.md
│   ├── PHASES.md
│   ├── PROJECT_REFERENCE.md
│   ├── Firewall_Rules_Auto_User_Guide.md   # FRA user guide
│   └── FRA_REPORTS_LOGVIEWER_REFERENCE.md  # FRA reports & log viewer reference
│
└── misc/                         # Development docs, tests, utilities, backups
    ├── requirements.txt          # Python dependencies (launcher expects this here)
    ├── FIREWALL_BACKUPS_*.md     # Implementation summaries (5 files)
    ├── FXOS_*.md                # FXOS feature documentation (6 files)
    ├── DEVICE_STATE_*.md         # State management docs
    ├── *.js, *.py               # Test scripts and utilities
    ├── *.bat, *.ps1             # Windows launcher scripts
    ├── server.py.*              # Server backups
    ├── package.json             # NPM package metadata
    └── README.md                # Misc README
```

### File Inventory Summary

| Path | Purpose | Key Responsibilities |
|------|---------|---------------------|
| `server.py` | Core orchestration (~9981 lines) | Flask app, API routes, job system, scheduler, FRA logic, credentials |
| `server.js` | Legacy backend | Deprecated Node.js implementation |
| `utils/helpers.py` | Formatting/validation (~68 lines) | `clean_command_output`, `prefix_to_netmask`, `is_valid_netmask`, `escape_html_for_server` |
| `utils/file_helpers.py` | File/device-state helpers (~202 lines) | `parse_inventory_file`, `get_cache_status`, `load/save_device_state`, `get_device_info_from_cache` |
| `parsers/output_parsers.py` | Output parsers (~385 lines) | `parse_show_route_output`, `parse_show_ip_output`, `parse_show_version_output`, `parse_fxos_chassis_detail` |
| `services/show_version_cache.py` | Show version cache (~105 lines) | `load/save_show_version_cache`, `extract_context_from_show_version_text`, `strip_device_output` |
| `ssh/client.py` | SSH & jumpbox layer (~831 lines) | `connect_via_jumpbox`, `JumpboxSessionManager`, `run_commands_on_device`, `run_failover_check_on_device` |
| `run_server.py` | Launcher | Checks prerequisites, installs deps, starts server |
| `app/web/static/js/app.js` | Core frontend (~2584 lines) | Navigation, credentials save, device collection, ping/packet-tracer/route/NAT tests |
| `app/web/static/js/firewall_backups.js` | Backup comparison (~2700 lines) | Archive listing, config file browsing, diff comparison, virtual scrolling |
| `app/web/static/js/global_rules.js` | ACL generation (~733 lines) | Normal/URL rule generation with deduplication |
| `app/web/static/js/show_version.js` | Version collection (~326 lines) | Show version fetch, display, CSV export |
| `app/web/static/js/firewall_rules_auto.js` | Rules Auto (Phase 1) (~1528 lines) | Base path, date folders, ensure/prepare/archive/deploy; calls `/api/firewall-rules-auto/*` |
| `app/web/templates/index.html` | Main UI (~1415 lines) | All page sections with DOM IDs |
| `app/web/static/css/styles.css` | Styling (~333 lines) | Dark theme, terminal styles, diff highlighting |

---

## 2. Webserver Architecture

### 2.1 Request Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           CLIENT (Browser)                                    │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         index.html (SPA Shell)                               │
│   - Bootstrap + FontAwesome loaded                                          │
│   - JS modules registered (app.js, firewall_backups.js, etc.)              │
│   - Hash-based routing (#path-finder, #firewall-rules-auto, #firewall-backups, etc.) │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                        JavaScript Modules                                   │
│  app.js ─ firewall_backups.js ─ show_version.js ─ firewall_rules_auto.js   │
│       │              │                  │                    │               │
│       ▼              ▼                  ▼                    ▼               │
│  ┌──────────────┐ ┌──────────────────┐ ┌─────────────────┐ ┌──────────────┐ │
│  │ Credentials  │ │ Compare Job Mgmt │ │ Version Fetch   │ │ Rule prep    │ │
│  │ Collection   │ │ Virtual Scroll   │ │ CSV Export      │ │ Date folders │ │
│  │ Path Finder  │ │ Diff Navigation│ │                 │ │ /api/fra/*   │ │
│  │ Live Tests   │ │ Search/Highlight│ │                 │ │              │ │
│  └──────────────┘ └──────────────────┘ └─────────────────┘ └──────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
                              fetch() / POST()
                                    │
══════════════════════════════════════╪═══════════════════════════════════════
                                    │ HTTP JSON
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                       server.py (Flask)                                     │
│                                                                             │
│  ┌──────────────────────────────────────────────────────────────────────┐    │
│  │ Session Management (threading.Lock)                                  │    │
│  │  SESSION_CREDS = {sid: {jumpbox, asa, ftd, _last_seen}}         │    │
│  │  SESSION_TTL_SECONDS = 30 min                                      │    │
│  └──────────────────────────────────────────────────────────────────────┘    │
│                                    │                                        │
│  ┌──────────────────────────────────────────────────────────────────────┐    │
│  │ Credential Validation Middleware                                    │    │
│  │  get_session_creds_required(device_types)                          │    │
│  │  get_session_creds_for_device_type(device_type)                   │    │
│  └──────────────────────────────────────────────────────────────────────┘    │
│                                    │                                        │
│  ┌──────────────────────────────────────────────────────────────────────┐    │
│  │ Route Handlers (see API Catalog)                                   │    │
│  │  /api/credentials          /api/backups/*                            │    │
│  │  /api/devices             /api/auto-select-firewalls                 │    │
│  │  /api/collect-*          /api/run-*                                │    │
│  │  /api/show-version-*     /api/preferred-role                      │    │
│  │  /api/firewall-rules-auto/*  (local FS: Prepare_Rules, date folders) │    │
│  └──────────────────────────────────────────────────────────────────────┘    │
│                                    │                                        │
│  ┌──────────────────────────────────────────────────────────────────────┐    │
│  │ Device Connection Layer (paramiko)                                  │    │
│  │  connect_via_jumpbox()                                             │    │
│  │  connect_to_device_via_jumpbox()  (direct-tcpip or shell-hop)      │    │
│  │  connect_to_device_via_jumpbox_shell()                              │    │
│  │  JumpboxSessionManager (connection pooling)                          │    │
│  └──────────────────────────────────────────────────────────────────────┘    │
│                                    │                                        │
│  ┌──────────────────────────────────────────────────────────────────────┐    │
│  │ Command Execution Layer                                             │    │
│  │  execute_command_wait_prompt()                                       │    │
│  │  execute_single_command()                                           │    │
│  │  run_commands_on_device()                                            │    │
│  │  run_failover_check_on_device()                                     │    │
│  │  run_show_version_on_device()                                        │    │
│  └──────────────────────────────────────────────────────────────────────┘    │
│                                    │                                        │
│  ┌──────────────────────────────────────────────────────────────────────┐    │
│  │ Parsing Layer                                                      │    │
│  │  parse_show_ip_output()           parse_show_route_output()        │    │
│  │  parse_show_version_output()        parse_fxos_chassis_detail()    │    │
│  │  parse_failover_state()                                            │    │
│  └──────────────────────────────────────────────────────────────────────┘    │
│                                    │                                        │
└────────────────────────────────────┼────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           FILESYSTEM                                       │
│                                                                             │
│  config/*.txt                    ← Device inventories                       │
│  data/collected_devices/*.json   ← Device data cache                       │
│  data/device_state/*.json        ← Per-device state                        │
│  data/session_logs/*.json        ← Test execution logs                     │
│  data/show_version_cache/*.json  ← Show version cache                       │
│  data/tmp_diff/<job_id>/         ← Compare job storage                      │
│    ├── diff.db                   ← SQLite (LEGACY: compare-configs-job only)│
│    ├── left.txt                  ← Extracted config (scalable compare)      │
│    ├── right.txt                 ← Extracted config (scalable compare)      │
│    ├── left_index.json           ← Byte offsets for left.txt               │
│    ├── right_index.json          ← Byte offsets for right.txt              │
│    ├── hunks.json                ← Diff hunk metadata                       │
│    └── metadata.json             ← Job metadata                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 Session & Credential Flow

```
Browser Session                    server.py
─────────────                    ────────
                                
1. User saves credentials   ────► POST /api/credentials
   {jumpbox, asa, ftd}          │
                                 SESSION_CREDS[sid] = {
                                   jumpbox: {...},
                                   asa: {...},
                                   ftd: {...},
                                   _last_seen: timestamp
                                 }
                                 │
2. Subsequent requests      ────► Any protected endpoint
   (Cookie: session)              │
                                 get_session_creds_required()
                                 │
                                 │ Validates TTL, returns creds
                                 ▼
                                
3. TTL expires (30 min)   ────► Next request
   Session purged                  │
                                 SESSION_CREDS[sid] = None
```

---

## 3. API Catalog

### 3.1 Credential Endpoints

| Endpoint | Method | Params | Response | Caller |
|----------|--------|--------|----------|--------|
| `/api/credentials` | POST | `{jumpbox, asa, ftd}` | `{status: 'success'}` | `app.js:saveIndividualCredentials()` |
| `/api/credentials/clear` | POST | - | `{status: 'success'}` | `app.js:clearCredentialsAll()` |

### 3.2 Device Management Endpoints

| Endpoint | Method | Params | Response | Caller |
|----------|--------|--------|----------|--------|
| `/api/devices` | GET | - | `[{hostname, ip, type, is_internet_gateway, cache, status, failover_state, preferred_role, ...}]` | `app.js:populateMockData()` |
| `/api/device-interfaces/<hostname>` | GET | - | `[{name, ip_address, netmask, status}]` | `app.js:populateInterfaceDropdown()` |
| `/api/collect-single-device` | POST | `{hostname}` | `{status, hostname, steps, message}` | `app.js:runSequential(kind='collect'), .collect-one-btn handler` |
| `/api/collect-data` | POST | `{all: bool} or {devices: []}` | `[{status, hostname, steps, message}, ...]` | **Legacy**: not used by current UI (frontend calls `/api/collect-single-device` sequentially instead) |
| `/api/failover-check` | POST | `{hostname} or {all: true}, {tabInstanceId}` | `{status, hostname, failover_state, steps}` | `app.js:runSequential(kind='failover'), .failover-one-btn handler` (UI sends `{hostname, tabInstanceId}` per-device; batch sets `{all:true, tabInstanceId}`; registers job with scope `"failover-check"`) |
| `/api/preferred-role` | POST | `{hostname, preferred_role}` | `{status, hostname, preferred_role}` | `app.js:preferred-role handler` |

### 3.3 Auto-Selection Endpoints

| Endpoint | Method | Params | Response | Caller |
|----------|--------|--------|----------|--------|
| `/api/auto-select-firewalls` | POST | `{src_ip, dst_ip}` | `{src: {status, match: {...}}, dst: {status, match: {...}}}` | `app.js:autoDetectFirewallsAndInterfaces()` |

### 3.4 Live Test Endpoints

| Endpoint | Method | Params | Response | Caller |
|----------|--------|--------|----------|--------|
| `/api/run-ping` | POST | `{hostname, command, direction}` | `{status, hostname, command, output, direction, timestamp}` | `app.js:executePingTest()` (creds: per-device-type via `get_session_creds_for_device_type`) |
| `/api/run-packet-tracer` | POST | `{hostname, command, direction}` | `{status, hostname, command, output, direction, timestamp}` | `app.js:executePacketTracerTest()` (creds: per-device-type via `get_session_creds_for_device_type`) |
| `/api/run-show-route` | POST | `{hostname, target_ip, command, direction}` | `{status, hostname, target_ip, command, output, direction, timestamp}` | `app.js:executeShowRouteTest()` (creds: per-device-type via `get_session_creds_for_device_type`) |
| `/api/run-show-nat` | POST | `{hostname, target_ip, command, direction}` | `{status, hostname, target_ip, command, output, direction, timestamp}` | `app.js:executeShowNatTest()` (creds: per-device-type via `get_session_creds_for_device_type`) |

### 3.5 Show Version Endpoints

| Endpoint | Method | Params | Response | Caller |
|----------|--------|--------|----------|--------|
| `/api/show-version/<hostname>` | GET | - | `{deviceName, ip, deviceType, serialNumber, hardwareOrModel, asaVersion, ftdVersion, fxosVersion, show_version_text, show_chassis_detail_text, collected_at}` | `show_version.js:.show-sn-btn handler` |
| `/api/show-version-collect` | POST | `{hostname}` | `{deviceName, ip, deviceType, ...}` | `show_version.js:collect handlers` |
| `/api/show-version-cache-missing` | POST | `{scope, selectedDevices}` | `{missing: [{hostname, ip, type}]}` | `show_version.js:generate CSV handler` |
| `/api/generate-show-version-csv` | POST | `{scope, selectedDevices}` | CSV file download | `show_version.js:generate CSV handler` |
| `/api/show-version-inventory` | GET | - | `[{deviceName, ip, deviceType, serialNumber, ...}]` | `app.js:populateFirewallInventory()` |

### 3.6 Firewall Rules Auto Endpoints

| Endpoint | Method | Params | Response | Caller |
|----------|--------|--------|----------|--------|
| `/api/firewall-rules-auto/configure` | POST | `{basePath, rolloverTime?, prepareTime?, prepareTargetMode?, auditRetentionDays?}` | `{status, message, basePath, rolloverTime, prepareTime, prepareTargetMode, auditRetentionDays}` | `firewall_rules_auto.js:saveAsDefault()` |
| `/api/firewall-rules-auto/config` | GET | - | `{status, basePath, normalizedBasePath, rolloverTime, prepareTime, prepareTargetMode, auditRetentionDays}` | `firewall_rules_auto.js:loadFromBackendConfig()` |
| `/api/firewall-rules-auto/create-date-folders` | POST | `{basePath}` | `{status, message, createdFolders, existingFolders, errors, logContent}` | `firewall_rules_auto.js:createDateFolders()` |
| `/api/firewall-rules-auto/ensure-folders` | POST | `{basePath}` | `{status, message, folders, log, logContent}` | `firewall_rules_auto.js:ensureFolders()`, auto-called when Firewall Rules Auto tab is first shown (per sessionStorage + path) |
| `/api/firewall-rules-auto/date-folders` | GET | `?basePath` | `{status, folders: [{name, isEmpty, isTagged, isPast}]}` | `firewall_rules_auto.js:refreshDateFolders()` |
| `/api/firewall-rules-auto/prepare` | POST | `{basePath, targetMode, targetDate?}` | `{status, message, filesScanned, rulesAccepted, rulesRejected, devicesWritten, outputFolder, logContent}` | `firewall_rules_auto.js:prepareRules()` — also writes `latest_results.log` with Detailed Stats, appends to `stats.log` with tag `manual_prepare` |
| `/api/firewall-rules-auto/archive-today` | POST | `{basePath}` | `{status, message, movedFrom, movedTo, recreated: false, errors, logContent}` — `recreated` is always `false` (today's folder is not recreated; run Ensure Folders to recreate) | `firewall_rules_auto.js:archiveToday()` — appends to `stats.log` with tag `manual_archive` and snapshot of `latest_results.log` |
| `/api/firewall-rules-auto/run-scheduled` | POST | `{basePath?}` | `{status, message, log, logContent}` | Manual test trigger; also exposed as `#fra-run-scheduled-btn` button (`firewall_rules_auto.js:runScheduled()`) — performs rollover + prepare, appends to `stats.log` with tag `manual_run_scheduled` |
| `/api/firewall-rules-auto/backup-folders` | GET | `?basePath` | `{status, folders: [...]}` | List backup subfolders (`firewall_rules_auto.js:showDisplayResults()`) |
| `/api/firewall-rules-auto/backup-subfolders` | GET | `?basePath&folder` | `{status, subfolders: [...]}` | List subfolders in backup folder (`firewall_rules_auto.js:onFolderSelected()`) |
| `/api/firewall-rules-auto/backup-files` | GET | `?basePath&folder&subfolder?` | `{status, files: [...]}` | List .txt/.log files in backup folder or subfolder (`firewall_rules_auto.js:onSubfolderSelected()`) |
| `/api/firewall-rules-auto/backup-read` | GET | `?basePath&folder&subfolder?&file` | `text/plain` | Read backup log file content (`firewall_rules_auto.js:onFileSelected()`) |
| `/api/firewall-rules-auto/scheduler-status` | GET | `?basePath` | `{status, schedulerRunning: boolean, rolloverTime, prepareTime, lastRolloverDate, lastPrepareDate}` | Get scheduler runtime status for UI badges (`firewall_rules_auto.js:updateScheduleBadges()`) |
| `/api/firewall-rules-auto/base-status` | GET | `?basePath` | `{status, messages, missingFolders}` | Check core folder existence under normalized base path (`firewall_rules_auto.js:updateBaseStatusBadge()`) |
| `/api/firewall-rules-auto/deploy` | POST | `{basePath, username, password, enableSecret}` | `{status, jobId}` | `firewall_rules_auto.js:deployRules()` — starts async deploy; credentials never stored; returns jobId for polling; writes `started` line to `DeployAuthLogs.log` |
| `/api/firewall-rules-auto/deploy/status` | GET | `?jobId&tabInstanceId=deploy` | `{status, currentDevice, totalDevices, deviceResults}` | Polled by `firewall_rules_auto.js` to update Deploy Terminal |
| `/api/firewall-rules-auto/deploy/cancel` | POST | `{tabInstanceId}` | `{cancelStatus}` | `firewall_rules_auto.js:stopDeploy()` — cancels running deploy |
| `/api/firewall-rules-auto/deploy/latest-results` | GET | `?basePath` | `{content: "..."}` | `firewall_rules_auto.js:loadDeployLatestResults()` — returns contents of `latest_deploy_results.log`; empty string if missing |
| `/api/firewall-rules-auto/audit-index/stats` | GET | `?basePath` | `{status, totalEntries, byCategory, lastIndexedAt, retentionDays}` | Index statistics for the audit index view (`firewall_rules_auto.js:startAuditPoll()`) |
| `/api/firewall-rules-auto/audit-index/entries` | GET | `?basePath&page&pageSize&category&device&search&dateFrom&dateTo` | `{status, entries: [...], totalCount, page, pageSize, totalPages}` | Paginated, filtered audit log entries query |
| `/api/firewall-rules-auto/audit-index/scan` | POST | `{basePath}` | `{status, folders_scanned, new_entries, errors}` | Manually trigger a full re-scan of all backup folders |
| `/api/firewall-rules-auto/audit-index/devices` | GET | `?basePath` | `{status, devices: [{deviceName, count}]}` | Distinct device names with counts for filter dropdown |
| `/api/firewall-rules-auto/audit-index/categories` | GET | - | `{status, categories: [{category, label}]}` | Static category list for filter dropdown |
| `/api/firewall-rules-auto/report/scan-stats` | GET | `?basePath` | `{availableDates, dateMin, dateMax, availableMetrics, hasPrepareData, hasDeployData, hasAnyData, statsEntryCount, deployEntryCount}` | `firewall_rules_auto.js:loadStatsInfo()` — returns parsed file metadata so frontend can dynamically configure metric checkboxes and date limits |
| `/api/firewall-rules-auto/report/download` | POST | `{basePath, metrics: [...], chartTypes: {...}, dateFrom?, dateTo?, tagFilter?}` | Excel `.xlsx` download (with headers `X-Report-Skipped-Lines`, `X-Report-Warning`) | `firewall_rules_auto.js:generateReport()` — generates 3-sheet executive dashboard report from `stats.log` + `Deploystats.txt` with native openpyxl charts |

### 3.6.1 Stats File Formats

**`Prepare_Rules/Results/latest_results.log`** (overwritten on each prepare):
```
Base path: <base_path>
Target folder: <YYYYMMDD>
Files scanned: N
Rules accepted: N
Rules rejected: N
Devices written: N
Unsupported devices: <list> (if any)
folder is empty (if empty)

=== Detailed Stats ===
Firewall rules:
  Accepted: N (ASA: N, FTD: N, Unknown Device: N)
  Rejected: N (ASA: N, FTD: N, Unknown Device: N)
Firewall config:
  Accepted: N (ASA: N, FTD: N, Unknown Device: N)
  Rejected: N (ASA: N, FTD: N, Unknown Device: N)
Delete rules:
  Accepted: N (ASA: N, FTD: N, Unknown Device: N)
  Rejected: N (ASA: N, FTD: N, Unknown Device: N)
Delete config:
  Accepted: N (ASA: N, FTD: N, Unknown Device: N)
  Rejected: N (ASA: N, FTD: N, Unknown Device: N)
Accepted total: N
Rejected total: N
```

**`Prepare_Rules/logs/stats.log`** (append-only, day-separated):
```
# Retention: keep forever; may change later
===== YYYY-MM-DD =====
[YYYY-MM-DD HH:MM:SS] tag=manual_prepare
  accepted_total: N
  rejected_total: N
  Firewall rules:
    Accepted: N (ASA: N, FTD: N, Unknown Device: N)
    Rejected: N (ASA: N, FTD: N, Unknown Device: N)
  ... (other 3 categories)
===== YYYY-MM-DD =====
[YYYY-MM-DD HH:MM:SS] tag=manual_archive
  snapshot of latest_results.log:
  Base path: ...
  Target folder: ...
  ...
===== YYYY-MM-DD =====
[YYYY-MM-DD HH:MM:SS] tag=manual_run_scheduled
  (includes rollover + prepare stats)
===== YYYY-MM-DD =====
[YYYY-MM-DD HH:MM:SS] tag=scheduled_prepare
  ... (same as manual_prepare)
===== YYYY-MM-DD =====
[YYYY-MM-DD HH:MM:SS] tag=scheduled_rollover
  snapshot of latest_results.log:
  ...
```

**`Prepare_Rules/logs/Deploystats.txt`** (append-only, day-separated, created per deploy):
```
===== YYYY-MM-DD =====
[YYYY-MM-DD HH:MM:SS] tag=manual basePath=<base_path> success=N partial=N failed=N errors=N skipped=N
<snapshot of latest_deploy_results.log contents>
hostname (ip): status (lines succeeded: N, failed: N)
hostname (ip): status (lines succeeded: N, failed: N)
```

**`Prepare_Rules/logs/DeployAuthLogs.log`** (append-only, two lines per deploy job):
```
<username> | YYYY-MM-DD HH:MM:SS | started | job_started
<username> | YYYY-MM-DD HH:MM:SS | <status> | <reason>
```
Statuses: `started`, `success`, `failed`, `cancelled`.
Reasons: `job_started`, `completed_ok`, `no_eligible_devices`, `all_devices_auth_failed`, `no_config_lines_sent`, `cancelled_by_user`, `forced_cancel_timeout`.

- `started` is written immediately when the deploy API returns HTTP 200.
- A final line is written when the deploy job finishes, with the computed outcome.
- Pipe (`|`) and backslash (`\\`) characters within values are escaped as `\\|` and `\\\\`; newlines as `\\n`.

> **Note:** Plaintext username retention in `DeployAuthLogs.log` may change in future versions.

**Tags and triggers:**
| Tag | Trigger |
|-----|---------|
| `manual_prepare` | Manual prepare button (`/api/firewall-rules-auto/prepare`) |
| `scheduled_prepare` | Scheduler-triggered prepare |
| `manual_run_scheduled` | Manual run scheduled button (`/api/firewall-rules-auto/run-scheduled`) |
| `manual_archive` | Manual archive today button (`/api/firewall-rules-auto/archive-today`) |
| `scheduled_rollover` | Scheduler-triggered rollover |

**Classification rules** (applies to prepare-log `latest_results.log` — for audit-index classification see §4.5):
- Device bucket (case-insensitive):
  - ASA: device starts with `fw`, `asa`, or `rmb-`
  - FTD: device starts with `ftd`
  - Unknown Device: anything else
- Category (rule text after `]` prefix, space normalized):
  - Starts with `no access-list` → Delete rules
  - Starts with `no ` → Delete config
  - Contains substring `access-list` → Firewall rules
  - Else → Firewall config
- Delete rules counts only as delete rules (not also firewall rules).

### 3.7 Firewall Backup Endpoints

| Endpoint | Method | Params | Response | Caller |
|----------|--------|--------|----------|--------|
| `/api/backups/list` | GET | `?limit=30\|100\|all` | `[{name, date}, ...]` | `firewall_backups.js:loadFirewallBackups()` |
| `/api/backups/config-files` | GET | `?archive=YYYYMMDD.tar.gz` | `{archive, files: [{name, internalPath}]}` | `firewall_backups.js:handleArchiveChange()` |
| `/api/backups/config` | GET | `?archive&internalPath` | `{archive, internalPath, content}` | `firewall_backups.js:handleCompareBackups()` |
| `/api/backups/compare-configs` | POST | `{leftArchive, leftInternalPath, rightArchive, rightInternalPath}` | `{left: [{lineNum, htmlText, diffType}], right: [...]}` | Legacy inline compare |
| `/api/backups/compare-configs-job` | POST | Same as above | `{jobId, totalRows}` | Legacy job-based compare |
| `/api/backups/compare-configs-job/<job_id>/range` | GET | `?start&limit` | `{left, right, totalRows}` | Legacy |
| `/api/backups/compare-configs-job/<job_id>/next` | GET | `?type&from&dir` | `{nextIndex}` | Legacy |
| `/api/backups/compare-configs-job/<job_id>` | DELETE | - | `{success}` | Legacy |
| `/api/backups/compare-job` | POST | Same as compare-configs | `{jobId, leftLineCount, rightLineCount, diffSummary}` | `firewall_backups.js:startScalableCompareJob()` |
| `/api/backups/compare-job/<job_id>/hunks` | GET | - | `{hunks: [{leftStart, leftEnd, rightStart, rightEnd, type}]}` | `firewall_backups.js:loadHunksMetadata()` |
| `/api/backups/compare-job/<job_id>/content` | GET | `?side=left\|right&startLine&limit` | `{side, startLine, lines: [{lineNum, text}], totalLines}` | `firewall_backups.js:fetchPaneContent()` |
| `/api/backups/compare-job/<job_id>/export.csv` | GET | `?includeEqual&hunkId` | CSV file download | `firewall_backups.js:downloadCSVReport()` |
| `/api/backups/compare-job/<job_id>/download-config/<side>` | GET | side=left\|right | Raw .txt download | `firewall_backups.js:downloadPaneTxt()` |
| `/api/backups/compare-job/<job_id>` | DELETE | - | `{success}` | `firewall_backups.js:beforeunload`, `startScalableCompareJob()` |

### 3.8 Global Rules Endpoints

| Endpoint | Method | Params | Response | Caller |
|----------|--------|--------|----------|--------|
| `/api/global-rules/inventory` | GET | - | `['hostname1', 'hostname2', ...]` | `global_rules.js:loadFirewallInventory()` |

### 3.9 Logs Endpoints

| Endpoint | Method | Params | Response | Caller |
|----------|--------|--------|----------|--------|
| `/api/logs` | GET | - | `[{id, filename, timestamp, hostname, device_type, test_type, command, size}]` | `app.js:loadLogs()` |
| `/api/logs/<log_id>` | GET | - | `{timestamp, hostname, device_type, test_type, command, output}` | `app.js:viewLog()` |

### 3.10 Cancel/Stop Endpoints

| Endpoint | Method | Params | Response | Caller |
|----------|--------|--------|----------|--------|
| `/api/data-collection/cancel` | POST | `{tabInstanceId}` | `{cancelStatus: 'requested'\|'none-running'}` | `app.js:stopDataCollection()` (when `dcCurrentKind !== 'failover'`) |
| `/api/failover-check` | POST | `{hostname}`, `{all:true}`, `{tabInstanceId}` | `{status, hostname, failover_state, steps}` | `app.js:runSequential(kind='failover')`, `check_single_device_failover()` |
| `/api/failover-check/selected` | POST | `{hostnames: [...]}, {tabInstanceId}` | `[{status, hostname, ...}, ...]` | `app.js:failover-check-all-btn` (batch selected devices) |
| `/api/failover-check/cancel` | POST | `{tabInstanceId}` | `{cancelStatus: 'requested'\|'none-running'}` | `app.js:stopDataCollection()` (when `dcCurrentKind === 'failover'`) |
| `/api/firewall-inventory/cancel` | POST | `{tabInstanceId}` | `{cancelStatus: 'requested'\|'none-running'}` | `app.js:stopFirewallInventory()` |
| `/api/path-finder/cancel` | POST | `{tabInstanceId, subscope}` | `{cancelStatus: 'requested'\|'none-running'}` | `app.js:stopPathFinder(subscope)` |

**Isolation:** Each browser tab gets a unique `tabInstanceId`. On first load, the server generates it (stored in Flask session) and injects it into the HTML as `window.__SERVER_TAB_INSTANCE_ID__`. The client fallback chain: 1) `sessionStorage` (survives refresh), 2) server-injected value, 3) `crypto.getRandomValues()`, 4) `crypto.randomUUID()`, 5) `sessionStorage` counter (`tab-1`, `tab-2`, ...). The chosen ID is persisted to `sessionStorage` and used globally as `window.TAB_INSTANCE_ID`. Jobs are keyed by `(session_id, tab_instance_id, scope[, subscope])`. Cancel only affects the requesting tab's jobs.

**Frontend routing:** `stopDataCollection()` checks `dcCurrentKind` (`'collect'` vs `'failover'`) and calls the matching cancel endpoint:
- `'collect'` → `POST /api/data-collection/cancel`
- `'failover'` → `POST /api/failover-check/cancel`
- `null` → falls back to `/api/data-collection/cancel` (legacy behavior)

**Scopes (must be distinct to prevent collateral cancel):**
- `"data-collection"` — used by `collect_single_device()` and `runSequential(kind='collect')`
- `"failover-check"` — used by `api_failover_check()` (batch + single) and `check_single_device_failover()`
- `"firewall-inventory"` — used by show-version batch operations
- `"path-finder"` — used by path finder tests (with `subscope` = `"src"` or `"dst"`)

**Semantics:**
- Best-effort cancel at safe checkpoints (before connect, between devices, before cache/log writes).
- Remaining queued devices marked as `Skipped (canceled)`.
- SSH channels/jumpbox connections tracked and force-closed on cancel.
- Cache/file writes strictly skipped if job is canceled.
- `FORCED_CANCEL_TIMEOUT = 30` — second cancel >30s after first escalates to `'forced'`.

---

## 4. Data Storage Model

### 4.1 In-Memory Storage

| Variable | Type | Lifetime | Purpose |
|----------|------|----------|---------|
| `SESSION_CREDS` | Dict[sid → creds] | 30 min TTL per session | Jumpbox, ASA, FTD credentials |
| `INTERFACE_INDEX` | List[dict] | Refreshed on collection | `{hostname, iface_name, iface_ip, netmask, prefix_length, failover_state, preferred_role}` |
| `INTERFACE_INDEX_BUILT` | Bool | Auto | Tracks if index needs rebuild |
| `ROUTE_INDEX` | List[dict] | Refreshed on collection | `{hostname, network, netmask, prefix_length, route_type, next_hop, interface, failover_state, preferred_role}` |
| `ROUTE_INDEX_BUILT` | Bool | Auto | Tracks if index needs rebuild |
| `JOBS_REGISTRY` | Dict[job_key → job_state] | Server lifetime | Thread-safe job tracking for cancel/stop. Keyed by `(session_id, tab_instance_id, scope[, subscope])`. Each entry: `{status: 'running'\|'cancel_requested'\|'done', resources: [ssh_connections], created_at}` |

### 4.2 Disk Storage

| Location | Format | Lifecycle | Contents |
|----------|--------|----------|----------|
| `config/asa_inventory.txt` | Text CSV | Manual | `hostname,ip,type,is_internet_gateway` |
| `config/ftd_inventory.txt` | Text CSV | Manual | Same format |
| `config/fxos_inventory.txt` | Text CSV | Manual | Same format |
| `config/global_rules_inventory.txt` | Text | Manual | One hostname per line |
| `data/collected_devices/<hostname>.json` | JSON | Until refreshed | `{metadata, parsed_data, raw_outputs, clean_outputs}` |
| `data/device_state/<hostname>.json` | JSON | Persistent | `{failover_state, failover_checked_at, preferred_role, preferred_role_set_at}` |
| `data/session_logs/<timestamp>_<hostname>_<test_type>.json` | JSON | Until purged | `{timestamp, hostname, device_type, test_type, direction, command, output}` |
| `data/show_version_cache/<hostname>_<ip>.json` | JSON | Until refreshed | `{deviceName, ip, deviceType, serialNumber, hardwareOrModel, asaVersion, ftdVersion, fxosVersion, context, show_version_text, show_chassis_detail_text, collected_at}` |
| `data/tmp_diff/<job_id>/` | Directory | 30 min TTL | Scalable compare job files (see Section 7.2) |
| `data/live_test_logs/` | — | — | LEGACY: present on disk but not used by Flask server (`session_logs` is used instead) |
| `data/tmp_report/` | `.xlsx` | Deleted after download | Temporary Excel report files generated by `/api/firewall-rules-auto/report/download`; created with UUID filename, cleaned up via `call_on_close` after `send_file` |
| `Prepare_Rules/Results/latest_results.log` | Text | Overwritten on each prepare | Base path, target date, counts, unsupported devices, **Detailed Stats section** (4 categories × ASA/FTD/Unknown Device subtotals for accepted/rejected) |
| `Prepare_Rules/Results/latest_deploy_results.log` | Text | Overwritten on each deploy | Deploy timestamp, total/success/partial/failed/error/skipped counts, per-device results |
| `Prepare_Rules/Output/DeploySuccess.log` | Text (append-only) | Keep forever | Successful deploy device entries with timestamp, hostname, IP, lines count |
| `Prepare_Rules/Output/DeployFailed.log` | Text (append-only) | Keep forever | Failed deploy lines/device entries with timestamp, hostname, IP, command, output snippet, prompt |
| `Prepare_Rules/Output/DeployError.log` | Text (append-only) | Keep forever | Deploy errors (connection/auth/timeout) with timestamp, hostname, IP, error message |
| `Prepare_Rules/logs/stats.log` | Text (append-only) | Keep forever (may change) | Day-separated entries with tags: `manual_prepare`, `scheduled_prepare`, `manual_run_scheduled`, `manual_archive`, `scheduled_rollover` |
| `Prepare_Rules/logs/Deploystats.txt` | Text (append-only) | Keep forever | Day-separated entries with tag=manual, basePath, deploy counts, and snapshot of `latest_deploy_results.log` |
| `Prepare_Rules/logs/DeployAuthLogs.log` | Text (append-only) | Keep forever | Per-deploy-job auth log: two lines per job (`started` + final). Statuses: started/success/failed/cancelled. Reasons include completed_ok, no_eligible_devices, all_devices_auth_failed, no_config_lines_sent, cancelled_by_user, forced_cancel_timeout. Pipe/newline/backslash escaped. Plaintext username retention may change later. |
| `data/firewall_rules_audit_index.db` | SQLite | 180 day TTL (configurable) | Persistent index of historical `prepare_audit.log` entries from `Prepare_Rules/Backup/*/logs/`. Table: `audit_log_entries` with columns `id`, `backup_folder`, `file_name`, `line_number`, `raw_line`, `command`, `device_name`, `ticket_prefix`, `category`, `indexed_at`. Unique key: `(backup_folder, file_name, line_number)`. Created/updated by audit indexer during scheduler ticks and startup scan. |

### 4.3 Device Cache Structure (`collected_devices/*.json`)

```json
{
  "metadata": {
    "hostname": "FW-CORE-01",
    "ip_address": "10.1.1.1",
    "device_type": "asa",
    "collected_at": "2026-04-07 10:30:00"
  },
  "parsed_data": {
    "show_ip": [
      {"name": "inside", "ip_address": "10.1.1.1", "netmask": "255.255.255.0", "status": "up"}
    ],
    "show_route": [
      {"network": "0.0.0.0", "netmask": "0.0.0.0", "prefix_length": 0, "route_type": "S*", "next_hop": "10.1.1.254", "interface": "outside"}
    ],
    "failover_state": "Active",
    "preferred_role": "Auto"
  },
  "raw_outputs": {"show ip": "...", "show route": "..."},
  "clean_outputs": {"show ip": "...", "show route": "..."}
}
```

### 4.4 Device State Structure (`device_state/*.json`)

```json
{
  "failover_state": "Active",
  "failover_checked_at": "2026-04-07 10:35:00",
  "preferred_role": "Auto",
  "preferred_role_set_at": "2026-04-07 09:00:00"
}
```

### 4.5 Audit Index Database (`firewall_rules_audit_index.db`)

**Location:** `data/firewall_rules_audit_index.db`

**Engine:** SQLite (same database system as project's existing `tmp_diff` usage)

**Lifecycle:** Persistent; records survive restarts; retention cleanup runs automatically via scheduler (default 180 days, configurable via `auditRetentionDays` in FRA config)

**Schema:**

```sql
CREATE TABLE IF NOT EXISTS audit_log_entries (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    backup_folder TEXT NOT NULL,
    file_name TEXT NOT NULL DEFAULT 'prepare_audit.log',
    line_number INTEGER NOT NULL,
    raw_line TEXT NOT NULL,
    command TEXT NOT NULL DEFAULT '',
    device_name TEXT DEFAULT '',
    ticket_prefix TEXT DEFAULT '',
    category TEXT NOT NULL DEFAULT 'unknown',
    indexed_at TEXT DEFAULT (datetime('now')),
    UNIQUE(backup_folder, file_name, line_number)
);
```

**Indexes:** `idx_audit_category`, `idx_audit_device`, `idx_audit_indexed_at`, `idx_audit_folder`

**Deduplication key:** `(backup_folder, file_name, line_number)` prevents duplicate indexing on re-scans.

**Categories:**
| Category | Classification Rule |
|----------|-------------------|
| `firewall_rules` | Command starts with `object`, `fqdn`, or `access-list` |
| `delete_firewall_rules` | Command starts with `no object`, `no fqdn`, or `no access-list` |
| `firewall_config` | Does not match any above rule, command does not start with `no` |
| `delete_firewall_config` | Command starts with `no` but not followed by `object`/`fqdn`/`access-list` |
| `malformed` | Line cannot be split into 3 parts by `: ` delimiter |

---

## 5. Key User Flows

### 5.1 Save Credentials

```
┌──────────────────────────────────────────────────────────────────────┐
│ 1. User enters credentials in #credentials-section                    │
│    - Jumpbox: host, port, username, password                        │
│    - ASA: username, password                                        │
│    - FTD: username, password                                        │
└──────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌──────────────────────────────────────────────────────────────────────┐
│ 2. User clicks #save-jumpbox-credentials (or similar)                │
│    → app.js:saveIndividualCredentials('jumpbox')                     │
└──────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌──────────────────────────────────────────────────────────────────────┐
│ 3. JS merges with sessionStorage, sends to server                │
│    POST /api/credentials                                           │
│    Body: {jumpbox: {...}, asa: {...}, ftd: {...}}                  │
└──────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌──────────────────────────────────────────────────────────────────────┐
│ 4. server.py stores in SESSION_CREDS[sid]                           │
│    Keys: jumpbox, asa, ftd, _last_seen                              │
└──────────────────────────────────────────────────────────────────────┘
```

### 5.2 Load Inventory (Data Collection Tab)

```
┌──────────────────────────────────────────────────────────────────────┐
│ 1. User navigates to #data-collection                               │
│    → app.js:showPage('data-collection')                           │
│    → app.js:populateMockData()                                    │
└──────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌──────────────────────────────────────────────────────────────────────┐
│ 2. GET /api/devices                                               │
│    → Returns all devices from inventory files with cache status     │
└──────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌──────────────────────────────────────────────────────────────────────┐
│ 3. app.js:renderDataCollectionTable(devices)                       │
│    → Populates #inventory-table-body with checkboxes                │
│    → Shows: hostname, ip, type, gateway, collected_at, status       │
│    → Shows: failover_state, preferred_role dropdown                │
└──────────────────────────────────────────────────────────────────────┘
```

### 5.3 Collect Data (Single Device)

```
┌──────────────────────────────────────────────────────────────────────┐
│ 1. User clicks "Collect" on a single device row                     │
│    → Event delegation in app.js catches .collect-one-btn click      │
│    → POST /api/collect-single-device {hostname: "FW-CORE-01"}       │
└──────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌──────────────────────────────────────────────────────────────────────┐
│ 2. server.py:collect_single_device(hostname)                         │
│    a. Parse inventory to get device_info                           │
│    b. Get SESSION_CREDS via get_session_creds_required()           │
│    c. connect_via_jumpbox(session_creds['jumpbox'])                │
│    d. connect_to_device_via_jumpbox(jumpbox_conn, device_info)     │
│       - FXOS: always shell-hop                                     │
│       - ASA/FTD: try direct-tcpip, fallback to shell-hop           │
│    e. run_commands_on_device(channel, type)                         │
│       - Commands: terminal pager 0, show ip, show route, show arp,  │
│         show interface                                            │
│    f. Parse outputs: parse_show_ip_output(), parse_show_route_output()│
│    g. Write to data/collected_devices/<hostname>.json              │
│    h. build_interface_index() to refresh INTERFACE_INDEX          │
└──────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌──────────────────────────────────────────────────────────────────────┐
│ 3. Response: {status, hostname, wroteCache, steps, message}        │
│    → app.js creates/updates per-device status line in #collection-log-box:        │
│      id=collect-line-<safe-hostname>, e.g. "FW-CORE-01 — Running" → "Completed" │
│    → If collect succeeded, app.js renders command steps under status line:      │
│      id=collect-steps-<safe-hostname>, filters steps where stage === 'cmd',     │
│      displays step.msg (e.g., "Executing: terminal pager 0", "Complete: show ip")│
│    → Refreshes inventory table via populateMockData()             │
└──────────────────────────────────────────────────────────────────────┘
```

### 5.4 Auto-Select Firewalls

```
┌──────────────────────────────────────────────────────────────────────┐
│ 1. User enters src_ip and dst_ip in #path-finder-section           │
│    → Debounced auto-detection triggers after 300ms                  │
│    → POST /api/auto-select-firewalls {src_ip, dst_ip}            │
└──────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌──────────────────────────────────────────────────────────────────────┐
│ 2. server.py:api_auto_select_firewalls()                           │
│    For each IP:                                                    │
│    a. find_firewall_for_ip_server(target_ip)                        │
│       - Uses INTERFACE_INDEX (primary) with longest prefix match    │
│       - Falls back to ROUTE_INDEX if no interface match           │
│    b. Filters by failover_state='Active' OR preferred_role='Active'│
│    c. Returns match or candidates list if ambiguous                │
└──────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌──────────────────────────────────────────────────────────────────────┐
│ 3. Response: {src: {status, match: {...}}, dst: {...}}            │
│    → app.js:updateFirewallSelections() sets dropdown values       │
│    → app.js:updateResultCards() updates firewall/interface display│
│    → app.js:buildCommandPreviews() generates CLI commands        │
└──────────────────────────────────────────────────────────────────────┘
```

### 5.5 Run Ping Test

```
┌──────────────────────────────────────────────────────────────────────┐
│ 1. User clicks #ping-src-run-btn                                   │
│    → app.js:executePingTest('src')                                │
│    → Reads #ping-src-fw-select, #ping-src-cmd                      │
│    → POST /api/run-ping {hostname, command, direction}             │
└──────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌──────────────────────────────────────────────────────────────────────┐
│ 2. server.py:api_run_ping()                                       │
│    a. Look up device in inventory                                  │
│    b. Get credentials                                             │
│    c. Connect via jumpbox → device                                 │
│    d. execute_single_command(channel, 'terminal pager 0')            │
│    e. execute_single_command(channel, command)                      │
│    f. save_test_log() to data/session_logs/                       │
│    g. Return {status, hostname, command, output, direction}          │
└──────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌──────────────────────────────────────────────────────────────────────┐
│ 3. app.js:executePingTest() receives response                      │
│    → Appends output to #ping-terminal-output                       │
│    → Scrolls to bottom                                            │
└──────────────────────────────────────────────────────────────────────┘
```

### 5.6 Firewall Rules Auto (Prepare Rules)

```
┌──────────────────────────────────────────────────────────────────────┐
│ 1. User navigates to #firewall-rules-auto                           │
│    → app.js:showPage('firewall-rules-auto') shows #firewall-rules-auto-section │
│    → firewall_rules_auto.js:onTabShown() may POST ensure-folders once │
│      per session+path (sessionStorage guard)                         │
└──────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌──────────────────────────────────────────────────────────────────────┐
│ 2. On load: GET /api/firewall-rules-auto/config                      │
│    → Fills #fra-base-path (if empty), #fra-schedule-time             │
│    → Optional: localStorage fra_base_path if server has no path      │
│    → If hash is already #firewall-rules-auto, onTabShown() runs      │
└──────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌──────────────────────────────────────────────────────────────────────┐
│ 3. User sets base path / Save as Default / Ensure Folders / Refresh  │
│    → POST /api/firewall-rules-auto/ensure-folders { basePath }       │
│    → GET /api/firewall-rules-auto/date-folders?basePath=...          │
│    → Populates #fra-date-folder                                      │
└──────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌──────────────────────────────────────────────────────────────────────┐
│ 4. User selects date folder, clicks Prepare Rules                    │
│    → POST /api/firewall-rules-auto/prepare { basePath, dateFolder }  │
│    → Updates counts (#fra-files-scanned, etc.) and #fra-results-content │
└──────────────────────────────────────────────────────────────────────┘
```

### 5.7 Firewall Backups: List Archives

```
┌──────────────────────────────────────────────────────────────────────┐
│ 1. User navigates to #firewall-backups                              │
│    → firewall_backups.js:loadFirewallBackups()                     │
│    → GET /api/backups/list?limit=30                               │
└──────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌──────────────────────────────────────────────────────────────────────┐
│ 2. server.py:api_backups_list()                                   │
│    a. get_session_creds_required(['jumpbox'])                      │
│    b. Connect to jumpbox via SSH                                   │
│    c. exec_command: ls -1 /opt/svc_fwauto_prod/compressed_backups │
│    d. Parse filenames matching YYYYMMDD.tar.gz                    │
│    e. Return [{name, date}, ...]                                  │
└──────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌──────────────────────────────────────────────────────────────────────┐
│ 3. JS populates #backup-left-select and #backup-right-select        │
│    Default: Left = newest (index 0), Right = previous (index 1)     │
└──────────────────────────────────────────────────────────────────────┘
```

### 5.8 Firewall Backups: List Config Files

```
┌──────────────────────────────────────────────────────────────────────┐
│ 1. User selects archive from dropdown                               │
│    → firewall_backups.js:handleArchiveChange('left', '20260401')  │
│    → GET /api/backups/config-files?archive=20260401.tar.gz         │
└──────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌──────────────────────────────────────────────────────────────────────┐
│ 2. server.py:api_backups_config_files()                            │
│    a. Validate archive name (must match YYYYMMDD.tar.gz)           │
│    b. Connect to jumpbox                                          │
│    c. exec_command: tar -tzf <archive>                             │
│    d. Filter paths starting with opt/svc_fwauto_prod/fwbkp/        │
│    e. Return {archive, files: [{name, internalPath}]}              │
└──────────────────────────────────────────────────────────────────────┘
```

### 5.9 Firewall Backups: Run Scalable Compare Job

```
┌──────────────────────────────────────────────────────────────────────┐
│ 1. User selects archives + config files, clicks Compare             │
│    → firewall_backups.js:startScalableCompareJob()                 │
│    → POST /api/backups/compare-job                                │
│       Body: {leftArchive, leftInternalPath, rightArchive,           │
│              rightInternalPath}                                   │
└──────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌──────────────────────────────────────────────────────────────────────┐
│ 2. server.py:api_backups_compare_job()                             │
│    a. Generate job_id (secrets.token_urlsafe(16))                  │
│    b. Create data/tmp_diff/<job_id>/ directory                     │
│    c. Connect to jumpbox                                          │
│    d. extract_config_to_file() for both configs                    │
│       → Streams tar.gz extraction to left.txt, right.txt           │
│    e. build_line_index() for both files                           │
│    f. run_external_diff() (git diff --no-index for >50MB)         │
│    g. save_hunks_to_json() with hunk metadata                     │
│    h. save metadata.json                                          │
│    i. cleanup_old_jobs()                                          │
│    j. Return {jobId, leftLineCount, rightLineCount, diffSummary}  │
└──────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌──────────────────────────────────────────────────────────────────────┐
│ 3. JS initiates virtual scroll                                     │
│    a. GET /api/backups/compare-job/<job_id>/hunks                 │
│    b. Builds O(1) diffTypeMap per side                           │
│    c. User scrolls → fetchPaneContent(side, startLine, limit)    │
│       → GET /api/backups/compare-job/<job_id>/content             │
└──────────────────────────────────────────────────────────────────────┘
```

### 5.10 Firewall Backups: Navigate Hunks

```
┌──────────────────────────────────────────────────────────────────────┐
│ 1. User clicks #diff-nav-added badge                               │
│    → firewall_backups.js:jumpToNextDiff('added')                  │
│    → lastFocusedPane determines which pane to navigate              │
└──────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌──────────────────────────────────────────────────────────────────────┐
│ 2. findNextDiffFromHunks(side, 'added', currentIndex)              │
│    → Uses pre-built diffTypeMap[side] for O(1) lookup             │
│    → Finds next line with diffType='added'                        │
│    → Returns line number or -1 if none                           │
└──────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌──────────────────────────────────────────────────────────────────────┐
│ 3. jumpToIndex(targetIndex, side)                                  │
│    → Scrolls pane to position                                     │
│    → Adds .diff-nav-active class for 2 seconds                   │
└──────────────────────────────────────────────────────────────────────┘
```

### 5.11 Firewall Backups: Export CSV

```
┌──────────────────────────────────────────────────────────────────────┐
│ 1. User clicks #download-csv-btn                                    │
│    → firewall_backups.js:downloadCSVReport()                      │
│    → GET /api/backups/compare-job/<job_id>/export.csv            │
└──────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌──────────────────────────────────────────────────────────────────────┐
│ 2. server.py:api_backups_compare_job_export_csv()                   │
│    a. Stream response                                             │
│    b. generate_csv_rows() generator yields aligned rows            │
│    c. Each row: [row_idx, left_line_num, left_text, left_type,    │
│                          right_line_num, right_text, right_type,  │
│                          hunk_id]                                │
└──────────────────────────────────────────────────────────────────────┘
```

### 5.12 Cancel/Stop a Running Operation

```
┌──────────────────────────────────────────────────────────────────────┐
│ 1. User clicks a Stop button (e.g. #dc-stop-btn)                   │
│    → app.js:stopDataCollection() (or stopFirewallInventory /      │
│      stopPathFinder)                                               │
│    → POST /api/data-collection/cancel {tabInstanceId}              │
│    → Aborts in-flight fetch() via AbortController.abort()          │
└──────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌──────────────────────────────────────────────────────────────────────┐
│ 2. server.py:api_data_collection_cancel()                          │
│    a. Looks up job by (session_id, tab_instance_id, 'data-collection')│
│    b. If no running job → return {cancelStatus: 'none-running'}    │
│    c. Else → update status to 'cancel_requested'                   │
│    d. Force-close tracked SSH connections/channels                 │
│    e. Return {cancelStatus: 'requested'}                           │
└──────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌──────────────────────────────────────────────────────────────────────┐
│ 3. Frontend (runSequential loop)                                   │
│    a. dcCancelRequested flag set to true                           │
│    b. Loop skips remaining devices with "Skipped (canceled)"       │
│    c. In-flight fetch() throws AbortError → caught as "Canceled"   │
│    d. Stop button hidden, Run button re-enabled                    │
│    e. Terminal shows "Canceled" status line (output preserved)     │
└──────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌──────────────────────────────────────────────────────────────────────┐
│ 4. Backend checkpoint guards                                       │
│    a. Before SSH connect: check_job_canceled() → return early      │
│    b. Between commands: check_job_canceled() → skip remaining cmds │
│    c. Before cache write: check_job_canceled() → skip write        │
│    d. Before log write: check_job_canceled() → skip write          │
│    e. Response includes {canceled: true} to signal frontend        │
└──────────────────────────────────────────────────────────────────────┘
```

**Multi-tab isolation:** Each browser tab has a unique `tabInstanceId`. A cancel in Tab A does NOT affect jobs running in Tab B. The backend looks up jobs by the full key `(session_id, tab_instance_id, scope[, subscope])`.

---

## 6. Per-File Reference

### 6.1 server.py (Python/Flask Backend)

**File:** `server.py` (~9981 lines)

#### Imports from Extracted Modules

| Module | Imported Names |
|--------|---------------|
| `utils.helpers` | `clean_command_output`, `prefix_to_netmask`, `is_valid_netmask`, `escape_html_for_server` |
| `utils.file_helpers` | `parse_inventory_file`, `get_cache_status`, `get_device_state_path`, `load_device_state`, `save_device_state`, `get_device_info_from_cache` |
| `parsers.output_parsers` | `parse_show_route_output`, `parse_show_ip_output`, `parse_show_version_output`, `parse_fxos_chassis_detail`, `extract_trailing_context` |
| `services.show_version_cache` | `get_show_version_cache_path`, `load_show_version_cache`, `save_show_version_cache`, `extract_context_from_show_version_text`, `strip_device_output` |
| `ssh.client` | `connect_via_jumpbox`, `connect_to_device_via_jumpbox`, `connect_to_device_via_jumpbox_shell`, `JumpboxSessionManager`, `run_commands_on_device`, `execute_command_wait_prompt`, `execute_single_command`, `run_failover_check_on_device`, `parse_failover_state` |

#### Major Regions

| Lines | Module/Region | Purpose |
|-------|---------------|---------|
| 1-36 | Imports | Standard library + Flask + external libs |
| 36-111 | Extracted Module Imports | Re-imports from `utils.*`, `parsers.*`, `services.*`, `ssh.*` |
| 112-159 | Configuration | `get_secret_key()`, `CredentialError`, PROJECT_ROOT derived |
| 160-291 | Job Management System | `_force_close_job()`, `register_job()`, `get_job()`, `update_job_status()`, `cancel_job()`, global `JOBS_REGISTRY` dict with threading |
| 292-376 | Session/Credential Management | `SESSION_CREDS`, `get_or_create_session_id()`, `get_session_creds_required()`, `get_session_creds_for_device_type()` |
| 377-595 | Device Collection | `collect_single_device()` (single device with step tracking) |
| 596-691 | Test Logging | `save_test_log()` (writes to `SESSION_LOGS_FOLDER`) |
| 613-691 | Interface Index Building | `build_interface_index()` (global `INTERFACE_INDEX` with lock) |
| 692-773 | Route Index Building | `build_route_index()` (global `ROUTE_INDEX` with lock) |
| 774-1121 | Auto-Selection Logic | `find_firewall_for_ip_route()`, `find_firewall_for_ip_server()` |
| 1122-2040 | API Route Handlers | `/` (index), `/api/credentials`, `/api/backups/*`, `/api/data-collection/cancel`, etc. |
| 2041-2255 | Compare Job DB Helpers | `create_job_db()`, `store_diff_rows()`, `fetch_diff_range()`, `cleanup_old_jobs()` (SQLite-based legacy) |
| 2256-3205 | Scalable Diff Infrastructure | `build_line_index()`, `extract_config_to_file()`, `run_external_diff()`, `run_difflib_diff()`, `parse_unified_diff()`, compare-job routes |
| 3206-4087 | Data Collection Routes | `/api/devices`, `/api/device-interfaces`, `/api/auto-select-firewalls`, `/api/collect-single-device`, `/api/collect-data`, `/api/run-ping`, `/api/run-packet-tracer`, `/api/run-show-route`, `/api/run-show-nat` |
| 4088-4792 | Failover & Misc Routes | `/api/failover-check`, `/api/preferred-role`, `/api/logs`, `/api/global-rules/inventory`, catch-all |
| 4793-5403 | Show Version Subsystem | `run_show_version_on_device()`, routes: `/api/show-version/<hostname>`, `/api/show-version-collect`, `/api/show-version-cache-missing`, `/api/generate-show-version-csv`, `/api/show-version-inventory` |
| 5404-8830 | FRA Logic | Config load/save, audit index (SQLite), stats log parsing, Excel report generation (`_fra_build_excel_report` ~500+ lines with openpyxl), deploy subsystem (SSH via Netmiko), backup prepare/rollover |
| 8830-9105 | FRA Scheduler | `_fra_scheduler_loop()` (daemon thread, 60s tick), `_start_scheduler()`, daily rollover + prepare guards |
| 9106-9400 | FRA Route Handlers | `/api/firewall-rules-auto/run-scheduled`, `/latest-results`, `/ensure-folders`, `/date-folders`, `/archive-today`, `/prepare` |
| 9401-9675 | FRA Backup/Status Routes | `/api/firewall-rules-auto/backup-folders`, `/backup-files`, `/backup-subfolders`, `/backup-read` |
| 9676-9760 | FRA Status Routes | `/api/firewall-rules-auto/scheduler-status`, `/base-status` |
| 9761-9941 | FRA Reports + Entry Point | `/api/firewall-rules-auto/report/scan-stats`, `/report/download`, `cleanup()`, `main()` |

### 6.2 app/web/templates/index.html

**File:** `app/web/templates/index.html` (~1241 lines)

#### Major Sections and DOM IDs

| Section | ID(s) | Purpose |
|----------|--------|---------|
| **Navigation** | `#sidebar`, `.nav-link` | Hash-based routing (#path-finder, #data-collection, etc.) |
| **Credentials Section** | `#credentials-section` | Jumpbox, ASA, FTD credential forms |
| Jumpbox Form | `#jumpbox-host`, `#jumpbox-port`, `#jumpbox-user`, `#jumpbox-pass`, `#save-jumpbox-credentials` | Jumpbox credentials |
| ASA Form | `#asa-user`, `#asa-pass`, `#save-asa-credentials` | ASA credentials |
| FTD Form | `#ftd-user`, `#ftd-pass`, `#save-ftd-credentials` | FTD credentials |
| **Global Rules Section** | `#global-rules-section` | ACL rule generation |
| Firewall Selection | `#firewall-checkbox-list`, `#select-all-firewalls`, `#select-none-firewalls` | Checkbox list, select buttons |
| Normal Rules | `#normal-rules-source-list`, `#normal-rules-dest-list`, `#normal-rules-port-list`, `#normal-rules-line-items`, `#normal-rules-output`, `#generate-normal-rules`, `#dedupe-normal-rules` | Source/dest/port inputs, line items, output |
| URL Rules | `#url-rules-source-list`, `#url-rules-dest-list`, `#url-rules-port-list`, `#url-rules-line-items`, `#url-rules-output`, `#generate-url-rules`, `#dedupe-url-rules` | URL-based ACL generation |
| **Firewall Rules Auto Section** | `#firewall-rules-auto-section` | Local filesystem rule preparation (`#fra-*`) |
| Base / schedule | `#fra-base-path`, `#fra-base-path-display`, `#fra-base-status-badge`, `#fra-rollover-status-badge`, `#fra-schedule-time`, `#fra-set-rollover-btn`, `#fra-unset-rollover-btn`, `#fra-prepare-status-badge`, `#fra-prepare-time`, `#fra-set-prepare-btn`, `#fra-unset-prepare-btn`, `#fra-prepare-target-mode`, `#fra-scheduled-target-display`, `#fra-save-default-btn` | Saved path banner, base status badge, schedule status badges, prepare target mode |
| Date folders / actions | `#fra-date-folder`, `#fra-ensure-folders-btn`, `#fra-prepare-btn`, `#fra-archive-today-btn`, `#fra-create-dates-btn`, `#fra-refresh-dates-btn`, `#fra-manual-target-mode`, `#fra-custom-date-container`, `#fra-custom-date-select` | Select `YYYYMMDD`, ensure, prepare, archive today, create 6 folders, refresh list, manual target mode, custom date selection |
| Results | `#fra-status`, `#fra-files-scanned`, `#fra-rules-accepted`, `#fra-rules-rejected`, `#fra-devices-written`, `#fra-output-folder`, `#fra-results-content` | Status alert, counts, log output |
| Display results | `#fra-display-results-btn`, `#fra-display-results-ui`, `#fra-display-back-btn`, `#fra-folder-search`, `#fra-folder-select`, `#fra-subfolder-select`, `#fra-file-select`, `#fra-clear-results-btn` | Log viewer with folder search, folder picker, subfolder picker, file picker (.txt/.log), back navigation (4-step), clear terminal button |
| Report config | `#fra-report-btn`, `#fra-report-config`, `#fra-report-metrics`, `#fra-report-chart-type`, `#fra-report-date-from`, `#fra-report-date-to`, `#fra-report-tag-filter`, `#fra-report-generate-btn`, `#fra-report-status`, `#fra-report-skipped` | Report button in action row, inline config panel with metric checkboxes, chart type selector, date range, tag filter, generate button, status/skipped message areas |
| **Data Collection Section** | `#data-collection-section` | Device inventory and collection |
| Controls | `#collect-selected-btn`, `#collect-all-btn`, `#failover-check-all-btn` | Collect Selected/All: sequential per-device via `/api/collect-single-device`; Failover Selected: sequential per-device via `/api/failover-check` on checked rows only; errors if none selected |
| Table | `#inventory-table-body` | Device rows with checkboxes |
| Status | `#collection-status-panel`, `#collection-log-box`, `#collection-progress-bar`, `#collection-status-badge` | Collection progress/log |
| Search | `#dc-inventory-search` | Filter inventory |
| Select All | `#dc-select-all-checkbox` | Master checkbox |
| **Firewall Inventory Section** | `#firewall-inventory-section` | Version inventory |
| Controls | `#show-version-selected-btn`, `#generate-show-version-csv-btn` | Batch actions |
| Table | `#fw-inventory-table-body` | Version rows |
| Status | `#fw-collection-status-panel`, `#fw-collection-log-box` | Collection log |
| Search | `#fw-inventory-search` | Filter table |
| **Path Finder Section** | `#path-finder-section` | Auto-selection and testing |
| IP Inputs | `#pf-src-ip`, `#pf-dst-ip`, `#pf-proto`, `#pf-src-port`, `#pf-dst-port` | Source/dest IP entry |
| Override Dropdowns | `#pf-src-fw-override`, `#pf-dst-fw-override` | Manual firewall selection |
| Result Cards | `#source-firewall`, `#source-interface`, `#source-subnet`, `#dest-firewall`, `#dest-interface`, `#dest-subnet` | Auto-selection results |
| Warnings | `#pf-warning`, `#pf-src-route-ambiguous`, `#pf-dst-route-ambiguous` | Status banners |
| Ping | `#ping-src-fw-select`, `#ping-src-cmd`, `#ping-src-run-btn`, `#ping-terminal-output` | Ping test UI |
| Packet-tracer | `#pt-src-fw-select`, `#pt-src-if-select`, `#pt-src-cmd`, `#pt-src-run-btn`, `#packet-terminal-output` | PT test UI |
| Show Route | `#showroute-src-fw-select`, `#showroute-src-ip-override`, `#showroute-src-cmd`, `#showroute-terminal-output` | Route test UI |
| Show Nat | `#shownat-src-fw-select`, `#shownat-src-ip-override`, `#shownat-src-cmd`, `#shownat-terminal-output` | NAT test UI |
| **Live Tests Section** | `#live-tests-section` | Alternative test interface |
| **Logs Section** | `#logs-section` | Session log viewer |
| **Firewall Backups Section** | `#firewall-backups-section` | Config comparison |
| Backup Selection | `#backup-left-select`, `#backup-right-select` | Archive dropdowns |
| Archive Search | `#backup-left-search-input`, `#backup-right-search-input` | Archive filter |
| Config Selection | `#backup-left-config-select`, `#backup-right-config-select` | Config file dropdowns |
| Config Search | `#backup-left-config-search-input`, `#backup-right-config-search-input` | Config filter |
| Controls | `#swap-backups-btn`, `#compare-backups-btn` | Swap and compare |
| Results | `#comparison-results-card`, `#left-backup-label`, `#right-backup-label` | Comparison header |
| Badges | `#diff-nav-added`, `#diff-nav-removed`, `#diff-nav-modified` | Diff navigation |
| CSV Export | `#download-csv-btn` | CSV download button |
| Panes | `#left-config-content`, `#right-config-content` | Config display (scrollable divs) |
| Pane Controls | `#left-config-search-input`, `#right-config-search-input`, `#lock-scroll-checkbox` | Search, sync scroll |
| Download | `#left-download-txt-btn`, `#right-download-txt-btn` | Raw config download |

### 6.3 app/web/static/js/app.js

**File:** `app/web/static/js/app.js` (~2584 lines)

#### Top-Level Responsibilities

1. **Navigation** - Hash-based routing, page show/hide
2. **Credentials** - Save individual credentials, sessionStorage merge, auto-rehydrate, idle expiry
3. **Device Collection** - Inventory loading, collection triggers, progress logging
4. **Auto-Selection** - Debounced firewall detection, result card updates
5. **Live Tests** - Ping, packet-tracer, show route, show nat execution
6. **Failover Check** - Per-device and batch failover checks
7. **Preferred Role** - Dropdown changes to set role

#### Key DOM IDs Read/Written

| ID | Operations | Purpose |
|----|------------|---------|
| `#pf-src-ip`, `#pf-dst-ip` | read, event listener | Path finder source/dest IPs |
| `#source-firewall`, `#dest-firewall` | write | Auto-selection result |
| `#collection-log-box` | append + in-place update | Collection status messages; per-device status lines (id=`collect-line-<safe>`) and command steps (id=`collect-steps-<safe>`) updated in-place during runs |
| `#inventory-table-body` | write (innerHTML) | Render device rows |
| `#dc-select-all-checkbox` | read, indeterminate | Master select checkbox |
| `#ping-terminal-output` | append | Ping command output |
| `.preferred-role-select` | event listener, fetch | Role change handler |

#### Key Functions

| Function | Lines | Purpose |
|----------|-------|---------|
| `showPage()` | 73-105 | Hash routing; calls `populateFirewallInventory` on firewall-inventory; `firewallRulesAutoOnTabShown` on firewall-rules-auto |
| `populateMockData()` | 111-186 | Load devices, populate dropdowns |
| `renderDataCollectionTable()` | 257-309 | Render inventory table rows |
| `initializePathFinder()` | 513-614 | Set up path finder auto-detection |
| `autoDetectFirewallsAndInterfaces()` | 632-776 | Call auto-select API, update UI |
| `updateFirewallSelections()` | 966-1035 | Set dropdown values from auto-select |
| `buildCommandPreviews()` | 1131-1188 | Generate ping/packet-tracer commands |
| `executePingTest()` | 2324+ | POST to run-ping with AbortController; toggles Run/Stop buttons |
| `executePacketTracerTest()` | ~2536 | POST to run-packet-tracer with AbortController; toggles Run/Stop buttons |
| `executeShowRouteTest()` | ~2365 | POST to run-show-route with AbortController; toggles Run/Stop buttons |
| `executeShowNatTest()` | ~2445 | POST to run-show-nat with AbortController; toggles Run/Stop buttons |
| `stopDataCollection()` | 21 | POST cancel endpoint, abort fetch, show warning badge |
| `stopFirewallInventory()` | 46 | POST cancel endpoint, abort fetch, show warning badge |
| `stopPathFinder(subscope)` | 70 | POST cancel endpoint with subscope, abort AbortController |
| `runSequential()` | 1830+ | Sequential per-hostname runner for collect/failover; AbortController + cancel loop skip |
| `setCollectRunning()` | 1773+ | Status badge + Stop button visibility + disable controls |
| `setFwCollectRunning()` | 580+ | Status badge + Stop button visibility + disable controls |
| `saveIndividualCredentials()` | ~1473 | Save creds with sessionStorage merge |
| `sanitizeId()` | 1503-1505 | Escape hostname to safe DOM id (`collect-line-<safe>`) |
| `setDeviceStatus()` | 1514-1553 | Update/create one per-device status line in `#collection-log-box` |
| `renderCollectSteps()` | 1555-1580 | Render command steps under status line (filters stage === 'cmd', displays step.msg) |

#### Cancel/Stop State

| Variable | Type | Purpose |
|----------|------|---------|
| `TAB_INSTANCE_ID` | string | Unique per-tab ID from `sessionStorage` (survives refresh, clears on close) |
| `dcCancelRequested` | bool | Flag for data collection cancel loop skip |
| `fwInventoryCancelRequested` | bool | Flag for firewall inventory cancel loop skip |
| `pfCancelControllers` | Dict[key → AbortController] | Path Finder AbortController per direction (e.g. `pf-src`, `pf-dst`) |
| `window._dcCurrentAbortController` | AbortController | In-flight Data Collection fetch controller |
| `window._fwCurrentAbortController` | AbortController | In-flight Firewall Inventory fetch controller |

### 6.4 app/web/static/js/firewall_backups.js

**File:** `app/web/static/js/firewall_backups.js` (~2700+ lines)

#### Top-Level Responsibilities

1. **Archive Listing** - Load from API, populate dropdowns, filter
2. **Config File Browsing** - Per-archive file listing with search
3. **Compare Job Management** - Create, poll, delete compare jobs
4. **Virtual Scrolling** - Render only visible lines for large configs
5. **Diff Navigation** - Jump to next/prev added/removed/modified
6. **Search/Highlight** - Find in config with match highlighting
7. **Export** - CSV download, raw .txt download
8. **Pane Synchronization** - Lock scroll, focused pane tracking

#### Key DOM IDs Read/Written

| ID | Operations | Purpose |
|----|------------|---------|
| `#backup-left-select`, `#backup-right-select` | read, populate | Archive dropdowns |
| `#backup-left-config-select`, `#backup-right-config-select` | read, populate | Config file dropdowns |
| `#comparison-results-card` | classList (show/hide) | Results visibility |
| `#left-config-content`, `#right-config-content` | innerHTML, scrollTop | Config line rendering |
| `#diff-nav-added`, `#diff-nav-removed`, `#diff-nav-modified` | click listener | Navigation badges |
| `#download-csv-btn` | click listener | Trigger CSV export |

#### Key Global State

```javascript
let firewallBackupsList = [];          // Archive list cache
let currentBackupLimit = '30';         // Current limit setting
let currentJobId = null;              // Active compare job ID
let currentTotalRows = 0;              // Total diff rows
let currentHunks = [];                 // Hunk metadata from job
let diffTypeMap = { left: {}, right: {} };  // O(1) diff type lookup
let virtualScroller = null;            // Virtual scroll instances
let lastFocusedPane = 'left';          // Which pane is focused
const CHUNK_SIZE = 3000;              // Rows per fetch
const INITIAL_RENDER_SIZE = 3000;     // Initial render count
```

### 6.5 app/web/static/js/global_rules.js

**File:** `app/web/static/js/global_rules.js` (~733 lines)

#### Top-Level Responsibilities

1. **Firewall Selection** - Load inventory, checkboxes
2. **Normal Rules Generation** - Source/dest/port → ACL rules
3. **URL Rules Generation** - FQDN-based ACL with object networks
4. **Deduplication** - Remove duplicate ACL lines

#### Key DOM IDs

| ID | Operations | Purpose |
|----|------------|---------|
| `#firewall-checkbox-list` | innerHTML | Firewall checkboxes |
| `#normal-rules-source-list`, `#normal-rules-dest-list`, `#normal-rules-port-list` | read, event | ACL source/dest/port inputs |
| `#normal-rules-line-items` | innerHTML | Per-line editor rows |
| `#normal-rules-output` | value (write) | Generated ACL output |
| `#url-rules-source-list`, `#url-rules-dest-list`, `#url-rules-port-list` | read, event | URL rule inputs |
| `#url-rules-output` | value (write) | Generated URL rules |

#### Key Functions

| Function | Purpose |
|----------|---------|
| `loadFirewallInventory()` | Fetch /api/global-rules/inventory |
| `syncNormalRuleLineItems()` | Parse textareas → editable rows |
| `syncUrlRuleLineItems()` | Parse URL inputs → editable rows |
| `generateNormalRules()` | Validate → generate ACL lines |
| `generateUrlRules()` | Validate → generate FQDN ACL + object networks |
| `deduplicateOutput()` | Remove duplicate lines |

### 6.6 app/web/static/js/show_version.js

**File:** `app/web/static/js/show_version.js` (~326 lines)

#### Top-Level Responsibilities

1. **Show Version Collection** - Per-device and batch collection
2. **Cache Handling** - Check cache first, collect if missing
3. **CSV Export** - Scope selection (A/B/C), download
4. **Cancel Support** - AbortController on all fetch calls; batch loop checks `fwInventoryCancelRequested`

#### Key DOM IDs

| ID | Operations | Purpose |
|----|------------|---------|
| `.show-sn-btn` | click (delegation) | Show serial number |
| `.refresh-version-btn` | click (delegation) | Refresh version data |
| `#show-version-selected-btn` | click | Batch collect selected |
| `#generate-show-version-csv-btn` | click | Generate CSV export |

#### Key Functions

| Function | Purpose |
|----------|---------|
| `formatShowVersionDetails()` | Format cached data for display |
| `_collectShowVersionForDevice(hostname, ip, controller)` | Shared fetch helper with AbortController signal |
| `.show-sn-btn` handler | Uses AbortController; stores in `window._fwCurrentAbortController` |
| `#show-version-selected-btn` handler | Batch loop with `fwInventoryCancelRequested` check + per-device AbortController |
| `.refresh-version-btn` handler | Uses AbortController; stores in `window._fwCurrentAbortController` |

### 6.7 app/web/static/js/firewall_rules_auto.js

**File:** `app/web/static/js/firewall_rules_auto.js` (~1528 lines)

#### Top-Level Responsibilities
- Manage base path input with localStorage persistence (read on load if server has no path; written on change/save)
- Fetch date folder list from backend
- Call ensure-folders and prepare-rules endpoints; optional one-time auto-ensure when the Firewall Rules Auto tab is shown (sessionStorage per normalized path)
- Render results summary and log content

#### DOM IDs Read/Written

| ID | Access | Purpose |
|----|--------|---------|
| `#fra-base-path` | read/write, localStorage | Base path input |
| `#fra-base-path-display` | write | Read-only banner showing current base directory |
| `#fra-schedule-time` | read/write | Rollover time (`HH:MM`) |
| `#fra-set-rollover-btn` | click | Set rollover schedule |
| `#fra-unset-rollover-btn` | click | Unset rollover schedule |
| `#fra-prepare-time` | read/write | Prepare time (`HH:MM`) |
| `#fra-set-prepare-btn` | click | Set prepare schedule |
| `#fra-unset-prepare-btn` | click | Unset prepare schedule |
| `#fra-prepare-target-mode` | read/write | Scheduler prepare target (today/tomorrow), persisted as `prepareTargetMode` in config |
| `#fra-scheduled-target-display` | read | Shows "Preparing folder: YYYYMMDD" based on prepareTargetMode |
| `#fra-manual-target-mode` | read/write | Manual prepare target (today/tomorrow/custom), independent from scheduler |
| `#fra-custom-date-container` | visibility | Hidden by default; shown when manual target is CUSTOM |
| `#fra-custom-date-select` | read | Custom date dropdown populated from `/api/firewall-rules-auto/date-folders` |
| `#fra-save-default-btn` | click | Save path via `/configure` |
| `#fra-base-status-badge` | write (class) | Base folder status (green `core files detected`, red `MISSING: ...`, red `ERROR`) |
| `#fra-rollover-status-badge` | write (class) | Rollover schedule status badge (ACTIVE green / NOT ACTIVE gray / ERROR red) |
| `#fra-prepare-status-badge` | write (class) | Prepare schedule status badge (ACTIVE green / NOT ACTIVE gray / ERROR red) |
| `#fra-date-folder` | write (select options) | Date folder dropdown |
| `#fra-ensure-folders-btn` | click | Ensure folders handler |
| `#fra-prepare-btn` | click | Prepare rules handler |
| `#fra-create-dates-btn` | click | Create 6 date folders (today..today+5) |
| `#fra-refresh-dates-btn` | click | Refresh date folders |
| `#fra-archive-today-btn` | click | Archive today's folder |
| `#fra-run-scheduled-btn` | click | Run scheduled task manually |
| `#fra-display-results-btn` | click | Show log viewer UI |
| `#fra-display-back-btn` | click | Step back in log viewer |
| `#fra-folder-search` | input | Search/filter backup folders |
| `#fra-folder-select` | change | Select backup folder |
| `#fra-subfolder-select` | change | Select subfolder in backup folder |
| `#fra-file-select` | change | Select log file to display |
| `#fra-clear-results-btn` | click | Clear terminal / results content |
| `#fra-status` | write | Status message |
| `#fra-files-scanned` | write | Files scanned count |
| `#fra-rules-accepted` | write | Rules accepted count |
| `#fra-rules-rejected` | write | Rules rejected count |
| `#fra-devices-written` | write | Devices written count |
| `#fra-output-folder` | write | Output folder path |
| `#fra-results-content` | write | Results/log display |
| `#fra-report-btn` | click | Toggle report configuration panel |
| `#fra-report-config` | visibility | Inline expandable report config panel (d-none) |
| `#fra-report-metrics` | read (checkboxes) | Metric selection checkboxes (7 metrics: accepted, rejected, success, partial, failed, errors, skipped) |
| `#fra-report-chart-type` | read | Chart type selector (Bar/Pie/Doughnut/Line, auto-filtered by metric compat) |
| `#fra-report-date-from` | read | Start date input (date type) |
| `#fra-report-date-to` | read | End date input (date type) |
| `#fra-report-tag-filter` | read | stats.log tag filter dropdown (all/manual_prepare/scheduled_prepare/...) |
| `#fra-report-generate-btn` | click | Generate and download report |
| `#fra-report-status` | write | Report status alert |
| `#fra-report-skipped` | write | Malformed lines skipped count display |
| `#fra-report-stats-status` | write | Stats summary text from scan-stats backend (available dates, metric presence) |

#### Key Functions

| Function | Purpose |
|----------|---------|
| `init()` | Set up event listeners for all buttons |
| `loadFromBackendConfig()` | Fetch persisted base path + rolloverTime + prepareTime from backend on load; then `applyLocalStoragePathFallback()`; if hash is `#firewall-rules-auto`, runs `onTabShown()` |
| `applyLocalStoragePathFallback()` | If `#fra-base-path` is still empty, fill from `localStorage` (`fra_base_path`) |
| `onTabShown()` | Exported as `window.firewallRulesAutoOnTabShown` — one-time `autoEnsureOnce` for `fraLastNormalizedBasePath` or `getBasePath()` (also invoked from `app.js:showPage` when navigating to this tab) |
| `autoEnsureOnce()` | POST ensure-folders once per session+path (`sessionStorage` guard) |
| `saveAsDefault()` | POST to /api/firewall-rules-auto/configure (saves path) |
| `setRolloverTime()` | POST rolloverTime via `/configure` |
| `unsetRolloverTime()` | POST rolloverTime: null via `/configure` |
| `setPrepareTime()` | POST prepareTime + prepareTargetMode via `/configure` |
| `unsetPrepareTime()` | POST prepareTime: null + prepareTargetMode via `/configure` |
| `updateScheduledTargetDisplay()` | Updates "Preparing folder: YYYYMMDD" based on scheduler target mode |
| `loadDateFoldersForCustomSelect()` | Populates custom date dropdown from `/date-folders` endpoint |
| `updateBasePathBanner()` | Update base directory banner display |
| `ensureFolders()` | POST to /api/firewall-rules-auto/ensure-folders |
| `refreshDateFolders()` | GET /api/firewall-rules-auto/date-folders |
| `prepareRules()` | POST `{basePath, targetMode, targetDate?}` to `/api/firewall-rules-auto/prepare`; targetMode defaults to tomorrow |
| `archiveToday()` | POST to /api/firewall-rules-auto/archive-today |
| `createDateFolders()` | POST to /api/firewall-rules-auto/create-date-folders |
| `runScheduled()` | POST to /api/firewall-rules-auto/run-scheduled (shows status + logContent in results area) |
| `updateResults()` | Update UI with summary counts |
| `showDisplayResults()` | Show log viewer UI with folder search/picker/file picker |
| `handleBack()` | Multi-step back navigation (file → folder → idle) |
| `hideDisplayResults()` | Hide log viewer UI and reset state |
| `updateScheduleBadges()` | Refresh schedule status badges (ACTIVE when time is set, NOT ACTIVE when not set, ERROR on failure) |
| `updateBaseStatusBadge()` | Check core folder existence (no .txt requirement); shows green when all core files detected, red with MISSING messages otherwise |
| `switchAuditView(showIndex)` | Toggle between Browse Files mode and Audit Index mode in the Log Viewer |
| `loadCategoryDropdown()` | Populate category filter dropdown from `/api/firewall-rules-auto/audit-index/categories` |
| `loadDeviceDropdown()` | Populate device filter dropdown from `/api/firewall-rules-auto/audit-index/devices` |
| `fetchAuditEntries()` | Fetch paginated, filtered entries from `/api/firewall-rules-auto/audit-index/entries` and render results table |
| `startAuditPoll()` | Start 60s interval polling `/api/firewall-rules-auto/audit-index/stats` for auto-refresh |
| `stopAuditPoll()` | Clear the auto-refresh poll timer |
| `triggerAuditScan()` | POST to `/api/firewall-rules-auto/audit-index/scan` to manually re-index all backup folders |
| `catLabel(cat)` | Map internal category string to display label |
| `toggleReportConfig()` | Toggle the inline report configuration panel visibility |
| `updateCompatibleChartTypes()` | Auto-filter chart type dropdown based on selected metrics (intersection of compatible types) |
| `generateReport()` | Gather metric/chart/date/tag selections, POST to `/api/firewall-rules-auto/report/download`, handle blob download, display skipped-line count from `X-Report-Skipped-Lines` header and backend warnings from `X-Report-Warning` header |
| `showReportStatus(msg, isError)` | Show report status alert in `#fra-report-status` |
| `showReportSkipped(count)` | Show malformed line count in `#fra-report-skipped` |
| `loadStatsInfo()` | GET `/api/firewall-rules-auto/report/scan-stats`, update date min/max, enable/disable metric checkboxes based on available data, show stats summary in `#fra-report-stats-status` |

**UI Elements:** `#fra-base-path` (input), `#fra-base-status-badge`, `#fra-rollover-status-badge`, `#fra-schedule-time` (time input), `#fra-prepare-status-badge`, `#fra-prepare-time` (time input), `#fra-date-folder` (select), `#fra-status` (alert).

**Audit Index View DOM IDs:** `#fra-view-toggle` (toggle button group), `#fra-view-browse-btn` / `#fra-view-index-btn` (mode buttons), `#fra-view-browse-body` / `#fra-view-index-body` (mode containers), `#fra-index-category` (category filter), `#fra-index-device` (device filter), `#fra-index-date-from` / `#fra-index-date-to` (date range), `#fra-index-search` (free-text search), `#fra-index-filter-btn` (apply filters), `#fra-index-count` (total count), `#fra-index-refresh-btn` (refresh), `#fra-index-scan-btn` (re-scan), `#fra-index-table` / `#fra-index-tbody` (results table), `#fra-index-page-info` (page indicator), `#fra-index-prev-btn` / `#fra-index-next-btn` (pagination).

**Report Config DOM IDs:** `#fra-report-btn` (action row toggle), `#fra-report-config` (inline panel), `#fra-report-metrics` (7 metric checkboxes), `#fra-report-chart-type` (auto-filtered chart type), `#fra-report-date-from` / `#fra-report-date-to` (date range), `#fra-report-tag-filter` (stats.log tag), `#fra-report-generate-btn`, `#fra-report-status` (alert), `#fra-report-skipped` (malformed line count), `#fra-report-stats-status` (stats summary).

**Base Path Status Badge:** `#fra-base-status-badge` shows core folder status. Checks folder existence only (no file content required). Runs on tab load. States: `all core files detected` (green), `MISSING: Prepare_Rules\Output, Prepare_Rules\Backup, ChangePrepBackup` (red) listing only the missing folders, `ERROR: <message>` (red) on access/permission issues.

**Schedule Status Badges:** `#fra-rollover-status-badge` and `#fra-prepare-status-badge` show state based on time being set (not scheduler runtime). States: `ACTIVE (Last: YYYYMMDD)` (green), `NOT ACTIVE` (gray), `ERROR` (red).
**Prepare Badge "Last":** The "Last:" value shows the folder date that was actually processed (e.g., D+1 when prepareTargetMode=tomorrow), not the run date. Stored in `Prepare_Rules/last_scheduled_prepare_date.txt`. Both scheduled and manual prepare jobs update this file after successful completion.

**Log Viewer Navigation:** 4-step browse flow (`idle → folder → subfolder → file → fileView`) with back navigation at each level. A parallel Audit Index mode (`indexView`) toggled via `switchAuditView()` replaces browse UI with a paginated, filterable entries table.

**Scheduled Rollover:** Uses `shutil.move` to move entire folder (not copy contents). After rollover on day D, ensures 6 active folders: D+1..D+6 (tomorrow through tomorrow+5).

**Date Folder Labels:** `refreshDateFolders()` shows `(past)`, `(empty)`, `(tagged)` based on folder metadata. Past folders are styled as muted.

### 6.8 app/web/static/css/styles.css

**File:** `app/web/static/css/styles.css` (~333 lines)

#### Key Classes and IDs

| Selector | Purpose |
|----------|--------|
| `.config-line` | Individual config lines in backup viewer |
| `.config-terminal` | Scrollable terminal container (black bg) |
| `.config-terminal-selected` | Glow effect on focused pane |
| `.diff-added` | Green background for added lines |
| `.diff-removed` | Red background for removed lines |
| `.diff-modified` | Yellow background for modified lines |
| `.diff-nav-active` | Orange outline for navigation target |
| `.config-search-match` | Yellow highlight for search matches |
| `.config-search-active` | Orange highlight for current search result |
| `.status-active` | Green status indicator |
| `.status-inactive` | Red status indicator |
| `.status-warning` | Orange/yellow status indicator |

### 6.9 utils/helpers.py

**File:** `utils/helpers.py` (~68 lines)

#### Functions

| Function | Signature | Purpose |
|----------|-----------|---------|
| `clean_command_output` | `(output: str) -> str` | Strips ANSI escape codes and trailing whitespace from device output |
| `prefix_to_netmask` | `(prefix: str) -> str` | Converts CIDR prefix length (e.g. "24") to dotted-quad netmask (e.g. "255.255.255.0") |
| `is_valid_netmask` | `(mask: str) -> bool` | Validates that a string is a proper dotted-quad netmask |
| `escape_html_for_server` | `(text: str) -> str` | Escapes HTML special characters in server-originated text |

**Dependencies:** `os`, `re`, `ipaddress`
**Imported by:** `server.py`, `ssh/client.py`, `parsers/output_parsers.py`

---

### 6.10 utils/file_helpers.py

**File:** `utils/file_helpers.py` (~202 lines)

#### Functions

| Function | Purpose |
|----------|---------|
| `parse_inventory_file(filepath)` | Reads `*_inventory.txt`, returns list of dicts: `{hostname, ip, type, is_gateway}` |
| `get_cache_status(hostname)` | Checks `data/collected_devices/<hostname>.json` exists and is non-empty |
| `get_device_state_path(hostname)` | Returns `Path` to `data/device_state/<hostname>.json` |
| `load_device_state(hostname)` | Reads device state JSON, returns dict with failover/preferred_role |
| `save_device_state(hostname, data)` | Writes device state dict to JSON file |
| `get_device_info_from_cache(hostname)` | Reads full cached device data from `data/collected_devices/` |

**Dependencies:** `json`, `Path` from `pathlib`
**Path convention:** Uses `Path(__file__).resolve().parent.parent / "data" / ...` (module-local computation)
**Imported by:** `server.py`

---

### 6.11 parsers/output_parsers.py

**File:** `parsers/output_parsers.py` (~385 lines)

#### Functions

| Function | Purpose |
|----------|---------|
| `parse_show_route_output(output)` | Parses Cisco `show route` output into structured route entries (network, nexthop, interface, type, preference) |
| `parse_show_ip_output(output)` | Parses Cisco `show ip` interface output into structured interface data |
| `parse_show_version_output(output)` | Parses `show version` output for model, serial, software version, uptime |
| `parse_fxos_chassis_detail(output)` | Parses FXOS `show chassis detail` output for chassis serial, model, status |
| `extract_trailing_context(output)` | Extracts trailing context lines from device output |

**Dependencies:** `re`, `prefix_to_netmask`, `is_valid_netmask` (from `utils.helpers`)
**Imported by:** `server.py`

---

### 6.12 services/show_version_cache.py

**File:** `services/show_version_cache.py` (~105 lines)

#### Functions

| Function | Purpose |
|----------|---------|
| `get_show_version_cache_path(hostname)` | Returns `Path` to `data/show_version_cache/<hostname>.json` |
| `load_show_version_cache(hostname)` | Loads cached show version data from JSON file |
| `save_show_version_cache(hostname, data)` | Saves show version data to JSON cache file |
| `extract_context_from_show_version_text(text)` | Extracts contextual metadata (model, serial, version) from raw show version text |
| `strip_device_output(output)` | Strips device prompt and control characters from raw output |

**Dependencies:** `json`, `Path` from `pathlib`
**Path convention:** `Path(__file__).resolve().parent.parent / "data" / ...`
**Imported by:** `server.py`

---

### 6.13 ssh/client.py

**File:** `ssh/client.py` (~831 lines)

#### Key Classes

| Class | Purpose |
|-------|---------|
| `JumpboxSessionManager` | Connection pool managing SSH jumpbox sessions with reconnection logic; stores `paramiko.SSHClient` instances keyed by session ID |

#### Key Functions

| Function | Purpose |
|----------|---------|
| `connect_via_jumpbox(session_id, target_host, target_port)` | Opens a direct-tcpip channel through the jumpbox to the target device; returns connected `paramiko.Channel` |
| `connect_to_device_via_jumpbox(session_id, device_creds)` | Full SSH connection through jumpbox using direct-tcpip channel forwarding + Paramiko transport |
| `connect_to_device_via_jumpbox_shell(session_id, device_type, device_creds)` | Shell-based jumpbox hop with device-specific login sequences (ASA `enable`, FTD `login as:`, FXOS `connect fxos`) |
| `run_commands_on_device(channel, commands, device_type)` | Sends multiple commands through an open channel, collects output for each |
| `execute_command_wait_prompt(channel, command, device_type, timeout)` | Sends command and waits for device-specific prompt pattern |
| `execute_single_command(session_id, device_creds, command, device_type)` | High-level one-shot: connect + execute + return output |
| `run_failover_check_on_device(channel, device_type)` | Runs failover show commands and returns structured failover state |
| `parse_failover_state(device_type, output)` | Parses `show failover` (ASA) or `show firewall` (FTD/FXOS) output for active/standby state |

**Dependencies:** `paramiko`, `time`, `re`, `Path`, `clean_command_output` (from `utils.helpers`)
**Imported by:** `server.py`

---

## 7. Naming Conventions & Invariants

### 7.1 UI Semantic Mapping (Backups Compare)

| UI Label | Semantic Meaning | Git Diff Equivalent |
|----------|----------------|-------------------|
| **Left Pane** | "New (After)" | NEW file |
| **Right Pane** | "Old (Before)" | OLD file |
| **Green (Added)** | Line exists in NEW only | `+` prefix |
| **Red (Removed)** | Line exists in OLD only | `-` prefix |
| **Yellow (Modified)** | Line changed between versions | Modified hunk |
| **Left-added** | Line added in NEW (not in OLD) | Lines that appeared |
| **Right-removed** | Line removed from OLD (not in NEW) | Lines that disappeared |

**Invariant:** In all comparison code:
- `left` side = NEW = AFTER
- `right` side = OLD = BEFORE
- Archive selected in left dropdown → newer backup
- Archive selected in right dropdown → older backup

### 7.2 Job Folder Layout (`data/tmp_diff/<job_id>`)

```
data/tmp_diff/<job_id>/                    # Scalable compare-job layout
├── left.txt             # Extracted NEW config (left pane / After)
├── right.txt            # Extracted OLD config (right pane / Before)
├── left_index.json      # Byte offset index for left.txt
│   └── [{offset, length}, ...]
├── right_index.json     # Byte offset index for right.txt
│   └── [{offset, length}, ...]
├── hunks.json           # Diff hunk metadata
│   └── [{leftStart, leftEnd, rightStart, rightEnd, type}, ...]
└── metadata.json        # Job metadata
    └── {jobId, leftArchive, rightArchive, leftLineCount,
         rightLineCount, createdAt, diffSummary}

# LEGACY: The following exists only for compare-configs-job (SQLite-based),
# NOT for scalable compare-job. May not be present for new jobs.
├── diff.db              # SQLite DB (compare-configs-job legacy only)
│   ├── diff_rows        # Table: row_index, left_*, right_*
│   └── metadata         # Table: key, value
```

### 7.3 SESSION_CREDS Structure

```python
SESSION_CREDS = {
    sid: {
        "jumpbox": {
            "host": str,
            "port": int,       # default 22
            "username": str,
            "password": str
        },
        "asa": {
            "username": str,
            "password": str
        },
        "ftd": {
            "username": str,
            "password": str
        },
        "_last_seen": float  # Unix timestamp
    }
}
```

**Invariant:** `_last_seen` is updated on every access; sessions older than `SESSION_TTL_SECONDS` (30 min) are purged.

### Per-Device Credential Validation

Endpoints that operate on a specific device use `get_session_creds_for_device_type(device_type)` to validate credentials:

| Device Type | Required Credentials |
|-------------|----------------------|
| `asa` | Jumpbox + ASA |
| `ftd` | Jumpbox + FTD |
| `fxos` | Jumpbox only (no device-specific creds) |

**Implication:** Saving Jumpbox+ASA is sufficient for ASA operations; saving Jumpbox+FTD is sufficient for FTD operations. The backend does NOT require ASA+FTD simultaneously.

This applies to:
- `POST /api/collect-single-device`
- `POST /api/failover-check` (single device path)
- `POST /api/run-ping`
- `POST /api/run-packet-tracer`
- `POST /api/run-show-route`
- `POST /api/run-show-nat`

Batch operations (`POST /api/collect-data`, batch failover) check Jumpbox first, then per-device type credentials — missing device-type creds result in per-device failures, not global failure.

### 7.4 Session Persistence & Credential Stability

Flask sessions are keyed by cookie (`session["sid"]`). The session cookie validity depends on a stable `app.secret_key`:

- **server.py** loads secret key via `get_secret_key()`:  
  1. If `FLASK_SECRET_KEY` env var set → use it  
  2. Else if `data/flask_secret_key.txt` exists → read from file  
  3. Else → generate random 32-byte key, save to file, use it

- **Session cookie settings:** `SESSION_COOKIE_SAMESITE = "Lax"`, `SESSION_COOKIE_HTTPONLY = True`

- **Implication:** If secret key changes (e.g., server restart without env var or key file), existing session cookies become invalid → new `sid` generated → saved credentials in `SESSION_CREDS` no longer accessible.

- **Client-side recovery:** On page load, `app.js` checks `sessionStorage.getItem('firewallCredentials')`. If jumpbox creds exist, it silently POSTs to `/api/credentials` to rehydrate the server-side session. This handles page refreshes without re-entering credentials.
- **Storage:** Credentials are stored in browser `sessionStorage` (not localStorage), so they automatically clear when the browser closes.
- **Idle expiry:** After 30 minutes of no user interaction (mouse/keyboard), credentials auto-clear via `startIdleTimer()`.
- **Clear button:** Each credential card has a Clear button that clears UI fields, sessionStorage, and server session via `/api/credentials/clear`.

**Troubleshooting: "Credentials not configured"**

1. **Inconsistent hostname** — Using `localhost` vs `127.0.0.1` creates different cookies. Use one consistently.
2. **Secret key changed** — Ensure `FLASK_SECRET_KEY` env is set, or `data/flask_secret_key.txt` persists across restarts.
3. **Server restarted without key file** — Re-save credentials after first startup if key file not present.
4. **Check /api/credentials returns success** — Use browser devtools to verify POST returns 200 before calling collection APIs.

### 7.5 Inventory File Format

```
# config/asa_inventory.txt
# Format: hostname,ip,type,is_internet_gateway
FW-CORE-01,10.1.1.1,asa,true
FW-DMZ-01,10.2.2.1,asa,false
```

- `type`: lowercase (`asa`, `ftd`, `fxos`)
- `is_internet_gateway`: lowercase boolean (`true`, `false`)

---

## 8. Legacy/Duplicate Implementations

### 8.1 server.js vs server.py

| Aspect | server.js (Legacy) | server.py (Active) |
|--------|--------------------|--------------------|
| Platform | Node.js/Express | Python/Flask |
| Status | DEPRECATED | ACTIVE |
| SSH | ssh2 library | paramiko |
| Session | In-memory | In-memory + TTL |
| Compare Jobs | Inline (all at once) | SQLite or file-based |
| Scalability | ~5000 line limit | 60k-500k+ lines |
| FXOS Support | No | Yes |
| Preferred Role | No | Yes |
| Virtual Scrolling | No | Yes |

**Action:** Do not use server.js for new development.

### 8.2 compare-configs vs compare-job

| Endpoint | Use Case | Storage | Scalability |
|----------|----------|---------|------------|
| `/api/backups/compare-configs` | Small configs (<5k lines) | Memory (full diff) | Limited |
| `/api/backups/compare-configs-job` | Medium configs | SQLite | ~10k lines |
| `/api/backups/compare-job` | Large configs (60k-500k+) | Filesystem | Unlimited |

**Action:** Frontend uses `compare-job` exclusively for new compares. The other endpoints remain for backward compatibility.

### 8.3 inline compare vs scalable job

The inline compare (`/api/backups/compare-configs`) returns the full diff immediately, loading all lines into memory. This is suitable only for small configs. The scalable compare (`/api/backups/compare-job`) extracts files to disk, computes diff using git or difflib, and allows chunked retrieval.

---

## 9. Update Procedure

### 9.1 When a Feature Changes

When modifying any part of the system, update this document in the following order:

#### Step 1: Identify Impacted Sections

| What Changed | Sections to Update |
|--------------|-----------------|
| New API endpoint | Section 3 (API Catalog), Section 5 (Key Flows) |
| Modified API endpoint | Section 3 (API Catalog), Section 5, Section 6 (Per-File) |
| New JS module | Section 6 (Per-File Reference) |
| New DOM ID | Section 6 (Per-File Reference - HTML section) |
| New disk storage path | Section 4 (Data Storage) |
| Changed naming convention | Section 7 (Naming Conventions) |
| Changed UI semantic | Section 7.1 (UI Semantic Mapping) |
| New user flow | Section 5 (Key User Flows) |

#### Step 2: Verification Checklist

After making changes, verify:

- [ ] New endpoints documented with method, params, response, caller
- [ ] New JS functions documented with DOM IDs read/written
- [ ] New disk paths documented with format and lifecycle
- [ ] If UI semantic changed, update comparison direction
- [ ] If job folder layout changed, update Section 7.2
- [ ] If new flows added, add to Section 5
- [ ] Run the app and test the new feature manually
- [ ] Verify API responses match documented shapes

#### Step 3: Diff Log Entry

Add entry to Section 10 (Diff Log) with:
- Date
- Change summary
- Files touched
- Endpoints impacted
- UI IDs impacted

### 9.2 Review Triggers

Review this document when:
- Adding or modifying any API endpoint
- Adding or modifying any frontend component
- Adding or modifying any disk storage path
- Adding or modifying any user flow
- Creating or modifying any naming convention
- Onboarding a new engineer

---

## 10. Diff Log

| Date | Change | Files Touched | Endpoints Impacted | UI IDs Impacted |
|------|--------|--------------|-------------------|----------------|
| (template) | Describe change | List files | List endpoints | List DOM IDs |
| 2026-04-07 | Initial document creation | All | All | All |
| 2026-04-07 | Doc accuracy audit — corrections: moved requirements.txt to misc/, added legacy live_test_logs/ note, clarified diff.db scope (compare-configs-job only), expanded show_version_cache response fields, added clarify note to filesystem diagram diff.db comment | `docs/PROJECT_REFERENCE.md` | None (doc-only changes) | None |
| 2026-04-07 | Data Collection UI refactor: updated callers to reflect runSequential sequential-per-device flow; marked /api/collect-data as legacy; updated failover button label; added per-device status line description; added sanitizeId/setDeviceStatus/runSequential to Key Functions; fixed misc/DEVICE_STATE_REFACTORING_SUMMARY.md label | `docs/PROJECT_REFERENCE.md`, `misc/DEVICE_STATE_REFACTORING_SUMMARY.md` | `/api/collect-single-device`, `/api/collect-data`, `/api/failover-check` | `#failover-check-all-btn`, `#collection-log-box` |
| 2026-04-07 | Session stability: implemented persistent Flask secret key (env var > key file > auto-generate); added SESSION_COOKIE_SAMESITE/Lax, HTTPONLY; added localStorage auto-rehydrate on page load; added new Section 7.4 Session Persistence & Credential Stability with troubleshooting | `server.py`, `app.js`, `docs/PROJECT_REFERENCE.md` | `/api/credentials` | None |
| 2026-04-07 | Per-device credential validation: changed /api/run-ping, /api/run-packet-tracer, /api/run-show-route, /api/run-show-nat to use get_session_creds_for_device_type() instead of get_session_creds_required(); added docs on per-device credential rules (ASA->Jumpbox+ASA, FTD->Jumpbox+FTD, FXOS->Jumpbox); merged /api/credentials now preserves existing credential blocks | `server.py`, `docs/PROJECT_REFERENCE.md` | `/api/run-ping`, `/api/run-packet-tracer`, `/api/run-show-route`, `/api/run-show-nat`, `/api/credentials` | None |
| 2026-04-07 | Firewall Rules Auto (Phase 1): added new tab, JS module, backend endpoints for rule preparation; fixed rollover bug (undefined src); added base directory banner at top of tab | `server.py`, `app/web/templates/index.html`, `app/web/static/js/firewall_rules_auto.js`, `docs/PROJECT_REFERENCE.md` | `/api/firewall-rules-auto/ensure-folders`, `/api/firewall-rules-auto/date-folders`, `/api/firewall-rules-auto/prepare` | `#firewall-rules-auto-section`, `#fra-base-path-display` |
| 2026-04-07 | Firewall Rules Auto: added base path persistence via config file; added /api/firewall-rules-auto/configure (POST) and /api/firewall-rules-auto/config (GET) endpoints; added "Save as Default" button to UI | `server.py`, `app/web/templates/index.html`, `app/web/static/js/firewall_rules_auto.js`, `docs/PROJECT_REFERENCE.md` | `/api/firewall-rules-auto/configure`, `/api/firewall-rules-auto/config` | `#fra-save-default-btn` |
| 2026-04-07 | Firewall Rules Auto stability: marker.write_text encoding explicit; added src.rmdir() after rollover move; replaced bare 'except: pass' with log_lines for folder creation errors; added 'input' listener for live banner update; added logContent alias in ensure-folders response | `server.py`, `app/web/static/js/firewall_rules_auto.js` | `/api/firewall-rules-auto/ensure-folders` | None |
| 2026-04-07 | Firewall Rules Auto: auto-call ensure-folders on tab load (guarded per sessionStorage); added "Archive Today" button; added /api/firewall-rules-auto/archive-today endpoint | `server.py`, `app/web/templates/index.html`, `app/web/static/js/firewall_rules_auto.js`, `docs/PROJECT_REFERENCE.md` | `/api/firewall-rules-auto/archive-today` | `#fra-archive-today-btn` |
| 2026-04-10 | Firewall Rules Auto: updated schedule time UI with dedicated Save button, renamed to "Archive / Rollover Run Time"; scheduler now backs up Prepare_Rules during run; archive-today now goes to ChangePrepBackup\YYYYMMDD without recreate; ensure-folders guarantees 3 future date folders | `server.py`, `app/web/templates/index.html`, `app/web/static/js/firewall_rules_auto.js`, `docs/PROJECT_REFERENCE.md` | `/api/firewall-rules-auto/configure`, `/api/firewall-rules-auto/run-scheduled` | `#fra-schedule-time`, `#fra-set-rollover-btn`, `#fra-unset-rollover-btn` |
| 2026-04-10 | Firewall Rules Auto: documentation — fixed duplicate §3.7 (Global Rules §3.8, Logs §3.9), added §5.6 FRA flow, architecture diagram + §6.1/§6.2 FRA rows, clarified archive-today `recreated`; lazy auto-ensure when tab shown + `app.js` hook; localStorage path fallback; Rule Preparation grid layout | `docs/PROJECT_REFERENCE.md`, `app/web/static/js/firewall_rules_auto.js`, `app/web/static/js/app.js`, `app/web/templates/index.html` | `/api/firewall-rules-auto/ensure-folders` (caller timing) | `#firewall-rules-auto-section` |
| 2026-04-12 | Firewall Rules Auto: added separate rolloverTime and prepareTime schedules with Set/Unset controls; scheduler now runs both jobs independently (separate last_run_date tracking); prepare job runs against today's folder; config uses explicit null for disabled schedules | `server.py`, `app/web/templates/index.html`, `app/web/static/js/firewall_rules_auto.js`, `docs/PROJECT_REFERENCE.md` | `/api/firewall-rules-auto/configure`, `/api/firewall-rules-auto/config`, `_fra_scheduler_loop` | `#fra-set-rollover-btn`, `#fra-unset-rollover-btn`, `#fra-set-prepare-btn`, `#fra-unset-prepare-btn` |
| 2026-04-21 | Credentials tab: replaced localStorage with sessionStorage (clears on browser close); added idle timer (30 min UI idle) auto-clear with `startIdleTimer()` + `checkIdleStatus()`; added `/api/credentials/clear` endpoint; wired Clear buttons to `clearCredentialsAll()` | `server.py`, `app.js`, `index.html`, `docs/PROJECT_REFERENCE.md` | `/api/credentials`, `/api/credentials/clear` | `#clear-jumpbox-credentials`, `#clear-asa-credentials`, `#clear-ftd-credentials` |
| 2026-05-01 | Cancel/Stop functionality: added JOBS_REGISTRY with thread-safe helpers; 3 cancel endpoints (data-collection, firewall-inventory, path-finder); cancel checks in all SSH endpoints (collect-single-device, failover-check, run-ping, run-packet-tracer, run-show-route, run-show-nat); cache/log writes skipped on cancel; frontend tabInstanceId, stopDataCollection/stopFirewallInventory/stopPathFinder handlers; executePingTest/PacketTracerTest/ShowRouteTest/ShowNatTest refactored with AbortController + Run/Stop button toggles; show_version.js cancel support with AbortController on per-device/batch/refresh operations; Stop button visibility hooked to setCollectRunning/setFwCollectRunning | `server.py`, `app.js`, `show_version.js`, `index.html`, `docs/PROJECT_REFERENCE.md` | `/api/data-collection/cancel`, `/api/firewall-inventory/cancel`, `/api/path-finder/cancel`, `/api/collect-single-device`, `/api/failover-check`, `/api/run-ping`, `/api/run-packet-tracer`, `/api/run-show-route`, `/api/run-show-nat`, `/api/show-version-collect` | `#dc-stop-btn`, `#fw-inventory-stop-btn`, `#ping-src-stop-btn`, `#ping-dst-stop-btn`, `#pt-src-stop-btn`, `#pt-dst-stop-btn`, `#showroute-src-stop-btn`, `#showroute-dst-stop-btn`, `#shownat-src-stop-btn`, `#shownat-dst-stop-btn` |
| 2026-05-20 | Doc accuracy audit — corrected file counts and line numbers: server.py (~7100→~8300), app.js (~2500→~2584), index.html (~1105→~1241), firewall_rules_auto.js (~350→~1082), styles.css (~390→~333), show_version.js (~285→~326); added missing files to repo tree (FirewallDeploy_inventory.txt, test_myers_diff.html, Firewall_Rules_Auto_User_Guide.md, FRA_REPORTS_LOGVIEWER_REFERENCE.md); updated misc/ directory listing; rewrote server.py Section 6.1 line-number table to match actual code; updated File Inventory Summary with line counts | `docs/PROJECT_REFERENCE.md` | None (doc-only changes) | None |
| 2026-05-20 | Audit log indexing: added persistent SQLite index for historical prepare_audit.log files under Prepare_Rules/Backup; 5 new API endpoints (audit-index/stats, entries, scan, devices, categories); startup scan + scheduler-tick incremental scan; 180-day configurable retention; enhanced Log Viewer with Audit Index toggle, paginated table (500 rows/page), category/device/date/search filters, auto-refresh polling | `server.py`, `app/web/static/js/firewall_rules_auto.js`, `app/web/templates/index.html`, `docs/PROJECT_REFERENCE.md` | `/api/firewall-rules-auto/audit-index/stats`, `/api/firewall-rules-auto/audit-index/entries`, `/api/firewall-rules-auto/audit-index/scan`, `/api/firewall-rules-auto/audit-index/devices`, `/api/firewall-rules-auto/audit-index/categories`, `/api/firewall-rules-auto/configure` (extended), `/api/firewall-rules-auto/config` (extended) | `#fra-view-toggle`, `#fra-view-browse-btn`, `#fra-view-index-btn`, `#fra-view-browse-body`, `#fra-view-index-body`, `#fra-index-category`, `#fra-index-device`, `#fra-index-date-from`, `#fra-index-date-to`, `#fra-index-search`, `#fra-index-filter-btn`, `#fra-index-count`, `#fra-index-refresh-btn`, `#fra-index-scan-btn`, `#fra-index-table`, `#fra-index-tbody`, `#fra-index-page-info`, `#fra-index-prev-btn`, `#fra-index-next-btn` |
| 2026-05-20 | Report feature: added Excel report generation from stats.log + Deploystats.txt via openpyxl; 7 pre-defined metrics with native Excel charts (Bar/Pie/Line/Doughnut); executive dashboard layout (navy title banner, 4 KPI cards, 2-column chart grid, heatmap, executive insights); 3 sheets (Overview, Chart Formula hidden, Detailed with freeze panes/autofilter); inline config panel with dynamic metric checkbox enable/disable via scan-stats endpoint; synchronous download with malformed-line reporting and backend warnings | `server.py`, `app/web/static/js/firewall_rules_auto.js`, `app/web/templates/index.html`, `misc/requirements.txt`, `docs/PROJECT_REFERENCE.md` | `/api/firewall-rules-auto/report/download`, `/api/firewall-rules-auto/report/scan-stats` | `#fra-report-btn`, `#fra-report-config`, `#fra-report-metrics`, `#fra-report-chart-type`, `#fra-report-date-from`, `#fra-report-date-to`, `#fra-report-tag-filter`, `#fra-report-generate-btn`, `#fra-report-status`, `#fra-report-skipped`, `#fra-report-stats-status` |

### Scheduler Behavior

**Startup:** `_start_scheduler()` is called during server startup (`main()` function). It checks if `rolloverTime` or `prepareTime` is saved in config. If either is set, the scheduler thread starts automatically.

**Runtime:** The background scheduler (`_fra_scheduler_loop`) runs every 60 seconds and checks if the current time matches either configured `rolloverTime` or `prepareTime`. Each runs independently with its own daily guard using persistent file `last_scheduled_rollover_date.txt`.

**Rollover job** (when `rolloverTime` matches):
1. Reads `<basePath>\Prepare_Rules\last_scheduled_rollover_date.txt` to check if already ran today
2. If not ran today, executes `_fra_scheduled_rollover()`:
   - Archives today folder to `ChangePrepBackup\YYYYMMDD` (or `YYYYMMDD_EMPTY` if empty)
   - Moves past non-empty date folders to `ChangePrepBackup`
   - Tags empty past folders with `_EMPTY__YYYYMMDD.txt` marker
   - Creates/ensures 6 active date folders (tomorrow..tomorrow+5), i.e. D+1..D+6
     - Example: If today is 20260414 and rollover runs, active folders become:
       20260415, 20260416, 20260417, 20260418, 20260419, 20260420
   - Writes `last_scheduled_rollover_date.txt` only on partial success
3. Backs up Prepare_Rules/Output and logs to `Prepare_Rules/Backup/<YYYYMMDD_HHMMSS>/` where YYYYMMDD is the processed target folder date, not the run day. Example: if run day is 20260414 and target is tomorrow (20260415), backup folder begins with `20260415_20260414_120000/`.

**Prepare job** (when `prepareTime` matches):
1. Loads base path and `prepareTargetMode` from config (default: tomorrow/D+1)
2. Computes target date: if `tomorrow` → D+1, if `today` → D
3. If target folder does not exist: logs error and skips (does not create folder)
4. Backs up Prepare_Rules/Output and logs to `Prepare_Rules/Backup/<YYYYMMDD_HHMMSS>/`
5. Executes prepare logic for target folder (`<base>/YYYYMMDD/`) using the same parsing as `POST /api/firewall-rules-auto/prepare`
6. Writes output to `Prepare_Rules/Output/<device>.txt` and logs to `prepare_audit.log`, `ExcludedLog.log`, `errors.log`

**Log output:**
- `[INFO] FRA Scheduler: no schedule times configured` - if neither rolloverTime nor prepareTime in config
- `[INFO] FRA Scheduler: started with schedule rollover: HH:MM, prepare: HH:MM` - if scheduler starts

### Archive Today Behavior

- Moves today's date folder to `<base>\ChangePrepBackup\<YYYYMMDD>\`
- Does NOT recreate today's folder (user must run Ensure Folders)
- Returns error if destination already exists (prevents duplicate archives)

### Ensure 6 Date Folders Policy

When only a few date folders remain (e.g., after archiving today), `ensure-folders` and the scheduler:
1. Run rollover (move past non-empty folders to ChangePrepBackup)
2. Create/ensure 6 folders for the next 6 days (tomorrow..tomorrow+5), i.e. D+1..D+6
   - Example: If today is 20260414, folders ensured are:
     20260415, 20260416, 20260417, 20260418, 20260419, 20260420
This ensures 6 folders always exist starting from tomorrow.

### Base Path Normalization

If the user enters or has saved a subfolder path (e.g., `...\Prepare_Rules` or `...\Prepare_Rules\Output`) instead of the correct parent directory, the backend automatically detects this and normalizes to the parent directory. The subfolder names detected are:
- `Prepare_Rules`, `Output`, `Backup`, `ChangePrepBackup`

The `/api/firewall-rules-auto/config` GET response returns both `basePath` (as entered) and `normalizedBasePath` (the actual path used for folder creation). This prevents issues where date folders would be created in the wrong location.

---

## Quick Reference

### Starting the Server

```bash
# Using launcher (recommended)
python run_server.py

# Or directly
python server.py

# Port (default 8000)
python server.py 9000
```

### Key API Testing

```bash
# Save credentials
curl -X POST http://localhost:8000/api/credentials \
  -H "Content-Type: application/json" \
  -d '{"jumpbox":{"host":"jb.example.com","port":22,"username":"user","password":"pass"}}'

# List devices
curl http://localhost:8000/api/devices

# Collect device
curl -X POST http://localhost:8000/api/collect-single-device \
  -H "Content-Type: application/json" \
  -d '{"hostname":"FW-CORE-01"}'

# Auto-select firewalls
curl -X POST http://localhost:8000/api/auto-select-firewalls \
  -H "Content-Type: application/json" \
  -d '{"src_ip":"10.1.1.100","dst_ip":"8.8.8.8"}'
```

### Environment Variables

| Variable | Default | Purpose |
|----------|---------|---------|
| `PORT` | `8000` | Server port |
| `FLASK_SECRET_KEY` | Random | Session encryption |
| `DISABLE_DIRECT_TCPIP` | `1` | Force shell-hop for all devices |
