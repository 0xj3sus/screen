# Window snapping with Super + Arrow on Ubuntu

Use **Tiling Assistant v36** to place the active window in either half or any corner of the screen. This guide targets **Ubuntu with GNOME Shell 42.x on X11**. Everything runs under your user account. The extension handles the shortcuts after login, so you can close the terminal when setup is complete.

## How the shortcuts work

Start with a normal, resizable window. Hold **Super** (the Windows key) throughout each gesture. For a corner, press the horizontal arrow first, release it, then press the vertical arrow while still holding Super. The extension uses the window’s current position to choose the next action.

| While holding Super | Result |
| --- | --- |
| Press Left | Left half |
| Press Right | Right half |
| Press Up | Maximize |
| Press Left, then Up | Top-left quarter |
| Press Right, then Up | Top-right quarter |
| Press Left, then Down | Bottom-left quarter |
| Press Right, then Down | Bottom-right quarter |

## 1. Check your desktop

Run these commands in a terminal inside your graphical session.

```bash
echo "$XDG_CURRENT_DESKTOP"
gnome-shell --version
echo "$XDG_SESSION_TYPE"
```

This guide applies when the output includes `GNOME`, `GNOME Shell 42.x` and `x11`. The original setup reported `ubuntu:GNOME`, `GNOME Shell 42.9` and `x11`. For another GNOME version, check the [project compatibility table](https://github.com/ubuntu/Tiling-Assistant#installation) before choosing an extension release.

## 2. Install Tiling Assistant v36

If you already installed v36 in your account, skip to step 3. If an earlier Python snapping script is running, stop it with **Ctrl+C** in its terminal to release the shortcuts.

```bash
bash <<'BASH'
set -e
zip_dir=$(mktemp -d)
trap 'rm -rf -- "$zip_dir"' EXIT
curl -fL 'https://extensions.gnome.org/review/download/34016.shell-extension.zip' \
  -o "$zip_dir/tiling-assistant-v36.zip"
gnome-extensions install "$zip_dir/tiling-assistant-v36.zip"
BASH
```

On **X11**, press **Alt+F2**, type `r`, and press **Enter** to reload GNOME Shell. Alternatively, save your work, log out and log back in.

## 3. Configure the arrow keys

Paste the whole block once. It selects **Tiling State**, assigns **Super+Down** to bottom tiling, and turns off the window-selection popup so you can move directly from a half to a quarter. It replaces the extension’s bindings for these actions with the arrows and keypad alternatives shown below. It also removes Super+Down from GNOME’s **Unmaximize** action to free that shortcut.

```bash
bash <<'BASH'
set -e
uuid='tiling-assistant@leleat-on-github'
schema='org.gnome.shell.extensions.tiling-assistant'
export GSETTINGS_SCHEMA_DIR="${XDG_DATA_HOME:-$HOME/.local/share}/gnome-shell/extensions/$uuid/schemas"

# Check the v36 setting before changing any shortcuts.
gsettings get "$schema" dynamic-keybinding-behaviour >/dev/null

# Disable first so the extension releases its overrides before we edit them.
gnome-extensions disable "$uuid"
trap 'gnome-extensions enable "$uuid"' EXIT

/usr/bin/python3 - <<'PY'
from gi.repository import Gio

wm = Gio.Settings.new("org.gnome.desktop.wm.keybindings")
keys = wm.get_strv("unmaximize")
if "<Super>Down" in keys:
    remaining = [key for key in keys if key != "<Super>Down"]
    if not wm.set_strv("unmaximize", remaining):
        raise SystemExit("Could not update the GNOME Unmaximize shortcut.")
    Gio.Settings.sync()
PY

gsettings set "$schema" restore-window '[]'
gsettings set "$schema" tile-left-half   "['<Super>Left', '<Super>KP_4']"
gsettings set "$schema" tile-right-half  "['<Super>Right', '<Super>KP_6']"
gsettings set "$schema" tile-maximize    "['<Super>Up', '<Super>KP_5']"
gsettings set "$schema" tile-bottom-half "['<Super>Down', '<Super>KP_2']"
gsettings set "$schema" dynamic-keybinding-behaviour 2
gsettings set "$schema" enable-tiling-popup false

gnome-extensions enable "$uuid"
trap - EXIT
gnome-extensions info "$uuid"
BASH
```

## 4. Verify the result

Focus a normal window. Hold Super, tap **Left** and watch the window move to the left half. Still holding Super, tap **Down**. The window should occupy the **bottom-left quarter**. Release Super. Use the table above to try the other positions, starting from a normal window each time.

| If you see this | What to check |
| --- | --- |
| `Extension … does not exist` after installation | Reload GNOME Shell as described in step 2, then run step 3. |
| `No such key “dynamic-keybinding-behavior”` | v36 uses the British spelling **`dynamic-keybinding-behaviour`**. Use the exact command in step 3. |
| Left and Right work, but the second arrow does nothing | Check that the extension is enabled, then run the settings check below. Stop any older script that also handles Super+Arrow. |
| A direct corner binding shows `Disabled` in Preferences | The two-arrow gesture uses **Tiling State**. A separate direct corner binding is optional. |

```bash
gnome-extensions info tiling-assistant@leleat-on-github
gsettings --schemadir "${XDG_DATA_HOME:-$HOME/.local/share}/gnome-shell/extensions/tiling-assistant@leleat-on-github/schemas" \
  get org.gnome.shell.extensions.tiling-assistant dynamic-keybinding-behaviour
```

The extension should show as active or enabled, and the last command should print **`2`**.

## References

- [Tiling Assistant on GitHub](https://github.com/ubuntu/Tiling-Assistant)
- [Official v36 download page](https://extensions.gnome.org/review/34016)
- [How dynamic keybindings select a corner](https://github.com/ubuntu/Tiling-Assistant/wiki/Dynamic-Keybindings)
