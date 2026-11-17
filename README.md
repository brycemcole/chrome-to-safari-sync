# Safari Sync

Safari Sync is a local macOS bridge that keeps Safari and Chromium-family browsers closer to the same browsing state. It syncs bookmarks, Favorites, Reading List entries, tab mirrors, tab groups, and history between Safari's local data stores and a Chromium extension.

It was built for people who use a Chromium browser day to day but still rely on Safari and iCloud to make bookmarks and history available on iPhone, iPad, and other Macs. The intent is to make Safari reflect what you do in Chromium without needing Safari open, while still keeping the implementation inspectable and self-hosted.

This is not a cloud service and it is not a Chrome Web Store package. It is an unpacked Manifest V3 extension plus a Python native messaging host.

## Run this example

On macOS with Python 3, Node.js, and Git installed, start with the non-mutating development check:

```sh
git clone https://github.com/brycemcole/chrome-to-safari-sync.git
cd chrome-to-safari-sync
./scripts/check.sh
```

Expect `checks passed`. This checks JavaScript and Python syntax without installing dependencies, registering a native host, or touching your browser data. It is a safe first check, not an end-to-end sync demo.

To actually sync, back up your Safari data first. Open `chrome://extensions` in your Chromium browser, enable **Developer Mode**, choose **Load unpacked**, select this directory, and copy the extension ID. Then run:

```sh
./setup.sh
```

Paste that ID when prompted, reload the extension, and open its popup to check native-host status and use **Sync Now**. Setup registers the native host against this checkout, so keep the directory in place. See [Install](#install) for the full steps. This live path writes Safari data; it is not a sandboxed example.

## Known limitations

- macOS only. The extension must be loaded unpacked and paired with the Python native messaging host; there is no Chrome Web Store installer or cloud service.
- Safari's private plist/database formats are a compatibility dependency, not a supported Apple API. OS changes or filesystem permission restrictions can break access; bookmark backups are not a substitute for a full browser-data backup.
- Open tabs and tab groups become generated Safari bookmark folders, not native Safari tabs, tab groups, or iCloud Tabs. Those generated folders are not synced back as ordinary Chromium bookmarks.
- Safari-to-Chromium history cannot preserve original visit timestamps through Chromium's extension API. That direction also needs the Chromium browser and extension to be active.
- Reading List support depends on the Chromium browser exposing its API. Safari/iCloud controls cross-device propagation; this project cannot promise its timing or delivery.
- The syntax check above does not verify sync correctness, native-host registration, permissions, or iCloud behavior. Test changes against copied Safari files using the [path overrides](#runtime-files).

## Why This Exists

Apple does not provide a public API for third-party browsers to write Safari bookmarks, Reading List, iCloud Tabs, or Safari history. Chromium also does not provide APIs that perfectly preserve external history timestamps when adding visits back into Chromium.

Safari Sync takes the pragmatic local route:

- A Chromium extension observes Chromium bookmarks, history, tabs, tab groups, and Reading List entries.
- A native messaging host reads and writes Safari's local bookmark plist and history database.
- Safari/iCloud then decides when those local Safari changes propagate to other Apple devices.

The project is useful when Safari is your cross-device source of truth, but another Chromium browser is where most browsing actually happens.

## Current Scope

- Safari bookmarks <-> Chromium bookmarks
- Safari Favorites <-> Chromium Bookmarks Bar
- Safari regular bookmarks <-> Chromium Other Bookmarks
- Safari Reading List <-> Chromium Reading List, when the browser exposes the API
- Chromium history -> Safari history with original visit timestamps where Chromium exposes them
- New Safari history visits -> Chromium history while the Chromium browser and extension are active
- Chromium tab groups -> generated Safari folders under `Tab Groups`
- Chromium open tabs -> generated Safari folders under `Open Tabs / <Browser>`

Generated `Open Tabs` and `Tab Groups` folders are intentionally not pushed back into Chromium as normal bookmarks.

## Supported Browsers

The installer knows the native messaging locations for:

- Google Chrome
- Google Chrome Beta
- Google Chrome Canary
- Chromium
- Brave
- Microsoft Edge
- Arc
- Helium

Other Chromium browsers can work if you add their native messaging host directory to `setup.sh`.

## Requirements

- macOS with Safari installed
- Python 3
- A Chromium-based browser with unpacked extension support and native messaging
- Node.js only for development checks

## Install

Clone the repo:

```sh
git clone https://github.com/brycemcole/chrome-to-safari-sync.git
cd chrome-to-safari-sync
```

Load the extension:

1. Open `chrome://extensions` in your Chromium browser.
2. Enable Developer Mode.
3. Click **Load unpacked**.
4. Select this repository directory.
5. Copy the extension ID shown by the browser.

Install the native messaging host:

```sh
./setup.sh
```

Paste the extension ID when prompted. If multiple browsers show different extension IDs for the unpacked extension, paste all IDs separated by spaces or commas.

Reload the extension from `chrome://extensions`.

## Runtime Files

By default, runtime state, logs, and Safari bookmark backups are stored in:

```text
~/Library/Application Support/Safari Sync
```

The native host reads and writes:

```text
~/Library/Safari/Bookmarks.plist
~/Library/Safari/History.db
```

You can override paths for testing:

```sh
SAFARI_SYNC_STATE_DIR=/tmp/safari-sync-state \
SAFARI_BOOKMARKS_PATH=/tmp/Bookmarks.plist \
SAFARI_HISTORY_PATH=/tmp/History.db \
./run.sh
```

## Configuration

The popup shows native-host status, last sync counts, a manual **Sync Now** button, pause/resume, and recent activity.

The options page supports:

- Bidirectional, Chromium -> Safari, or Safari -> Chromium modes
- History sync on/off
- Reading List sync on/off
- Chromium tab group mirroring on/off
- Chromium open tab mirroring on/off
- Custom generated folder names for open tabs and tab groups

Run the setup doctor when sync is not starting:

```sh
./doctor.sh
```

## Safety Model

Safari Sync is intentionally local and transparent, but it does write private browser data stores.

- Bookmark writes create timestamped backups in the runtime backup folder.
- History writes are insert-only and dedupe near-identical visits.
- Safari history rows use Safari's current history generation metadata instead of generation `0`.
- Safari-to-Chromium history cannot preserve original visit times because Chromium's extension API does not expose that capability.
- iCloud propagation is controlled by macOS/iCloud, not this project.

Before changing sync behavior, test against copied Safari files by setting `SAFARI_BOOKMARKS_PATH` and `SAFARI_HISTORY_PATH`.

## Development

Run checks:

```sh
./scripts/check.sh
./doctor.sh
```

Avoid `python3 -m py_compile` in the extension root. It creates `__pycache__`, and Chromium may reject unpacked extensions containing that directory.

Architecture notes live in [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md).

## Privacy

Do not commit runtime files. They can reveal bookmark URLs, open tabs, folder names, and browsing history.

Ignored local files include:

- `state.json`
- `sync.log`
- `backups/`
- Safari plist backups

## Project Intent

The project should stay boring and inspectable:

- Prefer deterministic local sync over cloud dependencies.
- Prefer reversible bookmark writes with backups.
- Treat Safari history writes as a narrow compatibility layer, not a general database migration tool.
- Keep browser-specific behavior isolated so new Chromium-family browsers can be added without changing the sync core.
