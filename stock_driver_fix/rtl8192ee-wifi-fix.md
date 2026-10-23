# Fixing a Realtek RTL8192EE wifi card that dies shortly after boot (Linux)

Written 2026-10-22 after fixing a ThinkPad X240 running Linux Mint 22.3 (kernel 6.8).
The same recipe applies to any laptop with this card (PCI ID `10ec:818b`) and
likely to its siblings (RTL8723BE, RTL8821AE, RTL8188EE) that use the in-kernel
`rtlwifi` driver.

> **Quick path:** `sudo ./fix-rtl8192ee-wifi.sh` (next to this file) does steps 1 to 3 below
> automatically, detects the card and root port itself, and tells you if a full power-off is
> needed. The rest of this guide explains what it does and how to do it by hand.

## Symptoms

- Wifi works for a few seconds to a few minutes after boot, then disappears.
  Scans show no networks, the connection drops, or the interface cannot be brought up.
- `dmesg` repeats `rtl8192ee: Init MAC failed`.
- Reloading the driver prints `Unable to change power state from D3cold to D0, device inaccessible`
  and `RTL8XXX did not boot from eeprom, check it !!`.
- `sudo lspci -vvv -s <card>` shows `!!! Unknown header type 7f` (config space reads as all-ones).
- A warm reboot does not help. Only a full power-off brings the card back.

## What is going on

The card's PCIe link idles into ASPM L1 (and L1 substates) and the card never wakes up.
Three things line up to cause it:

1. The BIOS enables ASPM L1 and L1.1/L1.2 on the root port the card hangs off.
2. The firmware tells Linux not to manage ASPM (`ACPI FADT declares the system doesn't
   support PCIe ASPM` in `dmesg`), so the kernel keeps the BIOS setting and `pcie_aspm=off`
   changes nothing.
3. The `rtl8192ee` driver's hardware init unconditionally programs the card to request L1
   when idle (`_rtl92ee_enable_aspm_back_door`), regardless of its `aspm` module option.
   Its inactive power save (`ips`) also powers the card off when idle and enables ASPM
   on the link at the same time, which triggers the hang even sooner.

Fix = stop the driver's power saving **and** refuse L1 on the root port.
Driver options alone were not enough on the X240.

## Step 1: identify the card and its root port

```bash
lspci -nn | grep -i realtek            # e.g. 03:00.0 ... RTL8192EE [10ec:818b]
CARD=0000:03:00.0
basename "$(dirname "$(readlink -f /sys/bus/pci/devices/$CARD)")"   # e.g. 0000:00:1c.1
sudo lspci -vvv -s 00:1c.1 | grep -E "LnkCtl:|L1SubCtl1"           # usually "ASPM L1 Enabled"
```

Useful log checks:

```bash
dmesg | grep -i -E "aspm|rtl8192ee|Init MAC"
journalctl -b -1 | grep -E "rtl8192ee|wlp" | head      # previous boot's timeline
```

## Step 2: turn off the driver's power management

`/etc/modprobe.d/rtl8192ee.conf`:

```
options rtl8192ee ips=0 swlps=0 fwlps=0 aspm=0
```

(`modinfo rtl8192ee | grep parm` lists the options; sibling drivers use the same names.)

`/etc/NetworkManager/conf.d/wifi-powersave-off.conf`:

```
[connection]
wifi.powersave = 2
```

If `lsinitramfs /boot/initrd.img-$(uname -r) | grep rtl8192ee` prints anything,
also run `sudo update-initramfs -u`. On Mint/Ubuntu the module is not in the initramfs.

## Step 3: refuse ASPM L1 on the root port at every boot

`/usr/local/sbin/rtl8192ee-noaspm` (make it executable, `chmod 755`).
Adjust `CARD` for the machine; the root port is found automatically.

```bash
#!/bin/bash
# Keep a Realtek PCIe wifi card's link out of ASPM L1 / L1 substates.
CARD=${1:-0000:03:00.0}
RP=$(basename "$(dirname "$(readlink -f /sys/bus/pci/devices/$CARD)")") || exit 1
RP=${RP#0000:}; DEV=${CARD#0000:}

# Root port Link Control: ASPM bits [1:0] = 0 (no L0s, no L1).
v=$(setpci -s $RP CAP_EXP+0x10.w) || exit 1
setpci -s $RP CAP_EXP+0x10.w=$(printf '%04x' $(( 0x$v & ~0x3 )))

# Root port L1 PM Substates Control 1: clear the four enable bits.
off=$(lspci -s $RP -vv | sed -n 's/.*Capabilities: \[\([0-9a-f]*\) v[0-9]*\] L1 PM Substates.*/\1/p')
if [ -n "$off" ]; then
	c=$(setpci -s $RP 0x$off+0x8.l)
	setpci -s $RP 0x$off+0x8.l=$(printf '%08x' $(( 0x$c & ~0xf )))
fi

# Card Link Control: ASPM bits [1:0] = 0, Clock PM enable (bit 8) = 0.
d=$(setpci -s $DEV CAP_EXP+0x10.w 2>/dev/null)
case "$d" in
	""|ffff) echo "card $DEV not readable (off the bus; power-cycle the machine)" ;;
	*) setpci -s $DEV CAP_EXP+0x10.w=$(printf '%04x' $(( 0x$d & ~0x103 ))) ;;
esac
echo "root port $RP LnkCtl $v -> $(setpci -s $RP CAP_EXP+0x10.w), card LnkCtl ${d:-n/a} -> $(setpci -s $DEV CAP_EXP+0x10.w 2>/dev/null || echo n/a)"
```

`/etc/systemd/system/rtl8192ee-noaspm.service` (runs before NetworkManager):

```ini
[Unit]
Description=Disable PCIe ASPM on the Realtek wifi link
DefaultDependencies=no
After=sysinit.target
Before=network-pre.target NetworkManager.service
Wants=network-pre.target

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/rtl8192ee-noaspm
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

Firmware can re-enable L1 after suspend, so re-run it on resume with
`/usr/lib/systemd/system-sleep/rtl8192ee-noaspm` (executable):

```sh
#!/bin/sh
[ "$1" = "post" ] && /usr/local/sbin/rtl8192ee-noaspm
exit 0
```

Enable it:

```bash
sudo systemctl daemon-reload
sudo systemctl enable rtl8192ee-noaspm.service
```

## Step 4: power-cycle, then verify

Once the card has hung it is gone from the bus until power is cut. Bus resets, link
retraining and PCI rescans do not recover it. Shut down fully (not Restart), wait ten
seconds, power on. Then check after ten or more minutes:

```bash
systemctl status rtl8192ee-noaspm.service        # active (exited), "LnkCtl 0042 -> 0040"
sudo lspci -vvv -s 00:1c.1 | grep LnkCtl           # ASPM Disabled
sudo lspci -vvv -s 03:00.0 | grep -c "Unknown header"   # 0
cat /sys/module/rtl8192ee/parameters/ips           # N
dmesg | grep -c "Init MAC failed"                  # 0
```

## Pitfalls met along the way

- Do not toggle wifi off in the desktop or with the airplane-mode key while testing.
  The soft block is saved in `/var/lib/systemd/rfkill/` and comes back after reboot.
  Clear it with `sudo rfkill unblock wlan`.
- If you unbind the driver while the card is hung, the kernel may runtime-suspend the
  root port (`power/runtime_status` = suspended); any rescan then finds nothing.
  `echo on > /sys/bus/pci/devices/<root port>/power/control` first.
- Secure Boot off, kernel headers and DKMS are only needed if you go for the out-of-tree
  driver (lwfinger/rtl8192ee or forks). It was not needed here, and it does not build on
  kernel 6.8 without more patches (`set_monitor_channel` gained a `net_device` argument).

## If it still fails

Plan B is to rebuild the in-kernel `rtl8192ee` module with the
`_rtl92ee_enable_aspm_back_door()` call removed from `rtl92ee_hw_init()` and install it
with DKMS, or to finish porting the out-of-tree driver.
