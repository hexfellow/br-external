# Build Root for Hexfellow Chassis Control Board

[English README](./README.md) | [中文 README](./README_cn.md)

## Build Process

1. Before you start building, remember to modify `overlay/usr/lib/systemd/system/ssh-key.service` to add your own SSH public key, so you can access the board more easily.

```bash
git clone https://github.com/buildroot/buildroot -b 2025.02.1 buildroot
git clone <this repo> br-external
cd buildroot
make BR2_EXTERNAL=../br-external hexfellow_geek_ctrl_defconfig
make
```

---

## How to Add Custom Packages to br-external

This section uses the `hexfellow-hello-world` package as an example to explain how to add new packages to this Buildroot external tree.

### Background

Buildroot uses the **br2-external** mechanism to support maintaining custom packages, board configurations, and overlays outside of the main source tree. This repository is a br2-external tree with the following core structure:

```
br-external/
├── Config.in          # Top-level Kconfig menu, registers all custom packages
├── external.mk        # Auto-includes all package/*/*.mk
├── external.desc      # Declares the external tree name (HEX_EMBEDDED)
├── package/           # All custom packages go here
│   ├── hexfellow-hello-world/
│   │   ├── Config.in
│   │   └── hexfellow-hello-world.mk
│   ├── rtl8821cs-wifi/
│   │   ├── Config.in
│   │   └── rtl8821cs-wifi.mk
│   └── ...
├── board/             # Board files (DTS, boot scripts, post-build, etc.)
├── configs/           # defconfig files
└── overlay/           # rootfs overlay (directly overlaid onto the target filesystem)
```

The following line in `external.mk` automatically loads all `.mk` files from subdirectories under `package/`:

```makefile
include $(sort $(wildcard $(BR2_EXTERNAL_HEX_EMBEDDED_PATH)/package/*/*.mk))
```

So you only need to worry about two things: **creating the package directory** and **registering it in the top-level Config.in**.

### Step 1: Create the Package Directory

Create a new directory under `package/` with the directory name as the package name. The directory needs at least two files:

```
package/<package-name>/
├── Config.in       # Kconfig configuration entry
└── <package-name>.mk       # Buildroot build rules
```

### Step 2: Write Config.in

`Config.in` defines the options for this package in `make menuconfig`. The simplest form is:

```kconfig
config BR2_PACKAGE_HEXFELLOW_HELLO_WORLD
	bool "hexfellow-hello-world"
	help
	  Simple hello world program in C, to demonstrate
	  how to add packages to buildroot.
```

Naming rules:
- The config option name must be `BR2_PACKAGE_` + package name in uppercase with hyphens replaced by underscores
- Example: package name `hexfellow-hello-world` → config option `BR2_PACKAGE_HEXFELLOW_HELLO_WORLD`

If your package depends on other packages, use `select` or `depends on`:

```kconfig
config BR2_PACKAGE_MY_APP
	bool "my-app"
	depends on BR2_PACKAGE_LIBCURL
	select BR2_PACKAGE_ZLIB
```

### Step 3: Write the .mk File

The `.mk` file is the core of the package, defining how to download, build, and install. Here's the `hexfellow-hello-world` example:

```makefile
# Download source from GitHub via git
HEXFELLOW_HELLO_WORLD_VERSION = 81cc11a
HEXFELLOW_HELLO_WORLD_SITE = https://github.com/hexfellow/hexfellow-hello-world.git
HEXFELLOW_HELLO_WORLD_SITE_METHOD = git

# CMakeLists.txt is in the c/ subdirectory, specify with _SUBDIR
HEXFELLOW_HELLO_WORLD_SUBDIR = c

# Use the cmake-package framework — handles configure / build / install automatically
$(eval $(cmake-package))
```

With the `cmake-package` framework, Buildroot automatically:
1. Runs `cmake` with the cross-compilation toolchain file to configure
2. Runs `make` to build
3. Runs `make install DESTDIR=$(TARGET_DIR)` to install

No need to manually write `_BUILD_CMDS` or `_INSTALL_TARGET_CMDS` — as long as the upstream `CMakeLists.txt` has the correct `install()` directives.

> **Note**: If the `CMakeLists.txt` is not in the repository root, use the `_SUBDIR` variable to specify the subdirectory path.

**Variable naming rules**: All variable prefixes are the package name in uppercase with hyphens replaced by underscores. For example, `hexfellow-hello-world` → `HEXFELLOW_HELLO_WORLD_`.

### Step 4: Register in the Top-Level Config.in

Edit `Config.in` and add a `source` directive:

```kconfig
menu "Packages"
source "$BR2_EXTERNAL_HEX_EMBEDDED_PATH/package/rtl8821cs-wifi/Config.in"
source "$BR2_EXTERNAL_HEX_EMBEDDED_PATH/package/rtl8821cs-bluetooth/Config.in"
source "$BR2_EXTERNAL_HEX_EMBEDDED_PATH/package/rk-uboot-tools/Config.in"
source "$BR2_EXTERNAL_HEX_EMBEDDED_PATH/package/hexfellow-hello-world/Config.in"
endmenu
```

### Step 5: Enable the Package and Build

```bash
cd buildroot

# Open the menu config, find and enable your package under External options → Packages
make menuconfig

# Build just this package (for debugging)
make hexfellow-hello-world

# Or do a full build
make
```

If you need to rebuild a package:

```bash
make hexfellow-hello-world-dirclean   # Clean the build directory
make hexfellow-hello-world-rebuild    # Rebuild
```

### Summary

Adding a new package takes just 3 steps:

1. **Create** `package/<package-name>/Config.in` and `package/<package-name>/<package-name>.mk`
2. **Register** by sourcing your `Config.in` in the top-level `Config.in`
3. **Enable** via `make menuconfig`, then `make`

`external.mk` will automatically discover the new `.mk` file — no additional changes needed.

## Todo
- Add a tutorial on how to add custom Rust packages
