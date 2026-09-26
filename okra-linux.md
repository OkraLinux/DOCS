# OkraLinux

OkraLinux 是 LiveCD 的构建项目。它把 Linux 内核、initramfs、根文件系统、SquashFS 和 Limine BIOS/UEFI ISO 装到一起。

本文是 [Okra 文档总仓库](README.md) 的一部分。总仓库地址是 `git@github.com:OkraLinux/DOCS.git`。

## 仓库布局

```text
okra-linux/
├── live-build/
│   ├── initramfs/       initramfs 暂存树
│   ├── iso-root/        ISO 暂存树和 Limine 配置
│   └── scripts/         可重复的构建和测试阶段
├── OKRAINSTALL/         三阶段 TUI 安装程序（默认）
└── kanina-installer/    Qt 6 安装器（可选）
```

内核、工作根文件系统和 Limine 默认在仓库外面：

```text
../linux-7.2/
../rootfs/
../limine/bin/limine
```

换位置时用 `KERNEL_TREE`、`ROOTFS`、`LIMINE_BIN` 和 `PROJECT_ROOT`。

## 构建 LiveCD

在 OkraLinux 仓库里：

```bash
SKIP_QEMU=1 live-build/scripts/build-livecd
```

流水线按这个顺序跑：

```text
build-kernel
build-rootfs
build-coreutils
build-dhcpcd
build-initramfs
build-squashfs
build-iso
```

ISO 写到 `okra-linux-livecd.iso`。不设置 `SKIP_QEMU=1` 时，流水线结束后会执行 `test-qemu`。

单独跑某一段：

```bash
live-build/scripts/build-kernel
live-build/scripts/build-rootfs
live-build/scripts/build-coreutils
live-build/scripts/build-initramfs
live-build/scripts/build-squashfs
live-build/scripts/build-iso
live-build/scripts/test-qemu
```

常用覆盖：

```bash
JOBS=4 SKIP_QEMU=1 live-build/scripts/build-livecd
FORCE_KERNEL=1 SKIP_QEMU=1 live-build/scripts/build-livecd
QEMU_MEMORY=4G QEMU_SMP=4 live-build/scripts/test-qemu
```

生成的 LiveCD 使用 SquashFS，上层是 OverlayFS，systemd 是 PID 1。Limine 命令行包含 VGA 输出和适合 VMware 的设置：

```text
console=tty0 nomodeset audit=0 systemd.show_status=false rd.live.image
```

生成的 ISO、SquashFS、initramfs、内核镜像和 QEMU 日志不要放进 Git。改构建脚本后，先用 `bash -n` 检查语法，再跑一遍 `SKIP_QEMU=1` 的完整构建。

## Fedora 上的构建

GitHub Actions 在 Fedora 容器里构建 LiveCD，工作流是 `.github/workflows/build-livecd-fedora.yml`。它做出基于 Fedora 的 rootfs 和 BIOS/UEFI ISO，校验 ISO，再上传为工作流产物。推送到 `main`、打开 pull request，或在 Actions 页手动 `workflow_dispatch` 都会触发。

本地等价命令：

```bash
FEDORA_ROOTFS=1 FEDORA_RELEASE=41 SKIP_QEMU=1 \
  live-build/scripts/build-livecd
```

托管的容器任务没有可靠的嵌套虚拟化，所以工作流不跑 QEMU。QEMU 启动测试放在本机，或放在自托管 runner 上。

## OKRAINSTALL（默认）

`OKRAINSTALL/` 是跨两次重启的三阶段 TUI 安装程序：

1. LiveCD：分区、复制 Base-OS、植入脚本与 `phase=2`、安装 GRUB、重启
2. 硬盘 Base-OS：本机配置、驱动、`phase=3`、重启。不写软件源，不装包
3. Final-OS：账号/时区、清理残留并删除标记。不做在线更新

详情见 [OKRAINSTALL](okrainstall.md) 和 [手册](handbook.md)。`build-rootfs` 默认 `INSTALL_OKRAINSTALL=1`。已安装磁盘用 GRUB；Live CD 仍用 Limine。

## Kanina 安装器（可选）

`kanina-installer/` 是 Qt 6 Widgets 程序，C++17，CMake 工程名 `kanina-installer`，版本 0.1.0。旁边有三份 systemd 单元：`kanina-installer.service`、`kanina-live.service`、`kanina-tui.service`，以及一份 `weston.ini`。默认不启用；设 `INSTALL_KANINA=1` 时打进 rootfs。与 OKRAINSTALL 共存时 tty1 仍走 OKRAINSTALL。
