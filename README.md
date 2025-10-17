Absolutely 👍 — here’s a clean, **Markdown-formatted setup note** you can save or reuse anywhere.
This version uses the **correct Wayland-native approach** (`LIBINPUT_IGNORE_DEVICE=1`), not the older permission or GRUB tricks.

---

# 🧭 Disable Internal Keyboard and Touchpad (Wayland / KDE / GNOME)

This guide permanently disables your **laptop’s internal keyboard and touchpad**
while keeping **external keyboards and mice fully functional**.
It uses the modern, safe method recommended by **libinput** and works across reboots.

---

## 🧩 Step 1 — Identify your internal devices

Run:

```bash
sudo libinput list-devices | grep -A5 "Device:"
```

Look for entries like:

```
Device:  AT Translated Set 2 keyboard
Kernel:  /dev/input/event3
...
Device:  FocalTechPS/2 FocalTech Touchpad
Kernel:  /dev/input/event4
```

Here,

* Internal keyboard → `/dev/input/event3`
* Internal touchpad → `/dev/input/event4`

---

## ⚙️ Step 2 — Create a udev rule to tell libinput to ignore them

Create a new file:

```bash
sudo vim /etc/udev/rules.d/99-ignore-internal-input.rules
```

Paste the following lines:

```text
# Ignore internal keyboard
KERNEL=="event*", SUBSYSTEM=="input", ATTRS{name}=="AT Translated Set 2 keyboard", ENV{LIBINPUT_IGNORE_DEVICE}="1"

# Ignore internal touchpad
KERNEL=="event*", SUBSYSTEM=="input", ATTRS{name}=="FocalTechPS/2 FocalTech Touchpad", ENV{LIBINPUT_IGNORE_DEVICE}="1"
```

Save and exit (`Esc` → `:wq` → Enter).

Then reload udev:

```bash
sudo udevadm control --reload-rules
sudo udevadm trigger
```

---

## 🔁 Step 3 — Reboot

Reboot your system:

```bash
sudo reboot
```

---

## ✅ Step 4 — Verify that it worked

### 1. Check via libinput:

```bash
sudo libinput debug-events
```

You should **not** see:

```
AT Translated Set 2 keyboard
FocalTechPS/2 FocalTech Touchpad
```

— only your external keyboard/mouse devices.

### 2. Confirm via udev:

```bash
udevadm info -a -n /dev/input/event3 | grep LIBINPUT_IGNORE_DEVICE
udevadm info -a -n /dev/input/event4 | grep LIBINPUT_IGNORE_DEVICE
```

Expected output:

```
E: LIBINPUT_IGNORE_DEVICE=1
```

---

## 🔓 Step 5 — To re-enable later

Simply comment out or delete the rule:

```bash
sudo vim /etc/udev/rules.d/99-ignore-internal-input.rules
```

Then reload rules and reboot:

```bash
sudo udevadm control --reload-rules
sudo udevadm trigger
sudo reboot
```

---

## 🧠 Notes

* Works perfectly on **Wayland** (KDE, GNOME, Sway, etc.).
* Doesn’t modify kernel drivers or `/dev/input` permissions.
* 100% safe — the devices still exist but are invisible to libinput and all applications.
* Survives reboots, system upgrades, and Wayland restarts.

---

✅ **Summary**

| Component                 | Disabled? | Method                             | Safe? |
| ------------------------- | --------- | ---------------------------------- | ----- |
| Internal Keyboard         | ✅         | `LIBINPUT_IGNORE_DEVICE` udev rule | ✅     |
| Internal Touchpad         | ✅         | `LIBINPUT_IGNORE_DEVICE` udev rule | ✅     |
| External Keyboard & Mouse | ❌         | Unaffected                         | ✅     |

---





# 🧭 Disable Internal Keyboard and Touchpad (Wayland / KDE / GNOME)

This guide permanently disables your **laptop’s internal keyboard and touchpad**
while keeping **external keyboards and mice fully functional**.

It uses the modern, safe method recommended by **libinput** (`LIBINPUT_IGNORE_DEVICE=1`)
and optionally makes it **conditional** — automatically re-enabling when no external keyboard is plugged in.

---

## 🧩 Step 1 — Identify your internal devices

Run:

```bash
sudo libinput list-devices | grep -A5 "Device:"
```

Look for entries like:

```
Device:  AT Translated Set 2 keyboard
Kernel:  /dev/input/event3
...
Device:  FocalTechPS/2 FocalTech Touchpad
Kernel:  /dev/input/event4
```

Note the device names exactly as shown — we’ll match by those names.

---

## ⚙️ Step 2 — Create a udev rule to make libinput ignore them

Create:

```bash
sudo vim /etc/udev/rules.d/99-ignore-internal-input.rules
```

Paste:

```text
# Ignore internal keyboard
KERNEL=="event*", SUBSYSTEM=="input", ATTRS{name}=="AT Translated Set 2 keyboard", ENV{LIBINPUT_IGNORE_DEVICE}="1"

# Ignore internal touchpad
KERNEL=="event*", SUBSYSTEM=="input", ATTRS{name}=="FocalTechPS/2 FocalTech Touchpad", ENV{LIBINPUT_IGNORE_DEVICE}="1"
```

Save (`Esc :wq`) and reload:

```bash
sudo udevadm control --reload-rules
sudo udevadm trigger
```

---

## 🔁 Step 3 — Reboot

```bash
sudo reboot
```

---

## ✅ Step 4 — Verify that it worked

### 1️⃣ Check that libinput ignores them:

```bash
sudo libinput debug-events
```

Only your **external** keyboard/mouse should appear — no “AT Translated Set 2 keyboard” or “FocalTech Touchpad”.

### 2️⃣ Confirm via udev:

```bash
udevadm info -a -n /dev/input/event3 | grep LIBINPUT_IGNORE_DEVICE
udevadm info -a -n /dev/input/event4 | grep LIBINPUT_IGNORE_DEVICE
```

Expected:

```
E: LIBINPUT_IGNORE_DEVICE=1
```

---

## 🧠 Step 5 — Optional: Make it conditional

*(auto-re-enable internal keyboard if no external one is detected)*

Create a small systemd service that dynamically enables or disables the udev rule.

```bash
sudo vim /usr/local/bin/toggle-internal-keyboard.sh
```

Paste:

```bash
#!/bin/bash
RULE_FILE="/etc/udev/rules.d/99-ignore-internal-input.rules"

# Detect external keyboard
if grep -qi "Keychron" /proc/bus/input/devices || grep -qi "Logitech" /proc/bus/input/devices; then
    echo "External keyboard detected → keeping internal keyboard disabled."
else
    echo "No external keyboard detected → re-enabling internal keyboard."
    sed -i 's/^/#/' "$RULE_FILE"
fi

# Reload udev and trigger changes
udevadm control --reload-rules
udevadm trigger
```

Make it executable:

```bash
sudo chmod +x /usr/local/bin/toggle-internal-keyboard.sh
```

Create the service:

```bash
sudo vim /etc/systemd/system/toggle-internal-keyboard.service
```

Paste:

```ini
[Unit]
Description=Toggle internal keyboard rule depending on external keyboard
After=graphical.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/toggle-internal-keyboard.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

Enable and run:

```bash
sudo systemctl daemon-reload
sudo systemctl enable toggle-internal-keyboard.service
sudo systemctl start toggle-internal-keyboard.service
```

Now, on every boot:

* If an **external keyboard** (e.g., *Keychron*, *Logitech*, etc.) is detected → internal keyboard stays disabled.
* If **no external keyboard** is present → udev rule is commented out (re-enabling internal keyboard automatically).

---

## 🔓 Step 6 — To re-enable everything manually

Just remove the rule:

```bash
sudo rm /etc/udev/rules.d/99-ignore-internal-input.rules
sudo udevadm control --reload-rules
sudo udevadm trigger
sudo reboot
```

---

## ✅ Summary

| Component                 | Disabled?   | Controlled By                      | Safe? |
| ------------------------- | ----------- | ---------------------------------- | ----- |
| Internal Keyboard         | ✅           | `LIBINPUT_IGNORE_DEVICE` udev rule | ✅     |
| Touchpad                  | ✅           | `LIBINPUT_IGNORE_DEVICE` udev rule | ✅     |
| External Keyboard & Mouse | ❌           | Unaffected                         | ✅     |
| Conditional Toggle        | ⚙️ Optional | `toggle-internal-keyboard.service` | ✅     |

---

💡 **This is the safest, Wayland-native way** to disable your laptop’s built-in keyboard and touchpad — no kernel tweaks, no permissions hacks, and it auto-adjusts when you connect or disconnect an external keyboard.
