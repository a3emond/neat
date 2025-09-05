# Neat – User-Space Cleanup Utility for macOS
Neat is a multi-layer macOS assistant (CLI + GUI) for managing user-created clutter. It removes app leftovers, oversized caches, old downloads, broken links, and developer build artifacts, with safe profiles, scheduling, and full transparency.


---

## Project Goals

- Provide safe, reversible cleanup of files left behind by apps and user activity.  
- Focus only on user-generated cruft (not system internals) to avoid conflicts with macOS periodic jobs.  
- Offer a simple one-click experience for casual users, with extensive settings for advanced users.  
- Support scheduled cleanups with flexible profiles (daily, weekly, monthly).  
- Be fully transparent and auditable (dry-run, reports, config).  

---

## Architecture

The project is structured into three layers:

### 1. Core (Swift Package)
Contains all scanning, planning, execution, and reporting logic.  
Exposes APIs for both CLI and GUI.

Components:
- Scanners – detect leftovers, caches, old downloads, broken links, duplicates, etc.
- Planner – applies config and profiles to decide actions.
- Executor – performs actions (Trash, archive, delete).
- Reporter – generates human-readable or JSON output.
- Config – loads settings from YAML/JSON and user overrides.
- Store – persists last scan results and logs.

### 2. CLI (SwiftPM target)
Command-line interface for scripting and automation.

Main commands:
- `scan` – find issues without modifying anything.
- `clean` – perform cleanup with chosen profile.
- `report` – generate reports (human or JSON).
- `schedule` – create or remove launchd jobs.
- `profiles` – list available cleanup profiles.
- `config` – show config paths, reload config.

### 3. GUI (SwiftUI App)
User-friendly graphical frontend.

Features:
- Dashboard showing reclaimable space and categories.
- Preferences panel for profiles, categories, thresholds, and exclusions.
- Scheduling interface for daily, weekly, monthly jobs.
- Advanced tab that mirrors CLI flags and exclusions.
- Restore tab for undoing recent cleanups (from Trash).

---

## Cleanup Categories

Cleanr focuses only on **user-space clutter**.  
They are grouped by type to balance safety and usefulness.

### Core Categories (default)
1. **App leftovers**  
   Orphaned folders in `~/Library/Preferences`, `~/Library/Application Support`, `~/Library/Caches`, `~/Library/Logs`, `~/Library/Saved Application State`.

2. **Oversized caches**  
   Per-app caches exceeding thresholds (configurable).

3. **Old Downloads and Desktop clutter**  
   Installers (`.dmg`, `.pkg`, `.zip`), archives, and media untouched for a configurable number of days.

4. **Large forgotten files**  
   Files above a size threshold (e.g., 500 MB) not opened for a configurable period.

5. **User logs and crash dumps**  
   User-owned logs in `~/Library/Logs` and crash dumps in `~/Library/DiagnosticReports`.

---

### Optional Categories
6. **Duplicate files**  
   Hash-based detection of duplicates in user folders.

7. **Old applications**  
   Applications not launched for a configurable period, detected via Spotlight metadata.

8. **Unused links**  
   Broken aliases and symlinks, plus stale Finder sidebar entries.

9. **Organizing functions**  
   - Sort Downloads into subfolders by type (installers, media, archives).  
   - Move old Desktop files into `Desktop/Old/<YYYY-MM>/`.  
   - Archive installers after use.  

10. **Stale application preferences**  
    Detect `.plist` files for apps no longer installed.

11. **Old backups and archives**  
    Detect duplicate or outdated `.bak`, `.old`, `.copy`, and redundant zip archives.

12. **Screenshots and screen recordings**  
    Detect screenshots older than N days on Desktop and move them to an archive folder.

13. **Unused fonts**  
    Detect fonts in `~/Library/Fonts` not accessed in a configurable period.

14. **Trash monitor**  
    Show Trash size and optionally purge items older than N days.

15. **Inactive LaunchAgents**  
    Detect `.plist` files in `~/Library/LaunchAgents` referencing missing apps.

---

### Advanced / Developer Categories
16. **Developer build caches**  
    - Xcode: `~/Library/Developer/Xcode/DerivedData`  
    - Gradle: `~/.gradle`  
    - Maven: `~/.m2`  
    - NuGet: `~/.nuget`  
    Detect build folders older than N days.

17. **Package manager leftovers**  
    - Homebrew: unused Cellar versions.  
    - Node.js: global `node_modules` with no linked projects.  
    - Python: `__pycache__` folders in project directories.

18. **Virtual machine and container images**  
    Detect large VM images (Parallels, VMware, UTM) and Docker volumes not used recently.

19. **Old log archives in dev tools**  
    Detect logs and crash dumps from simulators and test runs.

20. **Mail attachments (optional)**  
    Large cached attachments in `~/Library/Mail`, older than N days. Disabled by default.

---

## Safety Model

- Default action: move files to Trash.  
- Supports archive (move to `~/Archives/Cleanr`) or permanent delete for advanced users.  
- Dry-run mode shows findings without action.  
- Every action includes a reason string (e.g., "Orphaned application support folder").  
- Supports exclusion lists for paths, bundle identifiers, and app names.  
- Restore tab in GUI allows easy undo for recent actions.

---

## Configuration

Configuration is layered in the following order:
1. Built-in defaults.  
2. System config: `/Library/Application Support/Cleanr/config.yml`  
3. User config: `~/Library/Application Support/Cleanr/config.yml`  
4. CLI flags or GUI overrides.  

### Example config.yml

```yaml
profiles:
  daily:
    run: [leftovers, caches, screenshots, trash_monitor]
    rules:
      caches:
        minSizeMB: 50
        ignore: ["Xcode", "Docker Desktop"]

  weekly:
    run: [leftovers, caches, old_downloads, logs, unused_links, organizing]
    rules:
      old_downloads:
        minAgeDays: 60
        includeExtensions: ["dmg", "iso", "zip"]

  monthly:
    run: [leftovers, caches, old_downloads, big_files, duplicates, stale_prefs, organizing, dev_caches]
    rules:
      big_files:
        minSizeMB: 500
        minAgeDays: 180
      dev_caches:
        minAgeDays: 90

excludes:
  paths:
    - "~/Library/Application Support/Adobe"
    - "~/Library/Application Support/Microsoft"
  bundle_ids:
    - "com.apple.*"

actions:
  default: trash   # trash | archive | delete
  archiveDir: "~/Archives/Cleanr"

output:
  format: human    # human | json
```

---

## Scheduling

Scheduled cleanups are managed via launchd agents.  
Each profile can be scheduled independently:

- `~/Library/LaunchAgents/pro.aedev.cleanr.daily.plist`  
- `~/Library/LaunchAgents/pro.aedev.cleanr.weekly.plist`  
- `~/Library/LaunchAgents/pro.aedev.cleanr.monthly.plist`

The CLI (`cleanr schedule set/remove`) generates and manages these automatically.

---

## Example CLI Usage

```bash
# Scan leftovers and caches (dry-run)
cleanr scan --profile daily

# Perform weekly cleanup, with confirmation
cleanr clean --profile weekly

# Perform monthly cleanup without confirmation
cleanr clean --profile monthly --yes

# Scan for developer build caches older than 90 days
cleanr scan --category dev_caches

# Organize Downloads into subfolders
cleanr clean --category organizing

# List all profiles
cleanr profiles list

# Generate a JSON report for automation
cleanr report --json --profile monthly
```

---

## Development Roadmap

1. Scaffold SwiftPM workspace with Core, CLI, and GUI targets.  
2. Implement Leftovers scanner and dry-run reporting.  
3. Add Profiles, Config loader, and Executor.  
4. Implement scheduling with launchd integration.  
5. Extend to additional scanners (caches, downloads, large files, logs).  
6. Add organizing functions and unused link detection.  
7. Build GUI dashboard with scan and clean functionality.  
8. Add Preferences panel, scheduling, and restore tab.  
9. Implement developer-specific cleanups (Xcode, Docker, Homebrew).  
10. Extend with optional advanced features: duplicates, mail attachments, unused fonts.  

---

## License

To be decided.
