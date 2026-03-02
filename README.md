# ThinkPad W541 — Definitive Upstream Coreboot Guide with Working dGPU (K1100M / K2100M)

> **Firmware:** Upstream coreboot (NOT Libreboot/lbmk — under Libreboot the dGPU does not appear on the PCI bus at all)  
> **Board definition:** `lenovo/t540p` (covers W540/W541)  
> **GPU approach:** Hybrid — `libgfxinit` initialises the Intel HD 4600 iGPU + extracted Nvidia VGA ROM initialises the K1100M or K2100M  
> **Tested:** K1100M ✓ confirmed working | K2100M — should work in principle (same extraction method, different PCI ID), not personally tested  
> **Host OS:** Debian / Ubuntu  
> **External programmer:** Raspberry Pi 3/4 (linux_spi) **or** Raspberry Pi Pico (RP2040) running Libreboot serprog firmware  
> **Last validated:** February 2026

---

## Why Upstream Coreboot, Not Libreboot

This guide took about two days of work in total. The Libreboot path was taken first — it's well-documented, the tooling is polished, and the W541 is officially supported. The problem emerged when the dGPU was needed: under Libreboot the K1100M/K2100M **does not appear on the PCI bus at all** — fully absent from `lspci`, not merely disabled or power-gated. Libreboot's `w541_12mb` target uses NRI (libre raminit, no MRC blob) and does not load a VGA option ROM for the Quadro by design.

Switching to upstream coreboot turned out to be the more straightforward path once the blob extraction workflow was established. The `.config` structure is well-understood, the Nvidia VGA ROM mechanism works cleanly, and the result is a machine with both GPUs functional.

---

## Part 1 — Hardware You Need

### 1.1 Common to Both Programmer Methods

| Item | Notes | Approx. Cost |
|------|-------|-------------|
| SOIC-8 test clip | Pomona 5250 is best; generic clips work | ~£3–8 |
| SDK08 test clip (or IC test hook) | **Best option** for holding /CS on the idle chip high — individual spring-loaded probes make it easy to solder an inline 47Ω resistor to the single pin you need. An SOIC-8 test clip also works but engages all 8 pins unnecessarily. | ~£3–8 |
| Resistor for /CS idle line | 47Ω is the textbook value but anything from ~47Ω to 1kΩ works fine in practice — 1kΩ confirmed working. Solder inline to the SDK08 probe wire; a few spares | pennies |
| Female–female Dupont wires | 10–15 wires | ~£2 |
| IPA (isopropyl alcohol) | Clean chip legs before clipping | — |

> **Do not use a CH341A.** Many boards output 5V on data lines and will destroy the flash chip and potentially the PCH.

---

### 1.2 Method A — Raspberry Pi 3B / 3B+ / 4 Parts List

| Item | Notes | Approx. Cost |
|------|-------|-------------|
| Raspberry Pi 3B, 3B+, or 4 | With Raspberry Pi OS (Lite is fine) | ~£15–35 |
| MicroSD card (8 GB+) | With Raspberry Pi OS flashed | ~£5 |
| 5V PSU for the Pi | Stable power matters | — |
| Ethernet or WiFi | To SSH in and run commands — **a Pi with WiFi can run completely headless**: boot it without a screen, SSH in from your normal machine, and position it freely next to the open W541 with all wires connected to the flash chips. Much easier than working around a monitor. | — |

The Pi 3/4 has **two dedicated 3.3V pins** (physical pins 1 and 17) making the three-way VCC/WP/idle-CS distribution easy to crimp — two wires off pin 17 (VCC + /WP) and one wire off pin 1 (idle chip /CS via 47Ω).

---

### 1.3 Method B — Raspberry Pi Pico (RP2040) Parts List

| Item | Notes | Approx. Cost |
|------|-------|-------------|
| Raspberry Pi Pico (RP2040) | **NOT Pico 2 (RP2350)** — RP2350 has serprog stability issues | ~£4 |
| USB Micro-B cable | Pico to host | — |

The Pico is cheaper and needs no SD card or OS, but has only one 3.3V output pin (pin 36), requiring a four-way split or small breadboard for distribution.

---

## Part 2 — Setting Up Your Programmer

### Method A — Raspberry Pi 3/4

#### 2.1 Enable SPI on the Pi

SSH into the Pi (or work at the console):

```bash
sudo raspi-config
```

Navigate to **Interface Options → SPI** and enable it. Reboot when prompted.

Verify the SPI device:

```bash
ls /dev/spidev*
# Should show: /dev/spidev0.0  /dev/spidev0.1
```

#### 2.2 Build flashprog on the Pi

```bash
sudo apt install git build-essential libpci-dev libusb-1.0-0-dev libftdi1-dev \
  libgpiod-dev meson ninja-build pkg-config

git clone https://review.sourcearcade.org/flashprog.git
cd flashprog
meson setup builddir
ninja -C builddir
sudo ninja -C builddir install
```

Verify:

```bash
sudo flashprog -p linux_spi:dev=/dev/spidev0.0,spispeed=512
# Expected: "No EEPROM/flash device found." — nothing connected yet, this is correct
```

#### 2.3 Pi 3/4 GPIO Wiring to SOIC-8

The Pi has 3.3V logic on its GPIOs — no level shifter needed.

```
Pi GPIO header:                    SOIC-8 chip:

Pin 24 — GPIO8/SPI0_CE0 ────────── Pin 1  (/CS)
Pin 21 — GPIO9/SPI0_MISO ───────── Pin 2  (MISO/DO)
Pin 17 — 3.3V ──────────┬────────── Pin 3  (/WP)    ← must be driven high
                        └─────────── Pin 8  (VCC)
Pin 25 — GND ────────────────────── Pin 4  (GND)
Pin 19 — GPIO10/SPI0_MOSI ──────── Pin 5  (MOSI/DI)
Pin 23 — GPIO11/SPI0_SCLK ──────── Pin 6  (CLK)
Pin  1 — 3.3V ──────────[47Ω]───── idle chip Pin 1 (/CS)
                                    (Pin 7 /HOLD — see note; wiring to 3.3V is safe and recommended)
```

> **/WP (Pin 3) must be actively driven high** — confirmed required on the W541; without it, writes fail.  
> **/HOLD (Pin 7) — not wired in the tested setup and it worked fine.** However, /HOLD is an active-low signal: if it floats low even momentarily (due to noise, a weak pull-up, or marginal PCB trace resistance) the chip will pause mid-transfer and produce corrupt reads. If you see intermittent failures that IPA cleaning and slower speeds don't fix, add a wire from Pin 7 to 3.3V. It is always safe to drive /HOLD high — connecting it cannot cause harm.

**Two 3.3V distribution points:**
- **Physical Pin 17** → /WP (Pin 3) and VCC (Pin 8) — two wires, easy crimp
- **Physical Pin 1** → 47Ω resistor to idle chip /CS only — single wire

Full wiring table:

| Pi Physical Pin | Pi Signal | SOIC-8 Pin | Signal | Required? |
|:---:|:---:|:---:|:---:|:---:|
| 24 | GPIO8 / CE0 | 1 | /CS | Yes |
| 21 | GPIO9 / MISO | 2 | MISO (DO) | Yes |
| 17 | 3.3V | 3 | /WP | **Yes — must drive high** |
| 25 | GND | 4 | GND | Yes |
| 19 | GPIO10 / MOSI | 5 | MOSI (DI) | Yes |
| 23 | GPIO11 / SCLK | 6 | CLK | Yes |
| Pin 1 | 3.3V | 7 | /HOLD | Optional — not needed in tested setup, but wire it if you see intermittent failures |
| 17 | 3.3V | 8 | VCC | Yes |
| 1 | 3.3V → 47Ω | idle chip pin 1 | /CS (keep idle) | Yes |

#### 2.4 Pi flashprog command syntax

> **Verify your wiring against Libreboot's SPI guide** before first use — it covers the Pi 3/4 GPIO pinout in detail and is the authoritative reference for this platform: https://libreboot.org/docs/install/spi.html

Replace `serprog:dev=/dev/ttyACM0,spispeed=16M` in all subsequent flashprog commands with:

```bash
-p linux_spi:dev=/dev/spidev0.0,spispeed=1000
```

All other options (chip names, read/write/verify flags, filenames) are identical.

---

### Method B — Raspberry Pi Pico (RP2040)

#### 2.5 Flash serprog Firmware onto the Pico

Use Libreboot's pico-serprog firmware — it has multi-CS handling and tuned 12 mA drive strength for this platform. **Note: lbmk (Libreboot's build system) is used here only to compile the pico-serprog firmware for the Pico. The coreboot ROM itself is built entirely from upstream coreboot — lbmk plays no part in that.**

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

Install flashprog on the **host machine** (not the Pico):

```bash
sudo apt install flashprog
# If not available: sudo apt install flashrom  (commands are identical)
```

#### 2.6 Pico GPIO Wiring to SOIC-8

> **Critical:** GP3 = MOSI and GP5 = /CS. These are commonly documented in reverse. Using the wrong mapping gives garbage reads.

```
Pico (corner near USB):

Pin 4  — GP2  — SCK  ──────────────── SOIC-8 Pin 6
Pin 5  — GP3  — MOSI ──────────────── SOIC-8 Pin 5
Pin 6  — GP4  — MISO ──────────────── SOIC-8 Pin 2
Pin 7  — GP5  — /CS  ──────────────── SOIC-8 Pin 1
Pin 36 — 3V3(OUT) ──┬──────────────── SOIC-8 Pin 8 (VCC)
                    ├──────────────── SOIC-8 Pin 3 (/WP)  ← must be driven high
                    └──[47Ω]────────── idle chip Pin 1 (/CS)
Pin 38 — GND ──────────────────────── SOIC-8 Pin 4
                                       (Pin 7 /HOLD — see note; wiring to 3V3 is safe and recommended)
```

> **/WP (Pin 3) must be actively driven high.**  
> **/HOLD (Pin 7) — not wired in the tested setup and it worked.** The W541 PCB has a pull-up on this line, but it may be marginal in some setups. If the chip sees /HOLD go low mid-transfer it pauses and produces corrupt data. Adding a wire from Pin 7 to 3V3(OUT) costs nothing and can save debugging time. It is always safe to drive /HOLD high.

**Pin 36 carries 3 connections minimum, 4 if you add /HOLD:** VCC, /WP, 47Ω to idle chip /CS, and optionally /HOLD. **Pin 37 (3V3_EN) is NOT a power output — do not use it.**

Full table:

| SOIC-8 Pin | Signal | Pico Physical Pin | Pico GPIO | Required? |
|:---:|:---:|:---:|:---:|:---:|
| 1 | /CS | Pin 7 | GP5 | Yes |
| 2 | MISO | Pin 6 | GP4 | Yes |
| 3 | /WP | Pin 36 | 3V3(OUT) | **Yes — must drive high** |
| 4 | GND | Pin 38 | GND | Yes |
| 5 | MOSI | Pin 5 | GP3 | Yes |
| 6 | SCK | Pin 4 | GP2 | Yes |
| 7 | /HOLD | Pin 36 | 3V3(OUT) | Optional — not needed in tested setup, but wire it if you see intermittent failures |
| 8 | VCC | Pin 36 | 3V3(OUT) | Yes |

#### 2.7 Pico flashprog command syntax

> **Verify your wiring against Libreboot's SPI guide** — it covers the Pico serprog setup and GPIO pinout: https://libreboot.org/docs/install/spi.html

All flashprog commands use:

```bash
-p serprog:dev=/dev/ttyACM0,spispeed=16M
```

---

## Part 3 — W541 Dual Flash Chip Layout

| Chip | Location | Size | Role |
|------|----------|------|------|
| Chip 1 | Left, near white ribbon connector | **8 MiB** | `chip1.bin` / `bottom.rom` |
| Chip 2 | Right, closer to board edge | **4 MiB** | `chip2.bin` / `top.rom` |

Both chips share MOSI/MISO PCB traces at zero ohms. When you clip Chip 1 and power it, Chip 2 also powers up parasitically and its /CS pin **floats** — causing random interference with reads and writes. The fix is to hold pin 1 (/CS) of the idle chip HIGH through a resistor to 3.3V. **47Ω is the textbook value; anything from ~47Ω to 1kΩ works** — 1kΩ has been confirmed working. This is the only connection needed on the idle chip — everything else is already handled by the shared PCB traces.

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

## Part 4 — Disassembly and Reading the Factory ROM

### 4.1 Disassembly

The W541's flash chips are on the **underside of the motherboard**, which means the motherboard has to come out fully to access them — this is not a quick bottom-panel job. Search YouTube or iFixit for "ThinkPad W541 motherboard removal" before starting; there are good video guides. The process involves removing the keyboard, palmrest, display assembly, and several ribbon cables before the board lifts out.

Key steps:
1. Remove battery and AC power. Hold power button 10 seconds to discharge caps.
2. Remove bottom cover screws and pop the cover.
3. Disconnect all ribbon cables, antenna wires, and connectors as you work through the disassembly.
4. Lift the motherboard out. The two SOIC-8 flash chips are on the underside, near the LCD cable area — small, close together.
5. Clean chip legs with IPA before clipping.

**Do not fully reassemble until you've confirmed a successful boot test.** Bench-test first: just AC power + display cable.

### 4.2 Triple-Read Both Chips

**Chip 1 (8 MiB): full clip on Chip 1, idle hook on Chip 2 pin 1**

```bash
# Pi 3/4:
sudo flashprog -p linux_spi:dev=/dev/spidev0.0,spispeed=1000 -r chip1_r1.bin
sudo flashprog -p linux_spi:dev=/dev/spidev0.0,spispeed=1000 -r chip1_r2.bin
sudo flashprog -p linux_spi:dev=/dev/spidev0.0,spispeed=1000 -r chip1_r3.bin

# Pico:
# sudo flashprog -p serprog:dev=/dev/ttyACM0,spispeed=16M -r chip1_r1.bin
# (same for r2, r3)

sha256sum chip1_r1.bin chip1_r2.bin chip1_r3.bin
# All three must match. Size: exactly 8388608 bytes.
```

**Chip 2 (4 MiB): move clip to Chip 2, move idle hook to Chip 1 pin 1**

```bash
# Pi 3/4:
sudo flashprog -p linux_spi:dev=/dev/spidev0.0,spispeed=1000 -r chip2_r1.bin
sudo flashprog -p linux_spi:dev=/dev/spidev0.0,spispeed=1000 -r chip2_r2.bin
sudo flashprog -p linux_spi:dev=/dev/spidev0.0,spispeed=1000 -r chip2_r3.bin

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

> **Inconsistent reads?** Clean chip legs with IPA. Try `spispeed=512` (Pi) or `spispeed=4M` (Pico). Specify chip manually: `flashprog ... -c "W25Q64.V"`. Confirm idle chip's /CS is held high.

---

## Part 5 — Extract Blobs from the Factory Dump

> **All blob extraction is done from the factory dump you just read** — no tools need to run on the W541 itself.

### 5.1 Clone Coreboot and Build Utilities

```bash
git clone https://review.coreboot.org/coreboot.git
cd coreboot
git submodule update --init --checkout

# Build ifdtool (extracts IFD regions) and cbfstool (needed later)
cd util/ifdtool && make && cd ../..
cd util/cbfstool && make && cd ../..
```

### 5.2 Extract Flash Regions with ifdtool

```bash
util/ifdtool/ifdtool -x ../factory_12mb.bin
```

Produces:
```
flashregion_0_flashdescriptor.bin   (Intel Flash Descriptor / IFD)
flashregion_1_bios.bin              (BIOS region — Lenovo UEFI + embedded ROMs)
flashregion_2_intel_me.bin          (Intel ME firmware)
flashregion_3_gbe.bin               (Gigabit Ethernet NVM)
```

Note your MAC address before proceeding:
```bash
strings flashregion_3_gbe.bin | grep -E "([0-9A-Fa-f]{2}:){5}" | head -3
```

### 5.3 Extract the Nvidia K1100M VGA ROM and Intel VBT using UEFIExtract

UEFIExtract (from LongSoft's UEFITool project) is the proven reliable tool for this — it understands the UEFI volume structure inside `flashregion_1_bios.bin` and extracts all sections cleanly, including embedded PCI option ROMs.

#### 5.3a Get UEFIExtract

```bash
cd ~  # or wherever you're working
wget https://github.com/LongSoft/UEFITool/releases/download/A72/UEFIExtract_NE_A72_x64_linux.zip
unzip UEFIExtract_NE_A72_x64_linux.zip
chmod +x UEFIExtract
```

#### 5.3b Extract the Full BIOS Region Tree

```bash
./UEFIExtract flashregion_1_bios.bin all
# Creates: flashregion_1_bios.bin.dump/
# (a directory tree of every UEFI section, each as a separate body.bin)
```

#### 5.3c Find the Nvidia VGA ROM by PCI Vendor/Device ID

| GPU | PCI ID | little-endian in ROM | Status |
|-----|--------|---------------------|--------|
| Quadro K1100M | `10DE:0FF6` | `de 10 f6 0f` | ✓ **Tested and confirmed working** |
| Quadro K2100M | `10DE:11FC` | `de 10 fc 11` | Should work in principle — same extraction method, same coreboot config pattern, **not personally tested** |

The W541 factory BIOS actually contains **both** Nvidia option ROMs regardless of which GPU is fitted — the Python script will find and extract whichever ones are present. Only the ROM matching your actual GPU needs to go into the coreboot build.

This Python script walks the dump, parses every valid PCI option ROM header, and auto-extracts any Nvidia ROM it finds:

```bash
python3 - <<'EOF'
import os, struct

for root, dirs, files in os.walk('flashregion_1_bios.bin.dump'):
    for name in files:
        if name == 'body.bin':
            path = os.path.join(root, name)
            try:
                with open(path, 'rb') as f:
                    d = f.read()
                # Must start with PCI ROM magic 55 AA
                if len(d) > 0x1a and d[0] == 0x55 and d[1] == 0xAA:
                    # Pointer to PCI Data Structure is at offset 0x18 (little-endian word)
                    off = struct.unpack_from('<H', d, 0x18)[0]
                    if off + 8 <= len(d) and d[off:off+4] == b'PCIR':
                        vid = struct.unpack_from('<H', d, off+4)[0]
                        did = struct.unpack_from('<H', d, off+6)[0]
                        sz  = d[2] * 512
                        print(f'{path}  vendor=0x{vid:04x} device=0x{did:04x} size={sz} bytes')
                        if vid == 0x10DE:
                            outname = f'pci10de_{did:04x}.rom'
                            open(outname, 'wb').write(d[:sz] if sz else d)
                            print(f'  >>> EXTRACTED to {outname}')
            except:
                pass
EOF
```

Expected output will include lines for both Nvidia ROMs if the factory BIOS contains them:
```
...  vendor=0x10de device=0x0ff6 size=... bytes
  >>> EXTRACTED to pci10de_0ff6.rom    ← K1100M
...  vendor=0x10de device=0x11fc size=... bytes
  >>> EXTRACTED to pci10de_11fc.rom    ← K2100M
```

Rename whichever matches **your** GPU:

```bash
# K1100M:
mv pci10de_0ff6.rom nvidia_k1100m.rom
hexdump -C nvidia_k1100m.rom | head -3
strings nvidia_k1100m.rom | grep -i nvidia | head -3

# K2100M:
# mv pci10de_11fc.rom nvidia_k2100m.rom
# hexdump -C nvidia_k2100m.rom | head -3
# strings nvidia_k2100m.rom | grep -i nvidia | head -3
```

Both ROMs must start with `55 AA` and contain an NVIDIA string. Keep both extractions safe regardless — you only embed one in the coreboot build.

#### 5.3d Find and Extract the Intel VBT

The Intel GOP/VBT is embedded under a specific GUID path in the dump tree. The script above will have shown a line like:

```
flashregion_1_bios.bin.dump/1 7A9354D9-.../0 9E21FD93-.../0 Compressed section/
  0 Volume image section/0 A881D567-.../128 29206FC2-.../0 Raw section/body.bin
  vendor=0x8086 device=0x0406 size=... bytes
```

Copy that specific `body.bin` as your Intel VBT/VGA ROM:

```bash
cp "flashregion_1_bios.bin.dump/1 7A9354D9-0468-444A-81CE-0BF617D890DF/\
0 9E21FD93-9C72-4C15-8C4B-E77F1DB2D792/0 Compressed section/\
0 Volume image section/0 A881D567-6CB0-4EEE-8435-2E72D33E45B5/\
128 29206FC2-9EAB-4612-ACA1-1E3D098FB1B3/0 Raw section/body.bin" \
intel_vbt.bin

# Verify: look for $VBT signature
hexdump -C intel_vbt.bin | grep -m3 ""
strings intel_vbt.bin | grep -i vbt | head -3
```

> The exact directory path will match what the scan printed — the GUIDs above are from the W541 factory BIOS. If your dump path differs slightly (different section numbering), use the path the Python script reported for the `0x8086:0x0406` ROM.

### 5.5 Get mrc.bin (Haswell Memory Reference Code)

The MRC blob is the same across all Haswell platforms. If you have a working T440p coreboot setup, reuse that `mrc.bin` directly — it works on the W541.

```bash
# Check if already present in coreboot blobs submodule
ls 3rdparty/blobs/northbridge/intel/haswell/mrc.bin
```

If not available, use chromeos one:

```bash
cd util/chromeos
./crosfirmware.sh peppy
../cbfstool/cbfstool coreboot-*.bin extract -f mrc.bin -n mrc.bin -r RO_SECTION
cp mrc.bin ~/coreboot-blobs/
```

### 5.6 Intel ME — handled by the build

**No manual me_cleaner step is needed.** The `.config` includes `CONFIG_USE_ME_CLEANER=y` with `-S` args, so coreboot runs me_cleaner on the raw ME binary automatically at build time. Just pass `flashregion_2_intel_me.bin` directly — no pre-processing required.

---

## Part 6 — Organise Blobs

The `.config` uses absolute paths for all blobs. The simplest approach is a flat `coreboot-blobs/` directory alongside the coreboot tree:

```bash
mkdir -p ~/coreboot/coreboot-blobs

cp flashregion_0_flashdescriptor.bin  ~/coreboot/coreboot-blobs/
cp flashregion_2_intel_me.bin         ~/coreboot/coreboot-blobs/   # raw — no pre-cleaning needed
cp flashregion_3_gbe.bin              ~/coreboot/coreboot-blobs/
cp mrc.bin                            ~/coreboot/coreboot-blobs/

# VBT — must go to this exact path in the coreboot source tree
cp intel_vbt.bin /home/chrisf4lc0n/Downloads/coreboot/src/mainboard/lenovo/haswell/variants/w541

# Nvidia ROM — in coreboot-blobs alongside the others
cp nvidia_k1100m.rom  ~/coreboot/coreboot-blobs/pci10de,0ff6.rom    # K1100M
# cp nvidia_k2100m.rom  ~/coreboot/coreboot-blobs/pci10de,11fc.rom  # K2100M
```

The VBT is the only blob that must live inside the coreboot source tree — `src/mainboard/lenovo/t540p/data.vbt`. Everything else is referenced by absolute path in `.config` and can live wherever is convenient.

---

## Part 7 — Coreboot .config

This is the actual working `.config` used for this build. Save it as `.config` inside the coreboot source directory, then **update every absolute path** under `# Chipset — blob paths` to match your own directory layout before running `make olddefconfig`.

```
#
# Automatically generated file; DO NOT EDIT.
# coreboot configuration
#

#
# General setup
#
CONFIG_LOCALVERSION=""
CONFIG_CBFS_PREFIX="fallback"
CONFIG_COMPILER_GCC=y
# CONFIG_COMPILER_LLVM_CLANG is not set
# CONFIG_ANY_TOOLCHAIN is not set
# CONFIG_CCACHE is not set
# CONFIG_LTO is not set
# CONFIG_IWYU is not set
# CONFIG_FMD_GENPARSER is not set
# CONFIG_UTIL_GENPARSER is not set
# CONFIG_OPTION_BACKEND_NONE is not set
CONFIG_USE_OPTION_TABLE=y
# CONFIG_USE_CBFS_FILE_OPTION_BACKEND is not set
# CONFIG_STATIC_OPTION_TABLE is not set
CONFIG_COMPRESS_RAMSTAGE_LZMA=y
# CONFIG_COMPRESS_RAMSTAGE_LZ4 is not set
# CONFIG_COMPRESS_RAMSTAGE_ZSTD is not set
CONFIG_SEPARATE_ROMSTAGE=y
CONFIG_INCLUDE_CONFIG_FILE=y
CONFIG_COLLECT_TIMESTAMPS=y
# CONFIG_TIMESTAMPS_ON_CONSOLE is not set
CONFIG_USE_BLOBS=y
# CONFIG_USE_AMD_BLOBS is not set
# CONFIG_USE_QC_BLOBS is not set
# CONFIG_COVERAGE is not set
# CONFIG_UBSAN is not set
CONFIG_HAVE_ASAN_IN_ROMSTAGE=y
CONFIG_HAVE_ASAN_IN_RAMSTAGE=y
# CONFIG_ASAN is not set
# CONFIG_NO_STAGE_CACHE is not set
CONFIG_TSEG_STAGE_CACHE=y
# CONFIG_UPDATE_IMAGE is not set
# CONFIG_BOOTSPLASH_IMAGE is not set

#
# Software Bill Of Materials (SBOM)
#
# CONFIG_SBOM is not set
# end of Software Bill Of Materials (SBOM)
# end of General setup

#
# Mainboard
#

#
# Important: Run 'make distclean' before switching boards
#
# CONFIG_VENDOR_51NB is not set
# CONFIG_VENDOR_ACER is not set
# CONFIG_VENDOR_AMD is not set
# CONFIG_VENDOR_AOOSTAR is not set
# CONFIG_VENDOR_AOPEN is not set
# CONFIG_VENDOR_APPLE is not set
# CONFIG_VENDOR_ARM is not set
# CONFIG_VENDOR_ASROCK is not set
# CONFIG_VENDOR_ASUS is not set
# CONFIG_VENDOR_BIOSTAR is not set
# CONFIG_VENDOR_BOSTENTECH is not set
# CONFIG_VENDOR_BYTEDANCE is not set
# CONFIG_VENDOR_CAVIUM is not set
# CONFIG_VENDOR_CLEVO is not set
# CONFIG_VENDOR_COMPULAB is not set
# CONFIG_VENDOR_CWWK is not set
# CONFIG_VENDOR_DELL is not set
# CONFIG_VENDOR_EMULATION is not set
# CONFIG_VENDOR_ERYING is not set
# CONFIG_VENDOR_EXAMPLE is not set
# CONFIG_VENDOR_FACEBOOK is not set
# CONFIG_VENDOR_FOXCONN is not set
# CONFIG_VENDOR_FRAMEWORK is not set
# CONFIG_VENDOR_GETAC is not set
# CONFIG_VENDOR_GIGABYTE is not set
# CONFIG_VENDOR_GOOGLE is not set
# CONFIG_VENDOR_HARDKERNEL is not set
# CONFIG_VENDOR_HP is not set
# CONFIG_VENDOR_IBASE is not set
# CONFIG_VENDOR_IBM is not set
# CONFIG_VENDOR_INTEL is not set
# CONFIG_VENDOR_INVENTEC is not set
# CONFIG_VENDOR_KONTRON is not set
# CONFIG_VENDOR_LATTEPANDA is not set
CONFIG_VENDOR_LENOVO=y
# CONFIG_VENDOR_LIBRETREND is not set
# CONFIG_VENDOR_MITAC_COMPUTING is not set
# CONFIG_VENDOR_MSI is not set
# CONFIG_VENDOR_NOVACUSTOM is not set
# CONFIG_VENDOR_OCP is not set
# CONFIG_VENDOR_OPENCELLULAR is not set
# CONFIG_VENDOR_PACKARDBELL is not set
# CONFIG_VENDOR_PCENGINES is not set
# CONFIG_VENDOR_PINE64 is not set
# CONFIG_VENDOR_PORTWELL is not set
# CONFIG_VENDOR_PRODRIVE is not set
# CONFIG_VENDOR_PROTECTLI is not set
# CONFIG_VENDOR_PURISM is not set
# CONFIG_VENDOR_QOTOM is not set
# CONFIG_VENDOR_RAPTOR_CS is not set
# CONFIG_VENDOR_RAZER is not set
# CONFIG_VENDOR_RODA is not set
# CONFIG_VENDOR_SAMSUNG is not set
# CONFIG_VENDOR_SAPPHIRE is not set
# CONFIG_VENDOR_SIEMENS is not set
# CONFIG_VENDOR_SIFIVE is not set
# CONFIG_VENDOR_STARLABS is not set
# CONFIG_VENDOR_SUPERMICRO is not set
# CONFIG_VENDOR_SYSTEM76 is not set
# CONFIG_VENDOR_TI is not set
# CONFIG_VENDOR_TOPTON is not set
# CONFIG_VENDOR_UP is not set
# CONFIG_VENDOR_VIA is not set
CONFIG_MAINBOARD_FAMILY="ThinkPad W541"
CONFIG_MAINBOARD_PART_NUMBER="ThinkPad W541"
CONFIG_MAINBOARD_VERSION="1.0"
CONFIG_MAINBOARD_DIR="lenovo/haswell"
CONFIG_VGA_BIOS_ID="10de,0ff6"
CONFIG_DIMM_MAX=4
CONFIG_DIMM_SPD_SIZE=256
CONFIG_FMDFILE=""
CONFIG_NO_POST=y
CONFIG_MAINBOARD_VENDOR="LENOVO"
CONFIG_CBFS_SIZE=0x700000
CONFIG_LINEAR_FRAMEBUFFER_MAX_HEIGHT=1600
CONFIG_LINEAR_FRAMEBUFFER_MAX_WIDTH=2560
CONFIG_MAX_CPUS=8
CONFIG_ME_CLEANER_ARGS="-S"
CONFIG_ONBOARD_VGA_IS_PRIMARY=y
CONFIG_VARIANT_DIR="w541"
CONFIG_OVERRIDE_DEVICETREE=""
CONFIG_DEVICETREE="variants/$(CONFIG_VARIANT_DIR)/devicetree.cb"
# CONFIG_VBOOT is not set
CONFIG_VGA_BIOS=y
CONFIG_PCIEXP_ASPM=y
CONFIG_PCIEXP_L1_SUB_STATE=y
CONFIG_PCIEXP_CLK_PM=y
CONFIG_MAINBOARD_SMBIOS_MANUFACTURER="LENOVO"
CONFIG_ECAM_MMCONF_BASE_ADDRESS=0xf0000000
CONFIG_ECAM_MMCONF_BUS_NUMBER=64
CONFIG_MEMLAYOUT_LD_FILE="src/arch/x86/memlayout.ld"
# CONFIG_FATAL_ASSERTS is not set
CONFIG_USBDEBUG_HCD_INDEX=2
CONFIG_INTEL_GMA_VBT_FILE="src/mainboard/$(MAINBOARDDIR)/variants/$(VARIANT_DIR)/data.vbt"
CONFIG_PRERAM_CBMEM_CONSOLE_SIZE=0xc00
CONFIG_MAINBOARD_SMBIOS_PRODUCT_NAME="ThinkPad W541"
CONFIG_MAX_SOCKET=1
CONFIG_CMOS_DEFAULT_FILE="src/mainboard/$(MAINBOARDDIR)/cmos.default"
CONFIG_CMOS_LAYOUT_FILE="src/mainboard/$(MAINBOARDDIR)/cmos.layout"
CONFIG_PAYLOAD_CONFIGFILE=""
CONFIG_BOOT_DEVICE_SPI_FLASH_BUS=0
CONFIG_TPM_PIRQ=0x0
CONFIG_SEABIOS_PS2_TIMEOUT=5000
CONFIG_PXE_ROM_ID="8086,153a"
CONFIG_DCACHE_RAM_BASE=0xff7c0000
CONFIG_DCACHE_RAM_SIZE=0x10000
CONFIG_C_ENV_BOOTBLOCK_SIZE=0x40000
CONFIG_DCACHE_BSP_STACK_SIZE=0x2000
CONFIG_MAX_ACPI_TABLE_SIZE_KB=144
CONFIG_HAVE_INTEL_FIRMWARE=y
CONFIG_MRC_SETTINGS_CACHE_SIZE=0x10000
CONFIG_SPI_FLASH_INCLUDE_ALL_DRIVERS=y
CONFIG_SPI_FLASH_WINBOND=y
CONFIG_DRIVERS_INTEL_WIFI=y
CONFIG_IFD_BIN_PATH="coreboot-blobs/flashregion_0_flashdescriptor.bin"
CONFIG_ME_BIN_PATH="coreboot-blobs/flashregion_2_intel_me.bin"
CONFIG_GBE_BIN_PATH="coreboot-blobs/flashregion_3_gbe.bin"
CONFIG_CONSOLE_CBMEM_BUFFER_SIZE=0x20000
# CONFIG_ENABLE_DDR_2X_REFRESH is not set
CONFIG_PCIEXP_AER=y
CONFIG_VGA_BIOS_FILE="coreboot-blobs/pci10de,0ff6.rom"
CONFIG_SEABIOS_HARDWARE_IRQ=y
CONFIG_CARDBUS_PLUGIN_SUPPORT=y
CONFIG_SPI_FLASH_GIGADEVICE=y
CONFIG_SPI_FLASH_STMICRO=y
# CONFIG_DEBUG_SMI is not set
CONFIG_HAVE_IFD_BIN=y
# CONFIG_BOARD_LENOVO_THINKPAD_T440P is not set
CONFIG_BOARD_LENOVO_THINKPAD_W541=y
# CONFIG_BOARD_LENOVO_L520 is not set
# CONFIG_BOARD_LENOVO_M720Q is not set
# CONFIG_BOARD_LENOVO_M920Q is not set
# CONFIG_BOARD_LENOVO_M900 is not set
# CONFIG_BOARD_LENOVO_THINKCENTRE_M900_TINY is not set
# CONFIG_BOARD_LENOVO_S230U is not set
# CONFIG_BOARD_LENOVO_T470S is not set
# CONFIG_BOARD_LENOVO_T480 is not set
# CONFIG_BOARD_LENOVO_T480S is not set
# CONFIG_BOARD_LENOVO_T580 is not set
# CONFIG_BOARD_LENOVO_X280 is not set
# CONFIG_BOARD_LENOVO_T400 is not set
# CONFIG_BOARD_LENOVO_T500 is not set
# CONFIG_BOARD_LENOVO_R400 is not set
# CONFIG_BOARD_LENOVO_R500 is not set
# CONFIG_BOARD_LENOVO_W500 is not set
# CONFIG_BOARD_LENOVO_T410 is not set
# CONFIG_BOARD_LENOVO_T420 is not set
# CONFIG_BOARD_LENOVO_T420S is not set
# CONFIG_BOARD_LENOVO_T430S is not set
# CONFIG_BOARD_LENOVO_T431S is not set
# CONFIG_BOARD_LENOVO_T520 is not set
# CONFIG_BOARD_LENOVO_W520 is not set
# CONFIG_BOARD_LENOVO_T530 is not set
# CONFIG_BOARD_LENOVO_W530 is not set
# CONFIG_BOARD_LENOVO_T430 is not set
# CONFIG_BOARD_LENOVO_T60 is not set
# CONFIG_BOARD_LENOVO_Z61T is not set
# CONFIG_BOARD_LENOVO_R60 is not set
# CONFIG_BOARD_LENOVO_THINKCENTRE_A58 is not set
# CONFIG_BOARD_LENOVO_THINKCENTRE_M710S is not set
# CONFIG_BOARD_LENOVO_X131E is not set
# CONFIG_BOARD_LENOVO_X1_CARBON_GEN1 is not set
# CONFIG_BOARD_LENOVO_X200 is not set
# CONFIG_BOARD_LENOVO_X301 is not set
# CONFIG_BOARD_LENOVO_X201 is not set
# CONFIG_BOARD_LENOVO_X220 is not set
# CONFIG_BOARD_LENOVO_X220I is not set
# CONFIG_BOARD_LENOVO_X1 is not set
# CONFIG_BOARD_LENOVO_X230 is not set
# CONFIG_BOARD_LENOVO_X230T is not set
# CONFIG_BOARD_LENOVO_X230S is not set
# CONFIG_BOARD_LENOVO_X230_EDP is not set
# CONFIG_BOARD_LENOVO_X60 is not set
CONFIG_BOARD_LENOVO_HASWELL_COMMON=y
CONFIG_VBOOT_SLOTS_RW_AB=y
CONFIG_PS2K_EISAID="LEN0071"
CONFIG_PS2M_EISAID="LEN004A"
CONFIG_THINKPADEC_HKEY_EISAID="LEN0068"
CONFIG_GFX_GMA_PANEL_1_PORT="DP3"
CONFIG_D3COLD_SUPPORT=y
CONFIG_GFX_GMA_PANEL_1_ON_EDP=y
# CONFIG_DRIVERS_UART_8250IO is not set
CONFIG_PC_CMOS_BASE_PORT_BANK1=0x72
CONFIG_HEAP_SIZE=0x100000
# CONFIG_BOOTMEDIA_SMM_BWP is not set
# CONFIG_DRIVERS_EFI_FW_INFO is not set
# CONFIG_TPM_MEASURED_BOOT is not set
CONFIG_BOARD_ROMSIZE_KB_12288=y
# CONFIG_COREBOOT_ROMSIZE_KB_256 is not set
# CONFIG_COREBOOT_ROMSIZE_KB_512 is not set
# CONFIG_COREBOOT_ROMSIZE_KB_1024 is not set
# CONFIG_COREBOOT_ROMSIZE_KB_2048 is not set
# CONFIG_COREBOOT_ROMSIZE_KB_4096 is not set
# CONFIG_COREBOOT_ROMSIZE_KB_5120 is not set
# CONFIG_COREBOOT_ROMSIZE_KB_6144 is not set
# CONFIG_COREBOOT_ROMSIZE_KB_8192 is not set
# CONFIG_COREBOOT_ROMSIZE_KB_10240 is not set
CONFIG_COREBOOT_ROMSIZE_KB_12288=y
# CONFIG_COREBOOT_ROMSIZE_KB_16384 is not set
# CONFIG_COREBOOT_ROMSIZE_KB_24576 is not set
# CONFIG_COREBOOT_ROMSIZE_KB_32768 is not set
# CONFIG_COREBOOT_ROMSIZE_KB_65536 is not set
CONFIG_COREBOOT_ROMSIZE_KB=12288
CONFIG_ROM_SIZE=0x00c00000
CONFIG_HAVE_POWER_STATE_AFTER_FAILURE=y
CONFIG_HAVE_POWER_STATE_PREVIOUS_AFTER_FAILURE=y
CONFIG_POWER_STATE_OFF_AFTER_FAILURE=y
# CONFIG_POWER_STATE_ON_AFTER_FAILURE is not set
# CONFIG_POWER_STATE_PREVIOUS_AFTER_FAILURE is not set
CONFIG_MAINBOARD_POWER_FAILURE_STATE=0
# end of Mainboard

CONFIG_SYSTEM_TYPE_LAPTOP=y

#
# Chipset
#

#
# SoC
#
CONFIG_CHIPSET_DEVICETREE=""
CONFIG_CBFS_MCACHE_SIZE=0x4000
CONFIG_ROMSTAGE_ADDR=0x2000000
CONFIG_VERSTAGE_ADDR=0x2000000
CONFIG_SMM_TSEG_SIZE=0x800000
CONFIG_SMM_RESERVED_SIZE=0x100000
CONFIG_SMM_MODULE_STACK_SIZE=0x400
CONFIG_SERIRQ_CONTINUOUS_MODE=y
CONFIG_CPU_PT_ROM_MAP_GB=512
# CONFIG_VGA_BIOS_SECOND is not set
CONFIG_PRERAM_CBFS_CACHE_SIZE=0x4000
CONFIG_DOMAIN_RESOURCE_32BIT_LIMIT=0xf0000000
CONFIG_EHCI_BAR=0xe8000000
CONFIG_ACPI_CPU_STRING="CP%02X"
CONFIG_STACK_SIZE=0x2000
CONFIG_IED_REGION_SIZE=0x400000
CONFIG_INTEL_GMA_BCLV_OFFSET=0x48254
CONFIG_INTEL_GMA_BCLV_WIDTH=16
CONFIG_INTEL_GMA_BCLM_OFFSET=0xc8256
CONFIG_INTEL_GMA_BCLM_WIDTH=16
CONFIG_BOOTBLOCK_IN_CBFS=y
CONFIG_HAVE_MRC=y
CONFIG_MRC_FILE="coreboot-blobs/mrc.bin"
CONFIG_DCACHE_RAM_MRC_VAR_SIZE=0x30000
CONFIG_HPET_MIN_TICKS=0x80
CONFIG_FIXED_MCHBAR_MMIO_BASE=0xfed10000
CONFIG_FIXED_DMIBAR_MMIO_BASE=0xfed18000
CONFIG_FIXED_EPBAR_MMIO_BASE=0xfed19000
CONFIG_PCIEXP_COMMON_CLOCK=y
CONFIG_DISABLE_ME_PCI=y
CONFIG_CPU_INTEL_NUM_FIT_ENTRIES=6
CONFIG_SOC_PHYSICAL_ADDRESS_WIDTH=0
CONFIG_DEBUG_STACK_OVERFLOW_BREAKPOINTS=y
CONFIG_RAMSTAGE_CBFS_CACHE_SIZE=0x4000
CONFIG_CBFS_CACHE_ALIGN=8
# CONFIG_ALWAYS_ALLOW_ABOVE_4G_ALLOCATION is not set
CONFIG_FIXED_SMBUS_IO_BASE=0x400
CONFIG_UART_BITBANG_TX_DELAY_MS=5

#
# CPU
#
CONFIG_CPU_INTEL_HASWELL=y
CONFIG_CPU_INTEL_FIRMWARE_INTERFACE_TABLE=y
CONFIG_CPU_INTEL_COMMON=y
CONFIG_ENABLE_VMX=y
CONFIG_SET_IA32_FC_LOCK_BIT=y
CONFIG_SET_MSR_AESNI_LOCK_BIT=y
CONFIG_CPU_INTEL_COMMON_TIMEBASE=y
CONFIG_CPU_INTEL_COMMON_VOLTAGE=y
CONFIG_CPU_INTEL_COMMON_SMM=y
CONFIG_PARALLEL_MP=y
CONFIG_XAPIC_ONLY=y
# CONFIG_X2APIC_ONLY is not set
# CONFIG_X2APIC_RUNTIME is not set
# CONFIG_X2APIC_LATE_WORKAROUND is not set
CONFIG_UDELAY_TSC=y
CONFIG_TSC_MONOTONIC_TIMER=y
CONFIG_TSC_SYNC_MFENCE=y
CONFIG_HAVE_SMI_HANDLER=y
CONFIG_SMM_TSEG=y
CONFIG_SMM_PCI_RESOURCE_STORE_NUM_SLOTS=8
CONFIG_AP_STACK_SIZE=0x800
CONFIG_SMP=y
CONFIG_SSE=y
CONFIG_SSE2=y
CONFIG_SUPPORT_CPU_UCODE_IN_CBFS=y
CONFIG_USE_CPU_MICROCODE_CBFS_BINS=y
CONFIG_CPU_MICROCODE_CBFS_DEFAULT_BINS=y
# CONFIG_CPU_MICROCODE_CBFS_EXTERNAL_BINS is not set
# CONFIG_CPU_MICROCODE_CBFS_EXTERNAL_HEADER is not set
# CONFIG_CPU_MICROCODE_CBFS_NONE is not set

#
# Northbridge
#
CONFIG_NORTHBRIDGE_INTEL_HASWELL=y
# CONFIG_USE_NATIVE_RAMINIT is not set
# CONFIG_USE_BROADWELL_MRC is not set
CONFIG_HASWELL_HIDE_PEG_FROM_MRC=y

#
# Southbridge
#
# CONFIG_PCIEXP_HOTPLUG is not set
CONFIG_INTEL_DESCRIPTOR_MODE_REQUIRED=y
CONFIG_SOUTHBRIDGE_INTEL_LYNXPOINT=y
CONFIG_FINALIZE_USB_ROUTE_XHCI=y
CONFIG_SOUTHBRIDGE_INTEL_COMMON_RESET=y
CONFIG_SOUTHBRIDGE_INTEL_COMMON_RTC=y
CONFIG_SOUTHBRIDGE_INTEL_COMMON_PMCLIB=y
CONFIG_SOUTHBRIDGE_INTEL_COMMON_PMBASE=y
CONFIG_SOUTHBRIDGE_INTEL_COMMON_GPIO=y
CONFIG_SOUTHBRIDGE_INTEL_COMMON_EARLY_SMBUS=y
CONFIG_SOUTHBRIDGE_INTEL_COMMON_SMBUS=y
CONFIG_SOUTHBRIDGE_INTEL_COMMON_SPI=y
CONFIG_SOUTHBRIDGE_INTEL_COMMON_SPI_ICH9=y
CONFIG_SOUTHBRIDGE_INTEL_COMMON_PIRQ_ACPI_GEN=y
CONFIG_SOUTHBRIDGE_INTEL_COMMON_RCBA_PIRQ=y
CONFIG_HAVE_INTEL_CHIPSET_LOCKDOWN=y
CONFIG_SOUTHBRIDGE_INTEL_COMMON_FINALIZE=y
CONFIG_SOUTHBRIDGE_INTEL_COMMON_USB_DEBUG=y
CONFIG_INTEL_DESCRIPTOR_MODE_CAPABLE=y
# CONFIG_VALIDATE_INTEL_DESCRIPTOR is not set
CONFIG_INTEL_CHIPSET_LOCKDOWN=y
CONFIG_TCO_SPACE_NOT_YET_SPLIT=y
CONFIG_SOUTHBRIDGE_INTEL_COMMON_WATCHDOG=y
CONFIG_FIXED_RCBA_MMIO_BASE=0xfed1c000
CONFIG_RCBA_LENGTH=0x4000

#
# Super I/O
#

#
# Embedded Controllers
#
CONFIG_EC_ACPI=y
CONFIG_EC_LENOVO_H8=y
CONFIG_H8_BEEP_ON_DEATH=y
CONFIG_H8_FLASH_LEDS_ON_DEATH=y
CONFIG_H8_SUPPORT_BT_ON_WIFI=y
# CONFIG_H8_FN_CTRL_SWAP is not set
CONFIG_H8_HAS_BAT_THRESHOLDS_IMPL=y
CONFIG_H8_HAS_PRIMARY_FN_KEYS=y
CONFIG_H8_HAS_LEDLOGO=y
CONFIG_EC_LENOVO_PMH7=y

#
# Intel Firmware
#
CONFIG_HAVE_ME_BIN=y
# CONFIG_STITCH_ME_BIN is not set
# CONFIG_CHECK_ME is not set
CONFIG_ME_REGION_ALLOW_CPU_READ_ACCESS=y
CONFIG_USE_ME_CLEANER=y

#
# Please test coreboot with the original, unmodified ME firmware before using me_cleaner
#
CONFIG_MAINBOARD_USES_IFD_GBE_REGION=y
CONFIG_HAVE_GBE_BIN=y
# CONFIG_DO_NOT_TOUCH_DESCRIPTOR_REGION is not set
# CONFIG_LOCK_MANAGEMENT_ENGINE is not set
CONFIG_UNLOCK_FLASH_REGIONS=y
CONFIG_ACPI_FNKEY_GEN_SCANCODE=0
CONFIG_ARCH_X86=y
CONFIG_ARCH_BOOTBLOCK_X86_32=y
CONFIG_ARCH_VERSTAGE_X86_32=y
CONFIG_ARCH_ROMSTAGE_X86_32=y
CONFIG_ARCH_POSTCAR_X86_32=y
CONFIG_ARCH_RAMSTAGE_X86_32=y
CONFIG_ARCH_ALL_STAGES_X86_32=y
CONFIG_RESERVED_PHYSICAL_ADDRESS_BITS_SUPPORT=y
CONFIG_X86_TOP4G_BOOTMEDIA_MAP=y
CONFIG_POSTRAM_CBFS_CACHE_IN_BSS=y
CONFIG_PC80_SYSTEM=y
CONFIG_HAVE_CMOS_DEFAULT=y
CONFIG_POSTCAR_STAGE=y
CONFIG_BOOTBLOCK_SIMPLE=y
# CONFIG_BOOTBLOCK_NORMAL is not set
CONFIG_COLLECT_TIMESTAMPS_TSC=y
CONFIG_HAVE_CF9_RESET=y
CONFIG_DEBUG_HW_BREAKPOINTS=y
CONFIG_DEBUG_NULL_DEREF_BREAKPOINTS=y
# CONFIG_DUMP_SMBIOS_TYPE17 is not set
CONFIG_X86_BOOTBLOCK_EXTRA_PROGRAM_SZ=0
CONFIG_DEFAULT_EBDA_LOWMEM=0x100000
CONFIG_DEFAULT_EBDA_SEGMENT=0xF600
CONFIG_DEFAULT_EBDA_SIZE=0x400
# end of Chipset

#
# Devices
#
CONFIG_HAVE_VGA_TEXT_FRAMEBUFFER=y
CONFIG_HAVE_LINEAR_FRAMEBUFFER=y
CONFIG_VGA_ROM_RUN_DEFAULT=y
CONFIG_MAINBOARD_HAS_LIBGFXINIT=y
CONFIG_MAINBOARD_USE_LIBGFXINIT=y
# CONFIG_VGA_ROM_RUN is not set
# CONFIG_NO_GFX_INIT is not set
CONFIG_NO_EARLY_GFX_INIT=y

#
# Display
#
# CONFIG_VGA_TEXT_FRAMEBUFFER is not set
CONFIG_GENERIC_LINEAR_FRAMEBUFFER=y
CONFIG_LINEAR_FRAMEBUFFER=y
# CONFIG_BOOTSPLASH is not set
CONFIG_DEFAULT_SCREEN_ROTATION_INT=0
# end of Display

CONFIG_PCI=y
CONFIG_ECAM_MMCONF_SUPPORT=y
CONFIG_PCIX_PLUGIN_SUPPORT=y
CONFIG_AZALIA_HDA_CODEC_SUPPORT=y
CONFIG_AZALIA_USE_LEGACY_VERB_TABLE=y
CONFIG_PCIEXP_PLUGIN_SUPPORT=y
CONFIG_ECAM_MMCONF_LENGTH=0x04000000
CONFIG_PCI_ALLOW_BUS_MASTER=y
CONFIG_PCI_SET_BUS_MASTER_PCI_BRIDGES=y
CONFIG_PCI_ALLOW_BUS_MASTER_ANY_DEVICE=y
# CONFIG_PCIEXP_SUPPORT_RESIZABLE_BARS is not set
# CONFIG_PCIEXP_LANE_ERR_STAT_CLEAR is not set
# CONFIG_EARLY_PCI_BRIDGE is not set
CONFIG_SUBSYSTEM_VENDOR_ID=0x0000
CONFIG_SUBSYSTEM_DEVICE_ID=0x0000
# CONFIG_VGA_BIOS_DGPU is not set
CONFIG_INTEL_GMA_HAVE_VBT=y
CONFIG_INTEL_GMA_ADD_VBT=y
# CONFIG_SOFTWARE_I2C is not set
CONFIG_I2C_TRANSFER_TIMEOUT_US=500000
CONFIG_RESOURCE_ALLOCATION_TOP_DOWN=y
CONFIG_DRAM_SUPPORT_DDR3=y
# end of Devices

#
# Generic Drivers
#
CONFIG_CRB_TPM_BASE_ADDRESS=0xfed40000
# CONFIG_DRIVERS_EFI_VARIABLE_STORE is not set
# CONFIG_ELOG is not set
# CONFIG_DRIVERS_HWID_DMI is not set
CONFIG_CACHE_MRC_SETTINGS=y
# CONFIG_MRC_SETTINGS_PROTECT is not set
# CONFIG_DRIVERS_OPTION_CFR is not set
CONFIG_SMMSTORE=y
CONFIG_SMMSTORE_V2=y
CONFIG_SMMSTORE_SIZE=0x80000
CONFIG_SPI_FLASH=y
CONFIG_BOOT_DEVICE_SPI_FLASH_RW_NOMMAP=y
CONFIG_BOOT_DEVICE_SPI_FLASH_RW_NOMMAP_EARLY=y
CONFIG_SPI_FLASH_SMM=y
# CONFIG_SPI_FLASH_NO_FAST_READ is not set
CONFIG_SPI_FLASH_ADESTO=y
CONFIG_SPI_FLASH_AMIC=y
CONFIG_SPI_FLASH_ATMEL=y
CONFIG_SPI_FLASH_EON=y
CONFIG_SPI_FLASH_MACRONIX=y
CONFIG_SPI_FLASH_SPANSION=y
CONFIG_SPI_FLASH_SST=y
CONFIG_SPI_FLASH_ISSI=y
CONFIG_HAVE_EM100PRO_SPI_CONSOLE_SUPPORT=y
CONFIG_TPM_INIT_RAMSTAGE=y
# CONFIG_TPM_PPI is not set
CONFIG_NO_UART_ON_SUPERIO=y
# CONFIG_DRIVERS_UART_OXPCIE is not set
CONFIG_HAVE_USBDEBUG=y
CONFIG_HAVE_USBDEBUG_OPTIONS=y
# CONFIG_USBDEBUG is not set
# CONFIG_VPD is not set
# CONFIG_DRIVERS_EMULATION_QEMU_FW_CFG is not set
# CONFIG_DRIVERS_GENERIC_CBFS_SERIAL is not set
# CONFIG_DRIVERS_GENERIC_CBFS_UUID is not set
# CONFIG_DRIVERS_GENESYSLOGIC_GL9750 is not set
# CONFIG_DRIVERS_GENESYSLOGIC_GL9755 is not set
# CONFIG_DRIVERS_GENESYSLOGIC_GL9763E is not set
# CONFIG_DRIVERS_I2C_MAX98396 is not set
CONFIG_INTEL_DDI=y
CONFIG_INTEL_GMA_ACPI=y
CONFIG_VBT_CBFS_COMPRESSION_LZMA=y
# CONFIG_VBT_CBFS_COMPRESSION_LZ4 is not set
# CONFIG_VBT_CBFS_COMPRESSION_NONE is not set
CONFIG_VBT_CBFS_COMPRESSION_ALGORITHM="lzma"
CONFIG_GFX_GMA=y
CONFIG_GFX_GMA_DYN_CPU=y
CONFIG_GFX_GMA_GENERATION="Haswell"
CONFIG_GFX_GMA_PCH="Lynx_Point"
CONFIG_GFX_GMA_PANEL_2_PORT="Disabled"
CONFIG_GFX_GMA_ANALOG_I2C_PORT="PCH_DAC"
# CONFIG_DRIVERS_NXP_UWB_SR1XX is not set
CONFIG_DRIVERS_PS2_KEYBOARD=y
CONFIG_DRIVERS_MC146818=y
# CONFIG_USE_PC_CMOS_ALTCENTURY is not set
CONFIG_PC_CMOS_BASE_PORT_BANK0=0x70
CONFIG_MEMORY_MAPPED_TPM=y
CONFIG_TPM_TIS_BASE_ADDRESS=0xfed40000
# CONFIG_DRIVERS_SIL_3114 is not set
CONFIG_DRIVERS_WIFI_GENERIC=y
CONFIG_DRIVERS_MTK_WIFI=y
# end of Generic Drivers

#
# Security
#

#
# CBFS verification
#
# CONFIG_CBFS_VERIFICATION is not set
# end of CBFS verification

#
# Verified Boot (vboot)
#
# end of Verified Boot (vboot)

#
# Trusted Platform Module
#
CONFIG_TPM1=y
# CONFIG_TPM2 is not set
CONFIG_TPM=y
CONFIG_MAINBOARD_HAS_TPM1=y
# CONFIG_TPM_DEACTIVATE is not set
# CONFIG_DEBUG_TPM is not set
CONFIG_PCR_BOOT_MODE=1
CONFIG_PCR_HWID=1
CONFIG_PCR_SRTM=2
CONFIG_PCR_FW_VER=10
CONFIG_PCR_RUNTIME_DATA=3
# end of Trusted Platform Module

#
# Memory initialization
#
CONFIG_PLATFORM_HAS_DRAM_CLEAR=y
# CONFIG_SECURITY_CLEAR_DRAM_ON_REGULAR_BOOT is not set
# end of Memory initialization

# CONFIG_INTEL_TXT is not set
# CONFIG_STM is not set
# CONFIG_INTEL_CBNT_SUPPORT is not set
CONFIG_BOOTMEDIA_LOCK_NONE=y
# CONFIG_BOOTMEDIA_LOCK_CONTROLLER is not set
# CONFIG_BOOTMEDIA_LOCK_CHIP is not set
# end of Security

CONFIG_ACPI_HAVE_PCAT_8259=y
CONFIG_ACPI_INTEL_HARDWARE_SLEEP_VALUES=y
CONFIG_ACPI_SOC_NVS=y
CONFIG_ACPI_NO_CUSTOM_MADT=y
CONFIG_ACPI_COMMON_MADT_LAPIC=y
CONFIG_ACPI_COMMON_MADT_IOAPIC=y
CONFIG_HAVE_ACPI_TABLES=y
CONFIG_BOOT_DEVICE_SPI_FLASH=y
CONFIG_BOOT_DEVICE_MEMORY_MAPPED=y
CONFIG_BOOT_DEVICE_SUPPORTS_WRITES=y
CONFIG_RTC=y

#
# Console
#
CONFIG_BOOTBLOCK_CONSOLE=y
CONFIG_POSTCAR_CONSOLE=y
CONFIG_SQUELCH_EARLY_SMP=y
# CONFIG_SPKMODEM is not set
# CONFIG_CONSOLE_NE2K is not set
CONFIG_CONSOLE_CBMEM=y
# CONFIG_CONSOLE_CBMEM_DUMP_TO_UART is not set
# CONFIG_CONSOLE_SPI_FLASH is not set
# CONFIG_CONSOLE_I2C_SMBUS is not set
# CONFIG_EM100PRO_SPI_CONSOLE is not set
# CONFIG_DEFAULT_CONSOLE_LOGLEVEL_8 is not set
CONFIG_DEFAULT_CONSOLE_LOGLEVEL_7=y
# CONFIG_DEFAULT_CONSOLE_LOGLEVEL_6 is not set
# CONFIG_DEFAULT_CONSOLE_LOGLEVEL_5 is not set
# CONFIG_DEFAULT_CONSOLE_LOGLEVEL_4 is not set
# CONFIG_DEFAULT_CONSOLE_LOGLEVEL_3 is not set
# CONFIG_DEFAULT_CONSOLE_LOGLEVEL_2 is not set
# CONFIG_DEFAULT_CONSOLE_LOGLEVEL_1 is not set
# CONFIG_DEFAULT_CONSOLE_LOGLEVEL_0 is not set
CONFIG_DEFAULT_CONSOLE_LOGLEVEL=7
CONFIG_CONSOLE_USE_LOGLEVEL_PREFIX=y
CONFIG_CONSOLE_USE_ANSI_ESCAPES=y
CONFIG_HWBASE_DEBUG_CB=y
# end of Console

CONFIG_HAVE_ACPI_RESUME=y
CONFIG_RESUME_PATH_SAME_AS_BOOT=y
CONFIG_HAVE_MONOTONIC_TIMER=y
CONFIG_HAVE_OPTION_TABLE=y
CONFIG_IOAPIC=y
CONFIG_USE_WATCHDOG_ON_BOOT=y

#
# System tables
#
CONFIG_GENERATE_SMBIOS_TABLES=y
CONFIG_BIOS_VENDOR="coreboot"
CONFIG_MAINBOARD_SERIAL_NUMBER="123456789"
# end of System tables

#
# Payload
#
# CONFIG_PAYLOAD_NONE is not set
# CONFIG_PAYLOAD_ELF is not set
# CONFIG_PAYLOAD_FLAT_BINARY is not set
# CONFIG_PAYLOAD_BOOTBOOT is not set
# CONFIG_PAYLOAD_FILO is not set
# CONFIG_PAYLOAD_GRUB2 is not set
# CONFIG_PAYLOAD_SEAGRUB is not set
# CONFIG_PAYLOAD_LINUXBOOT is not set
CONFIG_PAYLOAD_SEABIOS=y
# CONFIG_PAYLOAD_UBOOT is not set
# CONFIG_PAYLOAD_EDK2 is not set
# CONFIG_PAYLOAD_LINUX is not set
CONFIG_PAYLOAD_FILE="payloads/external/SeaBIOS/seabios/out/bios.bin.elf"
CONFIG_SEABIOS_BOOTORDER_FILE=""
CONFIG_PAYLOAD_BUILD_SEABIOS=y
CONFIG_SEABIOS_STABLE=y
# CONFIG_SEABIOS_MASTER is not set
# CONFIG_SEABIOS_REVISION is not set
# CONFIG_SEABIOS_THREAD_OPTIONROMS is not set
CONFIG_SEABIOS_VGA_COREBOOT=y
# CONFIG_SEABIOS_ADD_SERCON_PORT_FILE is not set
CONFIG_PAYLOAD_VGABIOS_FILE="payloads/external/SeaBIOS/seabios/out/vgabios.bin"
CONFIG_SEABIOS_DEBUG_LEVEL=-1

#
# Using default SeaBIOS log level
#
CONFIG_PXE=y

#
# PXE Options
#
# CONFIG_PXE_ROM is not set
CONFIG_BUILD_IPXE=y
CONFIG_IPXE_STABLE=y
# CONFIG_IPXE_MASTER is not set
# CONFIG_IPXE_SERIAL_CONSOLE is not set
# CONFIG_IPXE_NO_PROMPT is not set
# CONFIG_IPXE_ADD_SCRIPT is not set
# CONFIG_IPXE_HAS_HTTPS is not set
# CONFIG_IPXE_TRUST_CMD is not set
# end of PXE Options

# CONFIG_COMPRESSED_PAYLOAD_NONE is not set
CONFIG_COMPRESSED_PAYLOAD_LZMA=y
# CONFIG_COMPRESSED_PAYLOAD_ZSTD is not set
# CONFIG_COMPRESSED_PAYLOAD_LZ4 is not set
CONFIG_COMPRESS_SECONDARY_PAYLOAD=y

#
# Secondary Payloads
#
CONFIG_COREINFO_SECONDARY_PAYLOAD=y
# CONFIG_GRUB2_SECONDARY_PAYLOAD is not set
CONFIG_MEMTEST_SECONDARY_PAYLOAD=y
CONFIG_NVRAMCUI_SECONDARY_PAYLOAD=y
# CONFIG_TINT_SECONDARY_PAYLOAD is not set
# CONFIG_COREDOOM_SECONDARY_PAYLOAD is not set

#
# Memtest86+
#
CONFIG_MEMTEST86PLUS_V6=y
# CONFIG_MEMTEST86PLUS_V5 is not set
CONFIG_MEMTEST86PLUS_ARCH_64=y
# CONFIG_MEMTEST86PLUS_ARCH_32 is not set
CONFIG_MEMTEST_STABLE=y
# CONFIG_MEMTEST_MAIN is not set
# CONFIG_MEMTEST_REVISION is not set
# end of Secondary Payloads
# end of Payload

#
# Debugging
#

#
# CPU Debug Settings
#
# CONFIG_DISPLAY_MTRRS is not set

#
# Vendorcode Debug Settings
#

#
# BLOB Debug Settings
#

#
# General Debug Settings
#
# CONFIG_DEBUG_CBFS is not set
CONFIG_HAVE_DEBUG_SMBUS=y
# CONFIG_DEBUG_SMBUS is not set
# CONFIG_DEBUG_MALLOC is not set
# CONFIG_DEBUG_CONSOLE_INIT is not set
# CONFIG_DEBUG_SPI_FLASH is not set
# CONFIG_DEBUG_BOOT_STATE is not set
# CONFIG_DEBUG_ADA_CODE is not set
CONFIG_HAVE_EM100_SUPPORT=y
# CONFIG_EM100 is not set
# CONFIG_DEBUG_ACPICA_COMPATIBLE is not set
# end of Debugging

CONFIG_RAMSTAGE_ADA=y
CONFIG_RAMSTAGE_LIBHWBASE=y
CONFIG_HWBASE_DYNAMIC_MMIO=y
CONFIG_HWBASE_DEFAULT_MMCONF=0xf0000000
CONFIG_HWBASE_DIRECT_PCIDEV=y
CONFIG_DECOMPRESS_OFAST=y

#
# Boot Logo Configuration
#
# CONFIG_BMP_LOGO is not set
CONFIG_PLATFORM_POST_RENDER_DELAY_SEC=5
CONFIG_PLATFORM_OFF_MODE_CHARGING_INDICATOR_LOGO_PATH="3rdparty/blobs/mainboard/$(MAINBOARDDIR)/off_mode_charging.bmp"
# CONFIG_FRAMEBUFFER_SPLASH_TEXT is not set
# end of Boot Logo Configuration

CONFIG_WARNINGS_ARE_ERRORS=y
CONFIG_MAX_REBOOT_CNT=3
CONFIG_RELOCATABLE_MODULES=y
CONFIG_GENERIC_GPIO_LIB=y
CONFIG_HAVE_BOOTBLOCK=y
CONFIG_HAVE_ROMSTAGE=y
CONFIG_HAVE_RAMSTAGE=y
```

> **Path note:** All `_PATH` and `_FILE` entries above use `/home/user/coreboot/...` as a placeholder. Replace with your actual username and directory layout. The VBT has no path entry here — coreboot picks it up automatically from `src/mainboard/lenovo/t540p/data.vbt` inside the source tree (placed in Part 6).

---

## Part 7a — Optional: PXE Boot via iPXE

If you want network boot, coreboot can build iPXE from source and embed it directly in the ROM. No pre-built ROM file is needed — the coreboot build system fetches and compiles iPXE automatically when `CONFIG_BUILD_IPXE=y` is set.

### How it works

The W541's onboard Gigabit Ethernet is an Intel I217-LM (`PCI ID 8086:153a`). Setting `CONFIG_PXE_ROM_ID="8086,153a"` tells coreboot to associate the compiled iPXE ROM with that specific NIC. SeaBIOS then finds it and makes the NIC available as a PXE boot option.

### What the iPXE options add

The default iPXE build covers standard PXE/DHCP network boot which is all you need for a local network boot server like iVentoy. HTTPS and trust/certificate options are not enabled — local network boot servers don't typically serve over HTTPS and there's no benefit to the extra complexity.

### Secondary payloads

The PXE config also adds two secondary payloads accessible from the SeaBIOS boot menu:
- **Coreinfo** — shows system information, CBFS contents, and memory map; useful for diagnostics
- **Memtest86+ v6** (64-bit) — full memory test

These are compressed and stored in CBFS, which is another reason for the larger `CBFS_SIZE`.

### To disable PXE / revert to base config

Remove or comment out the PXE block and secondary payload lines, and revert `CONFIG_CBFS_SIZE` back to `0x600000`.

---

## Part 8 — Build Coreboot

### 8.1 Install Build Dependencies

```bash
sudo apt install -y \
  build-essential gnat flex bison libncurses5-dev wget zlib1g-dev \
  git python3 libelf-dev nasm imagemagick pkg-config libssl-dev gcc-multilib libc6-dev-i386 \
  iasl unifont
```

### 8.2 Build the Toolchain

```bash
cd coreboot
make crossgcc-i386 CPUS=$(nproc)
# Takes 30–90 minutes on first run
```

### 8.3 Configure and Build

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

## Part 9 — Split the ROM for Dual-Chip Flashing

```bash
dd if=build/coreboot.rom of=bottom.rom bs=1M count=8
dd if=build/coreboot.rom of=top.rom    bs=1M skip=8

ls -la bottom.rom top.rom
# bottom.rom = 8388608 bytes  → Chip 1 (near white connector)
# top.rom    = 4194304 bytes  → Chip 2 (near board edge)
```

If you flash the wrong size to a chip, flashprog refuses with a size mismatch — no damage, just swap them.

---

## Part 10 — Flash

### 10.1 Flash Chip 1 (8 MiB)

Full clip on Chip 1, 47Ω hook on Chip 2 pin 1 → 3.3V.

```bash
# Pi 3/4:
sudo flashprog -p linux_spi:dev=/dev/spidev0.0,spispeed=1000 -w bottom.rom
sudo flashprog -p linux_spi:dev=/dev/spidev0.0,spispeed=1000 -v bottom.rom

# Pico:
# sudo flashprog -p serprog:dev=/dev/ttyACM0,spispeed=16M -w bottom.rom
# sudo flashprog -p serprog:dev=/dev/ttyACM0,spispeed=16M -v bottom.rom

# Must report: VERIFIED
```

### 10.2 Flash Chip 2 (4 MiB)

Move clip to Chip 2, move hook to Chip 1 pin 1 → 3.3V.

```bash
# Pi 3/4:
sudo flashprog -p linux_spi:dev=/dev/spidev0.0,spispeed=1000 -w top.rom
sudo flashprog -p linux_spi:dev=/dev/spidev0.0,spispeed=1000 -v top.rom

# Pico:
# sudo flashprog -p serprog:dev=/dev/ttyACM0,spispeed=16M -w top.rom
# sudo flashprog -p serprog:dev=/dev/ttyACM0,spispeed=16M -v top.rom

# Must report: VERIFIED
```

---

## Part 11 — First Boot Test

**Do not fully reassemble before testing.** The W541 can be bench-tested with just the motherboard partially installed. You only need:

- Motherboard seated in the chassis (or on a non-conductive surface)
- CPU and GPU cooler attached (thermal paste applied, fans connected)
- Power button ribbon connected
- Display cable connected and screen laid flat or propped up
- AC adapter connected (battery not required)
- USB keyboard plugged in

You do **not** need: the palmrest, keyboard, bottom cover, battery, or any other ribbons. This keeps the board accessible in case you need to re-clip for a recovery flash.

**Boot procedure:**

1. Disconnect the Pi/Pico from USB (remove all programmer wiring from the flash chips first).
2. Connect AC power.
3. Press the power button — you should see a coreboot/SeaBIOS/GRUB screen within 30 seconds.
4. Both GPUs should initialise: coreboot logs at the top of the screen will show iGPU init and the Nvidia VGA ROM executing.
5. Once you have confirmed a successful boot, proceed to the nvramtool step before full reassembly.

**If no display after 2 minutes:** The board is still accessible — re-attach clips and flash `factory_12mb.bin` back (split it the same way):

```bash
dd if=factory_12mb.bin of=factory_chip1.bin bs=1M count=8
dd if=factory_12mb.bin of=factory_chip2.bin bs=1M skip=8
# Flash factory_chip1.bin to Chip 1, factory_chip2.bin to Chip 2
# (same flashprog commands as Part 10, substitute filenames)
```

---

## Part 12 — Post-Flash: Enable Both GPUs with nvramtool

Coreboot's CMOS defaults may not enable the discrete GPU. `nvramtool` reads and writes coreboot's CMOS option table — it must run **on the W541 itself** (it needs access to the physical CMOS of the machine whose firmware you want to configure).

### 12.1 Build nvramtool

Build nvramtool on your **host/build machine** (or on the Pi if that's more convenient), then copy the binary to the W541 via USB stick, SCP, or any other method:

```bash
# On your build machine:
cd coreboot/util/nvramtool
make
# Binary: coreboot/util/nvramtool/nvramtool

# Transfer to the W541 (example via USB stick):
cp nvramtool /media/$USER/<usb>/
# On the W541 after boot:
# sudo cp /media/$USER/<usb>/nvramtool /usr/local/bin/
# sudo chmod +x /usr/local/bin/nvramtool
```

Alternatively, if the W541 has network access after first boot, build it directly on the machine:

```bash
# On the W541 itself (needs build-essential):
sudo apt install build-essential
cd coreboot/util/nvramtool   # or copy the source tree over
make
sudo make install
```

### 12.2 List Available CMOS Options

```bash
sudo nvramtool -l
# Look for graphics/GPU-related entries, e.g.:
#   hybrid_graphics_mode  (or similar)
#   peg_deassert_wake
```

The W541/T540p board typically exposes an option along the lines of:

```
graphics_mode = [auto | discrete | integrated]
```

### 12.3 Enable Both GPUs

```bash
# List current values
sudo nvramtool -a

# Enable hybrid/both GPU mode — check the exact option name from step 12.2
sudo nvramtool -w hybrid_graphics_mode=enable

# Alternatively, if the option is named differently:
sudo nvramtool -w graphics=both
```

After setting, reboot the W541. On next boot coreboot reads the CMOS value and enables both the iGPU and K1100M.

### 12.4 Verify Both GPUs Are Present

```bash
lspci | grep -E "VGA|3D|Display"
# Expected:
# 00:02.0 VGA compatible controller: Intel Corporation 4th Gen Core ... Integrated Graphics
# 01:00.0 3D controller: NVIDIA Corporation GK107GLM [Quadro K1100M]
```

---

## Part 13 — Post-Install: What Works and What Doesn't

### What works

- **Intel HD 4600 iGPU** — fully functional, internal display, libgfxinit init at boot
- **Nvidia K1100M dGPU** — appears on PCI bus, initialised by VGA ROM at boot, usable via nouveau or Nvidia 390xx driver ✓ tested
- **External displays via dock** — functional once the dGPU is up (dock outputs route through the Nvidia chip)
- **USB, storage, networking, audio** — all normal
- **Internal re-flashing** via `flashprog -p internal` — works once coreboot is running
- **General Linux desktop use** — stable

### What does not work

- **Suspend / sleep (S3)** — **confirmed broken**. The machine will appear to suspend but does not resume correctly. Do not rely on suspend.
- **Hibernate** — not tested; likely broken for the same reasons.

### Power draw

With the dGPU enabled via this method, **the K1100M is always active** — there is no runtime power-gating or Optimus-style switching off. Expect noticeably higher idle power draw and more fan activity compared to stock or Libreboot (iGPU-only). Battery life will be shorter than with Libreboot.

### Disabling suspend and hibernate in the OS

Since suspend is broken, disable it to prevent accidental triggering. The exact method varies by desktop environment and distro — search for how to disable suspend and lid-close sleep for your specific setup. On systemd-based systems `systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target` is the blunt instrument that works everywhere.

---

## Part 14 — dGPU Driver Setup

```bash
lspci | grep -E "VGA|3D|Display"
# 00:02.0 VGA compatible controller: Intel Corporation 4th Gen Core ... Integrated Graphics
# 01:00.0 3D controller: NVIDIA Corporation GK107GLM [Quadro K1100M]
```

**nouveau (default, no install needed):**
```bash
# GPU offload via PRIME
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

## Part 15 — Internal Re-Flashing (Future Updates)

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

**Nvidia ROM not found / Python script finds nothing:**
Confirm `UEFIExtract` ran successfully and `flashregion_1_bios.bin.dump/` was created. Re-run the Python script and check it's walking that directory. If the dump exists but no `0x10DE` ROM is found, check `flashregion_1_bios.bin` is the correct region (8 MiB BIOS region, not the full 12 MiB image). As a last resort the PCI sysfs method (`echo 1 | sudo tee /sys/bus/pci/devices/0000:01:00.0/rom && dd if=.../rom of=nvidia.rom`) can extract it from a running system with `nouveau` loaded.

**Reads fail or are inconsistent:**
IPA on chip legs. Try `spispeed=512` (Pi) or `spispeed=4M` (Pico). Specify chip: `flashprog ... -c "W25Q64.V"`. Confirm idle chip's /CS is held high.

**Boot shows black screen:**
Both chips verified? If yes, suspect the Nvidia VGA ROM — verify `55 aa` signature and `strings nvidia_k1100m.rom | grep -i nvidia`. Check `.config` VGA ROM path and PCI ID (`10de,0ff6` for K1100M, `10de,11fc` for K2100M — must match your actual GPU). Try building without the Nvidia ROM first to confirm the iGPU init works, then add it back.

**K1100M not in lspci after boot:**
Run the nvramtool step (Part 12) to enable both GPUs in CMOS. Also check ACPI power management: `sudo sh -c 'echo on > /sys/bus/pci/devices/0000:01:00.0/power/control'`.

**nvramtool option names don't match:**
Run `sudo nvramtool -l` to list all available CMOS options on your specific build. Option names depend on the coreboot board source at the time of your build. Cross-reference with `src/mainboard/lenovo/t540p/cmos.layout` in the coreboot source tree.

**build/coreboot.rom is wrong size:**
The flash map (board.fmd) must match the actual chip layout. The W541 is 12 MiB total (8+4). If the build produces something else, check `CONFIG_CBFS_SIZE` and `CONFIG_FMDFILE`.

---

## Appendix A — What Was Discarded / Doesn't Work

- **lbmk (Libreboot build system) for anything other than pico-serprog** — lbmk is used in this guide for one purpose only: building the serprog firmware UF2 for the Raspberry Pi Pico. The coreboot ROM is built entirely from upstream coreboot. If you're using a Pi 3/4, lbmk is not needed at all.
- **Running me_cleaner manually as a separate step** — not needed. `CONFIG_USE_ME_CLEANER=y` in the `.config` handles ME neutering at build time; just pass the raw `flashregion_2_intel_me.bin`.
- **intel_bios_dumper / PCI sysfs extraction from a running system** — unnecessary. Both the Nvidia ROM and the Intel VBT are reliably extracted from the factory dump using UEFIExtract without needing the W541 to be running at all.
- **stacksmashing pico-serprog** — use Libreboot's fork (multi-CS, tuned drive strength).
- **Raspberry Pi Pico 2 (RP2350)** — serprog stability issues; use original Pico (RP2040) only.
- **flashrom** — replaced by flashprog; use flashprog throughout.
- **Wrong Pico pinout (GP3↔GP5 swapped)** — GP5=/CS, GP3=MOSI as shown in Part 2.
- **Skipping /HOLD (Pin 7) wiring** — not wired in the tested setup and it worked. However the PCB pull-up may be marginal on some boards; if you see unexplained intermittent read failures, add a wire to 3.3V. It is always safe to do so.
- **Holding pin 7 of idle chip** — it's pin 1 (/CS) that matters for the idle chip. SDK08 test clips are ideal for this since you only need to probe a single pin, and you can solder the 47Ω resistor inline on the probe wire.
- **Single-chip assumption** — the W541 has two chips; any guide that doesn't address this produces unreliable results.
- **`CONFIG_BOARD_LENOVO_THINKPAD_W541` vs `T540P`** — the W541 uses the T540p board definition in upstream coreboot; check that the board Kconfig option matches what `make menuconfig` shows for your coreboot revision.

---

## Appendix B — Blob Summary

| Blob | Source | Destination in coreboot tree |
|------|--------|------------------------------|
| `mrc.bin` | T440p coreboot build (Haswell-universal) | `coreboot-blobs/mrc.bin` |
| `flashregion_2_intel_me.bin` | factory dump → ifdtool — passed raw; `CONFIG_USE_ME_CLEANER=y` cleans it at build time | `coreboot-blobs/flashregion_2_intel_me.bin` |
| `flashregion_3_gbe.bin` | factory dump → ifdtool | `coreboot-blobs/flashregion_3_gbe.bin` |
| `flashregion_0_flashdescriptor.bin` | factory dump → ifdtool | `coreboot-blobs/flashregion_0_flashdescriptor.bin` |
| `data.vbt` | factory dump → ifdtool → UEFIExtract → `$VBT` section | `src/mainboard/lenovo/t540p/data.vbt` (inside coreboot source tree) |
| `pci10de,0ff6.rom` | factory dump → ifdtool → UEFIExtract → Python PCI scan (K1100M) | `coreboot-blobs/pci10de,0ff6.rom` |
| `pci10de,11fc.rom` | factory dump → ifdtool → UEFIExtract → Python PCI scan (K2100M) — *not tested end-to-end* | `coreboot-blobs/pci10de,11fc.rom` |

---

## Appendix C — Programmer Comparison

| | Raspberry Pi 3/4 | Raspberry Pi Pico |
|---|---|---|
| Cost | ~£15–35 | ~£4 |
| Read speed | ~500 KB/s at 1MHz | ~2 MB/s at 16MHz |
| 3.3V distribution | 2 pins (1 and 17) — easy 2-way crimps | 1 pin (36) — needs 4-way split/breadboard |
| flashprog interface | `linux_spi` (native kernel driver) | `serprog` (USB serial) |
| Setup | Full OS, raspi-config, build flashprog | Flash UF2, done |
| Verdict | Easier wiring; good if you have one already | Cheapest option; slightly awkward 3.3V bus |

---

---

## Credits and References

**Coreboot .config** — based on and modified from the ThinkPad W541 coreboot + Tianocore guide by **Reventon1988** on Reddit:  
https://www.reddit.com/r/coreboot/comments/12oeag8/thinkpad_w541_coreboottianocore_guide/

**SPI wiring and programmer setup** — Libreboot's SPI flashing documentation is the authoritative reference for both the Pi 3/4 and Pico serprog wiring:  
https://libreboot.org/docs/install/spi.html

**Everything else** was cobbled together from various forum posts, the coreboot wiki, repair.wiki, and whatever turned up in searches. The Python scripting for ROM extraction in particular was assembled with heavy assistance from Claude.

*Validated hands-on, February 2026. K1100M confirmed working. K2100M untested.*
