# Getting the RK3399Pro NPU working on the ASUS Tinker Edge R under Armbian (mainline 6.12)

**TL;DR** — Four independent problems gate the NPU on this board. Three are on the AP side (DTB, a one-line clock-framework patch, and a missing U-Boot clock init); the fourth is NPU firmware built for a different board. With all four addressed the NPU enumerates as `2207:0019`, `rknn_server` runs, and RKNN inference works on stock Armbian 25.05 with Linux 6.12.21.

All the evidence below is from a real board, not from reading the vendor tree.

---

## 0. Hardware and software

| | |
|---|---|
| Board | ASUS Tinker Edge R (RK3399Pro: RK3399 + RK1808-class NPU die, 2 GB LPDDR3 on the NPU side) |
| OS | Armbian 25.05 (Debian 12), Linux 6.12.21 mainline (`6.12.21-current-rockchip64`) |
| Bootloader | Armbian U-Boot 2022.07 |
| NPU userspace | `npu_powerctrl`, `npu_upgrade`, `upgrade_tool`, `npu_transfer_proxy`, `librknn_api.so` 1.7.5 |

The NPU is a computer of its own. It has no storage: on every power-up the host has to push `MiniLoaderAll.bin`, `uboot.img`, `trust.img` and `boot.img` into its RAM over USB, at which point the NPU's own Linux (buildroot, 4.4.185) starts `rknn_server`. Two USB interfaces appear: a Rockchip NTB interface (class 24) used for RKNN transfers, and an ADB interface.

**On the Tinker Edge R the NPU is USB-attached** (nets `USB20_NPU` / `USB30_NPU` behind the board's GL3523 hub). Rockchip's reference boards use PCIe instead, which matters later.

---

## 1. Symptom

`lsusb` shows no Rockchip device at all, and `npu_upgrade` fails with `failed to wait device`. `dmesg` has no mention of the NPU. If you power the NPU (`npu_powerctrl -o`) you eventually see this in `dmesg`:

```
usb 1-1.4: new high-speed USB device number 6 using xhci-hcd
usb 1-1.4: Device not responding to setup address.
usb 1-1.4: device not accepting address 6, error -71
usb 1-1-port4: unable to enumerate USB device
```

That is the NPU's USB PHY asserting D+ but never answering a SETUP packet. It was present on both mainline 6.12 and the author's vendor 6.1 tree, which is what initially made it look like a kernel problem. It is not.

---

## 2. Root cause #1 — the Type-C driver is starving the USB PHYs

`fusb302` is present on I2C (0x22) but mainline 6.12 requires two things the board DTS did not provide.

**(a) `fcs,int_n-gpios`.** The 6.x driver asks `devm_gpiod_get(dev, "fcs,int_n")`; the ASUS DTS only had `int-n-gpios`, so probe failed with `-ENOENT`.

**(b) the `connector` child node.** This one is subtle. The driver resolves properties as:

```c
/* drivers/usb/typec/tcpm/fusb302.c */
fwnode = device_get_named_child_node(dev, "connector");
if (!fwnode)
        fwnode = fwnode_create_software_node(port_props, NULL);
```

and tcpm then reads the cable list from *that* node:

```c
/* drivers/usb/typec/tcpm/tcpm.c */
ret = fwnode_property_count_u32(fwnode, "extcon-cables");
if (ret > 0) { ... } else {
        dev_info(port->dev, "No cable types defined, using default cables");
        port->extcon_cables = default_supported_cables;   /* {EXTCON_NONE} */
}
```

The Tinker Edge R DTS has no `connector` child at all, so the extcon device tcpm registers advertises `{EXTCON_NONE}`. `rockchip-usb2phy` then registers an `EXTCON_USB_HOST` notifier on it and gets `-EINVAL`:

```
rockchip-usb2phy ff770000.syscon:usb2-phy@e450: register USB HOST notifier failed
rockchip-usb2phy ff770000.syscon:usb2-phy@e450: probe with driver rockchip-usb2phy failed with error -22
```

That single failure cascades: `usb2phy@e450` never probes → `fe800000.usb`, `fe380000.usb` and `fe3a0000.usb` sit in `devices_deferred` forever → the NPU's USB ports do not exist in the system.

### Fix

```dts
#include <dt-bindings/usb/pd.h>          /* PDO_FIXED() */

&i2c8 {
        fusb0: typec-portc@22 {
                compatible = "fcs,fusb302";
                reg = <0x22>;
                pinctrl-names = "default";
                pinctrl-0 = <&fusb0_int>;
                int-n-gpios = <&gpio1 RK_PA2 GPIO_ACTIVE_HIGH>;
                fcs,int_n-gpios = <&gpio1 RK_PA2 GPIO_ACTIVE_HIGH>;
                vbus-5v-gpios = <&gpio4 RK_PD1 GPIO_ACTIVE_HIGH>;
                vbus2-5v-gpios = <&gpio2 RK_PD2 GPIO_ACTIVE_HIGH>;
                status = "okay";

                usb_con: connector {
                        compatible = "usb-c-connector";
                        label = "USB-C";
                        data-role = "dual";
                        power-role = "dual";
                        try-power-role = "sink";
                        op-sink-microwatt = <1000000>;
                        sink-pdos = <PDO_FIXED(5000, 500, PDO_FIXED_USB_COMM)>;
                        source-pdos = <PDO_FIXED(5000, 1500, PDO_FIXED_USB_COMM)>;
                        /* USB, USB_HOST, SDP, CDP, DCP, SLOW_CHARGER, ACA, AUDIO_ADAPTER */
                        extcon-cables = <1 2 5 6 9 10 12 44>;
                };
        };
};
```

The same cable set is used by `rk3399-nanopc-t4`, `rk3399-orangepi-4` and `rk3399-kobol-helios64`, which is a good sanity check.

After this, `devices_deferred` is empty (except an unrelated `spdif-sound`) and all four USB controllers register.

---

## 3. Root cause #2 — `clk_enable_count` is read-only

The vendor tool `npu_powerctrl` enables the NPU reference clocks by writing:

```
/sys/kernel/debug/clk/clk_wifi_pmu/clk_enable_count
/sys/kernel/debug/clk/rk808-clkout2/clk_enable_count
```

Mainline registers those files read-only:

```c
debugfs_create_u32("clk_enable_count", 0444, root, &core->enable_count);
```

`npu_powerctrl` prints `Error writing ... value=1` and carries on, so the clocks are never actually enabled. Rockchip's vendor trees changed this, and `npu_powerctrl` depends on that ABI.

### Fix

Small patch to `drivers/clk/clk.c`, giving the file 0644 and making a write of 1 actually call `clk_prepare_enable()` (and a write of 0 release only refcounts taken through this file, so a driver-owned clock is not torn down):

```c
static int clk_enable_count_set(void *data, u64 val)
{
        struct clk_core *core = data;
        int ret = 0;

        if (val) {
                ret = clk_prepare_enable(core->hw->clk);
        } else if (core->enable_count > 0) {
                clk_disable_unprepare(core->hw->clk);
        }
        return ret;
}
...
debugfs_create_file("clk_enable_count", 0644, root, core, &clk_enable_count_fops);
```

This is worth upstreaming in some form: distros that carry the Rockchip NPU userspace need the vendor ABI, and 0444 silently breaks it.

---

## 4. Root cause #3 — Armbian U-Boot does not set the RK3399Pro reference clock

This is the one that cost the most time, and it is invisible from Linux.

ASUS's U-Boot contains this in `arch/arm/mach-rockchip/rk3399/rk3399.c`:

```c
#if defined(CONFIG_ROCKCHIP_RK3399PRO)
        struct rk3399_pmucru *pmucru = (void *)PMUCRU_BASE;

        /* set wifi_26M to 24M and disabled by default */
        writel(0x7f002000, &pmucru->pmucru_clksel[1]);
        writel(0x01000100, &pmucru->pmucru_clkgate_con[0]);
#endif
```

Decoded: `clk_wifi_div` (PMUCRU + 0x84) is driven from `xin24m` with divider 1 — **24 MHz** — and its gate is left closed for later. Armbian's U-Boot has no RK3399PRO block at all, so the same divider stays on `ppll/26` — **26 MHz**.

The NPU's USB PHY runs from that clock. At 26 MHz it can still assert the D+ pull-up (so the host sees *a* device) but its chirp/protocol timing is wrong, which is exactly the `error -71` / "Device not responding to setup address" signature above. Changing the kernel makes no difference, because the wrong frequency was already latched by the time Linux runs.

### Fix — no bootloader flashing required

A tiny userspace helper replicates the two writes before the NPU is powered:

```python
#!/usr/bin/env python3
# /usr/local/sbin/npu-refclk-24m
import mmap, os, struct
BASE, CLKSEL1, CLKGATE0 = 0xFF750000, 0x84, 0x100
fd = os.open("/dev/mem", os.O_RDWR | os.O_SYNC)      # O_RDWR - O_RDONLY is refused
m = mmap.mmap(fd, 4096, mmap.MAP_SHARED,
              mmap.PROT_READ | mmap.PROT_WRITE, offset=BASE)
struct.pack_into("<I", m, CLKSEL1, 0x7F002000)        # xin24m / 1 = 24 MHz
struct.pack_into("<I", m, CLKGATE0, 0x01000100)       # gate off; kernel opens it later
```

Call it from the firmware loader, immediately before `npu_upgrade`:

```sh
/usr/local/sbin/npu-refclk-24m || echo "warning: could not set NPU reference clock" >&2
```

Verified both ways: at 26 MHz the NPU never enumerates; at 24 MHz it enumerates every time.

*(A proper fix is either to add the RK3399PRO block to mainline/Armbian U-Boot, or to express it as an `assigned-clock-parents`/rate on `clk_wifi_pmu` in the DTS. Neither is a prerequisite — the userspace write works.)*

---

## 5. Root cause #4 — the "latest" NPU firmware is built for a different board

With the clock fixed, the NPU enumerates and the existing firmware push from `airockchip/RK3399Pro_npu` succeeds — but `rknn_init()` then fails:

```
E RKNNAPI: rknn_init, recv(MsgLoadAck) fail, -9(ERROR_PIPE) != 368!
```

`strace` shows the host library connecting to the proxy, uploading the 4.3 MB model, and the proxy closing the socket because the USB device vanished mid-transfer. On the NPU itself:

```
fan53555-regulator 1-001c: Failed to get chip ID!
fan53555-regulator 1-0040: Failed to get chip ID!
galcore ffbc0000.npu: failed to get regulator
```

The NPU's own device tree wants:

```dts
npu@ffbc0000 {
        npu-supply   = <&tcs452x_1c>;     /* tcs452x @ 0x1c */
        npu_1-supply = <&syr837_40>;      /* syr837  @ 0x40 */
};
```

Those are the PMICs on **Rockchip's RK3399Pro EVB**. The Tinker Edge R uses a single **fan53555 at 0x60**:

```dts
i2c@ff500000 {
        fan53555@60 {
                compatible = "fcs,fan53555";
                reg = <0x60>;
                regulator-name = "vdd_npu";
                regulator-min-microvolt = <800000>;
                regulator-max-microvolt = <850000>;
                ...
        };
};
```

Because the referenced regulators never register, `galcore` aborts its probe, the NPU core is never initialised, and the NTB transfer dies. Patching the device tree alone is **not** enough either: that firmware's kernel also rejects the board's chip (`Chip ID 0 with rev 0 not supported`).

### Fix — hybrid `boot.img`

The ASUS stock Debian image contains the correct firmware set. Extract it from the image:

```
GPT partition 8 (rootfs), start LBA 499712
dd if=Tinker_Edge_R-Debian-Buster-*.img of=rootfs.img bs=512 skip=499712 count=8120782
debugfs -R "dump /usr/share/npu_fw/boot.img out/boot.img" rootfs.img
```

Both `boot.img`s are Android boot v0, page size 2048, same kernel size (4,974,600 B), both Linux 4.4.185. The difference is the kernel contents and the ramdisk:

| boot.img | kernel | ramdisk (`rknn_server`) | result |
|---|---|---|---|
| airockchip (21,183,488 B) | airockchip | 1.7.5 | galcore fails, `ERROR_PIPE` |
| ASUS stock (20,307,968 B) | ASUS | 1.6.0 | works |
| **hybrid** | **ASUS** | **airockchip 1.7.5** | **works, 1.7.5** |

Rebuild with the ASUS header (ramdisk size rewritten), the ASUS kernel, the ASUS DTB and the airockchip ramdisk:

```python
import struct
ps = 2048
ru = lambda x: (x + ps - 1) // ps * ps
u32 = lambda d, o: struct.unpack_from("<I", d, o)[0]

asus = open("boot.img.asus", "rb").read()
rk   = open("boot.img.airockchip", "rb").read()
ks, rs_as, ss = u32(asus, 8), u32(asus, 16), u32(asus, 24)

kernel  = asus[ps:ps + ks]
second  = asus[ps + ru(ks) + ru(rs_as): ps + ru(ks) + ru(rs_as) + ss]
ramdisk = rk[ps + ru(u32(rk, 8)): ps + ru(u32(rk, 8)) + u32(rk, 16)]

hdr = bytearray(asus[:ps])
struct.pack_into("<I", hdr, 16, len(ramdisk))
pad = lambda b: b + bytes(ru(len(b)) - len(b))
open("boot.hybrid.img", "wb").write(bytes(hdr) + pad(kernel) + pad(ramdisk) + pad(second))
```

Flash with the ASUS loader/uboot/trust (the airockchip loader set also works, but the ASUS one matches the board):

```
NPU_FW_DIR=/usr/share/npu_fw npu_upgrade MiniLoaderAll.bin uboot.img trust.img boot.img
```

---

## 6. Result

```
$ uname -r
6.12.21-current-rockchip64-npu+

$ lsusb | grep 2207
Bus 002 Device 004: ID 2207:0019 Fuzhou Rockchip Electronics Company rk3xxx

$ npu_transfer_proxy devices
e4d8618710babd94    53a67580    USB_DEVICE
```

Inference (C demo from `RK3399Pro_npu`, mobilenet_v1, the bundled dog picture):

```
rknn_run
 --- Top5 ---
156: 0.853516     # Pekinese
155: 0.091980
205: 0.013626
284: 0.006485
194: 0.002020
```

And from Python 3.11, without the cp310 wheel, by calling the C API through `ctypes`:

```
rknn api : 1.7.5 (bb79b30 build: 2023-07-18 16:49:12)
rknn drv : 1.7.5 (bb79b30 build: 2023-07-18 10:49:14)
--- Top5 ---
156: 0.784668
155: 0.129272
```

(The small difference is the JPEG rescale in the Python path, not the NPU.)

---

## 7. Complete checklist

1. **DTS**: add `fcs,int_n-gpios`, a `connector` child with `usb-c-connector` + `extcon-cables`, `#include <dt-bindings/usb/pd.h>`, and `snps,dis-u3-autosuspend-quirk` on both dwc3 nodes.
2. **Kernel**: make `clk_enable_count` writable (0644) with real `clk_prepare_enable` semantics.
3. **Clock**: write `0x7f002000` → PMUCRU+0x84 and `0x01000100` → PMUCRU+0x100 before powering the NPU (24 MHz).
4. **Firmware**: `boot.img` must carry the ASUS kernel + DTB (fan53555 PMIC); the 1.7.5 ramdisk can be grafted on.
5. **Services**: `npu-firmware.service` (runs the clock fix + `npu_upgrade`) and `npu-transfer-proxy.service` enabled at boot.

## 8. Caveats

- The clock workaround is a userspace poke at `PMUCRU`; a kernel/U-Boot fix would be cleaner. If a future kernel driver enables/disables `clk_wifi_div`, re-check the gate.
- The hybrid firmware is not something the vendor ships. It works, but if you need features that depend on the airockchip kernel (rather than its ramdisk), you fall back to the ASUS 1.6.0 userspace.
- The NPU has no storage; every boot re-flashes it. Keep `boot.img.orig` backups.
- Long-run/high-rate inference stability was not characterised.

## 9. References

- frankurcrazy/tinker-edge-r — the 6.1 port and the research notes this work started from
- airockchip/RK3399Pro_npu — NPU firmware, `npu_transfer_proxy`, `librknn_api`
- ASUS `Tinker_Edge_R-Debian-Buster-v2.0.5-20220217.zip` — the stock firmware set
- Mainline: `drivers/usb/typec/tcpm/{fusb302,tcpm}.c`, `drivers/phy/rockchip/phy-rockchip-inno-usb2.c`, `drivers/clk/clk.c`
