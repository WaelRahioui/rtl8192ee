rtl8192ee
=========

Stand-alone (out-of-tree) driver for the Realtek RTL8192EE PCIe wireless adapter
(PCI ID `10ec:818b`), based on Realtek's vendor driver. This is a fork of
[lwfinger/rtl8192ee](https://github.com/lwfinger/rtl8192ee) with build and
runtime fixes for current kernels.

Builds and runs on Linux 6.8 (Ubuntu 24.04, Linux Mint 22). Older kernels should
still build but are no longer tested.

The in-kernel `rtl8192ee` driver (rtlwifi) works for most people. Use this one if
you need access-point mode, or want to compare it against the in-kernel driver.
Always `git pull` before reporting a build problem.

Requirements
------------

- `make`, `gcc` and the headers for the running kernel
  (`linux-headers-$(uname -r)` on Debian/Ubuntu/Mint)
- Secure Boot disabled, or your own MOK key to sign the module (it is unsigned)

Build and install
-----------------

    make
    sudo make install
    echo "blacklist rtl8192ee" | sudo tee /etc/modprobe.d/blacklist-rtl8192ee.conf
    echo "8192ee" | sudo tee /etc/modules-load.d/8192ee.conf

Or with DKMS, so the module is rebuilt automatically on kernel updates
(install the `dkms` package first):

    sudo dkms add .
    sudo dkms build 8192ee/1.1
    sudo dkms install 8192ee/1.1

To remove it again: `sudo dkms remove 8192ee/1.1 --all`.

Module options
--------------

On laptops where the card stops working after a while, disable the driver's
power management in `/etc/modprobe.d/8192ee.conf`:

    options 8192ee rtw_ips_mode=0 rtw_power_mgnt=0 rtw_pci_aspm_enable=0

`modinfo 8192ee` lists all options.

Known hardware issue: card drops off the PCIe bus
-------------------------------------------------

On some laptops (seen on a ThinkPad X240) the card stops responding entirely
after its PCIe link idles into ASPM L1: config space reads `0xff`, the in-kernel
driver logs `Init MAC failed`, and only a full power-off recovers the card. This
is independent of the driver. Fix it by clearing ASPM L1 (and the L1 PM
substates) on the card's root port at boot, for example with `setpci` from a
systemd oneshot service that runs before NetworkManager:

    setpci -s <root port> CAP_EXP+0x10.w=$(printf '%04x' $(( 0x$(setpci -s <root port> CAP_EXP+0x10.w) & ~0x3 )))

Changes for kernel 6.x in this fork
-----------------------------------

- `change_beacon` takes `struct cfg80211_ap_update` (6.4+).
- `set_monitor_channel` takes a `struct net_device` (6.8+).
- `regd_name` is a real flexible array member. Kernels since 6.5 build with
  `-fstrict-flex-arrays=3`, which made the old zero-length array a zero-byte
  object and FORTIFY_SOURCE then panicked in `strlen` while the module loaded.
- The module is compiled with `-fstrict-flex-arrays=0` and without UBSAN
  instrumentation: the vendor code has many `[1]`-style arrays that the
  kernel's bounds checker would otherwise report on every access.

Using as AP
-----------

Reference: TL-WN881ND v2. This device can broadcast on channels 1-13.
With hostapd, set the proper `ht_capab` for this device:

    HT_CAPAB=[RX-STBC1][SHORT-GI-40][SHORT-GI-20][DSSS_CCK-40][MAX-AMSDU-7935]

Optionally enable wideband if you have no neighbours. It increases throughput
but clients further away may fail to connect:

    HT_CAPAB=[HT40+][RX-STBC1][SHORT-GI-40][SHORT-GI-20][DSSS_CCK-40][MAX-AMSDU-7935]   (channels 1-7)
    HT_CAPAB=[HT40-][RX-STBC1][SHORT-GI-40][SHORT-GI-20][DSSS_CCK-40][MAX-AMSDU-7935]   (channels 5-13)

Changing transmit power
-----------------------

Transmit power cannot be changed with `iw`/`iwconfig` on this driver. It can be
changed at compile time in `hal/rtl8192e/rtl8192e_phycfg.c`:

- `transmit_power_boost` (default 2): added to the driver's computed power
  index; negative values reduce power, 0 disables.
- `transmit_power_override` (default 0): one fixed power index (1-63) for all
  transmissions, overriding the driver's calculation; 0 disables.

These may override your country's regulations, and running at high power for
long periods can overheat the card.
