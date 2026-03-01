# ThinkPad W541 — Definitive Upstream Coreboot Guide with Working dGPU (K1100M)

> **Firmware:** Upstream coreboot (NOT Libreboot/lbmk — Libreboot disables the K1100M)  
> **Board definition:** `lenovo/t540p` (covers W540/W541)  
> **GPU approach:** Hybrid — `libgfxinit` initialises the Intel HD 4600 iGPU + extracted Nvidia VGA ROM initialises the K1100M  
> **Host OS:** Debian / Ubuntu  
> **External programmer:** Raspberry Pi Pico (RP2040) running Libreboot serprog firmware  
> **Last validated:** February 2026

---

## Why Upstream Coreboot, Not Libreboot

Libreboot's `w541_12mb` target uses NRI (libre raminit, no MRC blob) and **does not load a VGA option ROM for the Quadro**. The K1100M appears on the PCI bus but boots disabled. If you need the K1100M functional — for external display outputs through the dock, or for GPU compute — you must use upstream coreboot with the full blob set including the extracted Nvidia VGA BIOS.

---

## Part 1 — Hardware You Need

### 1.1 Pico SPI Programmer Parts List

| Item | Notes | Approx. Cost |
|------|-------|-------------|
| Raspberry Pi Pico (RP2040) | **NOT Pico 2 (RP2350)** — RP2350 has serprog stability issues | ~£4 |
| SOIC-8 test clip | Pomona 5250 is best; generic clips work | ~£3–8 |
| Second SOIC-8 clip (or IC test hook) | For holding /CS on the idle chip high | ~£3–5 |
| 47Ω resistor | Series resistor on /CS line to idle chip; a few spares | pennies |
| Female–female Dupont wires | 10–15 wires | ~£2 |
| USB Micro-B cable | Pico to host | — |
| IPA (isopropyl alcohol) | Clean chip legs before clipping | — |

> **Do not use a CH341A.** Many boards output 5V on data lines and will destroy the flash chip and potentially the PCH.

---

## Part 2 — Converting the Pico to an SPI Programmer

### 2.1 Flash serprog Firmware

Use Libreboot's pico-serprog firmware — it has multi-CS handling and tuned 12 mA drive strength for this platform. Build via lbmk (you do not need to use lbmk for the coreboot build itself):

```bash
git clone https://codeberg.org/libreboot/lbmk
cd lbmk
./mk dependencies debian
./mk -b pico-serprog
# Output: bin/serprog_pico/serprog_pico.uf2
```

Flash onto the Pico:

1. Hold **BOOTSEL**, plug Pico into USB, release BOOTSEL — mounts as `RPI-RP2`
2. `cp bin/serprog_pico/serprog_pico.uf2 /media/$USER/RPI-RP2/`
3. Pico reboots. Verify: `sudo dmesg | grep ttyACM` → `ttyACM0: USB ACM device`

```bash
sudo usermod -aG dialout $USER   # avoid sudo on every flashprog call (re-login after)
```

### 2.2 Pico GPIO Pinout

> **Critical:** GP3 = MOSI and GP5 = /CS. These are commonly documented in reverse. Using the wrong mapping gives garbage reads.

```
Pico (corner near USB):

Pin 4  — GP2  — SCK  ──────────────── SOIC-8 Pin 6
Pin 5  — GP3  — MOSI ──────────────── SOIC-8 Pin 5
Pin 6  — GP4  — MISO ──────────────── SOIC-8 Pin 2
Pin 7  — GP5  — /CS  ──────────────── SOIC-8 Pin 1
Pin 36 — 3V3(OUT) ──┬──────────────── SOIC-8 Pin 8 (VCC)
                    ├──────────────── SOIC-8 Pin 3 (/WP)
                    ├──────────────── SOIC-8 Pin 7 (/HOLD)
                    └──[47Ω]────────── idle chip Pin 1 (/CS)
Pin 38 — GND ──────────────────────── SOIC-8 Pin 4
```

Full table:

| SOIC-8 Pin | Signal | Pico Physical Pin | Pico GPIO |
|:---:|:---:|:---:|:---:|
| 1 | /CS | Pin 7 | GP5 |
| 2 | MISO | Pin 6 | GP4 |
| 3 | /WP | Pin 36 | 3V3(OUT) |
| 4 | GND | Pin 38 | GND |
| 5 | MOSI | Pin 5 | GP3 |
| 6 | SCK | Pin 4 | GP2 |
| 7 | /HOLD | Pin 36 | 3V3(OUT) |
| 8 | VCC | Pin 36 | 3V3(OUT) |

**Pin 36 carries 4 connections:** VCC, /WP, /HOLD, and the 47Ω resistor to the idle chip's pin 1. Twist and solder, or use a small breadboard as a distribution bus. Pin 37 (3V3_EN) is NOT a power output — do not use it.

/WP and /HOLD are active-low; holding them high disables write protection and the hold function. The W541 PCB has onboard pull-ups but direct-driving from 3V3(OUT) is stronger and required per repair.wiki's note that the programmer must "strongly drive HOLD# high" on this platform.

---

## Part 3 — W541 Dual Flash Chip Layout

| Chip | Location | Size | Role |
|------|----------|------|------|
| Chip 1 | Left, near white ribbon connector | **8 MiB** | `chip1.bin` / `bottom.rom` |
| Chip 2 | Right, closer to board edge | **4 MiB** | `chip2.bin` / `top.rom` |

Both chips share MOSI/MISO PCB traces at zero ohms. When you clip Chip 1 and power it, Chip 2 also powers up parasitically and its /CS pin **floats** — causing random interference with reads and writes. The fix is to hold pin 1 (/CS) of the idle chip HIGH through a 47Ω resistor to 3V3(OUT). This is the only connection needed on the idle chip — everything else is already handled by the shared PCB traces.

**SOIC-8 pinout reference:**
```
      ┌───────┐
 /CS 1│●      │8 VCC
  DO 2│       │7 /HOLD
 /WP 3│       │6 CLK
 GND 4│       │5 DI
      └───────┘
```
Pin 1 = dot on IC package. Pin numbering goes anti-clockwise.

---

## Part 4 — Step 0: Extract Blobs BEFORE Flashing (Run on Stock BIOS)

**Do this while the W541 is still running the Lenovo factory BIOS.** Some extractions require the GPU to be active and initialised.

### 4.1 Extract the Nvidia K1100M VGA ROM (via PCI sysfs)

This is the **only working method** without specialised tools. Do not use Python-based extraction tools for this — use the PCI sysfs interface directly.

Boot the W541 into Linux (any live USB works). The Nvidia driver must have the GPU visible on the PCI bus.

```bash
# Find the K1100M's PCI address
lspci | grep -i nvidia
# Example: 01:00.0 3D controller: NVIDIA Corporation GK107GLM [Quadro K1100M]

# Enable ROM readback (replace 01:00.0 with your address)
echo 1 | sudo tee /sys/bus/pci/devices/0000:01:00.0/rom

# Extract the ROM
sudo dd if=/sys/bus/pci/devices/0000:01:00.0/rom of=nvidia_k1100m.rom bs=1k

# Disable ROM readback
echo 0 | sudo tee /sys/bus/pci/devices/0000:01:00.0/rom

# Verify: must start with 55 AA and contain "NVIDIA"
hexdump -C nvidia_k1100m.rom | head -3
strings nvidia_k1100m.rom | grep -i nvidia | head -3
```

The ROM must start with bytes `55 aa`. If the file is empty or missing the signature, the GPU was not active — try loading the nouveau or proprietary driver first:

```bash
sudo modprobe nouveau   # or: sudo modprobe nvidia
# Then retry the extraction
```

Copy `nvidia_k1100m.rom` somewhere safe (USB stick).

### 4.2 Extract the Intel VBT (Video BIOS Table)

The VBT contains display configuration specific to the W541's panel and output routing. Extract from the running system:

```bash
sudo apt install intel-gpu-tools
sudo intel_bios_dumper
# Creates: intel_bios_dumper.bin in current directory

# Extract VBT portion (at offset 0x8000, 8 KiB)
dd if=intel_bios_dumper.bin of=intel_vbt.bin bs=1 skip=32768 count=8192
```

Verify:
```bash
hexdump -C intel_vbt.bin | head -3
# Should contain "VBT" string near the start
strings intel_vbt.bin | head -5
```

---

## Part 5 — Disassembly and Reading the Factory ROM

### 5.1 Disassembly

1. Remove battery and AC power. Hold power button 10 seconds.
2. Remove bottom cover screws; pop the cover.
3. Disconnect the main battery connector.
4. The two SOIC-8 flash chips are near the LCD cable connector — small, close together.
5. Clean chip legs with IPA before clipping.

**Do not fully reassemble until you've confirmed a successful boot test.** Bench-test first: just AC power + display cable. If it boots, then button up.

### 5.2 Triple-Read Both Chips

Install flashprog on your host machine (the machine with the Pico attached, not the W541):

```bash
sudo apt install flashprog
# If not available: sudo apt install flashrom  (commands are identical)
```

**Chip 1 (8 MiB): full clip on Chip 1, idle hook on Chip 2 pin 1**

```bash
sudo flashprog -p serprog:dev=/dev/ttyACM0,spispeed=16M -r chip1_r1.bin
sudo flashprog -p serprog:dev=/dev/ttyACM0,spispeed=16M -r chip1_r2.bin
sudo flashprog -p serprog:dev=/dev/ttyACM0,spispeed=16M -r chip1_r3.bin
sha256sum chip1_r1.bin chip1_r2.bin chip1_r3.bin
# All three must match. Size: exactly 8388608 bytes.
```

**Chip 2 (4 MiB): move clip to Chip 2, move idle hook to Chip 1 pin 1**

```bash
sudo flashprog -p serprog:dev=/dev/ttyACM0,spispeed=16M -r chip2_r1.bin
sudo flashprog -p serprog:dev=/dev/ttyACM0,spispeed=16M -r chip2_r2.bin
sudo flashprog -p serprog:dev=/dev/ttyACM0,spispeed=16M -r chip2_r3.bin
sha256sum chip2_r1.bin chip2_r2.bin chip2_r3.bin
# All three must match. Size: exactly 4194304 bytes.
```

**Merge into a single 12 MiB backup:**

```bash
cat chip1_r1.bin chip2_r1.bin > factory_12mb.bin
ls -la factory_12mb.bin
# Must be 12582912 bytes
```

Keep `factory_12mb.bin` somewhere safe — it's your recovery image.

> **Inconsistent reads?** Clean chip legs with IPA. Try `spispeed=4M`. Specify chip manually: `flashprog ... -c "W25Q64.V"`. Confirm idle chip's /CS is held high.

---

## Part 6 — Extract Blobs from the Factory Dump

```bash
# Get upstream coreboot (needed for ifdtool and other utilities)
git clone https://review.coreboot.org/coreboot.git
cd coreboot
git submodule update --init --checkout

# Build ifdtool
cd util/ifdtool
make
cd ../..

# Extract all flash regions from the merged factory dump
util/ifdtool/ifdtool -x ../factory_12mb.bin
```

This produces:
```
flashregion_0_flashdescriptor.bin   (Intel Flash Descriptor / IFD)
flashregion_1_bios.bin              (BIOS region — contains the coreboot CBFS area)
flashregion_2_intel_me.bin          (Intel ME firmware)
flashregion_3_gbe.bin               (Gigabit Ethernet NVM — note your MAC)
```

### 6.1 Get mrc.bin (Haswell Memory Reference Code)

The MRC is the same binary across all Haswell platforms. If you have a working T440p coreboot setup, copy that `mrc.bin` directly — it will work on the W541.

Alternatively, extract it from any Haswell ThinkPad factory BIOS dump using UEFITool (look for the "Intel Reference Code" DXE module), or use the copy in coreboot's 3rdparty blobs if you have a valid binary from a prior build.

```bash
# Check if it exists in coreboot's blobs submodule
ls 3rdparty/blobs/northbridge/intel/haswell/mrc.bin
```

### 6.2 Neuter the Intel ME

```bash
git clone https://github.com/corna/me_cleaner.git

python3 me_cleaner/me_cleaner.py \
  -S -d flashregion_2_intel_me.bin \
  -O me_cleaned.bin
```

`-S` sets the HAP/AltMeDisable bit (ME shuts down after hardware init). `-d` strips non-essential ME modules. The result `me_cleaned.bin` replaces the original ME region.

---

## Part 7 — Place Blobs in the Coreboot Tree

```bash
cd coreboot  # if not already there

# Create blob directories
mkdir -p 3rdparty/blobs/northbridge/intel/haswell/
mkdir -p 3rdparty/blobs/southbridge/intel/lynxpoint/
mkdir -p src/mainboard/lenovo/t540p/

# Copy blobs
cp mrc.bin                          3rdparty/blobs/northbridge/intel/haswell/mrc.bin
cp ../me_cleaned.bin                3rdparty/blobs/southbridge/intel/lynxpoint/me.bin
cp ../flashregion_3_gbe.bin        3rdparty/blobs/southbridge/intel/lynxpoint/gbe.bin
cp ../flashregion_0_flashdescriptor.bin 3rdparty/blobs/southbridge/intel/lynxpoint/descriptor.bin
cp ../intel_vbt.bin                 src/mainboard/lenovo/t540p/data.vbt
cp ../nvidia_k1100m.rom             nvidia_k1100m.rom
```

---

## Part 8 — Coreboot .config

Save this as `.config` inside the coreboot directory before running `make olddefconfig`:

```
#
# Coreboot configuration for ThinkPad W541 (board: lenovo/t540p)
# Upstream coreboot — hybrid iGPU + K1100M dGPU
#

CONFIG_VENDOR_LENOVO=y
CONFIG_BOARD_LENOVO_THINKPAD_W541=y

# CPU / Chipset — Haswell + Lynx Point (8 Series PCH)
CONFIG_NORTHBRIDGE_INTEL_HASWELL=y
CONFIG_SOUTHBRIDGE_INTEL_LYNXPOINT=y
CONFIG_CPU_INTEL_HASWELL=y
CONFIG_CPU_MICROCODE_CBFS_EXTERNAL_BINS=y

# Memory init — proprietary Intel MRC blob
CONFIG_HAVE_MRC=y
CONFIG_MRC_FILE="3rdparty/blobs/northbridge/intel/haswell/mrc.bin"

# Graphics — hybrid: libgfxinit for Intel iGPU + Nvidia VGA ROM for K1100M
CONFIG_MAINBOARD_HAS_NATIVE_VGA_INIT=y
CONFIG_MAINBOARD_DO_NATIVE_VGA_INIT=y
CONFIG_INTEL_GMA_HAVE_VBT=y
CONFIG_INTEL_GMA_VBT_FILE="src/mainboard/lenovo/t540p/data.vbt"

# Run the extracted Nvidia VGA option ROM to initialise K1100M
CONFIG_DRIVERS_OPTION_ROM_REALMODE_x86=y
CONFIG_VGA_OPTION_ROM_FILE="nvidia_k1100m.rom"
CONFIG_VGA_OPTION_ROM_ID="10de,0ff6"

# Display framebuffer
CONFIG_FRAMEBUFFER_SET_VESA_MODE=y
CONFIG_FRAMEBUFFER_VESA_MODE_118=y

# Intel ME — neutered binary
CONFIG_HAVE_ME_BIN=y
CONFIG_ME_BIN_PATH="3rdparty/blobs/southbridge/intel/lynxpoint/me.bin"

# Gigabit Ethernet NVM
CONFIG_HAVE_GBE_BIN=y
CONFIG_GBE_BIN_PATH="3rdparty/blobs/southbridge/intel/lynxpoint/gbe.bin"

# Flash Descriptor
CONFIG_HAVE_IFD_BIN=y
CONFIG_IFD_BIN_PATH="3rdparty/blobs/southbridge/intel/lynxpoint/descriptor.bin"

# Flash map — 12 MiB total (8+4)
CONFIG_CBFS_SIZE=0x600000
CONFIG_FMDFILE="src/mainboard/lenovo/t540p/board.fmd"

# Embedded Controller
CONFIG_EC_LENOVO_PMH7=y
CONFIG_EC_LENOVO_H8=y

# Super I/O
CONFIG_SUPERIO_NSC_PC87382=y

# TPM
CONFIG_LPC_TPM=y
CONFIG_MAINBOARD_HAS_TPM1=y

# ACPI
CONFIG_GENERATE_ACPI_TABLES=y
CONFIG_BOARD_ACPI_SLEEP_GNVS=y

# Generic drivers
CONFIG_PS2_KEYBOARD=y

# Payload — choose one:
CONFIG_PAYLOAD_SEABIOS=y
CONFIG_SEABIOS_STABLE=y
# Or for GRUB:
# CONFIG_PAYLOAD_GRUB2=y
# Or for Tianocore UEFI:
# CONFIG_PAYLOAD_TIANOCORE=y
```

---

## Part 9 — Build Coreboot

### 9.1 Install Build Dependencies

```bash
sudo apt install -y \
  build-essential gnat flex bison libncurses5-dev wget zlib1g-dev \
  git python3 libelf-dev nasm imagemagick pkg-config libssl-dev \
  iasl unifont
```

### 9.2 Build the Toolchain

```bash
cd coreboot
make crossgcc-i386 CPUS=$(nproc)
# Takes 30–90 minutes on first run
```

### 9.3 Configure and Build

```bash
# Apply the .config you saved above
make olddefconfig

# (Optional) verify or tweak settings interactively
make menuconfig

# Build
make -j$(nproc)
```

Output: `build/coreboot.rom`

Verify the size:
```bash
ls -la build/coreboot.rom
# Must be exactly 12582912 bytes (12 MiB)
```

---

## Part 10 — Split the ROM for Dual-Chip Flashing

```bash
dd if=build/coreboot.rom of=bottom.rom bs=1M count=8
dd if=build/coreboot.rom of=top.rom    bs=1M skip=8

ls -la bottom.rom top.rom
# bottom.rom = 8388608 bytes  → Chip 1 (near white connector)
# top.rom    = 4194304 bytes  → Chip 2 (near board edge)
```

If you flash the wrong size to a chip, flashprog refuses with a size mismatch — no damage, just swap them.

---

## Part 11 — Flash

### 11.1 Flash Chip 1 (8 MiB)

Full clip on Chip 1, 47Ω hook on Chip 2 pin 1 → 3V3(OUT).

```bash
sudo flashprog -p serprog:dev=/dev/ttyACM0,spispeed=16M -w bottom.rom
sudo flashprog -p serprog:dev=/dev/ttyACM0,spispeed=16M -v bottom.rom
# Must report: VERIFIED
```

### 11.2 Flash Chip 2 (4 MiB)

Move clip to Chip 2, move hook to Chip 1 pin 1 → 3V3(OUT).

```bash
sudo flashprog -p serprog:dev=/dev/ttyACM0,spispeed=16M -w top.rom
sudo flashprog -p serprog:dev=/dev/ttyACM0,spispeed=16M -v top.rom
# Must report: VERIFIED
```

---

## Part 12 — First Boot Test

1. Disconnect the Pico from USB.
2. Connect AC power (no battery needed for bench test).
3. Power on — you should see a coreboot/SeaBIOS/GRUB screen within 30 seconds.
4. Both GPUs should initialise: coreboot logs at the top of the screen will show iGPU init and the Nvidia VGA ROM executing.

If no display after 2 minutes: re-attach clips and flash `factory_12mb.bin` back (split it the same way):

```bash
dd if=factory_12mb.bin of=factory_chip1.bin bs=1M count=8
dd if=factory_12mb.bin of=factory_chip2.bin bs=1M skip=8
sudo flashprog -p serprog:dev=/dev/ttyACM0,spispeed=16M -w factory_chip1.bin
# Move clip, hook
sudo flashprog -p serprog:dev=/dev/ttyACM0,spispeed=16M -w factory_chip2.bin
```

---

## Part 13 — Post-Install: dGPU in Linux

```bash
lspci | grep -E "VGA|3D|Display"
# 00:02.0 VGA compatible controller: Intel Corporation 4th Gen Core ... Integrated Graphics
# 01:00.0 3D controller: NVIDIA Corporation GK107GLM [Quadro K1100M]
```

**nouveau (default):**
```bash
# GPU offload
DRI_PRIME=1 <application>
```

**Proprietary Nvidia 390xx (legacy, EOL):**
```bash
sudo apt install nvidia-legacy-390xx-driver
```

**If thinkpad_acpi fails to load:**
```bash
# /etc/modprobe.d/thinkpad_acpi.conf
options thinkpad_acpi force_load=1
```

---

## Part 14 — Internal Re-Flashing (Future Updates)

Once coreboot is running, subsequent updates can be done internally:

```bash
sudo flashprog -p internal -w build/coreboot.rom
```

If your kernel restricts iomem access:
```bash
# Add to GRUB_CMDLINE_LINUX in /etc/default/grub:
GRUB_CMDLINE_LINUX="iomem=relaxed"
sudo update-grub
# Reboot, then retry
```

Internal flashing uses the full 12 MiB ROM — no splitting needed.

---

## Part 15 — Troubleshooting

**Nvidia ROM not found / empty file from PCI sysfs:**
The GPU must be active and claimed by a driver. `sudo modprobe nouveau` or load the proprietary driver, then retry the extraction. Verify the PCI address matches your GPU's address from `lspci`.

**Reads fail or are inconsistent:**
IPA on chip legs. Try `spispeed=4M`. Specify chip: `flashprog ... -c "W25Q64.V"`. Confirm idle chip's /CS is held high.

**Boot shows black screen:**
Both chips verified? If yes, suspect the Nvidia VGA ROM — verify `55 aa` signature and `strings nvidia_k1100m.rom | grep -i nvidia`. Check `.config` VGA ROM path and PCI ID (`10de,0ff6` for K1100M). Try building without the Nvidia ROM first to confirm the iGPU init works, then add it back.

**K1100M not in lspci after boot:**
If the VGA ROM executed but the GPU isn't showing under Linux, check ACPI power management: `sudo sh -c 'echo on > /sys/bus/pci/devices/0000:01:00.0/power/control'`.

**build/coreboot.rom is wrong size:**
The flash map (board.fmd) must match the actual chip layout. The W541 is 12 MiB total (8+4). If the build produces something else, check `CONFIG_CBFS_SIZE` and `CONFIG_FMDFILE`.

---

## Appendix A — What Was Discarded / Doesn't Work

- **Libreboot/lbmk for a working K1100M** — Libreboot NRI does not load Nvidia VGA ROMs; K1100M is effectively disabled. Use upstream coreboot as described in this guide.
- **Python-based Nvidia VGA extraction tools** — use PCI sysfs + `dd` as shown in Part 4.
- **stacksmashing pico-serprog** — use Libreboot's fork (multi-CS, tuned drive strength).
- **Raspberry Pi Pico 2 (RP2350)** — serprog stability issues; use original Pico (RP2040) only.
- **flashrom** — replaced by flashprog; use flashprog throughout.
- **Wrong Pico pinout (GP3↔GP5 swapped)** — GP5=/CS, GP3=MOSI as shown in Part 2.
- **Holding pin 7 of idle chip** — it's pin 1 (/CS) that matters. Pin 7 is already pulled up by the PCB.
- **Single-chip assumption** — the W541 has two chips; any guide that doesn't address this produces unreliable results.
- **`CONFIG_BOARD_LENOVO_THINKPAD_W541` vs `T540P`** — the W541 uses the T540p board definition in upstream coreboot; check that the board Kconfig option matches what `make menuconfig` shows for your coreboot revision.

---

## Appendix B — Blob Summary

| Blob | Source | Destination in coreboot tree |
|------|--------|------------------------------|
| `mrc.bin` | T440p coreboot build (Haswell-universal) | `3rdparty/blobs/northbridge/intel/haswell/mrc.bin` |
| `me.bin` | factory dump → ifdtool → me_cleaner | `3rdparty/blobs/southbridge/intel/lynxpoint/me.bin` |
| `gbe.bin` | factory dump → ifdtool | `3rdparty/blobs/southbridge/intel/lynxpoint/gbe.bin` |
| `descriptor.bin` | factory dump → ifdtool | `3rdparty/blobs/southbridge/intel/lynxpoint/descriptor.bin` |
| `data.vbt` | `intel_bios_dumper` on running system | `src/mainboard/lenovo/t540p/data.vbt` |
| `nvidia_k1100m.rom` | PCI sysfs on running stock BIOS | `./nvidia_k1100m.rom` (coreboot root) |

---

*Guide corrected from direct hands-on experience, February 2026.*
