# trash

A safe `rm` replacement for Linux that moves deleted files to a dated trash directory instead of permanently removing them. Transparent — no workflow changes needed.

## Requirements

- Linux (tested on Debian/Ubuntu/Alpine; any distro with bash 4.0+)
- Root access for install/uninstall
- Standard coreutils: `find`, `date`, `du`, `mv`, `cp`, `stat`, `awk`, `sed`, `wc`, `grep`

## Installation

**Quick install (wget):**

```bash
wget https://raw.githubusercontent.com/whonixnetworks/trash-bin/main/trash
chmod +x trash
sudo ./trash --install
```

**Or clone the repo:**

```bash
git clone https://github.com/whonixnetworks/trash-bin.git
cd trash-bin
sudo ./trash --install
```

This will:

- Create `/trash` with sticky-bit permissions (`1777`)
- Back up your existing `rm` binary as `rm.original`
- Replace `rm` with a safe wrapper that moves files to trash instead
- Install helper commands into `/usr/local/bin`
- Set up a daily cron job to auto-purge files older than 7 days

## Uninstallation

```bash
sudo ./trash --uninstall
```

Restores the original `rm` binary and removes all installed scripts. The `/trash` directory itself is **not** deleted — remove it manually if desired:

```bash
sudo bypass-rm -rf /trash
```

## Commands

| Command | Description |
|---------|-------------|
| `rm <file>` | Move file to trash instead of deleting |
| `bypass-rm <file>` | Permanently delete a file (bypass trash) |
| `trash-restore <name>` | Restore a matching file to the current directory |
| `trash-list` | List recently trashed files with size and date |
| `sudo trash-empty` | Permanently delete all trashed files |
| `trash-info` | Show trash status and command reference |

## How it works

When you `rm` a file it is moved to `/trash/YYYY-MM-DD/` under a unique name (`timestamp_random_originalname`). Metadata (original path, deletion time, user) is appended to a `.trash-info` file in the same dated directory. Files on separate filesystems (e.g. tmpfs `/tmp`) are handled with a copy-then-delete fallback so nothing is silently left behind.

A daily cron job removes dated directories older than `MAX_AGE_DAYS`.

## ⚠️ Warnings

Because this wrapper **replaces the `rm` binary system-wide**, _everything_ that calls `rm` is affected — not just your own terminal sessions.

### Disk usage will grow faster than expected

Package managers (`apt`, `pacman`, `dnf`), build systems (`make clean`), and system maintenance scripts all call `rm` internally. Every file they try to delete goes to `/trash` instead. On an active system this can accumulate gigabytes of trash silently, from operations you never directly initiated.

**Mitigations:**
- Keep `MAX_AGE_DAYS` low (the default of `7` is a reasonable ceiling)
- Run `sudo trash-empty` periodically or after major package operations
- Run `trash-list` after `apt upgrade` or similar to see what accumulated
- Monitor `/trash` size with `trash-info`

### Programs using syscalls are NOT intercepted

Any program that deletes files via `unlink()`, `remove()`, or similar C-level syscalls bypasses the wrapper entirely. Only programs that shell out to the `rm` binary are affected. This is expected behaviour but means the trash is not a complete safety net for all deletions.

### Package manager operations may leave ghost files

When you `apt remove` a package, its files are "deleted" to trash rather than truly removed. The package manager reports success, but the files remain on disk until the cron cleanup runs. This is generally harmless but means disk space is not freed immediately.

If you need to reclaim space after a package removal:

```bash
sudo trash-empty
```

## Configuration

Edit these variables at the top of the `trash` script before running `--install`:

| Variable | Default | Description |
|----------|---------|-------------|
| `TRASH_BASE` | `/trash` | Base directory for trash storage |
| `MAX_AGE_DAYS` | `7` | Auto-purge files older than this many days |
| `SIZE_LIMIT_WARNING` | `5GB` | Informational threshold shown in `trash-info` |
| `INSTALL_DIR` | `/usr/local/bin` | Where helper commands are installed |

Configuration is baked into the installed scripts at install time, so re-run `--install` after changing values.

## Notes

- `trash-restore` restores to your **current working directory**. The original path is shown so you can move it afterwards if needed. You will be prompted before overwriting an existing file.
- `sudo trash-empty` requires root because `/trash` uses sticky-bit permissions — only root can safely purge entries from all users.
- `bypass-rm` calls the backed-up original `rm` binary directly, so it permanently deletes without any wrapper overhead.
- Files with identical names never collide in trash — the `timestamp_randomhex_` prefix guarantees uniqueness.
- The `rm` wrapper silently accepts `-r`, `-f`, `-i` and similar flags (trashing is always safe regardless of flags), and propagates a non-zero exit code if any file could not be moved.

---

## License

MIT License - see [LICENSE](LICENSE) for details.

