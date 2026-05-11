# cleanmymac & cleanmypc

> Claude Code skills for **negotiated, case-by-case disk cleanup** on macOS and Windows.

Inspired by DaisyDisk + CleanMyMac (macOS) and CCleaner + WizTree (Windows), but with situational intelligence — Claude reads your actual disk usage and proposes deletions case-by-case instead of running a generic cleanup list. You stay in control of every `rm` / `Remove-Item`.

**Two flavors, same philosophy:**
- 🍎 **`/cleanmymac`** — macOS (Bash, `du`, `df`, `tmutil`)
- 🪟 **`/cleanmypc`** — Windows (PowerShell, `Get-PSDrive`, DISM, `vssadmin`)

## Why this exists

Generic cleaners (CleanMyMac, Mole, etc.) have hardcoded lists of "known junk". They're fast, but they don't know **your** setup — so they either nuke too aggressively or skip the real villains specific to how you use your Mac.

This skill flips that: it inspects your machine first (`du`, `df`, `tmutil`, `pgrep`), shows you a ranked list of what's actually taking space, and confirms every deletion before touching anything.

## What it does

A 5-phase playbook every time you invoke it:

1. **Diagnosis** (read-only) — runs `df -h ~`, `du -sh` on `~/Library/*`, Application Support, Group Containers, Caches, Xcode/simulators, Docker, Time Machine snapshots. No side effects.
2. **Categorization** — pure cache vs user data vs apps to uninstall vs system-settings-only (Photos, Notes).
3. **Proposal** — markdown table with sizes, descriptions, risk evaluation. Asks before deleting.
4. **Execution** — `rm -rf` for pure caches, **Trash-first** for big or risky deletions (rollback for days), full app removal pattern (9 paths), `xcrun simctl delete all` for simulators.
5. **Verification** — `du` and `df` before/after, running total of disk freed.

Real-time progress bar for deletions > 5 GB.

## Install

### macOS

```bash
mkdir -p ~/.claude/skills/cleanmymac
curl -fsSL https://raw.githubusercontent.com/lucasrosati/cleanmymac/main/SKILL.md \
  -o ~/.claude/skills/cleanmymac/SKILL.md
```

### Windows (PowerShell)

```powershell
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.claude\skills\cleanmypc" | Out-Null
Invoke-WebRequest `
  -Uri "https://raw.githubusercontent.com/lucasrosati/cleanmymac/main/skillwindows.md" `
  -OutFile "$env:USERPROFILE\.claude\skills\cleanmypc\SKILL.md"
```

Claude Code picks up skills from `~/.claude/skills/` (macOS) and `%USERPROFILE%\.claude\skills\` (Windows) automatically.

## Use

Inside Claude Code:

```
/cleanmymac   # macOS
/cleanmypc    # Windows
```

Or just describe the problem naturally — the skill auto-triggers on phrases like:

- "meu disco tá cheio"
- "limpar caches do mac" / "limpar o pc"
- "investigar o que ocupa system data"
- "que diabos tá ocupando esses 90 GB?"
- Sharing a screenshot of macOS Storage Settings or Windows Storage Sense

## Catalog of villains it knows how to handle

### macOS (`/cleanmymac`)

- **Browsers** (Chromium-based: Comet, Arc, Chrome, Brave, Edge) — caches Service Worker, GPU, blob storage, while preserving logins/bookmarks/cookies/extensions
- **iOS Simulators** — `xcrun simctl delete all` (canonical, cleans metadata)
- **Xcode** — DerivedData, Archives, iOS DeviceSupport
- **iCloud caches** — CloudKit, FileProvider, CloudDocs (regenerate on demand)
- **Apple Photos & Notes** — instructed via System Settings (`Apple ID → iCloud → toggle off → Delete from Mac`)
- **Time Machine local snapshots** — `tmutil deletelocalsnapshots`
- **Docker.raw** — guidance on shrinking VM image
- **Full app uninstall** — 9-path pattern (`/Applications`, `Group Containers`, `Containers`, `Application Support`, `Caches`, `Preferences`, `WebKit`, `HTTPStorages`, `LaunchAgents`)

### Windows (`/cleanmypc`)

- **Browsers** (same Chromium structure under `%LOCALAPPDATA%\<vendor>\<browser>\User Data\`)
- **Windows Update cache** — stop `wuauserv` → clear `SoftwareDistribution\Download`
- **Component store** (`WinSxS`) — `DISM /Online /Cleanup-Image /StartComponentCleanup /ResetBase`
- **Hibernation file** — `powercfg /h off` (with warning about Fast Startup)
- **Volume Shadow Copies** (System Restore) — `vssadmin delete shadows`
- **`Windows.old`** — Storage Sense or `cleanmgr`
- **WSL VHDX** — `wsl --shutdown` + `Optimize-VHD`
- **Docker Desktop WSL2** — Settings → Resources → Disk image cleanup
- **Visual Studio / NuGet / MSBuild** — DerivedData equivalents (`bin\`, `obj\`, `.vs\`, NuGet global packages)
- **Steam / Epic / Battle.net** — shader caches, downloading folders (not the games)
- **Temp folders** — `%TEMP%`, `C:\Windows\Temp`
- **Memory dumps** — `C:\Windows\Minidump`, `MEMORY.DMP`
- **Full app uninstall** — `winget uninstall` + `%LOCALAPPDATA%`, `%APPDATA%`, `%PROGRAMDATA%` leftover sweep

## What it WON'T touch

Explicitly listed in each skill as **off-limits**:

**macOS:**
- Browser dossiers: `Login Data`, `Bookmarks`, `Cookies`, `Web Data`, `Extensions`, `IndexedDB`, `Local Storage`, `Sessions`, `History`, `Favicons`
- `~/Library/Keychains/` (passwords)
- `~/Library/Mail/`
- `~/Library/Mobile Documents/` (iCloud Drive synced files)
- `~/Library/Biome/` (Siri/intelligence)
- iCloud-synced data without you explicitly disabling sync first

**Windows:**
- Same browser dossiers (under `%LOCALAPPDATA%\<vendor>\<browser>\User Data\Default\`)
- `C:\Windows\System32`, `C:\Windows\WinSxS` (manual deletion forbidden — use DISM)
- `pagefile.sys` (configure via System Properties, never delete manually)
- OneDrive / Google Drive / Dropbox sync folders
- `steamapps\common` (installed games)
- Registry edits without explicit confirmation + `reg export` backup

**Both:**
- Anything you mark as off-limits in Claude memory

## Personalize with absolute rules

The skill respects user-specific rules stored in Claude Code memory. Example: I keep my Flutter SDK at `~/development/flutter/` and don't want it touched in any cleanup. I told Claude once:

> "never delete Flutter SDK or pub-cache, even in disk audits"

Claude saved it as a feedback memory. Every `/cleanmymac` run from now on skips Flutter, even if it shows up in size rankings.

To add yours:

```
remember: never touch ~/some/path
```

## Sample output

```
## Diagnosis — who's eating disk

Volume Data: 135 GB used / 228 GB, 42 GB free.

### Top offenders in ~/Library
| Path | Size | Type | Action |
|---|---:|---|---|
| Application Support/Comet | 5.8 GB | Browser AI | Investigate (cache vs data) |
| Group Containers/group.com.apple.notes | 3.6 GB | iCloud Notes attachments | System Settings → disable sync |
| Developer/CoreSimulator | 4.1 GB | iOS Simulators | Safe to nuke if you test on device |
| Application Support/FileProvider | 2.0 GB | iCloud cache | Safe, regenerates |
| ... |
```

After investigation, propose targeted deletions. After approval, run with progress bar:

```
[██████████████████████████████] 100% — Concluído em 132s | Disco livre: 53Gi
```

## Real-world result (macOS dogfooding)

First session: **17 GB Photos + 3.4 GB browser caches + 7 GB simulators + 1.5 GB Telegram + 4 GB iCloud caches + 3.6 GB Notes attachments = ~36 GB freed** in one negotiated pass, with zero data loss and full transparency about every action.

## License

MIT — see [LICENSE](LICENSE).

## Contributing

Found a villain not in the catalog? PRs welcome. Add it to the "Catálogo de vilões conhecidos" section in `SKILL.md` (macOS) or `skillwindows.md` (Windows) with:
- Path
- What it is
- Whether it's safe to nuke (pure cache) or requires care (user data)
- The deletion pattern

## File layout

```
cleanmymac/
├── SKILL.md           ← macOS skill (Bash)
├── skillwindows.md    ← Windows skill (PowerShell)
├── README.md
└── LICENSE
```
