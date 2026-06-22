
# Obsidian Portable

## Table of Contents

- [Part 0: Quick Start (Reconstruction Steps for Clone)](#part-0-quick-start-reconstruction-steps-for-clone)
- [Architecture](#understanding-the-architecture)
- [Part 1: Building a Fresh Portable from Scratch](#part-1-building-a-fresh-portable-from-scratch)
- [Part 2: Upgrading an Existing Portable Instance](#part-2-upgrading-an-existing-portable-instance)
- [Part 3: Cloning Portable Instances](#part-3-cloning-portable-instances)
- [Troubleshooting](#troubleshooting)

---

## Part 0: Quick Start (Reconstruction Steps for Clone)

To run a clone of `https://github.com/sato-note/ObsidianPortable_1.12.7`:

1.  **Clone the repository:**
    ```powershell
    git clone https://github.com/sato-note/ObsidianPortable_1.12.7.git
    cd ObsidianPortable_1.12.7
    ```
2.  **Download installer:**
    *   URL: `https://github.com/obsidianmd/obsidian-releases/releases/download/v1.12.7/Obsidian-1.12.7.exe`
3.  **Extract executable:**
    *   Command: `7z x -t# -aoa Obsidian-1.12.7.exe 4.7z` (Extracts AMD64 resource container)
    *   Command: `7z e -aoa 4.7z Obsidian.exe` (Extracts `Obsidian.exe`)
4.  **Place executable:**
    *   Move `Obsidian.exe` to `App\Obsidian\Obsidian.exe`

---

## Architecture

### What is Obsidian Portable?

Obsidian Portable is a self-contained version of Obsidian that keeps all its data (vaults, settings, plugins, cache) in the same folder as the application. This allows you to:

- Run multiple isolated Obsidian instances side-by-side
- Switch between them with hotkeys
- Carry your setup on a USB drive
- Avoid conflicts between different vault configurations

### Directory Structure

A portable Obsidian instance has this structure:

```
ObsidianPortable/
├── ObsidianPortable.exe          # PortableApps launcher (entry point)
├── ObsidianPortable.ini          # Launcher settings (optional)
├── portable.txt                  # Marker file (can be empty)
├── Updater.bat                   # Auto-updater script
├── LICENSE
│
├── App/                          # Application binaries (REPLACEABLE)
│   ├── Obsidian/                 # Electron shell + Obsidian app (Layout A)
│   │   ├── Obsidian.exe          # Main executable (Electron shell)
│   │   ├── resources/
│   │   │   ├── obsidian.asar     # Obsidian app logic (versioned)
│   │   │   ├── app.asar          # Electron bootstrap
│   │   │   └── app.asar.unpacked/
│   │   ├── locales/              # Language packs
│   │   ├── *.dll                 # Chromium/Electron dependencies
│   │   ├── *.pak                 # Chromium resource packs
│   │   └── *.bin                 # V8 snapshots
│   │
│   ├── AppInfo/                  # PortableApps metadata
│   │   ├── AppInfo.ini           # Version info, app details
│   │   ├── AppIcon.ico           # App icon
│   │   └── Launcher/
│   │       └── ObsidianPortable.ini  # Launch configuration
│   │
│   └── Utils/                    # Helper tools
│       ├── 7za.exe               # 7-Zip for Updater.bat
│       ├── busybox.exe           # Unix tools for Updater.bat
│       ├── curl.exe              # HTTP client for Updater.bat
│       └── curl-ca-bundle.crt    # SSL certificates
│
└── Data/                         # User data (NEVER REPLACE)
    ├── Profile/                  # User data directory (passed via --user-data-dir)
    │   ├── obsidian.json         # Vault registry
    │   ├── obsidian-1.12.7.asar  # Auto-downloaded app update
    │   ├── IndexedDB/            # Vault databases
    │   ├── Local Storage/        # Browser storage
    │   ├── Cache/                # App cache
    │   └── ...                   # Other Electron profile data
    ├── settings/                 # PortableApps settings
    └── Temp/                     # Temporary files
```

### Key Insight: Two Separate Version Layers

Obsidian has **two versioned components** that update independently:

| Component | Location | What It Is |
|-----------|----------|------------|
| **Electron Shell** | `App/Obsidian/Obsidian.exe` | The Chromium/Electron runtime. Contains the browser engine, GPU rendering, etc. This is what the installer distributes. |
| **Obsidian App** | `Data/Profile/obsidian-X.Y.Z.asar` | The actual Obsidian application logic (editor, plugins, themes). Obsidian auto-downloads this on launch. |

When Obsidian launches, it checks `Data/Profile/` for the latest `obsidian-X.Y.Z.asar` and loads it instead of the bundled `App/Obsidian/resources/obsidian.asar`.

**The extension that required "latest installer version" was checking the Electron shell version, not the app version.** That's why replacing only `App/Obsidian/` fixed it.

### Why Minimal Changes Work

When upgrading, you only need to replace the **Electron shell** (`App/Obsidian/`). Everything else stays:

- `Data/Profile/` — your vaults, plugins, settings, cache (untouched)
- `App/AppInfo/` — portable launcher config (untouched)
- `App/Utils/` — updater tools (untouched)
- `ObsidianPortable.exe` — the launcher itself (untouched)
- Custom icons, configs, scripts (untouched)

---

## Part 1: Building a Fresh Portable from Scratch

Use this when you want a brand new portable instance with no existing data.

### Prerequisites

- **7-Zip** installed (default: `C:\Program Files\7-Zip\7z.exe`)
- **PowerShell 7+** (or Windows PowerShell 5.1)
- Internet connection
- An existing portable instance to copy the launcher from (e.g., `ObsidianPortable_1.11.5`)

### Step 1: Find the Latest Release

```powershell
# Check latest releases on GitHub
(Invoke-RestMethod -Uri "https://api.github.com/repos/obsidianmd/obsidian-releases/releases").tag_name | Select-Object -First 5
```

### Step 2: Download the Installer

```powershell
$version = "1.12.7"  # Change to your target version
$url = "https://github.com/obsidianmd/obsidian-releases/releases/download/v$version/Obsidian-$version.exe"
$outFile = "$env:TEMP\Obsidian-$version.exe"

Invoke-WebRequest -Uri $url -OutFile $outFile -UseBasicParsing
```

### Step 3: Extract the Installer (Two Levels Deep)

The installer is a nested archive. Extract using the PE resource container mode (`-t#`) or by extracting `$PLUGINSDIR\app-64.7z`.

Using resource ID method (matches Updater.bat logic):
```powershell
$extractDir = "$env:TEMP\obsidian-extract"
$appDir = "$env:TEMP\obsidian-app"

# Level 1: Extract resource container (4.7z is the 64-bit app archive)
New-Item -ItemType Directory -Path $extractDir -Force
& "C:\Program Files\7-Zip\7z.exe" x $outFile -o"$extractDir" -t# 4.7z -y

# Level 2: Extract the app files
New-Item -ItemType Directory -Path $appDir -Force
& "C:\Program Files\7-Zip\7z.exe" x "$extractDir\4.7z" -o"$appDir" -y
```

### Step 4: Create the Portable Directory Structure

```powershell
$dest = "C:\obs\ObsidianPortable_$version"

# Create directories
New-Item -ItemType Directory -Path "$dest\App\Obsidian" -Force
New-Item -ItemType Directory -Path "$dest\Data\Profile" -Force
New-Item -ItemType Directory -Path "$dest\Data\settings" -Force
New-Item -ItemType Directory -Path "$dest\Data\Temp" -Force
```

### Step 5: Copy Extracted App Files

```powershell
# Copy all Electron/Obsidian files to App/Obsidian/
Copy-Item -Path "$appDir\*" -Destination "$dest\App\Obsidian\" -Recurse -Force
```

### Step 6: Copy Portable Launcher and Metadata

Copy these from an existing portable instance (e.g., `ObsidianPortable_1.11.5`):

```powershell
$src = "C:\obs\ObsidianPortable_1.11.5"

# Copy AppInfo (launcher config, icons, metadata)
Copy-Item -Path "$src\App\AppInfo" -Destination "$dest\App\AppInfo" -Recurse -Force

# Copy the PortableApps launcher executable
Copy-Item -Path "$src\ObsidianPortable.exe" -Destination "$dest\ObsidianPortable.exe" -Force

# Copy LICENSE
Copy-Item -Path "$src\LICENSE" -Destination "$dest\LICENSE" -Force
```

### Step 7: Update Version in AppInfo.ini

Edit `App/AppInfo/appinfo.ini` and update the version numbers:

```ini
[Version]
PackageVersion=1.12.7.0
DisplayVersion=1.12.7
```

### Step 8: Verify the Launcher Config

Check `App/AppInfo/Launcher/ObsidianPortable.ini` to ensure it is configured for Layout A:

```ini
[Launch]
ProgramExecutable="Obsidian\Obsidian.exe"
CommandLineArguments=--user-data-dir="%PAL:DataDir%\Profile"
DirectoryMoveOK=yes
SupportsUNC=yes
```

The `--user-data-dir` flag tells Electron to store all user data in `Data/Profile/` instead of `%APPDATA%`.

### Step 9: Clean Up Temp Files

```powershell
Remove-Item "$env:TEMP\obsidian-extract" -Recurse -Force
Remove-Item "$env:TEMP\obsidian-app" -Recurse -Force
Remove-Item $outFile -Force
```

### Step 10: Launch

Run `ObsidianPortable.exe`. On first launch, Obsidian will:
1. Create profile data in `Data/Profile/`
2. Prompt you to create or open a vault
3. Auto-download the latest `obsidian-X.Y.Z.asar` into the profile

---

## Part 2: Upgrading an Existing Portable Instance

Use this when you have a working portable instance and want to update the Electron shell to a new version. **This is the minimal-change approach.**

### What Gets Replaced

| Path | Action | Why |
|------|--------|-----|
| `App/Obsidian/` | **DELETE and REPLACE** | Electron shell (the only thing that needs updating) |
| `App/AppInfo/AppInfo.ini` | **EDIT version** | Update displayed version number |

### What Stays Untouched

| Path | Why It's Safe |
|------|---------------|
| `Data/Profile/` | All vaults, plugins, settings, cache, auto-downloaded asar |
| `Data/settings/` | PortableApps settings |
| `App/AppInfo/Launcher/` | Launch configuration (unchanged between versions) |
| `App/AppInfo/AppIcon.ico` | Your custom icon |
| `App/Utils/` | Updater tools |
| `ObsidianPortable.exe` | Launcher binary |
| `ObsidianPortable.ini` | Your custom launch parameters |
| `*.ico`, `*.bat`, `*.txt` | Any custom files at root |

### Step 1: Backup

**Always backup before upgrading.**

```powershell
robocopy "C:\obs\obs-00" "C:\obs\obs-00-backup" /E /COPYALL /R:1 /W:1 /NFL /NDL /NP
```

### Step 2: Get the New Electron Shell

Either:
- **Option A:** Use an already-built portable (from Part 1) as the source:
  ```powershell
  $source = "C:\obs\ObsidianPortable_1.12.7\App\Obsidian"
  ```
- **Option B:** Download and extract fresh (Steps 1-3 from Part 1, using `$appDir` as source)

### Step 3: Replace App/Obsidian/

```powershell
$target = "C:\obs\obs-00"

# Remove old Electron shell
Remove-Item "$target\App\Obsidian" -Recurse -Force

# Copy new Electron shell
Copy-Item -Path "$source" -Destination "$target\App\Obsidian" -Recurse -Force
```

### Step 4: Update Version in AppInfo.ini

```powershell
(Get-Content "$target\App\AppInfo\AppInfo.ini") -replace 'DisplayVersion=.*', 'DisplayVersion=1.12.7' |
  Set-Content "$target\App\AppInfo\AppInfo.ini"
```

### Step 5: Verify

```powershell
# Check the new Obsidian.exe version
(Get-Item "$target\App\Obsidian\Obsidian.exe").VersionInfo.ProductVersion
# Expected: 1.12.7.0

# Check Data/Profile is still intact
Get-ChildItem "$target\Data\Profile" | Select-Object Name

# Check launcher config matches Layout A
Get-Content "$target\App\AppInfo\Launcher\ObsidianPortable.ini"
```

### Step 6: Launch

Run `ObsidianPortable.exe`. It should launch with the new Electron shell while loading all your existing vaults and settings from `Data/Profile/`.

---

## Part 3: Cloning Portable Instances

To create a new isolated instance from an existing one:

```powershell
# Full clone (includes all data)
robocopy "C:\obs\obs-00" "C:\obs\obs-new" /E /COPYALL /R:1 /W:1

# Or clone without data (fresh start, same config)
robocopy "C:\obs\obs-00" "C:\obs\obs-new" /E /XD "Data\Profile" /COPYALL /R:1 /W:1
```

Each clone is fully independent. Change the vault, plugins, or settings in one without affecting others.

---

## Troubleshooting

### "Extension requires latest installer version"

This error checks the **Electron shell version** (`Obsidian.exe`), not the Obsidian app version. Follow Part 2 to replace `App/Obsidian/`.

### Obsidian auto-downloads a different asar version

Normal behavior. Obsidian checks for updates on launch and downloads `obsidian-X.Y.Z.asar` into `Data/Profile/`. The Electron shell version and the app asar version can differ.

### Launcher doesn't find Obsidian.exe

Check `App/AppInfo/Launcher/ObsidianPortable.ini`:

```ini
ProgramExecutable="Obsidian\Obsidian.exe"
```

If it fails, confirm `Obsidian.exe` exists under `App\Obsidian\`.

### Directory Layouts

Ensure you follow **Layout A (Nested)**:
```
App/
├── Obsidian/        # Electron files nested here
│   ├── Obsidian.exe
│   └── resources/
└── AppInfo/
```
Launcher: `ProgramExecutable="Obsidian\Obsidian.exe"`


### Restoring from backup

```powershell
# Stop Obsidian first
Remove-Item "C:\obs\obs-00" -Recurse -Force
robocopy "C:\obs\obs-00-backup" "C:\obs\obs-00" /E /COPYALL /R:1 /W:1 /NFL /NDL /NP
```

### The Updater.bat script

The included `Updater.bat` auto-updates the Electron shell by:
1. Checking GitHub for the latest release.
2. Downloading the installer to `TMP`.
3. Extracting the nested resource (`4.7z` or `6.7z` depending on processor architecture) using 7-Zip (`-t#` mode).
4. Deleting and replacing `App/Obsidian/`.
5. Appending version info to `App/AppInfo/AppInfo.ini`.

Requires tools in `App/Utils/` (7za.exe, busybox.exe, curl.exe).

---

## Quick Reference
| Task | Command |
|------|---------|
| Check current version | `(Get-Item "App\Obsidian\Obsidian.exe").VersionInfo.ProductVersion` |
| Check latest release | `(Invoke-RestMethod -Uri "https://api.github.com/repos/obsidianmd/obsidian-releases/releases").tag_name \| Select -First 1` |
| Download installer | `Invoke-WebRequest -Uri "https://github.com/obsidianmd/obsidian-releases/releases/download/v{VER}/Obsidian-{VER}.exe" -OutFile "$env:TEMP\obs.exe"` |
| Extract level 1 | `7z x -t# -aoa obs.exe -o"$env:TEMP\ext1" {ARCH} -y` |
| Extract level 2 | `7z x -aoa "$env:TEMP\ext1\{ARCH}" -o"App\Obsidian" -y` |
| Backup instance | `robocopy "C:\obs\obs-XX" "C:\obs\obs-XX-backup" /E /COPYALL` |
| Replace shell | Delete `App/Obsidian/`, copy new files in |
| Update version | Edit `App/AppInfo/AppInfo.ini` → `DisplayVersion=X.Y.Z` |
