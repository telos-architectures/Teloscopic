# Teloscopic Observer Guard

**Teloscopic** is a Windows system-health and security triage dashboard packaged as a local `.exe`. It collects host telemetry, reduces common false positives, deduplicates noisy security events, and produces an HTML dashboard plus CSV exports for review.

Teloscopic is designed for local-first visibility. It does **not** replace antivirus, EDR, SIEM, or professional incident response.

---

## What it does

Teloscopic observes several layers of a Windows system and converts them into dashboard-ready health signals:

- **Identity**: running process identity, executable paths, Authenticode signer state, parent process context, and masquerading indicators.
- **Activity**: process pressure, TCP activity, network listeners, and public outbound connection context.
- **Security**: Windows Defender, PowerShell, System service-install events, optional Security log events, and optional Sysmon events.
- **Persistence**: registry run keys, startup folders, scheduled tasks, and services.
- **Drop-folder inventory**: scripts and executables in user-writable or startup-adjacent folders for manual review.
- **Content analysis**: bounded static inspection of scripts, text files, and executable metadata without executing files.
- **Baseline drift**: highlights new persistence entries or network listeners compared with prior Teloscopic snapshots.
- **Reputation overlay**: optional local reputation lists and optional online hash lookups.
- **MITRE-style tagging**: maps relevant findings to technique-style coverage outputs.

The main dashboard reports:

- Overall Health
- Risk
- Performance
- Integrity
- Priority Area
- Priority Action

---

## Why Teloscopic is different

Teloscopic is built to separate **raw observation** from **actionable risk**.

Native Windows files, expected signed vendor activity, expected signed user-application activity, and observer-generated telemetry are retained as context but excluded from dashboard risk unless there is a concrete issue. Security events are deduplicated into unique incidents before scoring, and content-analysis findings are risk-scored only when they are correlated with running process or persistence context.

The goal is not to create more alerts. The goal is to give a reviewer a cleaner map of what changed, what is active, and what deserves attention.

---

## Requirements

- Windows
- PowerShell 5.1 runtime
- Administrator elevation
- Optional: Sysmon installed and logging to `Microsoft-Windows-Sysmon/Operational`
- Optional: a local reputation JSON file
- Optional: a VirusTotal API key for online hash reputation lookups

> Teloscopic requires legitimate Administrator elevation for complete telemetry collection. It does not attempt execution-policy, UAC, AMSI, Defender, or EDR bypass.

---

## Download and run

Download `Teloscopic.exe` from the project Releases page.

Right-click the executable and choose **Run as administrator**, or launch it from an elevated PowerShell prompt:

```powershell
.\Teloscopic.exe
```

By default, Teloscopic runs in `Audit` mode, writes output files, and opens the HTML dashboard automatically.

---

## Common usage

Run a one-shot audit:

```powershell
.\Teloscopic.exe -Mode Audit
```

Write results to a chosen folder:

```powershell
.\Teloscopic.exe -Mode Audit -OutputRoot "$env:USERPROFILE\Desktop\TeloscopicRuns"
```

Run with Sysmon and Security log collection:

```powershell
.\Teloscopic.exe -Mode Audit -IncludeSysmon -IncludeSecurityLog
```

Run bounded monitor mode for 12 cycles, refreshing every 10 seconds:

```powershell
.\Teloscopic.exe -Mode Monitor -MaxMonitorCycles 12 -IntervalSeconds 10
```

Run audit first, then bounded monitor mode:

```powershell
.\Teloscopic.exe -Mode All -MaxMonitorCycles 12
```

Run without opening the dashboard automatically:

```powershell
.\Teloscopic.exe -Mode Audit -OpenDashboard:$false
```

Skip static content analysis:

```powershell
.\Teloscopic.exe -Mode Audit -SkipContentAnalysis
```

Start and stop optional `netsh trace` capture:

```powershell
.\Teloscopic.exe -Mode EtwStart
.\Teloscopic.exe -Mode EtwStop
```

---

## Modes

| Mode | Description |
|---|---|
| `Audit` | One-shot collection and dashboard generation. This is the default. |
| `Dashboard` | Alias-style mode that performs the same audit/dashboard generation path. |
| `Monitor` | Repeated activity/content refresh with periodic dashboard export. Requires `-MaxMonitorCycles` unless `-AllowContinuousMonitor` is used. |
| `All` | Runs `Audit` once, then enters `Monitor`. Requires `-MaxMonitorCycles` unless `-AllowContinuousMonitor` is used. |
| `EtwStart` | Starts optional `netsh trace` capture. |
| `EtwStop` | Stops optional `netsh trace` capture. |

---

## Command-line options

| Option | Default | Description |
|---|---:|---|
| `-Mode` | `Audit` | Selects `Audit`, `Dashboard`, `Monitor`, `All`, `EtwStart`, or `EtwStop`. |
| `-OutputRoot` | Auto | Root folder for run output. |
| `-EventLookbackHours` | `24` | How far back event log collectors should look. |
| `-DropFolderMaxFiles` | `2500` | Maximum drop-folder inventory rows. |
| `-IncludeSysmon` | Off | Includes `Microsoft-Windows-Sysmon/Operational` when present. |
| `-IncludeSecurityLog` | Off | Includes selected Windows Security log events. |
| `-StartEtwCapture` | Off | Starts lightweight `netsh trace` during audit or monitor execution. |
| `-OpenDashboard` | On | Opens the generated HTML dashboard after audit. |
| `-DisableTaskbarProgress` | Off | Disables packaged-EXE taskbar progress behavior. |
| `-IncludeObserverGeneratedSecurityRisk` | Off | Includes Teloscopic-generated PowerShell/Defender events in risk scoring. Normally leave off. |
| `-ObserverWindowMinutes` | `30` | Time window for identifying observer-generated activity. |
| `-IntervalSeconds` | `5` | Monitor refresh interval. Values under 1 are reset to 1. |
| `-TopN` | `15` | Number of top items used in dashboard tables. |
| `-SkipContentAnalysis` | Off | Skips bounded static inspection. |
| `-ContentMaxFiles` | `100` | Maximum files inspected by content analysis. |
| `-ContentMaxBytes` | `262144` | Maximum sampled bytes per inspected file. |
| `-MaxMonitorCycles` | `0` | Bounded monitor cycle count. Required for `Monitor` and `All` unless continuous mode is explicitly allowed. |
| `-AllowContinuousMonitor` | Off | Allows unbounded monitor execution. |
| `-DisableProcessLineage` | Off | Disables process-lineage enrichment. |
| `-DisableOutboundNetworkScoring` | Off | Disables public outbound network scoring. |
| `-DisableBaselineDiff` | Off | Disables cross-run baseline comparison. |
| `-BaselineStatePath` | Auto | Custom path for baseline state JSON. |
| `-DisableReputation` | Off | Disables local and online reputation scoring. |
| `-ReputationListPath` | Auto | Custom local reputation JSON path. |
| `-EnableOnlineReputation` | Off | Enables online hash reputation lookups. |
| `-VirusTotalApiKey` | Empty | API key used only when online reputation is enabled. |
| `-OnlineReputationMaxLookups` | `20` | Maximum online reputation lookups per run. |
| `-DisableMitreTagging` | Off | Disables MITRE-style tagging outputs. |

---

## Output location

Each run writes to a timestamped folder:

```text
Teloscopic_yyyyMMdd_HHmmss
```

When packaged as an `.exe`, Teloscopic tries these output roots in order:

1. The folder containing `Teloscopic.exe`
2. `%LOCALAPPDATA%\Teloscopic`
3. `%TEMP%\Teloscopic`

You can override this with `-OutputRoot`.

---

## Main output files

| File | Purpose |
|---|---|
| `Teloscopic_System-Health_Dashboard.html` | Main human-readable dashboard. |
| `Teloscopic_System-Health_Dashboard.csv` | Dashboard summary as CSV. |
| `Teloscopic_LayerSignals.csv` | Normalized signal inventory. |
| `Teloscopic_RiskRelations.csv` | Final scored risk relations. |
| `Teloscopic_SecurityIncidents.csv` | Deduplicated security incidents. |
| `Teloscopic_SecurityEvents_Raw.csv` | Raw collected security events. |
| `Teloscopic_ProcessActivity.csv` | Running process and activity telemetry. |
| `Teloscopic_NetworkActivity.csv` | TCP connection/listener rows. |
| `Teloscopic_Persistence.csv` | Run keys, startup folder items, scheduled tasks, and services. |
| `Teloscopic_ContentAnalysis.csv` | Static file/content analysis rows. |
| `Teloscopic_ContentFindings.csv` | Content findings selected for review/scoring. |
| `Teloscopic_BaselineDiff.csv` | New items since prior baseline snapshot. |
| `Teloscopic_ProcessLineage.csv` | Process parent/child relationship enrichment. |
| `Teloscopic_AttackCoverage.csv` | Technique-style coverage summary. |
| `Teloscopic_MitreFindings.csv` | MITRE-style finding mappings. |
| `Teloscopic_ReputationHits.csv` | Local or online reputation matches. |
| `Teloscopic_SystemRelations.csv` | Graph-ready system relation output. |
| `Teloscopic_TrendAnalysis.csv` | Dashboard trend rows. |
| `Teloscopic_TopLayerElements.csv` | Highest-priority layer elements. |
| `Teloscopic_Run.log` | Runtime log. |
| `Teloscopic_NetworkTrace.etl` | Optional `netsh trace` capture. |
| `Teloscopic_Baseline_State.json` | Cross-run baseline state. |

Do not publish run output publicly unless you have reviewed and sanitized it. Output files can contain usernames, process paths, command lines, network endpoints, file hashes, and security-event details.

---

## Reputation lists

Teloscopic can consume a local reputation JSON file. By default, it looks for `Teloscopic_Reputation.json` near the configured output root or executable/script path, unless `-ReputationListPath` is provided.

Example:

```json
{
  "BadHashes": [
    "0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef"
  ],
  "GoodHashes": [],
  "BadSigners": [
    "Example Suspicious Signer"
  ],
  "BadDomains": [
    "example-malicious-domain.test"
  ],
  "BadIPs": [
    "203.0.113.10"
  ]
}
```

Online reputation is disabled by default. When `-EnableOnlineReputation` and `-VirusTotalApiKey` are supplied, Teloscopic performs hash lookups against VirusTotal for a limited number of SHA-256 hashes. It does not upload files.

---

## Build the `.exe` from source

The script is intended to package cleanly as a Windows executable. One supported approach is `ps2exe`.

Example build command:

```powershell
Install-Module ps2exe -Scope CurrentUser

New-Item -ItemType Directory -Force .\dist | Out-Null

Invoke-ps2exe .\Teloscopic.ps1 .\dist\Teloscopic.exe `
  -noConsole `
  -requireAdmin `
  -x64 `
  -STA `
  -title "Teloscopic" `
  -description "Teloscopic System-Health Dashboard"
```

Recommended release artifact:

```text
dist/
  Teloscopic.exe
README.md
LICENSE
```

The `-requireAdmin` build flag is recommended so Windows requests elevation before collection begins.

---

## Exit behavior

| Condition | Behavior |
|---|---|
| Not elevated | Exits with code `740`. |
| `Monitor` or `All` without `-MaxMonitorCycles` or `-AllowContinuousMonitor` | Exits with code `2`. |
| Runtime failure | Logs details to `Teloscopic_Run.log` and returns a failure state. |
| Successful run | Writes dashboard/output files and logs `Teloscopic complete.` |

---

## Security and privacy notes

Teloscopic is a collection and triage tool. It reads local system state and writes local reports. Review the script before building or running it in sensitive environments.

Key safety properties:

- Static content analysis does not execute inspected files.
- Native Windows baseline items are not treated as dashboard risk by default.
- Expected signed vendor and expected signed user-application surfaces are gently labeled.
- Observer-generated telemetry is retained as context and excluded from risk unless explicitly included.
- Online reputation is off by default and limited by `-OnlineReputationMaxLookups` when enabled.
- Teloscopic does not attempt to bypass execution policy, UAC, AMSI, Defender, or EDR.

---

## Limitations

- Teloscopic is not malware removal software.
- Teloscopic is not an EDR replacement.
- Event log availability depends on local policy, retention, and permissions.
- Security scoring is triage-oriented and should be reviewed with operational context.
- Some protected Windows processes may not expose full path or command-line details.
- Optional Sysmon findings require Sysmon to be installed and configured before collection.

---

## Suggested workflow

1. Run `Teloscopic.exe -Mode Audit` as Administrator.
2. Open `Teloscopic_System-Health_Dashboard.html`.
3. Review `Priority Area` and `Priority Action`.
4. Inspect `Teloscopic_RiskRelations.csv` and `Teloscopic_TopLayerElements.csv`.
5. Confirm baseline drift in `Teloscopic_BaselineDiff.csv`.
6. Re-run after remediation or approved system changes.
7. Use `Monitor` mode for short, bounded follow-up observation.

---

## License

No license has been declared yet. Add a `LICENSE` file before accepting external contributions or distributing modified builds.

