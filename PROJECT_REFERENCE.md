# Firewall Path Tracer — Project Reference (Living Document)

> **Last Updated:** May 2026 (Management Reporting v2 — date range, cancel/retry, metadata, detailed metrics)  
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
├── server.py              # Primary Python/Flask backend (ACTIVE)
├── server.js              # Legacy Node.js/Express backend (DEPRECATED)
├── run_server.py          # Launcher script for Python server
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
│           │   └── firewall_rules_reports.js # Management reporting UI (ApexCharts)
│           └── vendor/
│               ├── bootstrap/        # Bootstrap 5
│               ├── fontawesome/     # Font Awesome icons
│               └── apexcharts/      # ApexCharts (charts library for reports)
│
├── config/
│   ├── asa_inventory.txt       # ASA device inventory (hostname,ip,type,is_gateway)
│   ├── ftd_inventory.txt       # FTD device inventory
│   ├── fxos_inventory.txt      # FXOS device inventory
│   └── global_rules_inventory.txt # Firewall hostnames for global rules
│
├── data/
│   ├── collected_devices/       # Device cache (hostname.json)
│   ├── device_state/            # Per-device state (failover, preferred_role)
│   ├── session_logs/            # Test execution logs (written by server.py)
│   ├── show_version_cache/      # Show version data cache
│   ├── tmp_diff/                # Scalable compare job storage
│   ├── reports_cache/           # Management reporting SQLite cache + job persistence
│   │   ├── cache.db             # SQLite: daily_metrics, rejection_categories, deploy_device_results, raw_entries
│   │   └── jobs/                # Report job persistence directory
│   └── live_test_logs/          # LEGACY: present on disk but unused by Flask server
│
├── docs/
│   ├── PROGRESS.md
│   └── PHASES.md
│
└── misc/                         # Development docs, tests, backups
│   └── requirements.txt          # Python dependencies (launcher expects this here)
    ├── FIREWALL_BACKUPS_*.md     # Implementation summaries
    ├── FXOS_*.md                # FXOS feature documentation
    ├── DEVICE_STATE_*.md         # State management docs
    └── *.js, *.py               # Test scripts, utilities
```

### File Inventory Summary

| Path | Purpose | Key Responsibilities |
|------|---------|---------------------|
| `server.py` | Primary backend | All API routes, SSH device connections, session management |
| `server.js` | Legacy backend | Deprecated Node.js implementation |
| `run_server.py` | Launcher | Checks prerequisites, installs deps, starts server |
| `app/web/static/js/app.js` | Core frontend | Navigation, credentials save, device collection, ping/packet-tracer/route/NAT tests |
| `app/web/static/js/firewall_backups.js` | Backup comparison | Archive listing, config file browsing, diff comparison, virtual scrolling |
| `app/web/static/js/global_rules.js` | ACL generation | Normal/URL rule generation with deduplication |
| `app/web/static/js/show_version.js` | Version collection | Show version fetch, display, CSV export |
| `app/web/static/js/firewall_rules_auto.js` | Rules Auto (Phase 1) | Base path, date folders, ensure/prepare/archive; calls `/api/firewall-rules-auto/*` |
| `app/web/static/js/firewall_rules_reports.js` | Management Reports UI | ApexCharts rendering, job polling, CSV download, chart type checkboxes, cancel/retry, metadata display, detailed metrics table, date range picker, Top N, animation toggle, CSV section filtering |
| `app/web/templates/index.html` | Main UI | All page sections with DOM IDs (extended with report config panel, date range, Top N, animations, prepare/deploy KPI rows, metadata section, error details, detailed metrics, cancel/retry buttons) |
| `app/web/static/css/styles.css` | Styling | Dark theme, terminal styles, diff highlighting, report styles, print CSS, config panel, metadata, error details, detailed metrics, spinner |
| `data/reports_cache/cache.db` | SQLite cache | Auto-created: daily_metrics, rejection_categories, deploy_device_results, raw_entries, weekly_metrics, monthly_metrics tables |

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
│  app.js ─ firewall_backups.js ─ show_version.js ─                            │
│  firewall_rules_auto.js ─ firewall_rules_reports.js                         │
│       │              │                  │                    │              │
│       ▼              ▼                  ▼                    ▼              │
│  ┌──────────────┐ ┌──────────────────┐ ┌─────────────────┐ ┌──────────────┐ │
│  │ Credentials  │ │ Compare Job Mgmt │ │ Version Fetch   │ │ Rule prep    │ │
│  │ Collection   │ │ Virtual Scroll   │ │ CSV Export      │ │ Date folders │ │
│  │ Path Finder  │ │ Diff Navigation│ │                 │ │ /api/fra/*   │ │
│  │ Live Tests   │ │ Search/Highlight│ │                 │ │              │ │
│  └──────────────┘ └──────────────────┘ └─────────────────┘ └──────────────┘ │
│       │                                                                    │
│       ▼                                                                    │
│  ┌────────────────────────────────────────────────────────────────────┐     │
│  │ firewall_rules_reports.js                                         │     │
│  │ ApexCharts rendering, job polling, CSV download, chart checkboxes │     │
│  │ Calls /api/fra-reports/*                                         │     │
│  └────────────────────────────────────────────────────────────────────┘     │
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
│     │  /api/firewall-rules-auto/*  (local FS: Prepare_Rules, date folders) │
   │  /api/fra-reports/*         (report generation, CSV export, cache)   │    │
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
| `/api/firewall-rules-auto/configure` | POST | `{basePath, rolloverTime?, prepareTime?, prepareTargetMode?}` | `{status, message, basePath, rolloverTime, prepareTime, prepareTargetMode}` | `firewall_rules_auto.js:saveAsDefault()` |
| `/api/firewall-rules-auto/config` | GET | - | `{status, basePath, normalizedBasePath, rolloverTime, prepareTime, prepareTargetMode}` | `firewall_rules_auto.js:loadFromBackendConfig()` |
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

**Classification rules:**
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

### 3.11 Management Reporting Endpoints

| Endpoint | Method | Params | Response | Caller |
|----------|--------|--------|----------|--------|
| `/api/fra-reports/generate` | POST | `{basePath, chartTypes?, dateRange?, frequency?, topN?, csvMetrics?, animations?, sessionId?}` | `{jobId, status: 'started'}` | `firewall_rules_reports.js:_onGenerate()` |
| `/api/fra-reports/status/<job_id>` | GET | - | `{jobId, status, progress, message, error}` | `firewall_rules_reports.js:_pollJob()` |
| `/api/fra-reports/data/<job_id>` | GET | `?sections=` (optional comma-separated filter) | `{daily, weekly, monthly, summary, metadata, error_details, granularity, frequency, top_n, animations, csv_metrics, rejection_breakdown, chart_types}` | `firewall_rules_reports.js:_fetchReportData()` |
| `/api/fra-reports/export/<job_id>` | GET | `?sections=` (optional comma-separated filter) | CSV file download | `firewall_rules_reports.js:_onExportCsv()`, `_onExportCsvSections()` |
| `/api/fra-reports/cancel/<job_id>` | POST | - | `{jobId, status: 'cancelled'}` | `firewall_rules_reports.js:_onCancel()` |
| `/api/fra-reports/retry/<job_id>` | POST | - | `{jobId, status: 'started'}` | `firewall_rules_reports.js:_onRetry()` |
| `/api/fra-reports/cleanup-jobs` | POST | `{maxAgeHours?}` (default 24) | `{removed: N}` | N/A (direct endpoint) |
| `/api/fra-reports/cache-status` | GET | - | `{cached, dayCount, lastUpdate}` | `firewall_rules_reports.js:_pollJob()` |
| `/api/fra-reports/refresh-cache` | POST | `{basePath}` | `{status, dayCount}` | N/A (direct endpoint) |

**New `generate` endpoint params:**
- `chartTypes` (array, optional): Chart types to render
- `dateRange` (object, optional): `{start: "YYYY-MM-DD", end: "YYYY-MM-DD"}` or `null` for all data
- `frequency` (string): `"daily"`, `"weekly"`, `"monthly"`, or `"custom"` (default `"daily"`)
- `topN` (number): Limit for error details (default 10)
- `csvMetrics` (array): CSV sections to include: `["summary", "daily", "weekly", "monthly", "error_details", "metadata"]`
- `animations` (boolean): Enable/disable ApexCharts animations (default true)
- `sessionId` (string, auto): Used for per-session concurrency limit (max 3 concurrent jobs)

**Auto-granularity thresholds** (for Custom date range):
- < 7 days → daily
- 7–30 days → daily
- 31–90 days → weekly
- > 90 days → monthly

**Report Data JSON Structure:**
```json
{
  "daily": [...],
  "weekly": [...],
  "monthly": [...],
  "summary": {
    "total_deploy_attempts": 23, "deploy_success_rate": 78.3,
    "deploy_success_count": 18, "deploy_partial_count": 2, "deploy_failed_count": 3,
    "total_rules_processed": 147, "total_rules_accepted": 137, "total_rules_rejected": 10,
    "accept_rate": 93.2, "total_days": 3
  },
  "metadata": {
    "job_id": "abc123def456",
    "generated_at": "2026-05-16T16:04:58",
    "date_range": {"start": "2026-05-01", "end": "2026-05-16"},
    "granularity": "daily",
    "total_days_parsed": 16,
    "total_rules_accepted": 137,
    "total_rules_rejected": 10,
    "total_deploy_attempts": 23,
    "date_range_start": "2026-05-01",
    "date_range_end": "2026-05-16"
  },
  "error_details": [
    {"date": "2026-05-14", "failed": 1, "errors": 0, "devices": 8}
  ],
  "granularity": "daily",
  "frequency": "daily",
  "top_n": 10,
  "animations": true,
  "csv_metrics": ["summary", "daily", "weekly", "monthly", "error_details", "metadata"],
  "rejection_breakdown": {"2026-05-14": 3},
  "chart_types": [...]
}
```

**CSV Export Format (with optional section filtering):**
```
Management Report - Firewall Rules Auto
Generated:,2026-05-16 16:04:58

=== Report Metadata ===
Job Id,abc123def456
Generated At,2026-05-16T16:04:58
...

=== Summary ===
Total Deploy Attempts,23
...

=== Daily Metrics ===
date,rules_accepted,...
2026-05-14,45,...

=== Weekly Aggregates ===
period,rules_accepted,...
2026-W20,137,...

=== Monthly Aggregates ===
period,...
2026-05,137,...

=== Error Details ===
date,failed,errors,devices
2026-05-14,1,0,8
```

**Session Concurrency:** Max 3 concurrent report generation jobs per session (identified by IP address or explicit `sessionId`). Attempts beyond the limit return HTTP 429. Cancel/failed/completed jobs release the slot. Retrying reuses the original session quota. Cleanup endpoint manually frees stale slots.

**Chart Types (user-selectable via checkboxes in UI):**
| Chart Type | Chart Style | Description |
|------------|-------------|-------------|
| `deploy_success_rate` | Line (green) | Daily deploy success rate % |
| `failed_partial` | Stacked Bar (red/orange) | Failed + Partial per day |
| `devices_processed` | Line (blue) | Devices processed per day |
| `rules_trend` | Line (green/red) | Accepted vs Rejected over time |
| `top_failing` | Pie | Deploy outcome distribution (Success/Partial/Failed) |
| `rejection_categories` | Pie | Rules Accepted vs Rejected total |
| `operational_trends` | Line (purple/green) | Total rules processed + deploy success |

**Job Lifecycle:**
1. `POST /api/fra-reports/generate` → returns `{jobId, status: 'started'}` (with optional `dateRange`, `frequency`, `topN`, `csvMetrics`, `animations`)
2. Background worker parses `stats.log`, `Deploystats.txt`, `DeployAuthLogs.log`
3. Worker filters by date range if specified, builds daily metrics → writes SQLite cache → computes aggregations
4. Auto-granularity: determines whether to return daily, weekly, or monthly based on date range span
5. Frontend polls `GET /api/fra-reports/status/<job_id>` every 5 seconds
6. On `status: 'completed'`, frontend calls `GET /api/fra-reports/data/<job_id>` for chart data + metadata + error details
7. User can cancel running job via `POST /api/fra-reports/cancel/<job_id>` (cooperative cancellation: worker checks status at each phase transition)
8. Failed/cancelled jobs can be retried via `POST /api/fra-reports/retry/<job_id>` (creates new job_id with same parameters)
9. User can download CSV via `GET /api/fra-reports/export/<job_id>` (supports `?sections=` filter)
10. Old jobs auto-cleaned from memory + disk after 48h (configurable via cleanup endpoint)

**File Watcher (optional):** `_fra_reports_start_watcher()` uses the `watchdog` library to monitor `stats.log` and `Deploystats.txt` for changes. Requires `watchdog>=4.0.0` in `misc/requirements.txt`. Activated in `main()` on server startup if base path is configured.

**Job Persistence:** Jobs are saved to disk at `data/reports_cache/jobs/<job_id>.json` on creation, completion, failure, and cancellation. On server restart, `_fra_reports_get_job()` falls back to disk if job not found in memory. Old job files cleaned up on startup (48h max age) and via cleanup endpoint.

**Session Concurrency:** Max 3 concurrent jobs per session (`SESSION_CONCURRENCY_LIMIT = 3`). Tracked via `SESSION_REPORT_JOBS` dict. Slots released when jobs complete, fail, or are cleaned up.

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
| `data/reports_cache/cache.db` | SQLite | Auto-created on server start | Tables: `daily_metrics` (date, rules_accepted, rules_rejected, deploy_total, deploy_success, deploy_partial, deploy_failed, deploy_errors, devices_processed, parsed_at), `rejection_categories` (date, category, device_type, count), `deploy_device_results` (date, hostname, status), `raw_entries` (id, source, entry_date, entry_type, raw_text, parsed_at) |
| `Prepare_Rules/Results/latest_results.log` | Text | Overwritten on each prepare | Base path, target date, counts, unsupported devices, **Detailed Stats section** (4 categories × ASA/FTD/Unknown Device subtotals for accepted/rejected) |
| `Prepare_Rules/Results/latest_deploy_results.log` | Text | Overwritten on each deploy | Deploy timestamp, total/success/partial/failed/error/skipped counts, per-device results |
| `Prepare_Rules/Output/DeploySuccess.log` | Text (append-only) | Keep forever | Successful deploy device entries with timestamp, hostname, IP, lines count |
| `Prepare_Rules/Output/DeployFailed.log` | Text (append-only) | Keep forever | Failed deploy lines/device entries with timestamp, hostname, IP, command, output snippet, prompt |
| `Prepare_Rules/Output/DeployError.log` | Text (append-only) | Keep forever | Deploy errors (connection/auth/timeout) with timestamp, hostname, IP, error message |
| `Prepare_Rules/logs/stats.log` | Text (append-only) | Keep forever (may change) | Day-separated entries with tags: `manual_prepare`, `scheduled_prepare`, `manual_run_scheduled`, `manual_archive`, `scheduled_rollover` |
| `Prepare_Rules/logs/Deploystats.txt` | Text (append-only) | Keep forever | Day-separated entries with tag=manual, basePath, deploy counts, and snapshot of `latest_deploy_results.log` |
| `Prepare_Rules/logs/DeployAuthLogs.log` | Text (append-only) | Keep forever | Per-deploy-job auth log: two lines per job (`started` + final). Statuses: started/success/failed/cancelled. Reasons include completed_ok, no_eligible_devices, all_devices_auth_failed, no_config_lines_sent, cancelled_by_user, forced_cancel_timeout. Pipe/newline/backslash escaped. Plaintext username retention may change later. |

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

**File:** `server.py` (~7100+ lines)

#### Major Regions

| Lines | Module/Region | Purpose |
|-------|---------------|---------|
| 26-47 | Configuration | PROJECT_ROOT, folders, directory creation |
| 49-58 | Flask App Setup | Flask initialization, static/template paths, secret key |
| 61-125 | Session/Credential Management | `SESSION_CREDS`, `get_or_create_session_id()`, `get_session_creds_required()`, `get_session_creds_for_device_type()` |
| 170-321 | Inventory & State Helpers | `parse_inventory_file()`, `get_cache_status()`, `get_device_state_path()`, `load_device_state()`, `save_device_state()` |
| 322-508 | Jumpbox Connection Layer | `connect_via_jumpbox()`, `connect_to_device_via_jumpbox()` (direct-tcpip + shell-hop), `JumpboxSessionManager` class |
| 511-862 | Shell-Hop Connection | `connect_to_device_via_jumpbox_shell()` (ASA, FTD, FXOS login sequences) |
| 863-1007 | JumpboxSessionManager | Connection pooling, reconnect handling, device chunking |
| 1009-1149 | Command Execution | `run_commands_on_device()`, `execute_command_wait_prompt()`, `execute_single_command()`, `run_failover_check_on_device()` |
| 1151-1508 | Parsing Functions | `parse_failover_state()`, `parse_show_ip_output()`, `parse_show_route_output()`, `parse_show_version_output()`, `parse_fxos_chassis_detail()`, `prefix_to_netmask()`, `is_valid_netmask()` |
| 1172-1279 | Device Collection | `collect_single_device()` (single device with step tracking) |
| 1637-1651 | Test Logging | `save_test_log()` |
| 1654-1797 | Index Building | `build_interface_index()`, `build_route_index()` (global INTERFACE_INDEX, ROUTE_INDEX) |
| 1825-2123 | Auto-Selection Logic | `find_firewall_for_ip_server()`, `find_firewall_for_ip_route()` |
| 2125-2127 | Root Route | `/` → serves index.html |
| 2141-2158 | `/api/credentials` | POST → store session credentials |
| 2161-2242 | `/api/backups/list` | GET → list archives from jumpbox |
| 2245-2325 | `/api/backups/config-files` | GET → list files in archive |
| 2328-2415 | `/api/backups/config` | GET → extract single config |
| 2418-2622 | `/api/backups/compare-configs` | POST → inline diff (small configs) |
| 2625-2895 | `/api/backups/compare-configs-job/*` | SQLite-based chunked compare (legacy) |
| 2902-3094 | Diff Job DB Helpers | `create_job_db()`, `get_job_db()`, `store_diff_rows()`, `fetch_diff_range()`, `find_next_diff()`, `cleanup_job()`, `cleanup_old_jobs()` |
| 3097-3449 | Scalable Diff Infrastructure | `build_line_index()`, `extract_config_to_file()`, `run_external_diff()`, `run_difflib_diff()`, `parse_unified_diff()` |
| 3492-3571 | Scalable Job Helpers | `get_scalable_job_metadata()`, `get_scalable_job_hunks()`, `get_scalable_job_content()` |
| 3573-3694 | `/api/backups/compare-job` | POST → create scalable job (extracts, diffs, stores) |
| 3657-3694 | `/api/backups/compare-job/<job_id>/*` | GET hunks/content, DELETE job |
| 3879-3934 | `/api/backups/compare-job/<job_id>/export.csv` | Stream CSV export |
| 3944-3962 | `/api/backups/compare-job/<job_id>/download-config/<side>` | Download raw .txt |
| 3965-4002 | `/api/devices` | GET → inventory with cache status |
| 4004-4019 | `/api/device-interfaces/<hostname>` | GET → cached interfaces |
| 4022-4050 | `/api/auto-select-firewalls` | POST → find firewalls by IP |
| 4052-4068 | `/api/collect-single-device` | POST → collect one device |
| 4070-4182 | `/api/collect-data` | POST → batch collection with JumpboxSessionManager (legacy UI path; current frontend uses sequential /api/collect-single-device via runSequential()) |
| 4184-4271 | `/api/run-ping` | POST → execute ping |
| 4273-4360 | `/api/run-packet-tracer` | POST → execute packet-tracer |
| 4362-4452 | `/api/run-show-route` | POST → execute show route |
| 4455-4545 | `/api/run-show-nat` | POST → execute show nat |
| 4547-4685 | `/api/failover-check` | POST → check failover state |
| 4687-4787 | `check_single_device_failover()` | Single device failover check with steps |
| 4789-4832 | `/api/preferred-role` | POST → set preferred role |
| 4834-4879 | `/api/logs` | GET → list logs, GET /logs/<id> |
| 4881-4901 | `/api/global-rules/inventory` | GET → firewall hostnames |
| 4903-4912 | Catch-all route | SPA support |
| 4914-5007 | Show Version Helpers | `get_show_version_cache_path()`, `load_show_version_cache()`, `save_show_version_cache()`, `run_show_version_on_device()` |
| 5096-5128 | `/api/show-version/<hostname>` | GET cached version |
| 5130-5246 | `/api/show-version-collect` | POST → collect version data |
| 5248-5301 | `/api/show-version-cache-missing` | POST → check what's missing |
| 5303-5412 | `/api/generate-show-version-csv` | POST → generate CSV download |
| 5414-5479 | `/api/show-version-inventory` | GET → inventory with version cache |
| 6273-6522 | Firewall Rules Auto helpers | `FRA_*` config paths, `_fra_load_config()`, `_fra_get_base_path()`, `_fra_perform_rollover()`, `_fra_backup_prepare_rules()`, date-folder metadata |
| 7078-7082 | Report config variables | `REPORTS_CACHE_DIR`, `REPORTS_CACHE_DB`, `REPORTS_JOBS_DIR`, `REPORTS_JOBS`, `REPORTS_JOBS_LOCK` |
| 6524-6587 | `/api/firewall-rules-auto/configure`, `/api/firewall-rules-auto/config` | POST save path + rolloverTime + prepareTime; GET returns `basePath`, `normalizedBasePath`, `rolloverTime`, `prepareTime` |
| 6589-6627 | `/api/firewall-rules-auto/create-date-folders` | POST → create today/tomorrow/day+2 folders |
| 6630-6716 | FRA scheduler | `_fra_scheduler_loop()`, `_start_scheduler()` (daemon thread, 60s tick) |
| 6718-6750 | `/api/firewall-rules-auto/run-scheduled` | POST manual scheduler test |
| 6752-6778 | `_fra_ensure_three_future_folders()` | Ensures 3 future `YYYYMMDD` folders from tomorrow |
| 6780-6840 | `/api/firewall-rules-auto/ensure-folders` | POST ensure folders + rollover + future folders |
| 6843-6856 | `/api/firewall-rules-auto/date-folders` | GET → list date folders with flags |
| 6858-6925 | `/api/firewall-rules-auto/archive-today` | POST → move today’s folder to `ChangePrepBackup/<YYYYMMDD>/` |
| 6928-7076 | `/api/firewall-rules-auto/prepare` | POST → scan date folder `*.txt`, write `Prepare_Rules/Output` |
| 9978-9996 | `_fra_reports_init_cache()` | Initialize SQLite cache, create 4 tables (daily_metrics, rejection_categories, deploy_device_results, raw_entries) |
| 9998-10050 | `_fra_reports_parse_stats_log(bp)` | Parse `Prepare_Rules/logs/stats.log` → list of days with entries containing accepted_total, rejected_total |
| 10052-10105 | `_fra_reports_parse_deploy_stats_log(bp)` | Parse `Prepare_Rules/logs/Deploystats.txt` → list of days with deploy entries (total_devices, success, partial, failed, errors) |
| 10107-10140 | `_fra_reports_parse_deploy_auth_log(bp)` | Parse `Prepare_Rules/logs/DeployAuthLogs.log` → auth entries (username, timestamp, status, reason, date) |
| 10142-10194 | `_fra_reports_build_daily_metrics()` | Merge stats + deploy + auth data into per-date dicts |
| 10196-10218 | `_fra_reports_write_cache()` | UPSERT daily_metrics rows into SQLite |
| 10220-10243 | `_fra_reports_read_cache()` | Read all daily_metrics from SQLite → list of dicts |
| 10245-10283 | `_fra_reports_aggregate()` | Aggregate daily metrics into weekly (`YYYY-Www`) or monthly (`YYYY-MM`) periods |
| 10285-10316 | `_fra_reports_compute_metrics()` | Compute derived summary (deploy_success_rate, accept_rate, totals) |
| 10318-10343 | Report job management | `_fra_reports_start_job()`, `_fra_reports_get_job()`, `_fra_reports_cleanup_job()` |
| 10345-10403 | `_fra_reports_generate_worker()` | Background thread: parse logs → build metrics → write cache → compute aggregates |
| 10405-10440 | `_fra_reports_generate_csv()` | Build CSV string: Summary + Daily + Weekly + Monthly sections |
| 10345-10370 | `/api/fra-reports/generate` | POST → async report generation → returns `jobId` |
| 10372-10390 | `/api/fra-reports/status/<job_id>` | GET → poll job progress (status, progress, message) |
| 10392-10413 | `/api/fra-reports/data/<job_id>` | GET → completed report JSON (daily, weekly, monthly, summary) |
| 10415-10440 | `/api/fra-reports/export/<job_id>` | GET → CSV file download |
| 10442-10468 | `/api/fra-reports/cache-status` | GET → cache metadata (cached, dayCount, lastUpdate) |
| 10470-10506 | `/api/fra-reports/refresh-cache` | POST → sync re-read logs → rebuild cache |
| 10508-10535 | `_fra_reports_start_watcher()` | Optional watchdog file system watcher for stats.log, Deploystats.txt |

### 6.2 app/web/templates/index.html

**File:** `app/web/templates/index.html` (~1300 lines)

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
| Reports Button | `#fra-reports-btn` | Toggle reports section visibility |
| Reports Section | `#fra-reports-section` (hidden by default `.d-none`) | Management reporting UI (charts, progress, export) |
| Reports Controls | `#fra-reports-generate-btn`, `#fra-reports-export-btn`, `#fra-reports-back-btn` | Generate, export CSV, back to FRA main |
| Reports Status | `#fra-reports-status`, `#fra-reports-progress-container`, `#fra-reports-progress-bar`, `#fra-reports-progress-message`, `#fra-reports-progress-pct` | Status alerts and animated progress bar |
| Summary Cards | `#fra-reports-summary`, `#fra-report-deploy-rate`, `#fra-report-rules-accepted`, `#fra-report-total-deploy`, `#fra-report-day-count` | 4 summary stat cards |
| Chart Checkboxes | `#fra-reports-chart-options`, `.fra-report-chart-cb` (7 checkboxes) | User-selectable chart types |
| Chart Container | `#fra-reports-charts`, `#fra-reports-chart-grid` | ApexCharts rendering area |
| CSV Export | `#fra-reports-csv-btn`, `#fra-reports-export-row` | CSV download button (shows after generation) |
| **Firewall Rules Auto Section** | `#firewall-rules-auto-section` | Local filesystem rule preparation (`#fra-*`) |
| Base / schedule | `#fra-base-path`, `#fra-base-path-display`, `#fra-base-status-badge`, `#fra-rollover-status-badge`, `#fra-schedule-time`, `#fra-set-rollover-btn`, `#fra-unset-rollover-btn`, `#fra-prepare-status-badge`, `#fra-prepare-time`, `#fra-set-prepare-btn`, `#fra-unset-prepare-btn`, `#fra-prepare-target-mode`, `#fra-scheduled-target-display`, `#fra-save-default-btn` | Saved path banner, base status badge, schedule status badges, prepare target mode |
| Date folders / actions | `#fra-date-folder`, `#fra-ensure-folders-btn`, `#fra-prepare-btn`, `#fra-archive-today-btn`, `#fra-create-dates-btn`, `#fra-refresh-dates-btn`, `#fra-manual-target-mode`, `#fra-custom-date-container`, `#fra-custom-date-select` | Select `YYYYMMDD`, ensure, prepare, archive today, create 6 folders, refresh list, manual target mode, custom date selection |
| Results | `#fra-status`, `#fra-files-scanned`, `#fra-rules-accepted`, `#fra-rules-rejected`, `#fra-devices-written`, `#fra-output-folder`, `#fra-results-content` | Status alert, counts, log output |
| Display results | `#fra-display-results-btn`, `#fra-display-results-ui`, `#fra-display-back-btn`, `#fra-folder-search`, `#fra-folder-select`, `#fra-subfolder-select`, `#fra-file-select` | Log viewer with folder search, folder picker, subfolder picker, file picker (.txt/.log), back navigation (4-step) |
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

**File:** `app/web/static/js/app.js` (~2500+ lines)

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

**File:** `app/web/static/js/show_version.js` (~285 lines)

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

**File:** `app/web/static/js/firewall_rules_auto.js` (~350 lines)

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
| `#fra-date-folder` | write (select options) | Date folder dropdown |
| `#fra-ensure-folders-btn` | click | Ensure folders handler |
| `#fra-prepare-btn` | click | Prepare rules handler |
| `#fra-create-dates-btn` | click | Create 6 date folders (tomorrow..tomorrow+5) |
| `#fra-refresh-dates-btn` | click | Refresh date folders |
| `#fra-archive-today-btn` | click | Archive today's folder |
| `#fra-run-scheduled-btn` | click | Run scheduled task manually |
| `#fra-display-results-btn` | click | Show log viewer UI |
| `#fra-display-back-btn` | click | Step back in log viewer |
| `#fra-folder-search` | input | Search/filter backup folders |
| `#fra-folder-select` | change | Select backup folder |
| `#fra-file-select` | change | Select log file to display |
| `#fra-status` | write | Status message |
| `#fra-files-scanned` | write | Files scanned count |
| `#fra-rules-accepted` | write | Rules accepted count |
| `#fra-rules-rejected` | write | Rules rejected count |
| `#fra-devices-written` | write | Devices written count |
| `#fra-output-folder` | write | Output folder path |
| `#fra-results-content` | write | Results/log display |

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

**UI Elements:** `#fra-base-path` (input), `#fra-base-status-badge`, `#fra-rollover-status-badge`, `#fra-schedule-time` (time input), `#fra-prepare-status-badge`, `#fra-prepare-time` (time input), `#fra-date-folder` (select), `#fra-status` (alert).

**Base Path Status Badge:** `#fra-base-status-badge` shows core folder status. Checks folder existence only (no file content required). Runs on tab load. States: `all core files detected` (green), `MISSING: Prepare_Rules\Output, Prepare_Rules\Backup, ChangePrepBackup` (red) listing only the missing folders, `ERROR: <message>` (red) on access/permission issues.

**Schedule Status Badges:** `#fra-rollover-status-badge` and `#fra-prepare-status-badge` show state based on time being set (not scheduler runtime). States: `ACTIVE (Last: YYYYMMDD)` (green), `NOT ACTIVE` (gray), `ERROR` (red).
**Prepare Badge "Last":** The "Last:" value shows the folder date that was actually processed (e.g., D+1 when prepareTargetMode=tomorrow), not the run date. Stored in `Prepare_Rules/last_scheduled_prepare_date.txt`. Both scheduled and manual prepare jobs update this file after successful completion.

**Log Viewer Navigation:** 4-step flow: folder → subfolder → file → content. Back button steps back through each level, clearing selections and terminal at each step.

**Scheduled Rollover:** Uses `shutil.move` to move entire folder (not copy contents). After rollover on day D, ensures 6 active folders: D+1..D+6 (tomorrow through tomorrow+5).

**Date Folder Labels:** `refreshDateFolders()` shows `(past)`, `(empty)`, `(tagged)` based on folder metadata. Past folders are styled as muted.

### 6.8 app/web/static/js/firewall_rules_reports.js

**File:** `app/web/static/js/firewall_rules_reports.js` (~750 lines)

#### Top-Level Responsibilities

1. **Management Reports UI** — show/hide reports section on button click
2. **Configurable Report Generation** — POST with date range, report type, Top N, animation toggle, CSV section filter
3. **Async Report Generation** — POST to `/api/fra-reports/generate`, poll status every 5 seconds via `setInterval`
4. **Cancel/Retry** — cancel running job, retry failed/cancelled jobs (creates new job_id preserving original params)
5. **ApexCharts Rendering** — 7 chart types with animation toggle support
6. **CSV Export** — full download or section-filtered via `?sections=` query param
7. **Prepare/Deploy KPI Cards** — separate rows: Prepare (accepted, rejected, accept_rate) and Deploy (success_rate, total_deployments, failed_count)
8. **Metadata Display** — collapsible section showing job ID, generation time, granularity, date range, totals
9. **Error Details Table** — expandable table of top-N failing days
10. **Detailed Metrics Table** — expandable per-period breakdown (daily/weekly/monthly based on granularity)

#### Key State

```javascript
var _state = {
  currentJobId: null,    // Active report generation job ID
  reportData: null,      // Cached report JSON (daily, weekly, monthly, summary, metadata, error_details)
  pollTimer: null,       // setInterval handle for job polling
  chartInstances: {},    // Rendered ApexCharts instances (keyed by chart type)
};
```

#### DOM IDs Read/Written

| ID | Access | Purpose |
|----|--------|---------|
| `#fra-reports-btn` | click | Show reports section |
| `#fra-reports-generate-btn` | click, disabled, text | Start report generation |
| `#fra-reports-export-btn` | click | Download CSV with section filter |
| `#fra-reports-back-btn` | click | Hide reports section |
| `#fra-reports-cancel-btn` | click | Cancel running job |
| `#fra-reports-retry-btn` | click | Retry failed/cancelled job |
| `#fra-reports-status` | classList, textContent | Status alerts (info/success/danger/warning) |
| `#fra-reports-progress-bar` | style.width, aria-valuenow | Animated progress bar |
| `#fra-reports-progress-message` | textContent | Current operation description |
| `#fra-reports-progress-pct` | textContent | Percentage display |
| `#fra-reports-spinner` | classList | Loading spinner overlay |
| `#fra-report-type` | value | Report type dropdown (prepare/deploy/both) |
| `#fra-report-date-range` | value, change | Date range preset selector |
| `#fra-report-date-start` | value | Custom date range start |
| `#fra-report-date-end` | value | Custom date range end |
| `#fra-custom-date-container` | classList | Custom date input visibility |
| `#fra-report-top-n` | value | Top N error details limit |
| `#fra-report-animations` | checked | Chart animation toggle |
| `.fra-csv-metric-cb` | checked | CSV section filter checkboxes |
| `#fra-reports-prepare-summary` | classList | Prepare KPI row visibility |
| `#fra-reports-deploy-summary` | classList | Deploy KPI row visibility |
| `#fra-report-prepare-accepted` | textContent | Prepare: rules accepted |
| `#fra-report-prepare-rejected` | textContent | Prepare: rules rejected |
| `#fra-report-prepare-accept-rate` | textContent | Prepare: accept rate % |
| `#fra-report-deploy-success-rate` | textContent | Deploy: success rate % |
| `#fra-report-deploy-total` | textContent | Deploy: total deployments |
| `#fra-report-deploy-failed` | textContent | Deploy: failed count |
| `#fra-reports-metadata-section` | classList | Metadata collapsible visibility |
| `#fra-reports-metadata-grid` | innerHTML | Metadata field population |
| `#fra-reports-error-section` | classList | Error details visibility |
| `#fra-reports-error-tbody` | innerHTML | Error rows |
| `#fra-error-count-badge` | textContent | Error count badge |
| `#fra-reports-detailed-metrics-section` | classList | Detailed metrics visibility |
| `#fra-reports-detailed-metrics-tbody` | innerHTML | Metric rows |
| `#fra-reports-chart-options` | classList | Chart checkbox visibility |
| `.fra-report-chart-cb` | checked | Per-chart type toggle |
| `#fra-reports-charts` | classList | Charts container visibility |
| `#fra-reports-chart-grid` | innerHTML (append) | Chart card grid |
| `#fra-reports-csv-btn` | click | CSV download trigger (no section filter) |
| `#fra-reports-export-row` | classList | CSV button visibility |
| `#fra-base-path` | value (read) | Base path for report generation |

#### Chart Rendering Functions

| Function | Purpose |
|----------|---------|
| `_buildLineChart(categories, series, colors)` | Generate ApexCharts line config (smooth curve, dark theme, datetime xaxis) |
| `_buildBarChart(categories, series, colors)` | Generate ApexCharts stacked bar config (4px radius, 60% column width) |
| `_buildPieChart(data)` | Generate ApexCharts pie config (dark theme, bottom legend, data labels) |
| `_renderChart(chartType, data, animations)` | Factory: creates chart card, selects chart builder, disables animations if toggled off |
| `_destroyCharts()` | Destroy all ApexCharts instances (called on regenerate/back) |

#### Key Functions

| Function | Purpose |
|----------|---------|
| `_init()` | Register DOM event listeners on page load (including cancel, retry, date range change, export sections) |
| `_onDateRangeChange()` | Show/hide custom date range inputs when preset changes |
| `_getDateRange()` | Compute date range object from preset or custom date inputs |
| `_getCsvMetrics()` | Read checked CSV section checkboxes |
| `_onShowReports()` | Show reports section with info message |
| `_onBack()` | Hide reports section, destroy charts, reset state |
| `_onGenerate()` | Read all config params (basePath, chartTypes, dateRange, frequency, topN, csvMetrics, animations), POST to `/api/fra-reports/generate` |
| `_onCancel()` | POST to `/api/fra-reports/cancel/<job_id>`, show retry button |
| `_onRetry()` | POST to `/api/fra-reports/retry/<job_id>`, start polling new job |
| `_startPolling()` | Begin 5-second interval polling for job status |
| `_stopPolling()` | Clear polling interval |
| `_pollJob()` | Fetch status, update progress, handle completed/failed/cancelled, show cancel/retry buttons, show spinner on completion |
| `_fetchReportData()` | GET `/api/fra-reports/data/<job_id>`, call `_renderReport()` |
| `_renderReport(data)` | Update Prepare/Deploy KPI cards, render metadata, error details, detailed metrics, charts based on granularity, show export button |
| `_renderMetadata(metadata)` | Populate metadata fields in collapsible section |
| `_renderErrorDetails(errorDetails)` | Build error detail table rows, update badge count |
| `_renderDetailedMetrics(data)` | Build detailed metrics table (daily/weekly/monthly based on granularity) |
| `_onExportCsv()` | Redirect to `/api/fra-reports/export/<job_id>` (full CSV) |
| `_onExportCsvSections()` | Redirect to `/api/fra-reports/export/<job_id>?sections=...` (filtered CSV) |
| `_showSpinner(show)` | Show/hide loading spinner overlay |

#### Chart Type to Visual Mapping

| Chart Type | Builder | Series | Purpose |
|------------|---------|--------|---------|
| `deploy_success_rate` | `_buildLineChart` | 1 series (green): daily success % | Track deployment health |
| `failed_partial` | `_buildBarChart` | 2 series (red, orange): failed + partial | Identify problem days |
| `devices_processed` | `_buildLineChart` | 1 series (blue): devices per day | Volume tracking |
| `rules_trend` | `_buildLineChart` | 2 series (green, red): accepted + rejected | Rule processing volume |
| `top_failing` | `_buildPieChart` | 3 slices: success, partial, failed | Overall deploy outcome |
| `rejection_categories` | `_buildPieChart` | 2 slices: accepted, rejected | Rule acceptance ratio |
| `operational_trends` | `_buildLineChart` | 2 series (purple, green): total rules + deploy success | Overall activity |

### 6.9 app/web/static/css/styles.css

**File:** `app/web/static/css/styles.css` (~550 lines)

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
| `#fra-reports-section .progress` | Report progress bar styling (dark bg, 4px radius) |
| `#fra-reports-section .form-check-inline` | Report chart checkbox inline layout |
| `#fra-reports-summary .card` | Summary card hover animation (translateY, shadow) |
| `#fra-reports-prepare-summary .card` | Prepare KPI card hover animation |
| `#fra-reports-deploy-summary .card` | Deploy KPI card hover animation |
| `#fra-reports-config-panel .card-body` | Config panel nested card padding |
| `#fra-reports-config-panel .form-label` | Config panel label color (slate) |
| `#fra-custom-date-container input[type="date"]` | Date input dark color-scheme |
| `#fra-reports-metadata-grid span` | Metadata field font size |
| `#fra-reports-error-table th/td` | Error table header/font styling |
| `#fra-reports-detailed-metrics-table th/td` | Detailed metrics table compact styling |
| `#fra-reports-spinner .spinner-border` | Loading spinner size (3rem) |
| `#fra-reports-charts .card` | Chart card hover shadow |
| `.fra-no-data` | No-data placeholder style |

#### Print CSS (`@media print`)

The print stylesheet hides all UI chrome (sidebar, nav, buttons, modals, `.d-none` elements) and shows all report content: Prepare/Deploy KPI cards, charts, metadata section, error details, and detailed metrics table. Config panel, progress, spinner, and export row are hidden. Cards use light backgrounds for print legibility.

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

| 2026-05-16 | Management Reporting feature: added 6 report API endpoints, ApexCharts vendor file, firewall_rules_reports.js module, SQLite cache (4 tables), log parsers (stats.log, Deploystats.txt, DeployAuthLogs.log), daily/weekly/monthly aggregation, CSV export, 7 chart types (line/bar/pie), progress bar with 5s job polling, optional watchdog file watcher, print CSS media query | `server.py`, `app/web/templates/index.html`, `app/web/static/js/firewall_rules_reports.js`, `app/web/static/css/styles.css`, `app/web/static/vendor/apexcharts/apexcharts.min.js`, `misc/requirements.txt`, `docs/PROJECT_REFERENCE.md` | `/api/fra-reports/generate`, `/api/fra-reports/status/<job_id>`, `/api/fra-reports/data/<job_id>`, `/api/fra-reports/export/<job_id>`, `/api/fra-reports/cache-status`, `/api/fra-reports/refresh-cache` | `#fra-reports-btn`, `#fra-reports-section`, `#fra-reports-generate-btn`, `#fra-reports-export-btn`, `#fra-reports-back-btn`, `#fra-reports-status`, `#fra-reports-progress-container`, `#fra-reports-progress-bar`, `#fra-reports-progress-message`, `#fra-reports-progress-pct`, `#fra-reports-summary`, `#fra-report-deploy-rate`, `#fra-report-rules-accepted`, `#fra-report-total-deploy`, `#fra-report-day-count`, `#fra-reports-chart-options`, `.fra-report-chart-cb`, `#fra-reports-charts`, `#fra-reports-chart-grid`, `#fra-reports-csv-btn`, `#fra-reports-export-row` |
| 2026-05-16 (v2) | Management Reporting enhancements: date range selection (presets + custom), configurable params (frequency, topN, animations, csvMetrics), auto-granularity thresholds, job persistence to disk, per-session concurrency limit (3), cancel/retry/cleanup endpoints, Prepare/Deploy KPI card separation, metadata section, error details table, detailed metrics table, loading spinner, file watcher activation in main(), cache invalidation function, section-filtered CSV export, animation toggle, chart group labels, print CSS improvements | `server.py`, `app/web/templates/index.html`, `app/web/static/js/firewall_rules_reports.js`, `app/web/static/css/styles.css`, `docs/PROJECT_REFERENCE.md` | `/api/fra-reports/generate` (enhanced), `/api/fra-reports/cancel/<job_id>`, `/api/fra-reports/retry/<job_id>`, `/api/fra-reports/cleanup-jobs`, `/api/fra-reports/data/<job_id>` (metadata + error_details), `/api/fra-reports/export/<job_id>` (?sections= filter) | `#fra-report-frequency`, `#fra-date-range-container`, `#fra-report-override-range`, `#fra-report-date-range`, `#fra-report-date-start`, `#fra-report-date-end`, `#fra-custom-date-container`, `#fra-report-top-n`, `#fra-report-animations`, `.fra-csv-metric-cb`, `#fra-reports-cancel-btn`, `#fra-reports-retry-btn`, `#fra-reports-spinner`, `#fra-reports-prepare-summary`, `#fra-reports-deploy-summary`, `#fra-report-prepare-accepted`, `#fra-report-prepare-rejected`, `#fra-report-prepare-accept-rate`, `#fra-report-deploy-success-rate`, `#fra-report-deploy-total`, `#fra-report-deploy-failed`, `#fra-reports-metadata-section`, `#fra-reports-error-section`, `#fra-reports-error-table`, `#fra-error-count-badge`, `#fra-reports-detailed-metrics-section`, `#fra-reports-detailed-metrics-table`, `#fra-reports-chart-prepare-group`, `#fra-reports-chart-deploy-group` |

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

# Generate management report (async, with options)
curl -X POST http://localhost:8000/api/fra-reports/generate \
  -H "Content-Type: application/json" \
   -d '{"basePath":"Z:\\Path\\To\\Firewall rules Auto","dateRange":{"start":"2026-05-01","end":"2026-05-16"},"frequency":"daily","topN":10,"animations":true}'

# Generate report with date range
curl -X POST http://localhost:8000/api/fra-reports/generate \
  -H "Content-Type: application/json" \
  -d '{"basePath":"Z:\\Path\\To\\Firewall rules Auto","dateRange":{"start":"2026-05-01","end":"2026-05-16"}}'

# Poll report job status
curl http://localhost:8000/api/fra-reports/status/<job_id>

# Get report data
curl http://localhost:8000/api/fra-reports/data/<job_id>

# Get report data (filtered sections only)
curl "http://localhost:8000/api/fra-reports/data/<job_id>?sections=summary,metadata"

# Download CSV export (full)
curl -o report.csv http://localhost:8000/api/fra-reports/export/<job_id>

# Download CSV export (filtered sections)
curl -o report.csv "http://localhost:8000/api/fra-reports/export/<job_id>?sections=summary,daily,error_details"

# Cancel running job
curl -X POST http://localhost:8000/api/fra-reports/cancel/<job_id>

# Retry failed/cancelled job
curl -X POST http://localhost:8000/api/fra-reports/retry/<job_id>

# Cleanup old jobs (default 24h)
curl -X POST http://localhost:8000/api/fra-reports/cleanup-jobs \
  -H "Content-Type: application/json" \
  -d '{"maxAgeHours":48}'

# Check cache status
curl http://localhost:8000/api/fra-reports/cache-status

# Refresh cache
curl -X POST http://localhost:8000/api/fra-reports/refresh-cache \
  -H "Content-Type: application/json" \
  -d '{"basePath":"Z:\\Path\\To\\Firewall rules Auto"}'
```

### Environment Variables

| Variable | Default | Purpose |
|----------|---------|---------|
| `PORT` | `8000` | Server port |
| `FLASK_SECRET_KEY` | Random | Session encryption |
| `DISABLE_DIRECT_TCPIP` | `1` | Force shell-hop for all devices |
