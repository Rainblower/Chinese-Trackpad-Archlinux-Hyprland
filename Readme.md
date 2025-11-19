# 🖐 How to Random Cnina Trackpad Work Properly on Arch Linux + Hyprland

This guide is based on real debugging steps, command outputs, and the final working solution.
The issue turned out to be caused by the device being incorrectly handled by the **magicmouse** driver, preventing multitouch and gestures from working.

---

# 📌 Problem

The trackpad with USB ID `05ac:0265` or another, appears in the system but behaves like a **regular mouse** without multitouch support.

In `hyprctl devices` it shows up as:

```
Mouse at ... hid-05ac:0265
Mouse at ... hid-05ac:0265-1
```

In `libinput` it appears as:

```
Capabilities: pointer gesture
```

But **not** as a real touchpad.

In `dmesg`, the device is assigned to the **magicmouse** driver:

```
magicmouse 0003:05AC:0265.xxxx: USB HID v1.00 Mouse
```

Since `magicmouse` only emulates mouse behavior, it breaks multitouch.

---

# 🧭 Goal

- Disable the `magicmouse` driver  
- Enable `hid-multitouch`  
- Bind the trackpad to the correct driver  

This restores full multitouch and gesture support.

---

# 🧪 Diagnostics

## Check device in libinput

```bash
sudo libinput list-devices | grep -A15 -i 05ac
```

Output:

```
Capabilities: pointer gesture
```

## Kernel logs

```bash
sudo dmesg | grep -i 05ac
```

Fragments:

```
magicmouse 0003:05AC:0265.0006: USB HID v1.00 Mouse
magicmouse 0003:05AC:0265.0007: USB HID v1.00 Mouse
```

## Check if multitouch driver is loaded

```bash
lsmod | grep hid_multitouch
```

Output:

```
hid_multitouch         36864  0
```

The module is loaded but **not used**.

---

# ✅ Fix

## 1. Blacklist the `magicmouse` driver

Create a blacklist file:

```bash
sudo nano /etc/modprobe.d/blacklist-magicmouse.conf
```

Add:

```
blacklist magicmouse
blacklist hid_magicmouse
```

Update initramfs:

```bash
sudo mkinitcpio -P
```

---

## 2. Unload the driver or reboot

Try unloading the magicmouse module:

```bash
sudo modprobe -r hid_magicmouse
sudo modprobe hid_multitouch
```

If unloading fails (module in use), reboot:

```bash
sudo reboot
```

---

## 3. Verify after reboot

Check kernel logs:

```bash
dmesg | grep -i 05ac
```

Expected result:  
**No more `magicmouse` lines** — instead you should see `hid-multitouch`.

Check libinput again:

```bash
sudo libinput list-devices
```

Now you should see:

```
Capabilities: touchpad gesture
```

---

## 4. (If needed) Force driver binding via udev

If the system **still** attempts to load `magicmouse`, create a udev rule:

```bash
sudo nano /etc/udev/rules.d/99-apple-trackpad.rules
```

Add:

```
ACTION=="add", SUBSYSTEM=="hid", KERNEL=="0003:05AC:0265.*", \
  RUN+="/bin/sh -c 'echo -n %k > /sys/bus/hid/drivers/magicmouse/unbind; \
                    echo -n %k > /sys/bus/hid/drivers/hid-multitouch/bind'"
```

Reload rules:

```bash
sudo udevadm control --reload
sudo udevadm trigger
```

Replug the dongle.

---

# 🎉 Result

After disabling `magicmouse` and enabling `hid-multitouch`, the trackpad works fully:

- Multitouch  
- Two- and three-finger gestures  
- Smooth scrolling  
- Native-like behavior under Hyprland  
- No cursor jitter  
- No mouse emulation issues  

---

# 📌 Optional additions

I can also prepare:

- Hyprland gesture bindings  
- libinput tuning (sensitivity, natural scrolling, tap-to-click)  
- Multitouch event debugging with `libinput debug-events`

Just ask if you want them added!

