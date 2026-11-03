# macperm

A terminal menu for allowing, denying or resetting macOS privacy permissions (microphone, camera, screen recording, Full Disk Access and more) for any app or command-line tool.

It writes straight into the macOS privacy (TCC) databases. That makes it useful when macOS stops showing permission prompts, so apps never appear in **System Settings → Privacy & Security**. It's also faster than clicking through Settings app by app.

> **Warning:** This edits macOS security databases directly and only works with **System Integrity Protection (SIP) disabled**. Only use it on a Mac you control, and only grant permissions to apps you trust.

## Requirements

- macOS (tested on macOS 27.2)
- SIP disabled (`csrutil status` should say `disabled`)
- [fzf](https://github.com/junegunn/fzf) for the menus: `brew install fzf`
- An admin password for permissions stored in the system database (see the table below)

## Install

```sh
git clone https://github.com/Somchandra17/macperm.git
cd macperm
chmod +x macperm
ln -s "$PWD/macperm" /usr/local/bin/macperm   # optional: run it from anywhere
```

## Usage

### Interactive

```sh
macperm
```

1. **Pick the app.** Type to search. The panel on the right shows what the highlighted app is currently allowed. Choose **Other** to enter a path or command name for a command-line tool, or for an app outside the Applications folders.
2. **Pick permissions.** The current setting is shown next to each one.
3. **Pick an action:** Allow, Deny, or Reset (removes the entry so the app asks again).
4. **Confirm.** macperm backs up the database, applies the change, restarts the privacy service, and offers to quit and reopen the app.

Keys in the permission list:

| Key | Action |
|---|---|
| Space or Tab | Mark or unmark the highlighted permission |
| Ctrl-A / Ctrl-D | Mark all / unmark all |
| Enter | Apply to every marked permission |
| Esc | Quit without changing anything |

### Non-interactive

```sh
macperm --app Dia --perm microphone,camera --allow
macperm --app com.hnc.Discord --perm screen-recording --allow --yes
macperm --app /opt/homebrew/bin/some-tool --perm full-disk-access --allow
macperm --app Slack --perm camera --reset
macperm --status Dia
macperm --list
```

`--app` takes an app name, a bundle ID, a path to a `.app`, or a command-line tool (a path, or a command name on your `PATH`).

| Option | Meaning |
|---|---|
| `-a, --app APP` | The app or tool to change |
| `-p, --perm KEYS` | Comma-separated permission keys |
| `--allow` / `--deny` / `--reset` | What to do with them |
| `-n, --dry-run` | Print the database changes without making them |
| `-y, --yes` | Skip the confirmation question |
| `-s, --status APP` | Show an app's current permissions |
| `-l, --list` | List permission keys |
| `--list-apps` | List the apps the menu offers |
| `-h, --help` | Show help |

Any step you leave out on the command line is asked for in a menu.

## Permissions

| Key | Permission | Database |
|---|---|---|
| `microphone` | Microphone | user |
| `camera` | Camera | user |
| `screen-recording` | Screen & System Audio Recording | system (password) |
| `system-audio` | System Audio Recording Only | user |
| `accessibility` | Accessibility | system (password) |
| `full-disk-access` | Full Disk Access | system (password) |
| `input-monitoring` | Input Monitoring | system (password) |
| `developer-tools` | Developer Tools | system (password) |
| `app-management` | App Management | user |
| `bluetooth` | Bluetooth | user |
| `contacts` | Contacts | user |
| `calendar` | Calendars | user |
| `reminders` | Reminders | user |
| `photos` | Photos | user |
| `speech` | Speech Recognition | user |
| `media` | Media & Apple Music | user |
| `desktop` | Desktop Folder | user |
| `documents` | Documents Folder | user |
| `downloads` | Downloads Folder | user |
| `network-volumes` | Network Volumes | user |
| `removable-volumes` | Removable Volumes | user |

## How it works

macOS stores privacy decisions in two SQLite databases:

- **Per-user:** microphone, camera, contacts, folders and so on. On macOS 27 it lives in a protected container under `/private/var/containers/Data/ProtectedSystem/<UUID>/Data/Library/Application Support/com.apple.TCC/TCC.db`, not in `~/Library/Application Support/com.apple.TCC/`. macperm finds it by asking the running per-user `tccd` process which file it has open, and falls back to the older locations.
- **System:** `/Library/Application Support/com.apple.TCC/TCC.db`, root-owned. It holds screen recording, Accessibility, Full Disk Access, Input Monitoring and Developer Tools, so writing there needs `sudo`.

For each change macperm:

1. Backs up the database it's about to change to `~/TCC-backups/`.
2. Reads the app's designated code-signing requirement (`codesign -d -r-`) and compiles it with `csreq`. The entry then only matches the real, correctly signed app, not something pretending to have the same bundle ID.
3. Writes the row: `auth_value` 2 for allowed, 0 for denied, or deletes it for reset.
4. Restarts `tccd` with `launchctl kickstart` so the change takes effect.

Apps usually need to be quit and reopened to pick up a new permission.

## Undoing changes

- Use `--reset` on the app and permission, or choose **Reset** in the menu.
- Or restore a backup from `~/TCC-backups/`:

  ```sh
  /usr/bin/sqlite3 "<database path>" ".restore '<backup file>'"   # add sudo for the system database
  ```

  Then restart your Mac, or run `launchctl kickstart -k gui/$(id -u)/com.apple.tccd`.

## Notes

- If SIP is enabled, macperm refuses to make changes. Use System Settings instead.
- An app without a valid code signature can't be tied to its entry. macperm warns you first, and macOS may ignore such entries.
- Entries for command-line tools are tied to the exact file. A tool installed with Homebrew may stop matching after `brew upgrade`; run macperm again for it.
- macperm calls `/usr/bin/sqlite3` directly, so another `sqlite3` earlier on your `PATH` (for example the Android SDK's) doesn't matter.
