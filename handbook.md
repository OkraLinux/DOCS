# Okra 手册

这份手册按构建盘上的当前代码写。发行版仓库、包管理器仓库和文档仓库是分开的。完整命令表仍以 [OkraPM](okrapm.md) 为准；这里写清各块怎么拼在一起，以及安装器和已安装系统实际会做什么。

## 1. Okra 是什么

Okra 是一套从源码搭起来的 Linux 发行版工作区，不是某一家上游发行版的再打包。现在能跑通的路径是：

1. 用 Linux 7.2 内核、自建的用户态和 systemd 262 做出 Base OS。
2. 用 Limine 把 Base OS 打成可启动的 Live CD。
3. Live CD 上的 OKRAINSTALL 把系统写到整块磁盘，并为这块磁盘装上 GRUB。
4. 磁盘第一次启动跑阶段 2，第二次启动跑阶段 3，然后安装器退出。

包管理器 Lunar / OAA 已经能在宿主机上构建和查询，但发行版还没有自己的软件源。安装器因此不写源、不装包、不做在线更新。

两套引导同时存在，职责不同：

| 介质 | 引导 | 原因 |
|---|---|---|
| Live CD | Limine | ISO 的 EFI 卷就是 Limine 被加载的那一卷，`boot():/boot/vmlinuz` 找得到内核和 initramfs |
| 已安装磁盘 | GRUB | 内核在 ext4 的 `/boot/vmlinuz`。Limine 的 `boot():` 只看见 EFI 卷，看不见 ext4 |

## 2. 仓库和工作区

| 路径 | 仓库或目录 | 作用 |
|---|---|---|
| `okra-linux/` | OkraLinux | Live CD 流水线、OKRAINSTALL、可选的 Kanina |
| `okrapm/` | OkraPM | Lunar、OAA、OPSIS、本地 HTTP 源 |
| `DOCS/` | `git@github.com:OkraLinux/DOCS.git` | 说明正文。代码仓库里的 README 只保留构建入口 |
| `linux-7.2/` | 内核树 | Live CD 默认从这里取内核 |
| `OKRALINUX/` | sysroot | 交叉编译得到的用户态，含 binutils、glibc、systemd |
| `rootfs/` | 工作根 | 构建用的根文件系统 |
| `limine/` | 引导器 | ISO 阶段使用 `limine/bin/limine` |
| `systemd-main/` | systemd 源码 | 当前构建没有链接 libmount |

工作区根上的 ISO、squashfs、qcow/raw 磁盘和 `.oaa` 不进 Git。文档改动写进 `DOCS/`。

## 3. 构建 Live CD

在 `okra-linux/` 里：

```bash
SKIP_QEMU=1 live-build/scripts/build-livecd
```

`build-livecd` 按顺序调用：

```text
build-kernel
build-rootfs
build-coreutils
build-dhcpcd
build-initramfs
build-squashfs
build-iso
```

不设置 `SKIP_QEMU=1` 时，流水线结束后执行 `test-qemu`。只改安装器或 initramfs 时，不必重编内核和 squashfs，跑 `build-initramfs` 再跑 `build-iso` 即可。squashfs 是阶段 1 的复制源；安装器脚本的新版本放在 initramfs 的 payload 里，启动 Live CD 时盖到 `/usr` 上。已经装到磁盘上的那一份不会跟着 ISO 变，除非重新安装，或直接改磁盘镜像里的 `/usr/lib/okrainstall/`。

常用覆盖：

```bash
JOBS=4 SKIP_QEMU=1 live-build/scripts/build-livecd
FORCE_KERNEL=1 SKIP_QEMU=1 live-build/scripts/build-livecd
KERNEL_TREE=../linux-7.2 ROOTFS=../rootfs LIMINE_BIN=../limine/bin/limine
```

产物是 `okra-linux/okra-linux-base-os-0.iso`（当前 Base OS 镜像）以及流水线名义上的 `okra-linux-livecd.iso`。改构建脚本后先 `bash -n`，再跑一次 `SKIP_QEMU=1` 的相关阶段。

Base OS 里没有 GNU grep，也没有 sed。安装器脚本只用 bash 的参数展开和 `case`。systemd 服务的默认 `PATH` 不含 `/bin` 和 `/sbin`，而 `mount` 在 `/bin/mount`，`ldconfig` 在 `/sbin/ldconfig`。OKRAINSTALL 在脚本开头和 continue 单元里都补上了这两段路径。

## 4. Live CD 怎么启动

Limine 配置在 `live-build/iso-root/limine.conf`。菜单项从 `boot():/boot/vmlinuz` 加载内核，从 `boot():/boot/initramfs-live.img` 加载 initramfs。命令行包含：

```text
console=tty0 console=ttyS0,115200n8 nomodeset loglevel=6 systemd.show_status=yes rd.live.image
```

`nomodeset` 让内核只用系统帧缓冲，VGA 窗口保持文本控制台。没有它时，fbcon 会等图形客户端，QEMU 窗口是黑的，串口却仍在打印。

initramfs 的 `init` 做这些事：挂上 live 介质，把 squashfs 挂到 `/run/live/rootfs`，准备 overlay，把 `okrainstall-payload` 拷进 `/usr`，再把 `mount` 链到 `/usr/bin`，然后切到真实根并启动 systemd。payload 里有安装器脚本、GRUB 主题和 `BOOTX64.EFI`。主题来自 `OKRA_GRUB_THEME`，默认是 `/home/forge/下载/Tachibana_Sherry_Fedora`。`build-initramfs` 用宿主机的 `grub2-mkimage` 生成 EFI 镜像，模块目录是 `/usr/lib/grub/x86_64-efi`。嵌入的 `early.cfg` 只有三行：

```text
search --no-floppy --file --set=root /boot/grub2/grub.cfg
set prefix=($root)/boot/grub2
configfile $prefix/grub.cfg
```

这张 EFI 应用不依赖固件里的 GRUB 前缀。它在所有分区里找 `/boot/grub2/grub.cfg`，找到后从那一块 ext4 读配置。

Live CD 上启用 `okrainstall-live.service`。它要求 `/run/live/rootfs` 存在，在 tty1 上运行阶段 1。

## 5. 安装到磁盘

阶段 1 把整块磁盘做成 GPT：

| 分区 | 大小 | 用途 |
|---|---|---|
| 1 | 1 MiB | BIOS boot（当前 QEMU 测试走 UEFI，这块是占位） |
| 2 | 512 MiB | EFI 系统分区。`BOOTX64.EFI` 放在这里 |
| 3 | 剩余空间 | ext4 根。内核、GRUB 配置、主题和用户态都在这里 |

复制用 rsync，源是 `/run/live/rootfs`。squashfs 里没有 `/dev`、`/proc`、`/sys`、`/run`、`/tmp`。`--exclude=/dev/*` 不会把缺失的目录变出来。阶段 1 在复制之后创建这些目录，并把 `/tmp` 设成 `1777`。否则内核在 pivot 之后按相对路径挂 `dev`，得到 `ENOENT`（日志里是 `devtmpfs: error mounting -2`）。

复制之后阶段 1 还做这些修补，因为它们不在 squashfs 里，live 环境的挂载也不会被 rsync 带走：

- `/usr/sbin/ldconfig` 指向 `../../sbin/ldconfig`。否则 `ldconfig.service` 报 203/EXEC。
- `/etc/group` 里若没有 `systemd-journal`，追加 `systemd-journal:x:190:`。
- 没有 `modprobe` 时，把 `modprobe@.service` 掩码到 `/dev/null`。内核配置没有 kmod。
- 把一批 `.mount` 单元掩码掉。systemd 没有 libmount，这些单元会以 `JOB_UNSUPPORTED` 结束。被掩码的是 `dev-hugepages.mount`、`dev-mqueue.mount`、`proc-sys-fs-binfmt_misc.automount`、`sys-fs-fuse-connections.mount`、`sys-kernel-config.mount`、`sys-kernel-debug.mount`、`sys-kernel-tracing.mount`。
- 写入 `/sbin/okra-init`，并在 GRUB 菜单里用 `init=/sbin/okra-init`。
- 把主题字体名用 bash 替换写成 `Sarasa Regular 28`。Base OS 没有 sed。
- 写入 `phase=2`，启用 `okrainstall-continue.service`，关掉 live 单元。

GRUB 菜单的内核命令行是：

```text
root=PARTUUID=<根分区> rw rootwait init=/sbin/okra-init console=ttyS0,115200n8 console=tty0 nomodeset systemd.show_status=yes
```

`console=` 的最后一项成为 `/dev/console`，所以 `tty0` 必须写在 `ttyS0` 后面，VGA 才是主控制台。串口仍然是 `ttyS0,115200n8`，两边都能看到内核日志。

## 6. okra-init 和 systemd

当前 systemd 构建没有 libmount。`mount_supported()` 取决于 `dlopen` 能否打开 libmount。打不开时，`mount_option_mangle()` 在挂载选项非空时返回 `EOPNOTSUPP`。`/dev` 的早期挂载选项正好是 `mode=0755,size=4m`。若 `/dev` 还不是挂载点，这一步是致命的，PID 1 会停在 `Freezing execution`。

`OKRALINUX/lib64/libmount.so.1` 是 Fedora 的库，和 Okra 的 libc 不是一套。不能把它链进 Okra 的 PID 1。

`/sbin/okra-init` 因此在 `exec` systemd 之前用 `/bin/mount` 挂上：

```text
proc, sysfs, devtmpfs /dev, tmpfs /run, tmpfs /dev/shm, devpts,
cgroup2, hugetlbfs, mqueue, debugfs, tracefs, configfs
```

对应的内核选项已经是 `=y`：`HUGETLBFS`、`POSIX_MQUEUE`、`DEBUG_FS`、`TRACING`、`FTRACE`、`CONFIGFS_FS`、`AUTOFS_FS`。`VFAT_FS` 和 `FUSE_FS` 是模块，而系统里没有 kmod，所以 EFI 分区和 fuse 在运行时挂不上。不要为了消掉日志就把这些都改成 `=y`。

systemd 起来之后，日志里仍会有两行，它们不是启动失败：

- `(sd-gens) Read-only bind remount failed, ignoring: Operation not supported`
- `Mount unit not supported, skipping *MountsFor=`

`libseccomp` 不在这套用户态里，于是 `bpf`、`sethostname` 之类的系统调用过滤行显示 `cannot be resolved`，systemd 会忽略。`ima: No TPM chip found` 是虚拟机没有 TPM。`clocksource: Watchdog remote CPU 1 read timed out` 是 KVM 上一次 IPI 没按时回来，只出现一行，不是 panic。`system.journal corrupted` 和 `EXT4-fs: recovery complete` 来自上一次直接断电。

`ldconfig` 修好之后，日志是 `Rebuild Dynamic Linker Cache skipped`，不再是 203/EXEC。

## 7. OKRAINSTALL

版本记在 `common.sh` 的 `OKRAINSTALL_VERSION`。入口是 `/usr/bin/okrainstall`。

```bash
okrainstall                 # 自己判断阶段
okrainstall --continue      # 硬盘上的 systemd 单元
okrainstall --stage 1|2|3   # 调试时强制阶段
```

判断规则：

- 有 `/run/live/rootfs`、`/run/live/media`，或内核命令行带 `rd.live.image`：阶段 1。
- 否则读 `/var/lib/okrainstall/phase`。`2` 走阶段 2，`3` 走阶段 3。
- 没有标记、又不是 live：退出并说明需要从 Live CD 重装。

标记：

| 路径 | 含义 |
|---|---|
| `/var/lib/okrainstall/phase` | `2` 或 `3`。存在就表示安装没完成 |
| `/var/lib/okrainstall/state.env` | 跨阶段状态，例如磁盘 UUID |
| `/var/lib/okrainstall/install.log` | 安装日志。阶段 3 删掉整个标记目录 |

`okrainstall-continue.service` 的 `After=` 只有 `local-fs.target`。它不等 `network-online.target`，也不等 `systemd-user-sessions.service`。后者自己 `After=network.target`，会把界面拖到网络后面。单元使用 `TTYPath=/dev/tty1`，和 `getty@tty1.service` 冲突，所以安装期间 tty1 上没有登录提示。

### 阶段 1

在 Live CD 的 tty1 上分区、格式化、复制 Base OS、植入 GRUB 和阶段标记，然后重启。界面是 bash TUI。复制过程有进度条。

### 阶段 2

在已安装系统上做本机配置，然后把标记改成 `3`，等窗口里按 Enter，再重启。

具体动作：

- 写 `machine-id`、`/etc/locale.conf`（`LANG=C.UTF-8`）、`/etc/vconsole.conf`（`KEYMAP=us`）、空的 hostname 文件（`okralinux`）。
- 对一组模块调用 `modprobe`。失败被吃掉。没有 kmod 时这一步是空操作。
- 跑 `/usr/lib/okrainstall/hooks/stage2.d/` 里的可执行钩子。
- 关掉 live 单元，确认 continue 单元仍启用。

阶段 2 不写 `/etc/lunar/repos.d`，不调用 `lunar sync`，不读 `stage2-packages.list`。仓库还不存在。

串口上先出现 `OKRAINSTALL stage2`，结束时出现 `stage2: waiting for Enter on tty1`。Enter 必须在 QEMU 窗口里按。`ui_pause` 读的是 tty1。

### 阶段 3

串口上先出现 `OKRAINSTALL stage3`，随后是 `okrainstall: tty1 stage 3 Account` 和 `okrainstall: tty1 input: Hostname`。窗口里依次要主机名、用户名、密码、时区，再确认。

确认之后：

- 写账户和时区。
- 跳过 `lunar sync` / `lunar update`。日志写 `no repository; skipping system update`。`hooks/stage3.d/` 仍会跑。
- 关掉 continue 单元，恢复 `getty@tty1.service`。
- 删掉安装器程序、单元和 `/usr/lib/okrainstall`。
- 删除阶段标记。
- 询问是否重启。

## 8. 界面画在哪

TUI 在 `lib/ui.sh`。它只用 bash 和 ANSI，不依赖 dialog。

systemd 会把 `TTYPath` 打开成标准输入和标准输出，但 `TIOCSCTTY` 失败时只记一条调试日志，进程没有控制终端。这时 `/dev/tty` 返回 `ENXIO`。旧脚本发现标准输出是终端，就把所有画面写到 `/dev/tty`，写入失败后窗口仍是内核日志。阶段 2 的 `read </dev/tty` 也会立刻失败，`|| true` 让它不等 Enter 就重启。

现在的 `ui_claim_console` 在第一次画画面时：

1. 把 `/proc/sys/kernel/printk` 的控制台级别降到 1，避免后面的网卡链路日志盖住画面。
2. 把 `/sys/class/tty/tty0/active` 写成 `1`，切到虚拟终端 1。
3. 把标准输入、输出、错误都接到 `/dev/tty1`。

读写走已经接好的标准输入和标准输出，不再打开 `/dev/tty`。每一帧额外写一行到 `/dev/kmsg`，所以串口能证明画面已经画到 tty1，但表单本身不在串口上。

QEMU 需要图形窗口（`-display gtk` 或默认窗口）和能把按键送进客户机的键盘。测试盘的启动命令里有 `-vga std`、`-serial mon:stdio` 和 virtio 键盘。看串口日志的那个终端不是安装界面。

## 9. OkraPM

OkraPM 是用户态包管理工具链，入口是 C++17 的 `lunar`。包格式是 `.oaa`（Okra Application Artifact，tar 归档）和早期的 `.okra`。数据目录默认 `/var/lib/lunar`。

构建：

```bash
cmake -S . -B build
cmake --build build -j2
ctest --test-dir build --output-on-failure
```

本地源：

```bash
python3 repo-server/server.py --root okrapm/repo --bind 0.0.0.0 --port 8765
```

客户机（QEMU 用户网络里宿主机是 `10.0.2.2`）：

```bash
lunar repo add okra http://10.0.2.2:8765 remote
lunar sync okra
lunar install GNU.gcc
```

当前 `okrapm/repo` 里有 `GNU.make` 4.4.1 和 `GNU.gcc` 16.2.1。这是开发仓库，不是安装器会自动配置的发行版源。命令、对象引用、事务、管道、`meta.yaml` 和 OPSIS 函数表见 [OkraPM](okrapm.md)。

安装器把「有仓库再装软件」留到以后。到那时阶段 2 再写仓库配置并调用 `lunar`，阶段 3 再做系统更新。在那之前不要把占位 URL `https://repo.okralinux.example/okra` 写回安装器。

## 10. Kanina

`kanina-installer/` 是可选的 Qt 6 Widgets 安装器，C++17，版本 0.1.0。`INSTALL_KANINA=1` 时打进 rootfs。默认不启用。和 OKRAINSTALL 同时存在时，tty1 仍走 OKRAINSTALL。

## 11. 在 QEMU 里看一次安装

当前测试盘是 `okra-linux/testdisk/fast-disk.img`，变量固件是 `okra-linux/OVMF_VARS.fd`。图形窗口加串口的启动方式与上次手动测试一致：OVMF、q35、4G 内存、2 个 CPU、virtio 磁盘、GTK 显示、`ttyS0` 串口。

一次完整安装的串口标记：

```text
阶段 1    Live CD 的 tty1 上操作，串口不一定有 OKRAINSTALL 行
重启后    Run /sbin/okra-init
          OKRAINSTALL stage2
          okrainstall: tty1 stage 2
          stage2: waiting for Enter on tty1
再重启    OKRAINSTALL stage3
          okrainstall: tty1 input: Hostname
```

阶段 2 的 Enter 和阶段 3 的账户都在 GTK 窗口里输入。阶段标记为 `2` 时，下一次启动是阶段 2，不是账户页。

磁盘镜像在 QEMU 打开时不要从宿主机写。`fuse2fs` 可以在虚拟机关闭后按分区偏移挂上 ext4（根分区从扇区 1052672 开始），但它不回放日志。只有超级块里没有 needs-recovery 时，这样改 `/usr/lib/okrainstall/*.sh` 才是安全的。

## 12. 现在故意不做的事

- 不把 Fedora 的 libmount 链进 Okra 的 systemd。
- 不把 `CONFIG_VFAT_FS` 改成内建，除非以后要在运行时挂 EFI。
- 不为了消除某一行日志去打开全部内核选项。
- 不在阶段 2 配置软件源或安装软件包。
- 不在阶段 3 做 `lunar update`。
- Live CD 继续用 Limine。已安装系统用 GRUB。
