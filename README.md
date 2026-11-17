# Frida Manager for iOS

A native iOS app for installing, managing, and launching
[Frida](https://frida.re/) server on **jailbroken** iPhones and iPads — the iOS
counterpart to [Android Frida Manager](https://github.com/abhilashnigam/Android-Frida-Manager).

It's for security researchers, mobile app testers, and reverse engineers who
want to manage `frida-server` from the device itself instead of over a cable.

---

## What it does

Working with Frida on a jailbroken iOS device usually means a repetitive loop:
find the right release, match it to your bootstrap's architecture, install it,
set the listen address, start/stop it, and repeat on every update. Frida Manager
does all of that from a native app — and controls `frida-server` **no matter how
it was installed**, whether by this app or from a repo via **Sileo** / **Zebra**.

### Features

**Server control**
- Start, stop, and see live status (whether frida is actually running).
- Starts frida as a **detached daemon in its own session**, so it keeps running
  after you close the app.
- Shows the installed `frida-server` version and flags when an update exists.
- Controls installs from **any source**: a package's `launchd` daemon
  (`re.frida.server`) is driven with `launchctl`; an app-started server is a
  tracked process (its PID is recorded for reliable status/stop).

**Take over a package daemon**
- If frida was installed from a repo, the app offers to **take over**: it unloads
  the `launchd` daemon and runs the binary directly under app control, so you get
  start/stop and address/port control. "Return control" restores the daemon.

**Versions**
- Lists `frida-server` releases from GitHub (with an Atom-feed fallback when the
  API is rate-limited).
- **Install** downloads the correct iOS **`.deb`** for your device and installs
  it with `dpkg` (frida ships the iOS server only as a package, not a raw binary).
- The installed version's row has an **Uninstall** button (`dpkg -r`).

**Network**
- Pick the listen address from a **dropdown of the device's interfaces**, plus
  `0.0.0.0` (all) and **USB only**.
- **USB only** runs frida with no `-l`, binding its default `localhost:27042`
  reachable over usbmux — not exposed to the network (the port field is disabled).
- For a package `launchd` daemon, saving edits the daemon's plist as root
  (backed up first) and reloads it.

**Other**
- **Diagnostics** panel: confirms the privileged shell works (uid 0), shows the
  detected jailbreak/arch, and a full on-device detection report.
- Optional **path anonymiser** for app-owned installs (random install path).
- In-app **logs** (refresh / clear / share), and light / dark / system theme.

---

## How it gets root (standalone app, no Settings bundle)

A SpringBoard-launched app runs as `mobile`, and `setuid(0)` from inside it fails
on a jailbreak — so the app can't control root-owned frida on its own. The package
installs a tiny **setuid-root helper** (`frida-manager-root`, mode `4755`, signed
as a platform binary). The app execs it to run privileged commands as root, and
uses its `--daemon` mode to launch frida detached. This keeps the project a plain
app — no preference bundle, no always-running daemon of its own.

> Because of this, the app **must be installed as a package** (`dpkg` / Sileo /
> Zebra). A plain side-load does not get the helper's setuid bit set by `root`, so
> privileged operations won't work.

Paths are resolved through the detected jailbreak prefix, so the same logic works
rootless (`/var/jb`) and rootful (`/`).

---

## Support

| Jailbreak layout | Examples                             | Prefix     |
|------------------|--------------------------------------|------------|
| Rootless         | Dopamine, palera1n rootless          | `/var/jb`  |
| Rootful          | palera1n rootful, checkra1n, unc0ver | `/` (root) |

- Minimum **iOS 15.0**.
- The install architecture is the **dpkg** architecture (`dpkg
  --print-architecture`): `iphoneos-arm64` on Procursus (rootless / newer
  rootful), `iphoneos-arm` on the legacy rootful bootstrap. Detected automatically.

---

## Install

### Add the repo (recommended)

Add the repo to your package manager, then install **Frida Manager** from it —
dependencies and the right architecture are resolved automatically:

- **Sileo:** [add this source](sileo://source/https://0x27042.abhilashnigam.nl/)
- **Zebra:** [add this source](zbra://sources/add/https://0x27042.abhilashnigam.nl/)
- **Any manager — add this URL manually:**
  `https://0x27042.abhilashnigam.nl/`

(The tap-to-add links only work on the device; the landing page at the URL above
has the buttons too.)

### Or download a `.deb` directly

Grab the matching package from the
[Releases page](https://github.com/abhilashnigam/iOS-Frida-Manager/releases/latest)
and install it with Sileo/Zebra or `dpkg -i`.

Install the variant that matches your device:

| Jailbreak | Scheme | dpkg arch |
|-----------|--------|-----------|
| Dopamine, palera1n rootless | rootless | `iphoneos-arm64` |
| palera1n rootful, newer Procursus rootful | rootful | `iphoneos-arm64` |
| checkra1n / unc0ver (legacy Bingner bootstrap) | rootful | `iphoneos-arm` |

If `dpkg -i` reports `package architecture (iphoneos-arm64) does not match system
(iphoneos-arm)`, use the `iphoneos-arm` build (same app — the arch field is just
the bootstrap's label).

Then open **Frida Manager**, go to **Versions**, install a `frida-server`
release, and use the **Server** tab to start it.

---

## Build from source

The source is the truth; pre-built packages are a convenience. No build system beyond Xcode is required —
just Xcode's iPhoneOS SDK and `ldid` + `dpkg` on the Mac:

```bash
brew install ldid dpkg
```

**Build every variant** (`{rootful,rootless}` × `{iphoneos-arm,iphoneos-arm64}`):

```bash
./scripts/build-all.sh        # -> packages/*.deb
```

Or one at a time:

```bash
./scripts/build-deb.sh rootful  iphoneos-arm      # legacy rootful bootstrap
./scripts/build-deb.sh rootful  iphoneos-arm64    # Procursus rootful
./scripts/build-deb.sh rootless iphoneos-arm64    # rootless jailbreaks
```

(`build-deb.sh` uses `dpkg-deb` when present and otherwise builds the `.deb` by
hand with `ar`/`tar`, so it works even without `dpkg`.)

**Install onto a USB-connected device** (tunnels over usbmux, installs with
`dpkg`, and resolves deps with `apt`):

```bash
SSHPASS=<root-password> ./scripts/install-deb.sh \
  packages/com.abhilashnigam.fridamanager_1.0.0_iphoneos-arm_rootful.deb
```

**Cut a release (publishes the Sileo/Zebra repo to GitHub Pages):**

The app is built on your Mac; a Gitea workflow then assembles the apt repo from
the pre-built `.debs` and pushes it to GitHub Pages — no macOS runner needed.

```bash
./scripts/stage-release.sh 1.0.0   # builds all .debs into dist/
git add dist && git commit -m "release 1.0.0" && git push
```

On that push, `.gitea/workflows/publish.yml` (on your Gitea `portfolio` ubuntu
runner) assembles the apt repo (`Packages`/`Release`/`debs/`/`index.html`) from
the committed `.debs` and force-pushes it to the GitHub repo's **`main`** branch.
It only packages pre-built `.debs`, so no macOS/Xcode is needed there.

Set GitHub → Settings → Pages → **Deploy from branch → `main` → `/ (root)`**,
add the custom domain `0x27042.abhilashnigam.nl` (with a matching DNS CNAME to
`abhilashnigam.github.io`), and the source is served at
`https://0x27042.abhilashnigam.nl/`.

To assemble the repo locally instead (any static host):

```bash
./scripts/build-repo.sh       # assembles ./repo (Packages, Release, debs/)
```

**Quick UI smoke-test (no package install):** `./scripts/compile-standalone.sh`
builds `build/Payload/FridaManager.app`, and `./deploy.sh` side-loads it. Note
that a side-load can't set the helper setuid, so privileged features won't work —
use the `.deb` for a real install.

---

## Layout

```
FridaManager/
  Core/       Jailbreak detection, setuid-aware shell runner, GitHub API,
              server + install control, settings, network interfaces,
              the observable repository
  Models/     Release / asset model
  UI/         SwiftUI screens: Server, Versions, Logs, Settings
  Resources/  Info.plist, entitlements.plist, app icons
helper/
  fmroot.c            Setuid-root helper (run-as-root + --daemon modes)
  entitlements.plist  platform-application, so setuid exec is allowed
scripts/
  build-deb.sh        Build one .deb (dpkg-deb, or ar/tar fallback)
  build-all.sh        Build every scheme/arch variant
  build-repo.sh       Assemble a Sileo/Zebra repo
  compile-standalone.sh   Build the .app
  install-deb.sh      Push + dpkg-install onto a USB device
control               Debian package metadata
deploy.sh             Side-load the .app (smoke-test only)
```

---

## Disclaimer

For use on devices you own or are authorised to test. Frida is a dynamic
instrumentation toolkit intended for security research, debugging, and development.

---

## License

Released under the [MIT License](LICENSE).
