# Adding a New Platform to SONiC

With the Accton ECS5550-30X as a worked example

How to read this document:

- **Part I – Base knowledge**: what a SONiC platform is, how the image boots,
  and the terms you'll meet.
- **Part II – Procedure**: the general steps for bringing up a brand-new board.
- **Part III – What to request from the vendor**: a checklist.
- **Part IV – Worked example**: the ECS5550-30X on
  `MarvellEmbeddedProcessors/marvell-sonic-buildimage` (branch `rls-202511.01`),
  mapped step by step onto Part II, with the problems we hit.
- **Appendix**: files referenced.

---

# Part I – Base knowledge

## 1. What a "platform" is in SONiC

A SONiC image is one generic Debian-based OS plus per-platform pieces. Adding a
platform means supplying every layer below:

| Layer | What it does | Where it lives in sonic-buildimage |
|---|---|---|
| Bootloader / boot image | Gets the kernel running on this board | `platform/<asic>/platform_<arch>.conf`, `sonic_fit.its` (arm), `installer/` |
| Device tree (arm only) | Tells the kernel what hardware exists | `.dtb` file, packed into the FIT |
| Kernel + drivers | SoC support, CPLD, i2c mux, fan, PSU, SFP, LED, watchdog, ASIC kernel module | `src/sonic-linux-kernel`, `platform/<asic>/sonic-platform-<vendor>/` |
| Platform API | Python `sonic_platform` package used by pmon and the CLI | platform package, or a `.whl` in the device folder |
| Device / HWSKU data | Port list, SAI profile, ASIC config, buffers/QoS | `device/<vendor>/<onie_platform>/` |
| SAI + SDK | Library that programs the switch ASIC | `platform/<asic>/sai.mk` (+ kernel module) |

The name that ties everything together is the **ONIE platform string**,
`<arch>-<vendor>_<model>-r<rev>` (e.g. `arm64-accton_ecs5550_30x-r0`).
- SONiC looks up `/usr/share/sonic/device/<onie_platform>/` at runtime.
- The installer only accepts platforms on its supported list.
- Platform packages are installed only on the matching platform.

Read it with `onie-sysinfo -p` in ONIE, or from `onie_platform=` in `/host/machine.conf`
on an installed system.

## 2. How a SONiC image gets onto the box and runs

```
build host                         switch
──────────                         ──────
make target/sonic-<asic>.bin  ──►  ONIE (or sonic-installer) runs the .bin
                                     │  installer reads onie_platform, checks platforms_asic,
                                     │  runs platform_<arch>.conf: picks disk, writes boot config
                                     ▼
                                   Bootloader: U-Boot (arm) / GRUB (x86)
                                     │  arm: loads sonic_<arch>.fit, bootm <addr>#<conf>
                                     ▼
                                   Linux kernel (+ dtb on arm) + initrd
                                     │  mounts fs.squashfs as root
                                     ▼
                                   SONiC host OS
                                     │  /usr/share/sonic/device/<onie_platform>/  ← device folder
                                     │  first boot: installs lazy platform package (drivers, sonic_platform)
                                     ▼
                                   containers: database, swss, syncd (loads libsai → ASIC),
                                               pmon (sonic_platform → drivers), bgp, lldp, …
```

Each arrow is a place where a new platform can fail. That's why bring-up
(Part II §8) goes one layer at a time.

## 3. Glossary

**ONIE** (Open Network Install Environment)
A small Linux that ships on the switch and installs a network OS. It identifies
the box via `onie_platform` / `onie_machine` in `/etc/machine.conf`. A SONiC
`.bin` is a self-extracting installer that ONIE (or `sonic-installer`) runs.

**U-Boot / GRUB**
The bootloader. Most arm switches use U-Boot; x86 switches use GRUB. U-Boot
runs commands stored in its environment.

**U-Boot environment ("env")**
Key/value variables stored in flash (an MTD partition), readable from Linux with
`fw_printenv` and writable with `fw_setenv`. `/etc/fw_env.config` tells those
tools where the env is: device, offset, size, erase size (e.g.
`/dev/mtd1 0x0 0x10000 0x10000`). The SONiC installer writes variables such as
`fit_addr`, `fit_name`, `fit_conf_name`, `sonic_image_1`, `sonic_image_2`,
`bootcmd`. A wrong env means the board won't boot SONiC.

**DTS / DTB (Device Tree Source / Blob)**
On arm there is no BIOS/ACPI to describe hardware, so the kernel is given a
*device tree*: a tree of nodes such as "an eMMC controller at address X",
"an i2c bus with an EEPROM at 0x54", "a CPLD", each with a `compatible`
string that selects the kernel driver.
- `.dts` = human-readable source; `.dtb` = compiled binary
  (`dtc -I dts -O dtb`).
- The dtb is **board-specific**. Two boards with the same CPU still need
  different dtbs because their peripherals differ.
- If a node's `compatible` has no matching driver in your kernel, that device
  simply doesn't appear. Nothing fails loudly.

**FIT (Flattened Image Tree)**
U-Boot's container format for booting. One `.fit` file holds several
images (kernels, initrds, dtbs). It also holds named *configurations*, each of
which picks one kernel + one initrd + one dtb.
- `.its` = text description (itself device-tree syntax), listing each image
  with `data = /incbin/("file")`, load address, entry address, and hash.
- `mkimage -f sonic_fit.its sonic_arm64.fit` builds it;
  `dumpimage -l file.fit` lists its contents.
- U-Boot boots a configuration with `bootm <fit_addr>#<conf_name>`.
- SONiC builds the FIT inside the image at build time (`build_debian.sh`,
  around line 773). A new arm board needs its own entries and configuration in
  `sonic_fit.its`.

**Load / entry address**
Where U-Boot copies an image in RAM before jumping to it. These must not
overlap each other, or the FIT itself (`fit_addr`), and must be inside valid RAM.
They're board-specific. Get them from the vendor or from a working image.

**initrd / initramfs**
Small root filesystem the kernel uses early in boot to mount the real
(squashfs) SONiC root.

**SAI** (Switch Abstraction Interface)
A standard C API (`sai*.h`) between SONiC and the ASIC vendor's library.
The vendor ships `libsai.so` (Marvell: package `mrvllibsai`). SAI has versions
(1.14, 1.15, 1.17 …); a SONiC branch's sairedis/syncd is compiled against a
specific SAI header version, so the vendor library must be close to it.

**SDK / CPSS / mvcpss.ko**
The ASIC vendor's low-level driver stack under SAI. On Marvell Prestera it's CPSS, and
`mvcpss.ko` is its kernel module. Like any module it must match the running kernel
(see *vermagic*).

**vermagic**
A string inside every `.ko`, for example `6.1.0-22-2-arm64 SMP mod_unload modversions aarch64`.
The kernel refuses modules whose vermagic doesn't match. Prebuilt modules only work on
the exact kernel they were built for. That's why you need driver **source**.

**syncd / swss / orchagent**
SONiC containers. swss turns config into ASIC_DB entries; syncd calls SAI
to program the hardware. If SAI can't initialize the ASIC, syncd fails and no
front-panel ports work.

**pmon / sonic_platform / platform API**
The pmon container runs daemons (xcvrd, psud, thermalctld, ledd, syseepromd,
pcied). They call the Python `sonic_platform` package, which reads the
platform kernel drivers via sysfs. `pmon_daemon_control.json` turns daemons
on and off per platform.

**PDDF** (Platform Driver Development Framework)
Generic drivers and platform API, driven by a JSON description of the
board's i2c/CPLD devices (`pddf-device.json`). It's an alternative to writing
per-board drivers and `sonic_platform` by hand.

**HWSKU**
A sub-folder of the device folder describing one port/speed configuration.
Selected by `default_sku` (`<HWSKU> <preset>`, preset e.g. `t1`, `l2`).

**platform_asic**
One-line file in the device folder. It must equal the image's
`TARGET_MACHINE` (e.g. `marvell-prestera`, `broadcom`). Otherwise
`build_image.sh` leaves the platform off the supported list and the installer
refuses it.

**sai.profile / vendor ASIC config**
`sai.profile` holds key=value settings passed to the vendor SAI. The ASIC config
format is vendor-specific:
- Marvell: `hwId`, plus `SAI-*.xml`, `ASK-Board/L1/PP-*.xml` and `profile.ini`.
- Broadcom: `*.config.bcm`.

They're tied to the SAI release that generated them.

**port_config.ini**
Maps SONiC port names (`Ethernet0`…) to ASIC lanes, alias, speed, FEC and
front-panel index.

**platform.json / hwsku.json**
Inventory of the chassis (fans, PSUs, thermals, SFPs) and the interface/breakout
definitions used by newer SONiC features (Dynamic Port Breakout, CLI).

**Lazy install**
Platform packages listed as `_LAZY_INSTALLS` are included in the image but only
installed at first boot, and only on the matching platform.

**QEMU multiarch build**
Building arm64 on an amd64 host runs arm64 binaries under QEMU user-mode
emulation, which is mostly single-threaded and 5–20× slower. `make configure`
and the build take hours regardless of CPU count. An arm64 build host avoids it.

---

# Part II – Procedure for a new platform

The steps follow the order bring-up usually happens in. Each one says what you
have to produce, and Part IV shows what that looked like for the ECS5550-30X.

## 4. Decide before you start

- **ASIC + SAI availability is the hard gate.** Without a vendor `libsai` for
  the ASIC, plus a SONiC branch whose SAI headers match it, the OS can boot
  but no front-panel port will ever work. Confirm this first.
- **Pick the SONiC branch to match the SAI**, not the other way round.
- **CPU architecture**: x86_64 (GRUB, no device tree) or arm64/armhf
  (U-Boot + dtb + FIT). §7 differs per arch.
- **ONIE present?** If not, you need ONIE ported to the board first, or a
  vendor-specific install path.
- **Is there a working vendor SONiC image?** If yes, you can extract most inputs
  from it (see Part IV). If not, everything below comes from the vendor or
  from you.
- **ONIE platform string**: use the same string for the device folder, the platform package,
  and the installer case.
- **Build host**: use an arm64 build host for arm targets if you can (see QEMU above).

## 5. Device folder: `device/<vendor>/<onie_platform>/`

The numbers show how many device folders in this repo contain each file, as a guide
to what's normal:

| File | Need | Purpose |
|---|---|---|
| `default_sku` | required (250) | `<HWSKU> <preset>` |
| `platform_asic` | required (244) | must equal `TARGET_MACHINE` |
| `installer.conf` | usual (199) | console speed, extra kernel cmdline |
| `platform.json` | strongly recommended (148) | inventory and interface/breakout definitions |
| `pmon_daemon_control.json` | usual (139) | which pmon daemons run |
| `sensors.conf` | common (118) | lm-sensors labels and limits |
| `pcie.yaml` | common (106) | expected PCIe devices for pcied (`pcieutil generate`) |
| `system_health_monitoring_config.json` | common (104) | what system-health ignores |
| `platform_components.json` | common (97) | firmware components for `fwutil` |
| `platform_env.conf` | ASIC-specific (69) | ASIC driver env (mostly Broadcom) |
| `media_settings.json` | common (49) | SerDes pre-emphasis per transceiver type (xcvrd) |
| `fancontrol`, `thermal_policy.json` | if SONiC does fan control | |
| `platform_reboot` | if reboot needs a CPLD/power-cycle | |
| `pddf_support` | if using PDDF | |

Per-HWSKU folder `device/<vendor>/<onie_platform>/<HWSKU>/`:

| File | Need | Purpose |
|---|---|---|
| `port_config.ini` | required (365) | name, lanes, alias, speed, index, (fec, autoneg) |
| `sai.profile` | required (362) | keys passed to the vendor SAI |
| vendor ASIC config | required | Marvell XML set / Broadcom `.config.bcm` / … |
| `hwsku.json` | recommended (178) | default breakout mode per port |
| `buffers.json.j2`, `buffers_defaults_t0/t1.j2`, `pg_profile_lookup.ini`, `qos.json.j2` | production configs (≈130–190) | MMU buffers, PFC, QoS. Values from the ASIC vendor |

Data that comes from the **hardware design** (schematics or the vendor):
- Front-panel port → ASIC lane mapping, lane polarity/swap, speed per port.
- SerDes tuning per port and per media type.
- i2c topology: which bus/mux channel reaches each SFP EEPROM, PSU, fan
  controller, temperature sensor, and the system EEPROM (base MAC lives here;
  SONiC reads it via `decode-syseeprom`).
- CPLD register map: SFP presence/reset/LP-mode, LEDs, fan PWM, PSU status,
  reset cause, watchdog.

The whole `device/` tree is packed by `sonic-device-data` into
`/usr/share/sonic/device/` in the image. No `.mk` change is needed for it.

## 6. Platform package: drivers + `sonic_platform`

Use the existing Marvell layout as the template:

```
platform/marvell-prestera/sonic-platform-marvell/
├── <board>/
│   ├── modules/ or utils/   # kernel drivers (CPLD, i2c mux, fan, PSU, SFP, LED, wdt)
│   ├── sonic_platform/      # Python platform API (chassis, sfp, psu, fan, thermal, eeprom, watchdog, component)
│   ├── service/             # systemd unit(s) for platform init (load modules, instantiate i2c devices)
│   ├── scripts/
│   └── setup.py             # builds sonic_platform-*.whl
└── debian/
    ├── control              # add a binary package sonic-platform-<board>
    ├── rules                # build modules against $(KVERSION), build the wheel
    ├── sonic-platform-<board>.install
    └── sonic-platform-<board>.postinst
```

Register it in the build (pattern from `platform-marvell.mk`):
```make
MYBOARD_PLATFORM = sonic-platform-myboard_1.0_$(CONFIGURED_ARCH).deb
$(MYBOARD_PLATFORM)_PLATFORM = arm64-vendor_myboard-r0       # = ONIE platform string
$(eval $(call add_extra_package,$(AC5X_RD98DX35xx_PLATFORM),$(MYBOARD_PLATFORM)))
```
Then add it to `one-image.mk`: `$(SONIC_ONE_IMAGE)_LAZY_INSTALLS += $(MYBOARD_PLATFORM)`.

Ways to reduce the work:
- **PDDF**: describe the devices in `pddf-device.json`, add `pddf_support`, and
  reuse the generic drivers and API, as long as the hardware fits its model.
- Start with **only `eeprom` + `sfp`** implemented and the other pmon daemons
  disabled. Add PSU/fan/thermal/LED afterwards.

## 7. Kernel, boot, and SAI integration

Kernel (`src/sonic-linux-kernel`):
- Enable any missing SoC / eMMC / Ethernet / i2c / SPI / GPIO options via
  `patch/kconfig-inclusions` (and `config.local/<arch>/config.sonic`).
- Add vendor kernel patches under `patches-sonic/` if the board needs them.
- Out-of-tree drivers go in the platform package (§6), built against the
  SONiC kernel headers.

arm (U-Boot):
1. Get or write the board `.dts`. Compile the `.dtb` and make sure it's in the
   rootfs before `mkimage` runs. Putting it in the device folder or the platform
   package both work.
2. Add kernel/initrd/fdt images and a `conf_<board>` to
   `platform/<asic>/sonic_fit.its` with safe load addresses.
3. Add a `case` entry for the ONIE platform string to
   `platform/<asic>/platform_<arch>.conf`: boot disk (`mmc_bus` / `scsi_bus`),
   `fit_conf_name`, `fit_addr`, `fdt_addr`, `initrd_addr`, `FW_ENV_DEFAULT`.
   Without an entry, the installer falls back to defaults (another board's dtb
   and addresses), and the board won't boot.
4. Verify the U-Boot env location (`/proc/mtd`, vendor docs) before the
   installer writes it. A wrong `fw_env.config` can corrupt the env.

x86 (GRUB):
- No dtb/FIT. `installer.conf` sets `CONSOLE_SPEED`, and optionally
  `CONSOLE_PORT` / `CONSOLE_DEV`, plus `ONIE_PLATFORM_EXTRA_CMDLINE_LINUX`.
- The default x86 installer usually works. Platform quirks go in
  `installer.conf` or the platform package.

SAI (`platform/<asic>/sai.mk`):
- Point the SAI version / URL at the vendor package (or a local mirror), and
  make sure the syncd docker depends on it.
- If the SAI needs a kernel module (e.g. mvcpss), build it from source in the
  platform package for the SONiC kernel.

Build:
- Raise package parallelism in `rules/config.user` (git-ignored), e.g.
  `SONIC_CONFIG_BUILD_JOBS = 8`.
- `make configure PLATFORM=<asic> PLATFORM_ARCH=<arch>`, then
  `make target/sonic-<asic>-<arch>.bin`.
- Commit local build-system fixes to a branch. Uncommitted patches and
  `config.user` won't follow the repo to another build server.

## 8. Bring-up order and checks

Do one layer at a time, with a serial console attached. Install with
`sonic-installer` when a SONiC image is already on the box (it keeps the old
image as a fallback). An ONIE install wipes the old image.

| Step | Goal | Check |
|---|---|---|
| 1 | Image installs and kernel boots | console shows login; `uname -r`; `dmesg` free of panics |
| 2 | Management network | `ip -br addr`, SSH in |
| 3 | Storage / partitions sane | `df -h /host`, `sonic-installer list` |
| 4 | Platform drivers load | `lsmod`, `i2cdetect -l`, `/sys/bus/i2c/devices/` |
| 5 | System EEPROM | `show platform syseeprom`, `decode-syseeprom -m` (base MAC) |
| 6 | Platform API | `python3 -c "import sonic_platform.platform as p; c=p.Platform().get_chassis(); print(c.get_name())"` |
| 7 | pmon daemons, one by one | `docker exec pmon supervisorctl status`; `show platform psustatus/fan/temperature` |
| 8 | Transceivers | `sfputil show presence`, `show interfaces transceiver eeprom` |
| 9 | SAI / syncd starts | `docker ps` (syncd Up), `/var/log/syslog` for `create_switch` errors |
| 10 | Ports and link | `show interfaces status`, cable a port, check oper up and speed/FEC |
| 11 | Traffic / L2/L3 | ping across ports, VLAN, BGP as needed |
| 12 | Regression | `sonic-mgmt` platform tests (`tests/platform_tests/`) if available |

Tips:
- If full init fails, start syncd with a **minimal `port_config.ini`** (a few
  ports). That separates lane-map errors from SAI or driver errors.
- Keep the fallback image until steps 1–3 pass. On U-Boot, `run sonic_image_2`
  at the prompt boots the previous image.
- Record every value you take from the vendor (addresses, i2c buses, lane map)
  and where it came from.

## 9. Upstreaming (optional)

To get the platform into `sonic-net/sonic-buildimage`:
- The device folder, platform package and `.mk` changes go in one PR. ASIC/SAI
  version bumps go in a separate PR.
- Closed vendor binaries (SAI, SDK) are fetched by URL, not committed.
- Expect reviewers to ask for `platform.json`, `hwsku.json`, a working
  platform API, and passing platform tests.

---

# Part III – What to request from the platform vendor

Ask for **source**, not just binaries. Binaries are tied to one kernel and one
SONiC branch and become unusable as soon as you move.

## 10. Checklist

### 10.1 Identity and baseline
- [ ] ONIE platform string and machine name (`onie_platform`, `onie_machine`).
- [ ] The SONiC branch, commit and kernel version their image is based on.
- [ ] A working reference image (for comparison, extraction and fallback).
- [ ] If no ONIE: an ONIE port or an alternative install procedure.

### 10.2 Hardware design
- [ ] Schematics, or at least: port→lane map, lane polarity/swap, i2c tree,
      CPLD register map, EEPROM address and format (TLV).
- [ ] Hardware spec / block diagram (CPU, RAM, ASIC, management PHY).

### 10.3 Boot
- [ ] Device tree **source** (`.dts`) for the board, plus any kernel patches it needs.
- [ ] U-Boot env layout: the `fw_env.config` line and the MTD partition map.
- [ ] RAM size and safe load addresses for FIT, kernel, dtb and initrd.
- [ ] Boot storage (eMMC / SATA / USB), device path, partition expectations.
- [ ] Console port and baud rate, how to reach the U-Boot prompt, and how to recover.
- [ ] Required kernel command-line arguments (e.g. hugepages, `pci=pcie_bus_safe`).
- [ ] Required kernel config options (SoC, eMMC, Ethernet, i2c, SPI, GPIO).

### 10.4 ASIC (SAI / SDK)
- [ ] `libsai` built for **your** SONiC branch (state the SAI version you need),
      with release notes.
- [ ] SDK kernel module **source** (Marvell: mvcpss / CPSS driver), or a build
      for your exact kernel.
- [ ] Firmware files the SAI loads (SerDes, service CPU, PHY).
- [ ] HWSKU files for each port configuration, generated for that SAI release.
- [ ] SerDes tuning per port and media type (`media_settings.json`), and FEC per speed.
- [ ] Buffer / QoS / PFC parameters for the ASIC and port configuration.
- [ ] Redistribution licence for the SAI/SDK binaries.

### 10.5 Platform peripherals
- [ ] Kernel driver **source**: CPLD/FPGA, i2c mux, fan, PSU, thermal sensors,
      SFP/QSFP (optoe), LEDs, watchdog, system EEPROM.
- [ ] `sonic_platform` Python **source**, or a PDDF `pddf-device.json`.
- [ ] `platform.json` / `hwsku.json` (inventory and breakout capability).
- [ ] Supported pmon daemons (`pmon_daemon_control.json`).
- [ ] Module load order and any platform init service.
- [ ] Expected PCIe devices (`pcie.yaml`), sensor limits (`sensors.conf`).
- [ ] Firmware components (CPLD/FPGA/BIOS/U-Boot) for `platform_components.json`.
- [ ] Reboot / power-cycle method and reset-cause registers.
- [ ] Debian packaging / `.mk` integration, if they have it.

### 10.6 Support
- [ ] Known issues and test results for their image.
- [ ] Contact for SAI questions (`hwId`, config schema changes between releases).

---

# Part IV – Worked example: Accton ECS5550-30X

## 11. The situation

| Item | Value |
|---|---|
| Board | Accton ECS5550-30X, 24x10G + 6x100G |
| ONIE platform string | `arm64-accton_ecs5550_30x-r0` |
| CPU | Marvell CN9130 (ap807), arm64, 4 cores, 4 GB RAM |
| ASIC | Marvell Aldrin2 (`hwId=ALDRIN2`) |
| Vendor image on the box | SONiC `Lite-1.10.0-rc1_202405_arm64_ols` (202405, Debian 12, kernel 6.1.0-22-2), syncd/swss Up |
| Vendor SAI | `mrvllibsai 1.15.1-1` (SAI 1.15.1, Release P5.0.0) |
| Target tree | `marvell-sonic-buildimage` `rls-202511.01`: Debian 13, kernel 6.12.41, Marvell SAI 1.17.1-21, sairedis headers SAI 1.17.5 |

This is the **easy case** from §4: a working vendor SONiC image was on the box,
so most inputs could be copied from it instead of requested.

Status (2026-11-13): device folder, FIT entry and installer case are written
and statically verified. **Not yet built, not yet booted.**

## 12. Mapping Part II to what was done

| Part II step | ECS5550-30X |
|---|---|
| §4 Decide | Vendor SAI exists but is 1.15.1, older than the tree's 1.17.x (open risk). arm64 + U-Boot. ONIE present. Reference image present → extract. |
| §5 Device folder | Copied from the vendor image; 2 files changed (§13.4) |
| §6 Platform package | **Skipped.** Vendor modules are built for kernel 6.1, and there's no source → no fan/PSU/SFP/LED yet |
| §7 Kernel | Not changed; `.config` for this SoC not yet checked |
| §7 Boot | Vendor dtb + new FIT entry + new installer case (§13.4) |
| §7 SAI | Tree's own Marvell SAI 1.17.1-21 used as-is. Whether it supports Aldrin2 is unknown |
| §7 Build | `rules/config.user` `SONIC_CONFIG_BUILD_JOBS = 8`; configure under QEMU (slow) |
| §8 Bring-up | Pending |

## 13. Steps in detail

### 13.1 Identify the box (don't assume)
On the running vendor image:
```
cat /host/machine.conf          # onie_platform=arm64-accton_ecs5550_30x-r0
cat /etc/sonic/sonic_version.yml
docker ps -a                    # syncd/swss Up → the vendor SAI works on this box
ls /usr/share/sonic/device
```
Lesson: the first SAI extract (notes in `ecs5550-54x-202405/`) came from an **ECS5550-54X**
(48x10G). The target is a **30X**. They use different HWSKU XMLs and port maps.

### 13.2 Extract SAI and board data (read-only)
Output: a local work directory (`~/sai-extract-30x/`, since removed; the files that are still needed are in this repo, see the appendix). Method: [SAI_EXTRACT_GUIDE.md](SAI_EXTRACT_GUIDE.md).
- The SAI lives in the syncd **container image**, not on the host. Run
  `docker run --rm --network none --entrypoint sh <syncd image>`, then re-pack
  with `dpkg -L` + `dpkg -s` + `dpkg-deb --root-owner-group -b`.
- Result: `mrvllibsai_1.15.1-1_arm64.deb`; the sha256 of `libsai.so` matches the container.
- Pitfall: when rebuilding `DEBIAN/control` from `dpkg -s`, remove the whole
  `Conffiles:` block, including its indented lines. Otherwise `dpkg-deb` fails.
- Also saved: the device folder (tgz), `mvcpss.ko`, platform package file
  lists, the `sonic_platform` Python sources, and version information.

### 13.3 Extract boot information
```
cat /etc/fw_env.config; cat /proc/mtd
ls -l /sys/block/                     # eMMC = mmc0:0001 → /dev/mmcblk0
sudo fw_printenv                      # contains passwords/keys: don't publish
ls /host/image-*/boot/                # vendor dtbs, sonic_fit.its, sonic_arm64.fit
```

| Item | Value |
|---|---|
| Boot storage | eMMC `mmc0:0001`, SONiC on partition 3 |
| U-Boot env | `/dev/mtd1 0x0 0x10000 0x10000` |
| `fit_addr` | `0x10000000` |
| kernel load/entry | `0x5480000` |
| dtb load | `0x5400000` |
| `initrd_addr` | `0xa00000` (ramdisk used in place, uncompressed) |
| FIT config | `conf_ecs5550-30x` |
| dtb | `ecs5550-30x.dtb` (compatible `marvell,cn9130-db-A`, plus Accton CPLD nodes) |

### 13.4 Files added/changed in `~/marvell-sonic-buildimage`

1. `device/accton/arm64-accton_ecs5550_30x-r0/` (new)
   - `Accton-ECS5550-30X/`: copied unchanged (sai.profile, profile.ini,
     port_config.ini, SAI/ASK XMLs + md5). The `.md5` files use Marvell's own
     scheme, not a plain `md5sum`, so don't regenerate them.
   - `default_sku` (`Accton-ECS5550-30X l2`), `installer.conf`,
     `system_health_monitoring_config.json`, `plugins/`,
     `sonic_platform-1.0-py3-none-any.whl`: copied unchanged.
   - `platform_asic`: **changed** `marvell` → `marvell-prestera` (= this
     tree's `TARGET_MACHINE`).
   - `pmon_daemon_control.json`: **changed**. Turns off ledd, xcvrd, psud,
     thermalctld and pcied, whose drivers can't load on kernel 6.12.
   - `ecs5550-30x.dtb`: vendor dtb. It ends up at `/usr/share/sonic/device/...`
     in the rootfs before `mkimage` runs.
2. `platform/marvell-prestera/sonic_fit.its`: added `kernel_aldrin2`,
   `ramdisk_aldrin2`, `fdt_ecs5550-30x`, `conf_ecs5550-30x`.
3. `platform/marvell-prestera/platform_arm64.conf`: added an
   `arm64-accton_ecs5550_30x-r0)` case (eMMC bus, FIT conf name) and an
   address block (`fdt_addr`, `fit_addr`, `initrd_addr`, `FW_ENV_DEFAULT`).
4. `rules/config.user` (git-ignored): `SONIC_CONFIG_BUILD_JOBS = 8`. This tree defaults to 1.

The FIT config name matches the vendor's (`conf_ecs5550-30x`) on purpose. The
installer's fallback `sonic_image_2` boots the old image with the *current*
`fit_conf_name`, so the vendor image stays bootable.

### 13.5 Verification done (static only)
- `sh -n` / `dash -n` on `platform_arm64.conf` (ONIE runs it with sh).
- Ran the case block against the 30X's real `machine.conf`. It picks the expected values.
- `mkimage` 2025.01 built a FIT from the modified `.its` (dummy kernel/initrd);
  `dumpimage` shows `conf_ecs5550-30x` → kernel 0x05480000, dtb 0x05400000.
- `dtc` decodes the vendor dtb.
- Simulated `build_image.sh` device list: the 30X is included.

### 13.6 Next steps
```
# after configure finishes
make target/sonic-marvell-prestera-arm64.bin
# on the switch: install, keeping the vendor image as fallback
sudo sonic-installer install <image>.bin
```
Keep the serial console attached (115200 8N1). If the new image doesn't boot,
type `run sonic_image_2` at the U-Boot prompt to boot the vendor image. Then follow §8.

## 14. Problems met and lessons learned

| Problem | Cause | Lesson |
|---|---|---|
| Extracted SAI/XML didn't fit the target | Pulled from a 54X, target is a 30X | Check `machine.conf` before extracting |
| SSH login failed | Password case differed | Confirm credentials; stop after 2 failures to avoid lockout |
| `dpkg-deb` error on syncd re-pack | `Conffiles:` continuation lines left in control | Strip the whole block |
| Board not offered by installer | `platform_asic` ≠ `TARGET_MACHINE` | Use the tree's ASIC name, not the vendor's |
| `.md5` ≠ `md5sum` of XML | Marvell-specific checksum | Copy as-is |
| Vendor kernel modules unusable | Built for 6.1.0-22-2; tree uses 6.12.41 (vermagic) | Ask for driver **source** |
| SAI version gap | Vendor 1.15.1 vs tree headers 1.17.5 | Match the branch to the SAI, or get a newer SAI |
| No boot config for the board | Installer `case` had no entry → wrong dtb/addresses | Add the board to `platform_<arch>.conf` and `sonic_fit.its` |
| `make configure` very slow | arm64 slave image built under QEMU | arm64 build host, or `CROSS_BLDENV=1` (untested here) |
| Build runs one package at a time | `SONIC_CONFIG_BUILD_JOBS = 1` in this tree | Override in `rules/config.user` |
| Local fixes not on other servers | `Makefile.work` / Dockerfile patches and `config.user` are uncommitted / git-ignored | Commit to a branch or export a patch |

Open risks for this bring-up:
- SAI: the tree's Marvell SAI (1.17.1-21) may not support `hwId=ALDRIN2`, and
  the 1.15 XMLs may not match it. syncd may fail even if the OS boots.
- Kernel: the 6.12 `.config` was not checked for eMMC (Xenon), mvpp2 and the
  i2c drivers this board needs.
- The dtb was made for kernel 6.1. The Accton CPLD nodes have no driver on 6.12.
- No fan/PSU/SFP/LED support until the platform drivers are rebuilt from source (§6).

---

# Appendix – Files referenced

| Path | Content |
|---|---|
| `ecs5550-save: docs/ecs5550-54x-202405/` | Notes from the first extract (ECS5550-54X, SAI 1.14.0-1): REPORT, COMPAT, BRANCH, BUILD |
| `ecs5550-save: docs/SAI_EXTRACT_GUIDE.md` | How the SAI and board files were pulled from the vendor image |
| `ecs5550-save: device/accton/arm64-accton_ecs5550_30x-r0/` | Vendor `ecs5550-30x.dtb`, `sonic_platform` whl, SAI/ASK XMLs from the 30X |
| local `~/sai-extract-30x/` (removed) | Raw 30X extract: SAI 1.15.1-1 debs, mvcpss.ko, package lists, vendor `sonic_fit.its`, `ecs5550-54x.dtb` |
| `~/marvell-sonic-buildimage/device/accton/arm64-accton_ecs5550_30x-r0/` | New device folder |
| `~/marvell-sonic-buildimage/platform/marvell-prestera/sonic_fit.its` | FIT description (edited) |
| `~/marvell-sonic-buildimage/platform/marvell-prestera/platform_arm64.conf` | Installer boot config (edited) |
| `~/marvell-sonic-buildimage/rules/config.user` | Local build parallelism (git-ignored) |
