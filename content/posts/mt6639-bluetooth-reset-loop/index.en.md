---
author: Douglas Santos
title: "Ghost Bluetooth on Linux: how btusb wedges the MediaTek MT6639 trying to load firmware that isn't there."
date: "2026-09-09"
description: "Bluetooth never worked on this board and I was sure the missing firmware was the problem. It was — but what actually broke the hardware was the driver trying to fix it forever. Here is the whole investigation, from a false negative in dmesg to a 174 millisecond race at boot."
draft: false
slug: "mt6639-bluetooth-reset-loop"
tags:
  - posts
---

I bought an ASUS ProArt X870E-Creator WiFi and Wi-Fi worked on the first boot. Bluetooth did not. No adapter ever showed up, and worse: the device was not in `lsusb` at all. Not failing, not erroring — absent.

My first guess was the obvious one, and it was half right: the Bluetooth firmware for this chip does not ship in `linux-firmware`. What I did not expect is that the missing firmware is not what breaks the hardware. What breaks it is what the driver does about it.

In this article I will walk through the whole investigation, including the two moments where I confidently reached the wrong conclusion. I think those parts are more useful than the fix.

| | |
|---|---|
| Board | ASUS ProArt X870E-Creator WiFi rev 2 |
| BIOS | 2402 |
| OS | Bazzite (Fedora 44 Atomic) |
| Kernel | 7.2.3-ogc3.1.fc44.x86_64 |
| Bluetooth | MediaTek MT6639, USB `0489:e13a` |
| Wi-Fi | MediaTek MT7927, PCIe `14c3:7927` |

## The symptom: an adapter that does not exist

No adapter. `bluetoothctl show` hangs without printing anything, because bluetoothd sits waiting on a controller that is never going to appear. And `dmesg` had absolutely no messages from the Bluetooth subsystem — no success, no firmware failure, no USB error. Complete silence.

Complete silence is a strange result. If the firmware were missing, I should see the driver complaining. If the device were faulty, I should see USB complaining. Seeing nothing means the kernel never even tried.

## The false negative that cost me a whole round

Here is the first mistake, and it is embarrassing in how simple it is.

```bash
dmesg | grep -iE 'btusb|bluetooth: hci'
```

This came back empty. I read "empty" as "there are no Bluetooth messages". Wrong. What was actually happening is that `kernel.dmesg_restrict=1` makes `dmesg` return **zero lines of any kind** without root. There were no messages at all to filter — not about Bluetooth, not about USB, not about anything.

```bash
$ dmesg | wc -l
0
$ sysctl -n kernel.dmesg_restrict
1
```

An empty `grep` over empty input looks exactly like an empty `grep` over a thousand lines. The practical lesson: use `journalctl -k`, which works without sudo and gives you the real log.

```bash
$ journalctl -k -b 0 --no-pager | wc -l
1893
```

One thousand eight hundred and ninety three lines I had been ignoring. And everything was in there.

## The device did not disappear — it wedged

With the real log in hand the picture changed completely. The device **is** there, on port `1-6`, right next to the board's LED controller. It is detected electrically. It just does not answer anything.

```
usb 1-6: new high-speed USB device number 4 using xhci_hcd
usb 1-6: device descriptor read/64, error -110
usb 1-6: device descriptor read/64, error -110
usb 1-6: new high-speed USB device number 5 using xhci_hcd
usb 1-6: device descriptor read/64, error -110
usb 1-6: device descriptor read/64, error -110
usb usb1-port6: attempt power cycle
usb 1-6: new high-speed USB device number 6 using xhci_hcd
usb 1-6: Device not responding to setup address.
usb 1-6: device not accepting address 6, error -71
usb 1-6: new high-speed USB device number 7 using xhci_hcd
usb 1-6: Device not responding to setup address.
usb 1-6: device not accepting address 7, error -71
usb usb1-port6: unable to enumerate USB device
```

Translating: `-110` is a timeout, `-71` is a protocol error. The hub sees something plugged in, tries to read the device descriptor, gets no answer, cuts and restores power to the port, tries again, gives up. Four attempts, sixty three seconds, nothing.

This also explains why `btusb` was not loaded, and why that was **not** the problem. udev loads the module when a matching modalias shows up. Since nothing enumerated, there is no modalias, so the module does not load. Loading it by hand works with no errors and creates no HCI device at all:

```bash
$ sudo modprobe btusb && ls /sys/class/bluetooth/
# (empty)
```

The software stack is spotless. It is the hardware that never shows up.

## The cause: btusb retries forever

Bazzite keeps previous boots in the journal, and that is where it gets interesting. I swept every recorded boot looking for the device, and in two of them it **worked** — it enumerated normally. So I went to look at what had happened:

```
[    3.068951] usb 1-6: New USB device found, idVendor=0489, idProduct=e13a
[    3.069092] usb 1-6: Product: Wireless_Device
[    8.221018] usbcore: registered new interface driver btusb
[    8.233139] Bluetooth: hci0: Failed to load firmware file (-2)
[    8.233145] Bluetooth: hci0: Failed to set up firmware (-2)
[    8.564037] usb 1-6: reset high-speed USB device number 4 using xhci_hcd
[    8.817354] Bluetooth: hci0: Failed to load firmware file (-2)
[    9.147127] usb 1-6: reset high-speed USB device number 4 using xhci_hcd
[    9.402354] Bluetooth: hci0: Failed to load firmware file (-2)
[    9.727020] usb 1-6: reset high-speed USB device number 4 using xhci_hcd
```

See the pattern? The firmware fails with `-2` (ENOENT, file not found), and `btusb` **resets the device over USB and tries again**. Then it fails again, resets again, tries again. Every 0.58 seconds. No backoff, no retry limit, never gives up.

On that boot this ran for 13 minutes until I shut the machine down. That was 1335 resets.

And that is what wedges the chip. The full chain goes like this:

1. Clean cold start: the controller enumerates normally, about 3 seconds into boot.
2. `btusb` binds at around 8 seconds, creates `hci0`, requests the firmware.
3. The file is not there. It returns `-2`.
4. **`btusb` resets the device and tries again. And again. Indefinitely.**
5. After a few hundred resets, the controller firmware wedges.
6. From then on the port detects the device but it no longer answers anything.
7. This **survives reboots**, because the standby power rail keeps the chip alive.

Point 7 is what makes it cruel. You reboot, no change. You reinstall the OS, no change. The chip stays wedged because it was never actually powered down.

## The numbers line up

What convinced me this was really the cause, and not a coincidence, was counting. If the reset loop is a consequence of the firmware failure, the two numbers have to be identical:

| Boot | Resets of port 1-6 | Firmware failures |
|---|---:|---:|
| -8 | 398 | 398 |
| -1 | 1335 | 1335 |

They match exactly. One reset per failure, in both boots.

And the recurrence pattern across boots tells the rest of the story:

| Boot | Port 1-6 | Reading |
|---|---|---|
| -8 | enumerated | clean start, 398 resets, wedges |
| -7 to -3 | no events | **five dead boots**, caused by -8 |
| -2 | failed to enumerate | wedged |
| -1 | enumerated | freed by a power cut, 1335 resets, wedges again |
| 0 | failed to enumerate | wedged |

Two complete cycles. Every time I freed the chip by cutting power, it came back, entered the loop, and wedged again. I was reproducing the problem without knowing it.

To free it, you have to actually cut power: enable **ErP in S4+S5** in the BIOS and use `poweroff` (not `reboot`, which never cuts standby), or unplug the machine for about 10 seconds.

## By the way, the slow boot was the same bug

I had a second complaint I thought was unrelated: boot was taking almost two minutes. It was related.

`systemd-udev-settle.service` sits at the **root of the boot's critical chain** — everything waits on it. And the enumeration retries on port 1-6 keep udev busy:

| State of port 1-6 | Boots | udev-settle |
|---|---|---:|
| device absent | -7 to -3 | 8.6 - 9.7 s |
| device wedged | -2, 0 | **67.8 s** |

About 59 seconds of penalty, matching the 63 second window of the retries. Once the controller came up properly, boot went from **1 min 52 s to 41 s**. Two symptoms, one bug.

## Where to put firmware when /usr is read-only

Bazzite is an immutable system: `/usr` is read-only, so you cannot just drop the file into `/usr/lib/firmware`. The standard route is to use `/var/lib/firmware` and point the kernel at it with a boot parameter:

```bash
sudo install -Dm644 BT_RAM_CODE_MT6639_2_1_hdr.bin \
  /var/lib/firmware/mediatek/mt7927/BT_RAM_CODE_MT6639_2_1_hdr.bin

sudo rpm-ostree kargs --append=firmware_class.path=/var/lib/firmware
```

I did that, rebooted, and Bluetooth worked. End of article, right?

No. I went to check the log and the firmware had **failed with `-2` again** — and the device came up anyway. That made no sense at all, and the explanation is the most interesting part of the whole thing.

## The race I won by 174 milliseconds

On ostree systems, `/var` is a separate subvolume mounted by a systemd unit **after** switch-root. And `btusb` probes right inside that window.

| Time | Event |
|---:|---|
| 7.294 s | switch-root |
| 8.677 s | `btusb` registers the interface driver |
| **8.688 s** | **first firmware attempt fails, `-2`** |
| 9.021 s | `btusb` resets the device and reschedules |
| **9.235 s** | **`var.mount` completes** |
| 9.409 s | the retry **finds** the firmware |
| 28.758 s | `Device setup in 19036182 usecs` |
| 28.930 s | `AOSP extensions version v1.00` |

So it worked for the wrong reason. The first iteration of the reset loop — the same loop that wedges the chip — is exactly what saved it, because `/var` mounted in the middle of it. The margin was 174 milliseconds.

That is reproducible luck, not a guarantee. If the retry landed 200 ms earlier, the loop would start and the chip would wedge. And there is a cruel detail on top: there is a non-empty `/var` stub underneath the mount point in the deployment, so the lookup fails **silently** instead of reporting a missing directory.

## The real fix

The solution is to put the firmware somewhere already readable at switch-root. `/etc` qualifies: it is a bind mount of the deployment's own subvolume, set up by `ostree-prepare-root` while still inside the initrd. It is available at 7.294 s, more than a second before `btusb` probes.

```bash
sudo install -Dm644 BT_RAM_CODE_MT6639_2_1_hdr.bin \
  /etc/firmware/mediatek/mt7927/BT_RAM_CODE_MT6639_2_1_hdr.bin

sudo rpm-ostree kargs \
  --replace=firmware_class.path=/var/lib/firmware=/etc/firmware
```

A nice detail: SELinux labels the file `cpucontrol_conf_t`, because `/etc/firmware` is already a path the policy knows about (used for CPU microcode). And it blocks nothing — the loaded policy does not even define the `firmware_load` permission.

To validate after the reboot, both numbers have to come out zero:

```bash
journalctl -k -b 0 | grep -c 'Failed to load firmware file'
journalctl -k -b 0 | grep -c 'reset high-speed USB device'
```

## The staged /etc trap — I almost gave up here

This was the second moment where I confidently reached the wrong conclusion, and it was worth a scare.

After `rpm-ostree kargs`, I went to check the new deployment before rebooting. `/etc/firmware` **was not there**. And the deployment is read-only, so I could not even put it there by hand.

This looked fatal. With the boot parameter now pointing only at `/etc/firmware`, a missing file means the first attempt **and the retry** both fail, since `/var` is no longer in the search path. In other words: worse than not touching anything.

Before reverting, I decided to compare the two `/etc` trees. The new deployment's was missing 23 entries the running one had:

```
bazzite    cardwire   cni        crypttab   firmware
fstab      group-     gshadow-   hostname   iwd
locale.conf localtime passwd-    sddm.conf.d shadow-
subgid-    subuid-    vconsole.conf
```

Look at `fstab` sitting there in the middle.

That is what settles it. If finalization did not merge `/etc`, **no ostree system could boot after an upgrade**, because it would have no `fstab`. Therefore, a staged deployment's `/etc` is the pristine one from the new commit, and the three-way merge is deferred to `ostree admin finalize-staged`, which runs at shutdown.

In other words: inspecting a staged `/etc` will **always** look like your local changes are gone. That is expected, not a defect. My file was going to be carried over along with `fstab` and `hostname`, and it was.

What got me out of that hole was not prior ostree knowledge, it was looking for a piece of evidence that would settle the question instead of betting on a hunch.

## Where this firmware comes from anyway

The Bluetooth blob is not in `linux-firmware`. MR !946 was closed because the project only accepts blobs submitted by the rights holder — it has to come from MediaTek itself. The Wi-Fi one got in through MR !1055, and that is exactly why the Wi-Fi half of the chip works out of the box and the Bluetooth half does not.

You can extract it from ASUS's Windows driver packages with [extract_firmware.py](https://github.com/jetm/mediatek-mt7927-dkms) from the `jetm/mediatek-mt7927-dkms` project. I extracted it from two different packages, independently, and the files come out **byte for byte identical**:

| Package | Container | Size |
|---|---|---:|
| Bluetooth V1.1147.0.610 | `mtkbt_v2.dat` | 571349 B |
| Wi-Fi V5706054 | `mtkwlan.dat` | 571349 B |

```
sha256  2135f2c4220cfa6e8eb9fdf430517098c13b862a95844bbff0153240a768efa8
path    mediatek/mt7927/BT_RAM_CODE_MT6639_2_1_hdr.bin
built   20260611041233
```

Two observations that save time. The path is `mediatek/mt7927/`, not `mediatek/mt6639/` — confirm it directly against the module's strings with `modinfo -F firmware btmtk` instead of inferring it from the error message, because that convention has already moved between kernel versions. And a hash of `669c5c99...` at roughly 688 KB circulates out there; it did not reproduce here from either package. Two independent extractions agreeing carry more weight than one loose community figure.

## What not to do — read before reproducing

- **Do not run `modprobe -r btusb`.** The MT6639 firmware hangs during a module reload and the device disappears from `lsusb` persistently. That is how this whole story started. Reboot instead.
- **Do not install `WIFI_*.bin` files into a firmware path.** `linux-firmware` already ships the correct ones, compressed, in `/usr/lib/firmware/mediatek/mt7927/`. A loose copy silently shadows the newer blob and breaks Wi-Fi that currently works. Only the Bluetooth file should be installed.
- **Do not expect a reboot to free the controller.** The standby rail keeps it powered. Only a real power cut clears it.
- **Do not let the loop run once you see the firmware failure.** Every reset cycle risks wedging the chip again and costs another power cycle. Shut down as soon as `Failed to load firmware file (-2)` shows up.

## What should change in the kernel

A firmware file that does not exist will still not exist on the next attempt. Retrying the same request thousands of times cannot possibly succeed — and here it does active harm, because it pushes the controller into a state that survives reboots and needs physical intervention.

A retry limit, or a backoff, or simply not retrying on `-2` when the previous attempt failed for the same reason, would turn this from a wedged device into a log line saying the firmware is missing. I am taking this to `linux-bluetooth`.

## Conclusion

What I take away from this investigation is not the final command, which fits in two lines. It is two other things.

The first is that absence of evidence is not evidence of absence, and a silent tool lies. An empty `grep` convinced me for quite a while that there was no log, when in fact there was no permission. It is always worth confirming that the tool is actually handing you data before you interpret its silence.

The second is that the second scare — the apparently empty staged `/etc` — resolved because I went looking for evidence that would settle the question instead of trusting what I thought ostree did. That `fstab` in the list was worth more than any certainty of mine about how the system works.

And in the end, the missing firmware really was the problem. It just did not break anything on its own: the driver did, trying with infinite persistence to fix something that could not be fixed that way.

If you have this chip and landed here wondering why your Bluetooth does not work, I hope this article saves you the days it cost me. Any questions or corrections, just reach out.

See you next time!
