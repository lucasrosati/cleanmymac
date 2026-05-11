# cleanmymac

> Claude Code skill for **negotiated, case-by-case macOS disk cleanup**.

Inspired by DaisyDisk + CleanMyMac, but with situational intelligence — Claude reads your actual disk usage and proposes deletions case-by-case instead of running a generic cleanup list. You stay in control of every `rm`.

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

```bash
mkdir -p ~/.claude/skills/cleanmymac
curl -fsSL https://raw.githubusercontent.com/lucasrosati/cleanmymac/main/SKILL.md \
  -o ~/.claude/skills/cleanmymac/SKILL.md
```

That's it. Claude Code picks up skills from `~/.claude/skills/` automatically.

## Use

Inside Claude Code:

```
/cleanmymac
```

Or just describe the problem naturally — the skill auto-triggers on phrases like:

- "meu disco tá cheio"
- "limpar caches do mac"
- "investigar o que ocupa system data"
- "que diabos tá ocupando esses 90 GB?"
- Sharing a screenshot of macOS Storage Settings

## Catalog of villains it knows how to handle

- **Browsers** (Chromium-based: Comet, Arc, Chrome, Brave, Edge) — caches Service Worker, GPU, blob storage, while preserving logins/bookmarks/cookies/extensions
- **iOS Simulators** — `xcrun simctl delete all` (canonical, cleans metadata)
- **Xcode** — DerivedData, Archives, iOS DeviceSupport
- **iCloud caches** — CloudKit, FileProvider, CloudDocs (regenerate on demand)
- **Apple Photos & Notes** — instructed via System Settings (`Apple ID → iCloud → toggle off → Delete from Mac`)
- **Time Machine local snapshots** — `tmutil deletelocalsnapshots`
- **Docker.raw** — guidance on shrinking VM image
- **Full app uninstall** — 9-path pattern (`/Applications`, `Group Containers`, `Containers`, `Application Support`, `Caches`, `Preferences`, `WebKit`, `HTTPStorages`, `LaunchAgents`)

## What it WON'T touch

Explicitly listed in the skill as **off-limits**:

- Browser dossiers: `Login Data`, `Bookmarks`, `Cookies`, `Web Data`, `Extensions`, `IndexedDB`, `Local Storage`, `Sessions`, `History`, `Favicons`
- `~/Library/Keychains/` (passwords)
- `~/Library/Mail/`
- `~/Library/Mobile Documents/` (iCloud Drive synced files)
- `~/Library/Biome/` (Siri/intelligence)
- iCloud-synced data without you explicitly disabling sync first
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

## Real-world result

First session: **17 GB Photos + 3.4 GB browser caches + 7 GB simulators + 1.5 GB Telegram + 4 GB iCloud caches + 3.6 GB Notes attachments = ~36 GB freed** in one negotiated pass, with zero data loss and full transparency about every action.

## License

MIT — see [LICENSE](LICENSE).

## Contributing

Found a villain not in the catalog? PRs welcome. Add it to the "Catálogo de vilões conhecidos" section in `SKILL.md` with:
- Path
- What it is
- Whether it's safe to nuke (pure cache) or requires care (user data)
- The deletion pattern
