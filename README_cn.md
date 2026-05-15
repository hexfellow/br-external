# Build Root for Hexfellow Chassis Control Board

[English README](./README.md) | [中文 README](./README_cn.md)

## 构建流程

1. 在你开始构建之前，记得先修改 `overlay/usr/lib/systemd/system/ssh-key.service` 添加你自己的 SSH 公钥，这样你就可以更容易地访问板子。当然也可以不改，然后使用 usb 串口登录系统后面再加进来

```bash
git clone https://github.com/buildroot/buildroot -b 2025.02.1 buildroot
git clone <this repo> br-external
cd buildroot
make BR2_EXTERNAL=../br-external hexfellow_geek_ctrl_defconfig
make
```


---

## 临时测试软件

开发期间需要经常改变自己的软件包，每次都用 buildroot 显然不太方便。我们可以在电脑上完成交叉编译然后 scp 到板子上进行测试。这里用一个新建的 cargo 项目为例。

首先，你需要已经完成 [构建流程](#构建流程)，并且准备好交叉编译工具链。

```bash
cargo new hello-world
cd hello-world
echo 'fn main() {
    println!("Hello, world!");
}' > src/main.rs
export BR=/home/kisonhe/buildroot/ # 这里假设你已经完成了构建流程，需要是绝对路径。不能是相对路径
env RUSTFLAGS='-C target-cpu=cortex-a53' CARGO_TARGET_AARCH64_UNKNOWN_LINUX_GNU_LINKER=$BR/output/host/bin/aarch64-linux-gcc CARGO_BUILD_TARGET=aarch64-unknown-linux-gnu PKG_CONFIG_SYSROOT_DIR=$BR/output/staging PKG_CONFIG=$BR/output/host/usr/bin/pkg-config cargo build --release # A53 to support RK3576 also
scp ./target/aarch64-unknown-linux-gnu/release/hello-world root@172.18.25.50: # 改成你自己的板子 IP
```

然后，你就可以 ssh 进入系统，运行 `./hello-world` 看到输出 `Hello, world!` 了。

默认只读挂载了 `/`, 如果需要把东西放到 `/userdata` 和 `/root` 之外的地方，可以运行 `mount -o remount,rw /`。文件写完之后一定要运行 `sync` 同步一下，不然断电后数据可能丢失。

---

## 永久添加软件包（二进制）

你可以将刚刚编译好的 `hello-world` 放到 `overlay/usr/bin` 中。并向 `overlay/usr/lib/systemd/system` 中添加一个服务文件，这样系统启动时就会自动运行 `hello-world`。

---

## 永久添加软件包（源码形式）

本节以 `hexfellow-hello-world` 包为例，讲解如何向这个 Buildroot external tree 添加新的软件包。此节并非必要，如果你只是想临时测试软件，可以看上一节。

### 背景知识

Buildroot 使用 **br2-external** 机制来支持在主代码树之外维护自定义的包、板级配置和 overlay。我们这个仓库就是一个 br2-external tree，其核心结构如下：

```
br-external/
├── Config.in          # 顶层 Kconfig 菜单，注册所有自定义包
├── external.mk        # 自动 include 所有 package/*/*.mk
├── external.desc      # 声明 external tree 的名称 (HEX_EMBEDDED)
├── package/           # 所有自定义包都放在这里
│   ├── hexfellow-hello-world/
│   │   ├── Config.in
│   │   └── hexfellow-hello-world.mk
│   ├── rtl8821cs-wifi/
│   │   ├── Config.in
│   │   └── rtl8821cs-wifi.mk
│   └── ...
├── board/             # 板级文件 (DTS、boot 脚本、post-build 等)
├── configs/           # defconfig 文件
└── overlay/           # rootfs overlay (会直接覆盖到目标文件系统)
```

`external.mk` 中的这行代码会自动加载 `package/` 下所有子目录中的 `.mk` 文件：

```makefile
include $(sort $(wildcard $(BR2_EXTERNAL_HEX_EMBEDDED_PATH)/package/*/*.mk))
```

所以你只需要关注两件事：**创建包目录** 和 **在顶层 Config.in 中注册**。

### 第一步：创建包目录

在 `package/` 下新建一个目录，目录名即为包名。目录中至少需要两个文件：

```
package/<包名>/
├── Config.in       # Kconfig 配置项
└── <包名>.mk       # Buildroot 构建规则
```

### 第二步：编写 Config.in

`Config.in` 定义了这个包在 `make menuconfig` 中的选项。最简形式如下：

```kconfig
config BR2_PACKAGE_HEXFELLOW_HELLO_WORLD
	bool "hexfellow-hello-world"
	help
	  Simple hello world program in C, to demonstrate
	  how to add packages to buildroot.
```

命名规则：
- 配置项名称必须是 `BR2_PACKAGE_` + 包名大写、连字符换下划线
- 例如包名 `hexfellow-hello-world` → 配置项 `BR2_PACKAGE_HEXFELLOW_HELLO_WORLD`

如果你的包依赖其他包，可以用 `select` 或 `depends on`：

```kconfig
config BR2_PACKAGE_MY_APP
	bool "my-app"
	depends on BR2_PACKAGE_LIBCURL
	select BR2_PACKAGE_ZLIB
```

### 第三步：编写 .mk 文件

`.mk` 文件是包的核心，定义了如何下载、构建和安装。下面以 `hexfellow-hello-world` 为例：

```makefile
# 从 GitHub 通过 git 下载源码
HEXFELLOW_HELLO_WORLD_VERSION = 81cc11a
HEXFELLOW_HELLO_WORLD_SITE = https://github.com/hexfellow/hexfellow-hello-world.git
HEXFELLOW_HELLO_WORLD_SITE_METHOD = git

# CMakeLists.txt 在 c/ 子目录，用 _SUBDIR 指定
HEXFELLOW_HELLO_WORLD_SUBDIR = c

# 使用 cmake-package 构建框架，自动处理 configure / build / install
$(eval $(cmake-package))
```

使用 `cmake-package` 框架后，Buildroot 会自动：
1. 用交叉编译工具链的 toolchain file 调用 `cmake` 进行配置
2. 调用 `make` 构建
3. 调用 `make install DESTDIR=$(TARGET_DIR)` 安装

因此不需要手动编写 `_BUILD_CMDS` 和 `_INSTALL_TARGET_CMDS`，只要上游项目的 `CMakeLists.txt` 中有正确的 `install()` 指令即可。

> **注意**：如果源码的 `CMakeLists.txt` 不在仓库根目录，需要用 `_SUBDIR` 变量指定子目录路径。

**变量命名规则**：所有变量前缀为包名大写、连字符换下划线，例如 `hexfellow-hello-world` → `HEXFELLOW_HELLO_WORLD_`。

### 第四步：在顶层 Config.in 中注册

编辑 `Config.in`，添加一行 `source` 指令：

```kconfig
menu "Packages"
source "$BR2_EXTERNAL_HEX_EMBEDDED_PATH/package/rtl8821cs-wifi/Config.in"
source "$BR2_EXTERNAL_HEX_EMBEDDED_PATH/package/rtl8821cs-bluetooth/Config.in"
source "$BR2_EXTERNAL_HEX_EMBEDDED_PATH/package/rk-uboot-tools/Config.in"
source "$BR2_EXTERNAL_HEX_EMBEDDED_PATH/package/hexfellow-hello-world/Config.in"
endmenu
```

### 第五步：启用包并构建

```bash
cd buildroot

# 打开菜单配置，在 External options → Packages 下找到并勾选你的包
make menuconfig

# 单独构建这个包（调试用）
make hexfellow-hello-world

# 或者完整构建
make
```

如果需要重新构建某个包：

```bash
make hexfellow-hello-world-dirclean   # 清除构建目录
make hexfellow-hello-world-rebuild    # 重新构建
```

### 完整流程小结

添加一个新包只需 3 步：

1. **创建** `package/<包名>/Config.in` 和 `package/<包名>/<包名>.mk`
2. **注册** 在顶层 `Config.in` 中 source 你的 `Config.in`
3. **启用** 通过 `make menuconfig` 勾选，然后 `make`

`external.mk` 会自动发现新的 `.mk` 文件，不需要额外修改。

## Todo
- 增加如何添加自定义 Rust 包的教程