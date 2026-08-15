# OpenWrt Flashing Guide for JioExtender JE6000

Installation guide for flashing OpenWrt onto the **JioExtender JE6000**, a
MediaTek MT7981B WiFi 6 range extender. This guide assumes basic familiarity
with UART serial connections and TFTP servers.

## ⚠️ Important Warnings

> [!WARNING]
> Flashing custom firmware carries inherent risks. Make sure you understand
> these steps fully before proceeding.

> [!CAUTION]
> **DO NOT CONNECT VCC** when attaching your USB to TTL adapter. Doing so will
> permanently damage your board.

> [!IMPORTANT]
> Flashing erases the vendor dual boot layout. The `ubi` partition is
> reformatted and both vendor slots are destroyed. Confirm you can reach the
> U-Boot menu over serial **before** you start, as that is the only recovery
> path afterwards.

> [!IMPORTANT]
> **Back up every MTD partition before flashing.** `Factory` holds the WiFi
> calibration EEPROM and `MFG` holds the MAC base address; neither can be
> regenerated if lost, and without them the board will not have working radios
> or its original addresses. Do this from the initramfs, before running
> `sysupgrade`.

### Backing up the vendor partitions

Boot the initramfs first (steps 1-8), then dump all nine partitions. The
device is reachable at `192.168.1.1`.

From your computer, pull each one over SSH:

```sh
for i in 0 1 2 3 4 5 6 7 8; do
    ssh root@192.168.1.1 "cat /dev/mtd$i" > "je6000-mtd$i.bin"
done
```

Or dump them on the device first and copy them off in one go:

```sh
ssh root@192.168.1.1 'for i in 0 1 2 3 4 5 6 7 8; do dd if=/dev/mtd$i of=/tmp/mtd$i.bin; done'
scp root@192.168.1.1:/tmp/mtd*.bin .
```

The LuCI web interface at `192.168.1.1` can do the same thing without a
terminal: **System → Backup / Flash Firmware → Save mtdblock contents**, then
pick each partition in turn and download it.

Expected sizes, useful for confirming the dumps are complete:

| dev | partition | size |
|---|---|---|
| mtd0 | BL2 | 1 MiB |
| mtd1 | u-boot-env | 512 KiB |
| mtd2 | Factory | 2 MiB |
| mtd3 | FIP | 2 MiB |
| mtd4 | ubi | 140 MiB |
| mtd5 | ubi2 | 88 MiB |
| mtd6 | BDF | 2 MiB |
| mtd7 | MFG | 2 MiB |
| mtd8 | Jio-Reserved | 2 MiB |

> [!CAUTION]
> `MFG` contains your device's MAC address and factory-provisioned
> credentials. Keep these dumps private and do not publish them.

---

## Hardware Preparation

### Step 1: UART Connection

Connect your USB to TTL adapter to the UART headers on the board.

* **TX** to **RX**
* **RX** to **TX**
* **GND** to **GND**

### Step 2: Establish Serial Console

Use a serial terminal emulator such as **PuTTY**, **Tera Term**, **MobaXterm**
or **gtkterm**.

### Step 3: Serial Settings

* **Baud Rate:** `115200`
* **Data bits:** `8`
* **Stop bits:** `1`
* **Parity:** `None`
* **Flow control:** `None`

---

## Accessing the Bootloader

### Step 4: Enter the U-Boot Menu

Power on the extender while watching the serial console. The **MediaTek U-Boot
Boot Menu** appears with entries `1` through `9`, `a` and `0`:

```
*** U-Boot Boot Menu ***

    1. Startup system (Default)
    2. Upgrade firmware
    3. Upgrade ATF BL2
    4. Upgrade ATF FIP
    ...
    9. Start Web failsafe
    a. Change boot configuration
    0. U-Boot console

Press UP/DOWN to move, ENTER to select, ESC to quit
```

Use the arrow keys to select **`0. U-Boot console`** and press Enter.

### Step 5: Shell Authentication Bypass

Press `Enter` **6 times**.

*(This accounts for 2 retries each for the username and password, eventually
landing you on the default authentication shell.)*

### Step 6: Login

Enter the default factory credentials:

* **Username:** `cheetah12`
* **Password:** `RtFQm@tb9P(K6vy2`

---

## Flashing OpenWrt

### Step 7: Prepare TFTP Server

Set a static IP on your computer's ethernet interface:

* **IP Address:** `192.168.1.2`
* **Gateway:** `192.168.1.1`

Host the **initramfs image** in the root directory of your TFTP server.

### Step 8: Load and Run Initramfs

From the U-Boot console, load the image into RAM and bypass signature
verification:

```sh
setenv ipaddr 192.168.1.1
setenv serverip 192.168.1.2
tftpboot 0x46000000 openwrt-mediatek-filogic-jioextender_je6000-initramfs-kernel.bin
fdt addr $(fdtcontroladdr)
fdt rm /signature
bootm
```

Alternatively, select **`2. Upgrade firmware`** from the boot menu and transfer
the initramfs image that way.

### Step 9: Flash Sysupgrade Image

Once OpenWrt is running from RAM, copy the `sysupgrade` image to `/tmp` (via
`scp` or a local web server) and flash it:

```sh
sysupgrade -n /tmp/openwrt-mediatek-filogic-jioextender_je6000-squashfs-sysupgrade.bin
```

You can also use the LuCI web interface at `192.168.1.1` and the built-in
*Backup / Flash Firmware* page.

> [!NOTE]
> Flash from an initramfs built from **this** tree. It contains the
> `uboot-envtools` entry that generates `/etc/fw_env.config`. Without it
> `fw_setenv` fails during the upgrade, `bootcmd` is never written, and the
> board falls through to the U-Boot web failsafe on the next boot.

### Step 10: Restore the Boot Delay

The upgrade helper sets `bootdelay` to `0`, which removes the boot menu
interrupt window. Set it back once OpenWrt is up:

```sh
fw_setenv bootdelay 3
```

Verify the environment was written correctly:

```sh
fw_printenv bootcmd bootdelay
```

---

## Recovery

If the board drops to the U-Boot web failsafe, the firmware is on flash but
`bootcmd` was not written. From the U-Boot console:

```sh
setenv bootcmd 'ubi read 46000000 kernel;fdt addr $(fdtcontroladdr);fdt rm /signature;bootm 0x46000000'
setenv bootdelay 3
setenv ipaddr ''
saveenv
run bootcmd
```

---

## Hardware

| | |
|---|---|
| SoC | MediaTek MT7981B, 2x Cortex-A53 @ 1.3 GHz |
| RAM | 512 MB DDR4 |
| Flash | 256 MB SPI-NAND, Winbond W25N02KV, NMBM |
| Ethernet | 1x 1000M, SoC integrated GbE PHY on gmac1 |
| WiFi | MT7981 2x2 2.4 GHz + 2x2 5 GHz (DBDC, two shared dual-band antennas) |
| USB | none usable, both connectors are power inputs |
| LEDs | red / green / blue status |
| Buttons | reset |
| Serial | 115200 8N1, 3.3V |

## Flash Layout

Nine partitions on the NMBM-mapped device, matching the vendor bootloader's
`mtd list` and `mtdparts` exactly.

| partition | offset | size |
|---|---|---|
| BL2 | `0x000000` | 1024k |
| u-boot-env | `0x100000` | 512k |
| Factory | `0x180000` | 2048k |
| FIP | `0x380000` | 2048k |
| ubi | `0x580000` | 143360k |
| ubi2 | `0x9180000` | 90112k |
| BDF | `0xe980000` | 2048k |
| MFG | `0xeb80000` | 2048k |
| Jio-Reserved | `0xed80000` | 2048k |

The raw flash is 256 MiB. NMBM maps the lower 240 MiB and reserves the top
16 MiB for bad block management, so the partitions end at `0xef80000`.

The MTD `u-boot-env` partition is unused. The live environment is a UBI volume
inside `ubi`, hence `UBOOTENV_IN_UBI := 1`.

## MAC Addresses

A six byte base address sits at offset 0 of the `MFG` partition and is read
through a `mac-base` nvmem cell. Offsets match the stock firmware:

| offset | used by |
|---|---|
| base + 0 | ethernet, label MAC |
| base + 1 | unused (single port) |
| base + 2 | 2.4 GHz |
| base + 3 | 5 GHz |

`label-mac-device` is deliberately not set. It resolves a static `mac-address`
device tree property, which does not exist when the address comes from nvmem,
so the label MAC is set from `02_network` instead.

## Notes on the Vendor Firmware

The stock firmware ships MediaTek's **unmodified reference device tree**
(`model = "MediaTek MT7981 RFB"`), so it describes hardware this board does not
have.

**No DSA switch.** The JE6000 has one port and no MT7531. An image expecting
one fails with `mt7530-mdio ... probe ... failed with error -110`. The port is
the SoC's integrated PHY on `gmac1` in `gmii` mode.

**No usable USB.** The vendor sets `xhci` to `disabled`. The controller itself
works if enabled, probing and registering a root hub reporting `Powered`, but
no device ever enumerates because neither connector is a host port. One powers
the wall-adapter variant, the other allows powering the unit when wall mounted.
`xhci` is therefore left disabled.

**`spi1` is reference-board boilerplate.** The vendor enables it only for a
`silabs,proslic_spi` telephony chip that is not present on this board.

## Known Issues

* **Vendor SKU power limits are not applied.** The 2 MiB `BDF` partition holds
  a vendor power limit table, a header plus an MD5 and a plain text CSV of per
  channel and per rate limits for both bands. mainline mt76 has no consumer for
  it, so transmit limits come from the configured regulatory domain instead.
  Set the country code accordingly.

## Status

| | |
|---|---|
| Ethernet | working, 1 Gbps |
| WiFi 2.4 / 5 GHz | working, both radios, distinct MACs |
| LEDs | working, all three colours |
| Reset button | working |
| Flash, NMBM, all partitions | working |
| sysupgrade, boot from NAND | working |
| USB | not applicable, no host port |
