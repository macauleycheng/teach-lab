# SAI deb for SONiC: how to build the ECS5550 one, and how to make one

This guide has two parts:

- **Part A: Build the ECS5550 SAI deb.** Step-by-step commands for
  `mrvllibsai_1.17.5-1_arm64.deb`, the SAI adapter for the Accton/Edgecore
  ECS5550-30X/54X (Marvell Aldrin2, CPSS 4.3.12) in `marvell-sonic-buildimage`.
- **Part B: How to make a SAI deb.** What any `libsai` package must satisfy so
  SONiC can build syncd against it and run it, and how to hook it into
  sonic-buildimage. Every rule here comes from a problem we actually hit; the
  "Why" column says which.

Read section 1 first. It explains how SONiC uses the SAI deb, which both parts
rely on.

---

## 1. How SONiC uses the SAI deb

```
platform/<asic>/sai.mk         defines the package: MRVL_SAI = mrvllibsai_<ver>_<arch>.deb
        │                      and where it comes from (download / copy / build)
        ▼
target/debs/<distro>/mrvllibsai_<ver>_<arch>.deb
        │
        ├─► syncd package build        rules.mk: $(SYNCD)_DEPENDS += $(MRVL_SAI)
        │     syncd links -lsai          the deb is installed in the build container,
        │     (src/sonic-sairedis/syncd)  syncd is linked against /usr/lib/libsai.so,
        │                                 dpkg-shlibdeps records the dependency
        │
        ├─► libsaithrift / saiserver   (ENABLE_SYNCD_RPC=y) also links libsai
        │
        └─► docker-syncd-<asic>.gz     docker-syncd-mrvl-prestera.mk:
              the deb is installed      $(DOCKER_SYNCD_BASE)_DEPENDS += $(MRVL_SAI)
              into the syncd container
                         │
                         ▼  on the switch
              syncd → dlopen/ld.so loads /usr/lib/libsai.so
                    → sai_api_initialize(), sai_api_query(), create_switch(...)
                    → vendor SDK (CPSS) → kernel module (mvcpss.ko) → ASIC
```

Consequences:

- The deb is used **twice**: at build time to link syncd and saiserver, and
  at run time inside the syncd container. It has to work in both places.
- The syncd container's Debian release decides which glibc `libsai.so` runs
  against. In this tree the syncd container is **bookworm (glibc 2.36)**, even
  though the host image is trixie.
- syncd only uses the SAI C API (`sai_*` functions and the SAI headers). The
  header version the deb was built against must match the tree's
  `src/sonic-sairedis/SAI` (here SAI **1.17.5**).

---

# Part A: Build the ECS5550 SAI deb

## 2. What you need

| Item | Where |
|---|---|
| Build tree | `marvell-sonic-buildimage`, branch `ecs5550-30x` (git88) |
| Adapter source | `platform/marvell-prestera/sai-ecs5550/` (in the build tree) |
| SAI headers | `src/sonic-sairedis/SAI` (submodule, SAI 1.17.5) |
| CPSS bundle | private repo `git88.accton.com/macauley_cheng/ecs5550-save`, file `platform/marvell-prestera/sai-ecs5550/cpss-bundle.tar.gz` |
| Toolchain | Debian **bookworm** with `gcc-aarch64-linux-gnu` (a throwaway docker container is enough) |

The CPSS bundle is Marvell proprietary. It must **not** be committed to the build
tree. Add these lines to the build tree's `.git/info/exclude`:

```
/platform/marvell-prestera/sai-ecs5550/sdk
/platform/marvell-prestera/sai-ecs5550/out
/platform/marvell-prestera/sai-ecs5550/cpss-bundle.tar.gz
/platform/marvell-prestera/sai-ecs5550/extra_objs.txt
/platform/marvell-prestera/sai-ecs5550/build
/platform/marvell-prestera/sai-ecs5550/deb
/platform/marvell-prestera/sai-ecs5550/doc/sai_cpss_mapping.md
```

## 3. (Only when CPSS changes) Build the CPSS bundle on the EC_VOB machine

The bundle contains the CPSS headers, the CPSS archives (`*.a`) and a list of
extra objects. It is made on the machine with the EC_VOB tree.

1. Build the SDK **with `-fPIC`** and a clean build folder, so every object is
   really recompiled:

   ```bash
   make CPSS_USER_CFLAGS=-fPIC TARGET=armv8 \
        TARGET_SPECIFIC_CFLAGS=-DECS5550_AL2 \
        TARGET_SPECIFIC_CFLAGS+=-DSYS_CPNT_TX_RX_USE_TWO_DMA_BLOCK \
        TARGET_SPECIFIC_CFLAGS+=-DSYS_CPNT_EXTRA_RX_COPY_DMA_BLOCK \
        FAMILY=DX INCLUDE_MPD=Y MPD_INCLUDE_SLAVE_IMAGE=Y INCLUDE_TM=y INCLUDE_GALTIS=y \
        CONFIG_DMA_LEN=12 -s appDemo BUILD_FOLDER=$CPSS_OUTPUT
   ```

2. **Check that the objects are PIC before packing.** This one object is enough:

   ```bash
   ar x $CPSS_OUTPUT/cpssEnabler/libs/CPSSENABLER.a osGlobalDb.o
   readelf -rW osGlobalDb.o | grep -c R_AARCH64_ADR_GOT_PAGE    # must be > 0
   ```

   Use `readelf -rW`. Without `-W`, readelf cuts the relocation names short
   (`R_AARCH64_ADR_GOT`) and a grep for the full name finds nothing.

3. Pack the bundle:

   ```bash
   cd platform/marvell-prestera/sai-ecs5550
   make CPSS_SDK=<EC_VOB>/src/kernel/driver/dev_driver/marvell/cpss_4.3.12 \
        CPSS_OUTPUT=$CPSS_OUTPUT \
        DRIVER_PROC_MK=<EC_VOB>/project_build_environment/ecs5550/user/process/driver_proc/Makefile.am \
        bundle                         # -> build/cpss-bundle.tar.gz
   ```

   `make bundle` only **packs** existing `.a` files. It does not recompile. If the
   SDK was built without `-fPIC`, a new bundle is still non-PIC. That happened
   once: the new tarball had a different size, but every `.a` inside was
   byte-identical to the old one.

4. Commit the new `cpss-bundle.tar.gz` to `ecs5550-save`.

## 4. Build the deb

From the build tree root:

```bash
# 1. Get the CPSS bundle (once, or when it changes)
git clone https://git88.accton.com/macauley_cheng/ecs5550-save.git /tmp/ecs5550-save
cp -a /tmp/ecs5550-save/platform/. platform/

# 2. Unpack it next to the adapter
cd platform/marvell-prestera/sai-ecs5550
tar -xzf cpss-bundle.tar.gz           # -> sdk/  out/  extra_objs.txt
cd -

# 3. Build in a throwaway bookworm container (matches the syncd container's glibc)
R=$PWD; S=$R/platform/marvell-prestera/sai-ecs5550
docker run --rm -v $S:/sai -v $R/src/sonic-sairedis/SAI:/saihdr:ro \
    publicmirror.azurecr.io/debian:bookworm bash -c "
  apt-get -qq update && apt-get -qq install -y make gcc gcc-aarch64-linux-gnu \
      libc6-dev-arm64-cross dpkg-dev binutils-aarch64-linux-gnu >/dev/null
  cd /sai
  make SAI_ROOT=/saihdr syntax                    # SAI part only, quick
  make SAI_ROOT=/saihdr BUILD_DIR=/tmp/t test     # host test: '181 checks, 0 failures'
  make SAI_ROOT=/saihdr CROSS_COMPILE=aarch64-linux-gnu- \
       CPSS_SDK=/sai/sdk CPSS_OUTPUT=/sai/out CPSS_EXTRA_OBJS= deb
  chown -R $(id -u):$(id -g) /sai/build /sai/deb"
```

Result: `platform/marvell-prestera/sai-ecs5550/deb/mrvllibsai_1.17.5-1_arm64.deb`.

`CPSS_EXTRA_OBJS=` (empty) is required. All 122 objects in `extra_objs.txt` are
byte-identical copies of members of the CPSS archives. Linking both gives 88
"multiple definition" errors.

## 5. Check the deb

```bash
D=platform/marvell-prestera/sai-ecs5550/deb/mrvllibsai_1.17.5-1_arm64.deb
dpkg-deb -I $D                      # Package: mrvllibsai, Version: 1.17.5-1, Architecture: arm64
dpkg-deb --ctrl-tarfile $D | tar -xO ./shlibs   # libsai 1 mrvllibsai (>= 1.17.5-1)
dpkg-deb -c $D                      # /usr/lib/libsai.so.1.17.5, libsai.so.1 -> , libsai.so ->
T=$(mktemp -d); dpkg-deb -x $D $T; L=$T/usr/lib/libsai.so.1.17.5
aarch64-linux-gnu-readelf -d $L | grep -E 'SONAME|NEEDED'          # libsai.so.1; libc, libm only
aarch64-linux-gnu-objdump -T $L | grep -oE 'GLIBC_[0-9.]+' | sort -uV | tail -1   # <= GLIBC_2.36
aarch64-linux-gnu-nm -D --defined-only $L | awk '{print $3}'       # only sai_* functions
rm -rf $T
```

Reference values for the working build (2026-11-17): deb 12,947,272 bytes;
`libsai.so.1.17.5` 46,814,968 bytes; NEEDED `libc.so.6` and `libm.so.6`; highest
glibc `GLIBC_2.34`; 13 exported `sai_*` symbols.

## 6. Put it into the SONiC image

`ECS5550_SAI ?= y` in `platform/marvell-prestera/sai.mk` (the default) makes the
build copy the deb from `sai-ecs5550/deb/`.

```bash
sudo -v
make target/sonic-marvell-prestera-arm64.bin 2>&1 | tee build-ecs5550.log
```

Two things stop the new deb from being picked up:

1. **A copy already in `target/debs/`.** The copy step (`SONIC_COPY_DEBS`)
   doesn't run again while `target/debs/bookworm/mrvllibsai_1.17.5-1_arm64.deb`
   exists. After rebuilding the deb, replace that copy:
   ```bash
   cp -p platform/marvell-prestera/sai-ecs5550/deb/mrvllibsai_1.17.5-1_arm64.deb target/debs/bookworm/
   ```
2. **An existing image file.** The top-level `Makefile` reports
   `'target/sonic-marvell-prestera-arm64.bin' is up to date` whenever that file
   exists. Move it aside, along with its `__rfs.squashfs`, to force a rebuild.

On the switch, after installing the image:

```bash
docker exec syncd dpkg -l mrvllibsai          # 1.17.5-1
docker logs syncd | tail
sudo grep -E 'syncd#|orchagent' /var/log/syslog | grep -iE 'err|create_switch'
show interfaces status
```

To go back to Marvell's prebuilt SAI: `make ECS5550_SAI=n …`.

## 7. Troubleshooting (errors we hit)

| Error | Cause | Fix |
|---|---|---|
| `relocation R_AARCH64_ADR_PREL_PG_HI21 against symbol 'stderr@@GLIBC_2.17' … can not be used when making a shared object; recompile with -fPIC` / `final link failed: bad value` | CPSS archives built without `-fPIC` | Rebuild the SDK with `CPSS_USER_CFLAGS=-fPIC` and a clean build folder, then repack (§3) |
| 88 × `multiple definition of 'mvHwsComphyWait'` … `first defined here` | `extra_objs.txt` objects are also archive members | `CPSS_EXTRA_OBJS=` (empty) |
| `dpkg-shlibdeps: error: no dependency information found for /usr/lib/libsai.so.1 (used by …/saiserver)` | deb had no `DEBIAN/shlibs` | Makefile `deb` target writes `libsai 1 mrvllibsai (>= <ver>)` to `DEBIAN/shlibs`; rebuild deb and replace the copy in `target/debs/` |
| `make: 'target/…bin' is up to date` | top-level Makefile rule has no prerequisites | Move the old `.bin` (and `__rfs.squashfs`) aside |
| image build: `cp …/sai-ecs5550/deb/mrvllibsai_… : No such file` | deb not built, `ECS5550_SAI=y` | Build the deb (§4), or `ECS5550_SAI=n` |
| syncd: `Cant use parameters from file [...SAI-*.xml], retVal : 23` | (Marvell prebuilt SAI 1.17.1) vendor XMLs were made for SAI 1.15 | Not an issue with the adapter; it doesn't use those XMLs. With `ECS5550_SAI=n` you need XMLs generated for that SAI release |

---

# Part B: How to make a SAI deb

## 8. Requirements for the library (`libsai.so`)

| # | Requirement | Check | Why |
|---|---|---|---|
| 1 | Built for the target CPU (`aarch64`, `x86_64`, …) | `file libsai.so` | syncd for arm64 can't load an x86 library |
| 2 | Built against the **same SAI headers** as the tree (`src/sonic-sairedis/SAI/inc/saiversion.h`) | compare `SAI_MAJOR/MINOR/REVISION`; syncd logs `SAI API vendor version` and `headers version` | Structs, enums and attribute IDs change between SAI releases. A Marvell 1.15 library or its configs fail on 1.17 |
| 3 | Links against a glibc **no newer** than the syncd container's (bookworm: 2.36) | `objdump -T libsai.so \| grep -o 'GLIBC_[0-9.]*' \| sort -uV \| tail -1` | A library built on trixie (glibc 2.41) may not load in a bookworm container. Build it on the same Debian release as the container |
| 4 | Shared object, with **every** object compiled `-fPIC`, including vendor SDK archives | `readelf -rW x.o \| grep ADR_GOT_PAGE` (aarch64) | Non-PIC objects can't be linked into a `.so` |
| 5 | No unresolved symbols left for run time | link with `-Wl,--no-undefined` | Otherwise syncd fails only when the missing function is first called on the switch |
| 6 | Exports the SAI entry points (`sai_api_initialize`, `sai_api_query`, `sai_api_uninitialize`, `sai_log_set`, `sai_object_type_query`, `sai_switch_id_query`, `sai_query_*`, …) and preferably nothing else | `nm -D --defined-only libsai.so`; use a linker version script | Exporting the whole SDK can clash with symbols in syncd and other libraries |
| 7 | Run-time dependencies are only system libraries, or are shipped in the same deb | `readelf -d libsai.so \| grep NEEDED` | Marvell's package also ships `libXdkCpss.so`. Anything not in the container makes `ld.so` fail when syncd starts |
| 8 | Any firmware, configs or tools the library loads at run time are shipped in the same deb | e.g. Marvell's deb installs `/opt/marvell/firmware/*.fw`, `/fwFiles/phy/*.hdr`, `/usr/bin/mrvlcmd` | Missing firmware only shows up at `create_switch` time |
| 9 | Kernel side (e.g. `mvcpss.ko`) matches the library | build the module from source for the image kernel, in the platform package | A `.ko` only loads on the kernel it was built for (vermagic) |

## 9. Requirements for the package (deb)

| # | Requirement | Example (ECS5550) | Why |
|---|---|---|---|
| 1 | Package name = the one `sai.mk` and `rules.mk` use | `Package: mrvllibsai` | syncd, saiserver and the syncd docker depend on `$(MRVL_SAI)`. Keeping the name means no Dockerfile or rules change |
| 2 | File name `<name>_<version>_<arch>.deb` matching `sai.mk` | `mrvllibsai_1.17.5-1_arm64.deb` | slave.mk builds the target path from these variables |
| 3 | Library in `/usr/lib/` with the `libsai.so` dev link | `/usr/lib/libsai.so.1.17.5`, `libsai.so.1 ->`, `libsai.so ->` | syncd links with `-lsai`, which finds `libsai.so` |
| 4 | **If the library has a SONAME, ship `DEBIAN/shlibs`** (or `symbols`) | `libsai 1 mrvllibsai (>= 1.17.5-1)` | `dpkg-shlibdeps` (for syncd, saiserver) must know which package provides `libsai.so.1`. Without it the saiserver build fails. Marvell's prebuilt `libsai.so` has no SONAME, which is why it never needed one |
| 5 | `postinst` runs `ldconfig` | `#!/bin/sh` / `ldconfig` | Refreshes the loader cache for the new SONAME links |
| 6 | Files owned by root | `dpkg-deb --build --root-owner-group …` | Re-packed vendor debs otherwise keep the vendor's build uid/gid |
| 7 | `Architecture:` matches the library | `arm64` | apt/dpkg refuse wrong-arch packages |
| 8 | `Depends:` lists real run-time package dependencies | `libc6` | Keeps the container consistent |

Minimal packaging, the same steps as the adapter's `make deb`:

```bash
P=build/pkg; V=1.17.5; DV=1.17.5-1
rm -rf $P && mkdir -p $P/DEBIAN $P/usr/lib
install -m 0755 build/libsai.so.$V $P/usr/lib/
ln -sf libsai.so.$V $P/usr/lib/libsai.so.1
ln -sf libsai.so.1  $P/usr/lib/libsai.so
cat > $P/DEBIAN/control <<EOF
Package: mrvllibsai
Version: $DV
Architecture: arm64
Maintainer: <you>
Section: libs
Priority: optional
Provides: libsai
Depends: libc6
Description: SAI $V for <board>
EOF
printf '#!/bin/sh\nldconfig\n' > $P/DEBIAN/postinst && chmod 0755 $P/DEBIAN/postinst
printf 'libsai 1 mrvllibsai (>= %s)\n' $DV > $P/DEBIAN/shlibs
dpkg-deb --build --root-owner-group $P deb/mrvllibsai_${DV}_arm64.deb
```

Link the library itself with a SONAME and a version script:

```bash
aarch64-linux-gnu-gcc -shared -Wl,-soname,libsai.so.1 -Wl,--version-script=libsai.map \
    -Wl,--no-undefined -o build/libsai.so.1.17.5 <sai objects> \
    -Wl,--start-group <vendor SDK .a files> -Wl,--end-group -lpthread -lrt -lm -ldl
```

`libsai.map`:

```
{
  global: sai_*;
  local:  *;
};
```

## 10. Hooking the deb into sonic-buildimage

`platform/<asic>/sai.mk` defines the package and where it comes from. Pick one
of three source types:

| Type | When | sai.mk |
|---|---|---|
| `SONIC_ONLINE_DEBS` | Vendor publishes the deb on a URL (Marvell's default) | `$(MRVL_SAI)_URL = https://…/$(MRVL_SAI)` |
| `SONIC_COPY_DEBS` | Deb built outside the SONiC container and copied in (ECS5550) | `$(MRVL_SAI)_PATH = $(PLATFORM_PATH)/sai-ecs5550/deb` |
| `SONIC_DPKG_DEBS` / `SONIC_MAKE_DEBS` | Built inside the SONiC build from source with `debian/` or a Makefile | `$(MRVL_SAI)_SRC_PATH = …` |

The ECS5550 `sai.mk`, with a switch to keep the original:

```make
ECS5550_SAI ?= y

ifeq ($(ECS5550_SAI),y)
MRVL_SAI_VERSION = 1.17.5-1
ECS5550_SAI_DEB_PATH ?= $(PLATFORM_PATH)/sai-ecs5550/deb
MRVL_SAI = mrvllibsai_$(MRVL_SAI_VERSION)_$(PLATFORM_ARCH).deb
$(MRVL_SAI)_PATH = $(ECS5550_SAI_DEB_PATH)
SONIC_COPY_DEBS += $(MRVL_SAI)
else
# … original SONIC_ONLINE_DEBS definition …
endif
$(MRVL_SAI)_SKIP_VERSION=y
$(eval $(call add_conflict_package,$(MRVL_SAI),$(LIBSAIVS_DEV)))
```

- `_SKIP_VERSION=y` keeps the version-pinning files (`files/build/versions/…`)
  from forcing another version.
- `add_conflict_package` prevents installing the real SAI and the virtual-switch
  SAI (`libsaivs-dev`) at the same time.
- `rules.mk` and `docker-syncd-*.mk` only reference `$(MRVL_SAI)`. If the
  package name stays the same, nothing else needs to change.

## 11. Alternative: re-package a SAI from a working vendor image

If the board already runs a vendor SONiC image, its SAI deb can be re-packed from
the syncd container image. The procedure is in [SAI_EXTRACT_GUIDE.md](SAI_EXTRACT_GUIDE.md)
(`dpkg -L` + `dpkg -s` + `dpkg-deb --root-owner-group -b`). Requirements §8 and
§9 still apply. In particular, a re-packed vendor SAI only works with the same
SAI header version and the same kernel module it came with. That's why
`mrvllibsai 1.15.1-1` from the ECS5550 vendor image (SAI 1.15) can't simply be
dropped into this SAI 1.17.5 tree.

## 12. Files referenced

| Path | Content |
|---|---|
| `marvell-sonic-buildimage/platform/marvell-prestera/sai.mk` | SAI package definition, `ECS5550_SAI` switch |
| `marvell-sonic-buildimage/platform/marvell-prestera/sai-ecs5550/` | Adapter source, Makefile (`syntax`, `test`, `deb`, `bundle`) |
| `marvell-sonic-buildimage/platform/marvell-prestera/rules.mk` | syncd / libsaithrift depend on `$(MRVL_SAI)` |
| `marvell-sonic-buildimage/platform/marvell-prestera/docker-syncd-mrvl-prestera.mk` | syncd docker installs `$(MRVL_SAI)` |
| `git88: macauley_cheng/ecs5550-save` | CPSS bundle, SAI-to-CPSS mapping document, vendor board files |
| `ecs5550-save: docs/SAI_EXTRACT_GUIDE.md` | Re-packing a SAI deb from a vendor image |
| `ecs5550-save: docs/NEW_PLATFORM_GUIDE.md` | Adding the ECS5550-30X platform |
