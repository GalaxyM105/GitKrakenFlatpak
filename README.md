# GitKraken Flatpak

This is the unofficial community-maintained Flatpak for [GitKraken](https://www.gitkraken.com/).
It is **not verified by, affiliated with, or supported by Axosoft, LLC** (see the [Flathub page](https://flathub.org/en/apps/com.axosoft.GitKraken)).

The manifest downloads the official `gitkraken-amd64.deb` at install time using Flatpak's `extra-data` mechanism, then extracts it inside the sandbox. This means the version shipped by the Flatpak is normally the same as the current official Linux `.deb`, but updates depend on the Flatpak maintainers bumping the manifest.

## Contents

- [Build / install from this repo](#build--install-from-this-repo)
- [How the version is kept in sync](#how-the-version-is-kept-in-sync)
- [Limitations compared with the official `.deb`](#limitations-compared-with-the-official-deb)
  - [OAuth / browser login and the `gitkraken://` URL scheme](#oauth--browser-login-and-the-gitkraken-url-scheme)
  - [External editors, diff/merge tools and terminals](#external-editors-diffmerge-tools-and-terminals)
  - [System D-Bus / power management](#system-d-bus--power-management)
  - [Warnings that are not Flatpak-specific](#warnings-that-are-not-flatpak-specific)
- [Optional workarounds](#optional-workarounds)
  - [Option A: Register the `gitkraken://` handler on the host](#option-a-register-the-gitkraken-handler-on-the-host)
  - [Option B: Enable automatic `gitkraken://` registration from inside the Flatpak](#option-b-enable-automatic-gitkraken-registration-from-inside-the-flatpak)
  - [Option C: Open external editors / terminals](#option-c-open-external-editors--terminals)
  - [Option D: System D-Bus / power management](#option-d-system-d-bus--power-management)
- [What the tested workarounds fixed](#what-the-tested-workarounds-fixed)
- [Reporting issues](#reporting-issues)

## Build / install from this repo

```bash
# Add Flathub and the freedesktop SDK if you have not already
flatpak remote-add --user --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo
flatpak install --user flathub org.freedesktop.Platform//24.08 org.freedesktop.Sdk//24.08

# Build and install the app from this manifest
flatpak-builder --install --user --force-clean build-dir com.axosoft.GitKraken.json
```

## How the version is kept in sync

The manifest is an `extra-data` build. During `flatpak install` the official `.deb` is downloaded from `release.gitkraken.com` / `api.gitkraken.dev` and its SHA-256 and size are checked against the values in `com.axosoft.GitKraken.json`. If the upstream `.deb` is newer than the manifest, the install fails until a maintainer updates those fields. You can check the current upstream version on the [Flathub page](https://flathub.org/en/apps/com.axosoft.GitKraken).

## Limitations compared with the official `.deb`

Running a proprietary Electron app inside a Flatpak sandbox necessarily removes some abilities that the unsandboxed `.deb` has by default. The upstream maintainers have intentionally left these as sandbox limitations rather than opening large holes in the default permissions.

### OAuth / browser login and the `gitkraken://` URL scheme

GitKraken uses the `gitkraken://` URL scheme to finish OAuth flows (GitHub, GitLab, Bitbucket, etc.). On a normal `.deb` install, GitKraken can call `xdg-settings` to register itself as the default handler for that scheme. Inside the Flatpak:

- `xdg-settings` is not provided by the runtime.
- `xdg-settings` modifies the host's MIME associations, so shipping it inside the sandbox would require a permission to run commands on the host.
- As a result, clicking "Sign in with ..." may show `LaunchProcess: failed to execvp: xdg-settings` and you may have to copy the token from the browser back into GitKraken manually.

**This is a sandbox limitation, not a bug.** Upstream issue: [flathub/com.axosoft.GitKraken#156](https://github.com/flathub/com.axosoft.GitKraken/issues/156).

### External editors, diff/merge tools and terminals

GitKraken can open files in an external editor, run a diff/merge tool, or open a terminal. In the Flatpak these lists are usually empty and custom commands like `code` or `gnome-terminal` do not work because the sandbox cannot see or launch host programs or other Flatpaks.

**This is also a sandbox limitation, not a bug.** Upstream issues: [flathub/com.axosoft.GitKraken#12](https://github.com/flathub/com.axosoft.GitKraken/issues/12), [flathub/com.axosoft.GitKraken#88](https://github.com/flathub/com.axosoft.GitKraken/issues/88).

### System D-Bus / power management

Electron's `powerMonitor` expects to talk to `org.freedesktop.login1` (systemd-logind) and `org.freedesktop.UPower` over the system D-Bus. Without permission to reach those services you may see:

```
Failed to connect to the bus: Failed to connect to socket /run/dbus/system_bus_socket
```

This is normally non-fatal, but it means suspend/resume inhibition and battery-aware behavior may not work the same way as the `.deb`.

The default manifest does **not** grant system-bus access. You can optionally add it as described in [Option D](#option-d-system-d-bus--power-management).

### Warnings that are not Flatpak-specific

Some console warnings appear because the test/development environment lacks services that a normal desktop has. These warnings are also seen when running the official `.deb` binary directly on the same machine and cannot be fixed by the Flatpak manifest:

- `fusermount3: fuse device not found` / `Can't get document portal` — the kernel FUSE module is not loaded.
- `ContextResult::kTransientFailure: Failed to send GpuControl.CreateCommandBuffer` — no GPU / DRI device is passed through, so Chromium falls back to software rendering.
- `Failed to load RealtimeKit property` / `PipeWire` / `AT-SPI` warnings — RealtimeKit, PipeWire or the accessibility bus are not running.
- `state: unavailable` / `UnhandledPromiseRejectionWarning` — application-level network/login state messages, not sandbox errors.

## Optional workarounds

These are **not enabled by default** because each one punches a hole in the sandbox. Only enable the ones you actually need, and prefer the least-privileged option first.

### Option A: Register the `gitkraken://` handler on the host

This is the safest option. It requires no extra sandbox permissions. After installing the Flatpak, run this once on the host:

```bash
xdg-settings set x-scheme-handler/gitkraken com.axosoft.GitKraken.UrlHandler.desktop
```

This makes the host's default browser (or any app that fires `gitkraken://...`) open the Flatpak. GitKraken may still try to call `xdg-settings` internally and log a warning, but the scheme association will already be correct.

### Option B: Enable automatic `gitkraken://` registration from inside the Flatpak

If you want GitKraken's own "Set as default URL handler" flow to work automatically, you need to give the sandbox two extra things:

1. The `org.freedesktop.Flatpak` D-Bus permission, so it can run `flatpak-spawn --host`.
2. An `xdg-settings` wrapper inside the sandbox that forwards the call to the host.

#### Security note

`org.freedesktop.Flatpak` lets the sandbox spawn arbitrary commands on the host. The `xdg-settings` wrapper lets GitKraken change your host MIME associations. This is the same hole used for external-editor support. The upstream maintainers have chosen not to ship this by default.

#### How to enable

Add to `finish-args` in `com.axosoft.GitKraken.json`:

```json
"--talk-name=org.freedesktop.Flatpak"
```

Then add the following wrapper to `gitkraken-build.sh` before the `bsdtar` line:

```sh
cat > xdg-settings-wrapper <<'EOF'
#!/bin/sh
# Forward xdg-settings to the host, mapping the internal desktop file names
# to the exported Flatpak app IDs.

app_id="com.axosoft.GitKraken"
mapped_args=""
for arg in "$@"; do
    case "$arg" in
        gitkraken-url-handler.desktop)
            arg="${app_id}.UrlHandler.desktop"
            ;;
        gitkraken.desktop)
            arg="${app_id}.desktop"
            ;;
    esac
    mapped_args="$mapped_args \"$arg\""
done
eval set -- "$mapped_args"
unset app_id mapped_args arg

exec flatpak-spawn --host sh -c '
    XDG_DATA_DIRS="${XDG_DATA_DIRS:-/usr/local/share:/usr/share}"
    XDG_DATA_DIRS="$XDG_DATA_DIRS:/var/lib/flatpak/exports/share:$HOME/.local/share/flatpak/exports/share"
    export XDG_DATA_DIRS
    unset XDG_CONFIG_HOME XDG_DATA_HOME
    exec xdg-settings "$@"
' _ "$@"
EOF
install -Dm0755 xdg-settings-wrapper "${FLATPAK_DEST}/bin/xdg-settings"
```

After rebuilding, `xdg-settings set x-scheme-handler/gitkraken ...` will no longer fail and `gitkraken://` links will open the Flatpak.

### Option C: Open external editors / terminals

To let GitKraken launch a host editor or terminal, grant the `org.freedesktop.Flatpak` D-Bus permission and use `flatpak-spawn --host` as the custom command.

#### Security note

This gives the sandbox the same host-execution hole as Option B. It is not enabled by default.

#### How to enable

1. Add the permission. You can do this without rebuilding by using [Flatseal](https://github.com/tchx84/flatseal) or the command line:

   ```bash
   flatpak override --user --talk-name=org.freedesktop.Flatpak com.axosoft.GitKraken
   ```

2. In GitKraken's settings, set the custom editor/terminal command. For example:

   - Host VS Code: `flatpak-spawn --host code`
   - Host terminal: `flatpak-spawn --host gnome-terminal`
   - Another Flatpak (e.g. VS Code: Flatpak): `flatpak-spawn --host flatpak run com.visualstudio.code`

If you want this to persist for all users, add `--talk-name=org.freedesktop.Flatpak` to the `finish-args` in the manifest and rebuild.

### Option D: System D-Bus / power management

If you want to silence the `Failed to connect to socket /run/dbus/system_bus_socket` messages and let Electron's `powerMonitor` talk to `systemd-logind` and `UPower`, add these targeted system-bus permissions to the `finish-args`:

```json
"--system-talk-name=org.freedesktop.login1",
"--system-talk-name=org.freedesktop.UPower"
```

#### Security note

This is much narrower than `--socket=system-bus`, which would expose the entire system bus, but it does give the app access to session/seat information, suspend/resume inhibitors, and battery/charging state. It is not enabled by default.

## What the tested workarounds fixed

The following was verified by building the Flatpak with the `org.freedesktop.Flatpak` permission and the `xdg-settings` wrapper (but **without** the `login1`/`UPower` system-bus permissions), and by running the official `gitkraken-amd64.deb` on the same machine:

- With the `xdg-settings` wrapper + `org.freedesktop.Flatpak`, the `LaunchProcess: failed to execvp: xdg-settings` error disappeared.
- With `org.freedesktop.Flatpak`, `flatpak-spawn --host <command>` worked from inside the sandbox, so external editor/terminal commands can be configured.
- Without `--system-talk-name=org.freedesktop.login1` and `--system-talk-name=org.freedesktop.UPower`, the `Failed to connect to socket /run/dbus/system_bus_socket` error still appeared. That error is only silenced by adding the `login1`/`UPower` system-bus permissions described in [Option D](#option-d-system-d-bus--power-management).
- The remaining warnings (`fusermount3`, GPU fallback, RealtimeKit, PipeWire, AT-SPI, `state: unavailable`, etc.) also appeared in the official `.deb` log. They are caused by the test environment, not by the Flatpak sandbox.

In other words, Options B and C resolve the `xdg-settings` and external-editor/terminal errors, but the system-bus error requires Option D. The rest is environment noise and does not affect functionality on a normal desktop.

## Reporting issues

- Upstream Flatpak manifest and issues: [flathub/com.axosoft.GitKraken](https://github.com/flathub/com.axosoft.GitKraken)
- Official GitKraken support: https://www.gitkraken.com/contact
- Flathub page: https://flathub.org/en/apps/com.axosoft.GitKraken
