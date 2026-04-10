# CLAUDE.md — FFU Builder (FFUFLUENT)

This file documents the codebase structure, development conventions, and workflows for AI assistants working in this repository.

---

## Project Overview

**FFU Builder** automates the creation of Windows Full Flash Update (FFU) images — sector-level snapshots that deploy a complete Windows installation (OS + updates + drivers + apps) in under two minutes. It targets enterprise and IT-pro deployment scenarios for Windows 10, 11, Server 2022/2025, and LTSC editions including ARM64 (Copilot+ PCs).

- **GitHub:** `rbalsleyMSFT/FFU`
- **License:** MIT
- **Latest release:** 2603.2 (2026-03-16)
- **Language:** PowerShell 7+ only (no compiled code, no package managers)
- **Platform:** Windows 11/10 with Hyper-V enabled, run as Administrator

---

## Repository Layout

```
FFUFLUENT/
├── FFUDevelopment/               # All source code lives here
│   ├── BuildFFUVM.ps1            # Primary CLI build script (~7,600 lines)
│   ├── BuildFFUVM_UI.ps1         # WPF GUI launcher (~36 KB)
│   ├── BuildFFUVM_UI.xaml        # WPF UI definition (~77 KB)
│   ├── Create-PEMedia.ps1        # Standalone WinPE media creator
│   │
│   ├── FFU.Common/               # Shared utility modules
│   │   ├── FFU.Common.psd1       # Module manifest
│   │   ├── FFU.Common.Core.psm1  # Logging, process execution, BITS transfers
│   │   ├── FFU.Common.Drivers.psm1
│   │   ├── FFU.Common.Drivers.Microsoft.psm1
│   │   ├── FFU.Common.Drivers.Dell.psm1
│   │   ├── FFU.Common.Winget.psm1  # WinGet integration (~77 KB)
│   │   ├── FFU.Common.Parallel.psm1
│   │   └── FFU.Common.Cleanup.psm1
│   │
│   ├── FFUUI.Core/               # WPF UI logic modules
│   │   ├── FFUUI.Core.psd1
│   │   ├── FFUUI.Core.psm1
│   │   ├── FFUUI.Core.Applications.psm1
│   │   ├── FFUUI.Core.Config.psm1        # Config management (~69 KB)
│   │   ├── FFUUI.Core.Drivers.psm1
│   │   ├── FFUUI.Core.Drivers.Dell.psm1
│   │   ├── FFUUI.Core.Drivers.HP.psm1
│   │   ├── FFUUI.Core.Drivers.Lenovo.psm1
│   │   ├── FFUUI.Core.Drivers.Microsoft.psm1
│   │   ├── FFUUI.Core.Handlers.psm1      # UI event handlers (~53 KB)
│   │   ├── FFUUI.Core.Initialize.psm1    # UI init (~49 KB)
│   │   ├── FFUUI.Core.Shared.psm1        # Shared UI utilities (~49 KB)
│   │   ├── FFUUI.Core.WindowsSettings.psm1
│   │   └── FFUUI.Core.Winget.psm1
│   │
│   ├── Apps/
│   │   ├── Orchestration/        # Scripts that run inside the Hyper-V VM
│   │   │   ├── Orchestrator.ps1          # Master in-VM script
│   │   │   ├── Install-Win32Apps.ps1
│   │   │   ├── Install-StoreApps.ps1
│   │   │   ├── Run-Sysprep.ps1
│   │   │   ├── Run-DiskCleanup.ps1
│   │   │   └── Invoke-AppsScript.ps1
│   │   ├── Office/               # ODT config (DeployFFU.xml, DownloadFFU.xml)
│   │   ├── AppList_Sample.json   # WinGet app list template
│   │   └── AppList_InboxAppsSample_Win11_24H2.json
│   │
│   ├── config/
│   │   └── Sample_default.json   # Master configuration template (UTF-16 LE)
│   ├── unattend/                 # Unattend.xml templates (x64, arm64)
│   ├── Autopilot/                # Autopilot config templates
│   ├── PPKG/                     # Provisioning package templates
│   ├── Drivers/                  # Local driver storage (user-populated)
│   ├── PEDrivers/                # WinPE-specific drivers
│   ├── WinPECaptureFFUFiles/     # Scripts bundled into WinPE capture ISO
│   ├── WinPEDeployFFUFiles/      # Scripts bundled into WinPE deploy ISO
│   ├── VM/                       # Hyper-V VM disk storage
│   └── Docs/
│       ├── BuildFFUVM_flowchart.md  # Mermaid diagram of build process
│       └── BuildDeployFFU.docx
│
├── docs/                         # Jekyll documentation site (just-the-docs theme)
│   ├── _config.yml
│   ├── index.md
│   ├── quickstart.md
│   ├── prerequisites.md
│   ├── build.md                  # Build parameters reference (~48 KB)
│   ├── drivers.md
│   ├── applications.md
│   ├── winget.md
│   ├── ui_overview.md
│   └── ...
│
├── image/                        # Image assets for docs
├── README.md                     # Project overview and release notes
├── ChangeLog.md                  # Full version history (all releases)
└── LICENSE                       # MIT
```

---

## Architecture

The build process has three distinct execution contexts:

### 1. Host machine (BuildFFUVM.ps1 / UI)
Runs on the Windows host with Hyper-V. Responsible for:
- Downloading Windows ESD/ISO, ADK, updates, drivers, apps
- Creating and managing the Hyper-V VM
- Mounting/injecting content into the offline VHDX
- Launching the VM to run in-VM orchestration
- Capturing the FFU via WinPE
- Building the USB deployment drive

### 2. In-VM orchestration (Apps/Orchestration/)
Scripts that execute inside the Hyper-V VM in Windows audit mode:
- `Orchestrator.ps1` — entry point, coordinates the sequence
- `Install-Win32Apps.ps1` — installs WinGet packages
- `Install-StoreApps.ps1` — installs Store apps
- `Run-DiskCleanup.ps1` — disk cleanup before capture
- `Run-Sysprep.ps1` — sysprepping the image
- `Invoke-AppsScript.ps1` — runs any user-provided custom script

### 3. WinPE environments
- **Capture WinPE:** boots the VM after audit mode to capture the FFU via DISM
- **Deploy WinPE:** boots the target device to apply the FFU from USB

### Module dependency graph
```
BuildFFUVM.ps1
    └── FFU.Common (Core, Drivers, Winget, Parallel, Cleanup)

BuildFFUVM_UI.ps1
    └── FFUUI.Core (Config, Handlers, Initialize, Shared, Applications,
                    Drivers*, WindowsSettings, Winget)
        └── FFU.Common
```

---

## Entry Points

| Script | Purpose | How to run |
|--------|---------|-----------|
| `BuildFFUVM_UI.ps1` | WPF GUI — recommended for most users | Right-click → Run with PowerShell 7 (as Admin) |
| `BuildFFUVM.ps1` | CLI / scripting / automation | `.\BuildFFUVM.ps1 -ConfigFile .\config\default.json` |
| `Create-PEMedia.ps1` | Create WinPE media without a full build | `.\Create-PEMedia.ps1 [params]` |

**Minimum requirements to run:**
- Windows 11 or 10 with Hyper-V role enabled
- PowerShell 7+
- Windows ADK (auto-downloaded if `UpdateADK = $true`)
- Run as Administrator
- Long paths enabled: `HKLM:\SYSTEM\CurrentControlSet\Control\FileSystem\LongPathsEnabled = 1`

---

## Configuration System

Configuration is JSON-based. The canonical template is `FFUDevelopment/config/Sample_default.json` (encoded UTF-16 LE with BOM).

Key parameters (all PascalCase):

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `WindowsRelease` | int | `11` | 10, 11, 2022, 2025 |
| `WindowsVersion` | string | `"24h2"` | e.g. `"24h2"`, `"23h2"` |
| `WindowsSKU` | string | `"Pro"` | `"Pro"`, `"Enterprise"`, `"Education"`, etc. |
| `WindowsArch` | string | `"x64"` | `"x64"` or `"arm64"` |
| `WindowsLang` | string | `"en-us"` | Language tag |
| `FFUDevelopmentPath` | string | `"C:\FFUDevelopment"` | Root working directory |
| `FFUCaptureLocation` | string | `"C:\FFUDevelopment\FFU"` | Where FFU files are saved |
| `VMLocation` | string | `"C:\FFUDevelopment\VM"` | Hyper-V VM disk location |
| `VMSwitchName` | string | `""` | Hyper-V virtual switch name |
| `Memory` | long | `4294967296` (4 GB) | VM RAM in bytes |
| `Processors` | int | `4` | VM vCPU count |
| `Disksize` | long | `32212254720` (30 GB) | VM disk size in bytes |
| `InstallApps` | bool | `true` | Run WinGet app installation |
| `InstallDrivers` | bool | `true` | Inject OEM drivers |
| `InstallOffice` | bool | `true` | Include Microsoft 365 Apps |
| `UpdateLatestCU` | bool | `true` | Include latest cumulative update |
| `UpdateLatestDefender` | bool | `true` | Include latest Defender updates |
| `UpdateEdge` | bool | `true` | Include latest Edge |
| `UpdateOneDrive` | bool | `true` | Include latest OneDrive |
| `BuildUSBDrive` | bool | `true` | Create USB deployment drive |
| `CompactOS` | bool | `true` | Apply CompactOS compression |
| `AllowVHDXCaching` | bool | `true` | Cache VHDX to avoid re-downloading |
| `AppListPath` | string | `"...\Apps\AppList.json"` | Path to WinGet app list JSON |
| `ConfigFile` | string | `null` | Path to load another JSON config |
| `LogicalSectorSizeBytes` | int | `512` | 512 or 4096 (4Kn drives) |

**App list format** (`AppList.json`):
```json
[
  { "name": "Microsoft Teams", "id": "Microsoft.Teams", "source": "winget" },
  { "name": "Company Portal", "id": "9WZDNCRFJ3PZ", "source": "msstore" }
]
```

---

## PowerShell Code Conventions

### Naming
- **Functions:** PascalCase verb-noun (`WriteLog`, `Invoke-Process`, `Start-BitsTransferWithRetry`)
- **Parameters:** PascalCase (`-LogText`, `-FilePath`, `-Priority`)
- **Script-scoped vars:** `$script:VariableName` prefix
- **Local vars:** camelCase or PascalCase (mixed in codebase; follow existing style in the file)

### Module structure
- Each `.psm1` starts with a `<# .SYNOPSIS / .DESCRIPTION #>` block for the module
- All exported functions have comment-based help (`.SYNOPSIS`, `.DESCRIPTION`, `.PARAMETER`, `.EXAMPLE`)
- `#Requires` statements at the top of `.ps1` scripts
- Manifests (`.psd1`) explicitly list `FunctionsToExport`

### Error handling
- Use `try/catch/finally` blocks, not bare `-ErrorAction Stop` alone
- Log errors via `WriteLog` before rethrowing or continuing
- Mutexes (`System.Threading.Mutex`) protect shared log file writes

### Logging
All logging goes through `WriteLog` from `FFU.Common.Core`:
```powershell
WriteLog "Message text here"   # Writes to log file with timestamp
```
Log path must be initialized with `Set-CommonCoreLogPath -Path $logFilePath` before any calls.

### BITS transfers
Use `Start-BitsTransferWithRetry` (not raw `Start-BitsTransfer`) — it handles retries and priority:
```powershell
Start-BitsTransferWithRetry -Source $url -Destination $dest
```

### Process execution
Use `Invoke-Process` (not `Start-Process` or `&`) for external tools — it captures output and handles errors consistently.

### UI (WPF)
- XAML defined in `BuildFFUVM_UI.xaml`, loaded at runtime
- Event handlers are in `FFUUI.Core.Handlers.psm1`
- UI state managed through `FFUUI.Core.Shared.psm1` shared variables
- Always dispatch UI updates on the WPF Dispatcher thread

---

## Testing

There is **no automated test suite** (no Pester tests). Testing is manual and integration-based — running an actual build against a VM.

When making changes:
1. Verify PowerShell syntax: `$null = [System.Management.Automation.Language.Parser]::ParseFile($file, [ref]$null, [ref]$err)`
2. Check for PSScriptAnalyzer issues if available: `Invoke-ScriptAnalyzer -Path ./FFUDevelopment`
3. Test the full build path in a Hyper-V environment with the target Windows version
4. Test both CLI (`BuildFFUVM.ps1`) and UI (`BuildFFUVM_UI.ps1`) flows if the change touches shared modules

---

## Development Workflow

### Branching model
- `main` — stable, release branch
- `UI` — ongoing UI feature development (merged to main via PRs)
- Feature branches: `claude/...`, contributor branches

### Commit style
Short imperative summaries, e.g.:
- `Fixes working directory handling`
- `Uses ADK BCDBoot to prevent issues with devices that have updated Secureboot certificates`
- `Updates changelog for version 2603.2`

### Release naming
Releases use `YYMM.patch` format: `2603.2` = March 2026, patch 2. Track changes in `ChangeLog.md`.

### Git remote
The local proxy remote is: `http://local_proxy@127.0.0.1:45750/git/jansene/FFUFLUENT`

---

## Key Files for AI Assistants

When understanding or modifying the build logic, read these files first:

| File | Why |
|------|-----|
| `FFUDevelopment/BuildFFUVM.ps1` | The monolithic main build script — most build behavior lives here |
| `FFUDevelopment/FFU.Common/FFU.Common.Core.psm1` | Core shared utilities (logging, process execution, BITS) |
| `FFUDevelopment/FFU.Common/FFU.Common.Winget.psm1` | WinGet integration (large, complex) |
| `FFUDevelopment/FFUUI.Core/FFUUI.Core.Config.psm1` | Config serialization/deserialization for the UI |
| `FFUDevelopment/FFUUI.Core/FFUUI.Core.Handlers.psm1` | All WPF event handlers |
| `FFUDevelopment/FFUUI.Core/FFUUI.Core.Initialize.psm1` | UI startup and data binding |
| `FFUDevelopment/Apps/Orchestration/Orchestrator.ps1` | In-VM orchestration entry point |
| `FFUDevelopment/config/Sample_default.json` | Complete parameter reference |
| `docs/build.md` | Human-readable parameter documentation |
| `ChangeLog.md` | Full history of what changed and why |

---

## Common Pitfalls

- **Config files are UTF-16 LE with BOM** — do not save as UTF-8 or the script will fail to parse them
- **Paths use Windows backslash** — even in JSON values, paths are Windows-style (`C:\\FFUDevelopment\\...`)
- **Admin elevation required** — scripts will fail silently or with cryptic errors without `#Requires -RunAsAdministrator`
- **Hyper-V module must be loaded** — `#Requires -Modules Hyper-V, Storage` is enforced at script top
- **BuildFFUVM.ps1 is very long (~7,600 lines)** — when reading, use line offsets; search for function names with Grep first
- **The UI and CLI share FFU.Common but not FFUUI.Core** — changes to shared modules affect both surfaces
- **In-VM scripts run in a different execution context** — they cannot call host-side functions; all communication is via shared network file paths or VM integration services
- **`WriteLog` requires `Set-CommonCoreLogPath` first** — calling it before initialization drops messages to `Write-Warning` only
