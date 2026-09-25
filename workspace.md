# 构建工作区

这块盘是 Okra 的本地构建工作区，不是单独发布的代码仓库。说明文档的正文在 `git@github.com:OkraLinux/DOCS.git`，也就是 [Okra 文档总仓库](README.md)。

## 正在维护的目录

| 路径 | 作用 |
|---|---|
| `okrapm/` | 包管理器。见 [OkraPM](okrapm.md) |
| `okra-linux/` | LiveCD。见 [OkraLinux](okra-linux.md) |
| `linux-7.2/` | 内核源码。LiveCD 默认从这里取内核 |
| `rootfs/` | 工作根文件系统 |
| `OKRALINUX/` | 交叉编译 sysroot，里面有 binutils、glibc、systemd 的安装结果 |
| `limine/` | 引导器。ISO 阶段使用 `limine/bin/limine` |
| `glibc-2.44/`、`binutils-2.47/`、`busybox-1.37.0/`、`coreutils-9.12/`、`systemd-main/` | 上游源码树 |
| `gcc-build/`、`binutils-build/` | 工具链构建目录 |
| `sources/` | 下载下来的上游压缩包 |
| `iso-build/` | ISO 组装中间产物 |

## 镜像和包文件

工作区根上已经有这些产物，它们不该进入文档仓库或发行版 Git：

- `okra-linux/okra-linux-livecd.iso`
- `okra-linux/okra-linux-base-os-0.iso`
- `okralinux-devtools-1.0.iso`
- `okralinux.vmdk`
- `rootfs.img`、`rootfs.cpio.gz`、`rootfs.tar.gz`
- `gcc-16.2.1.oaa`、`fastfetch-2.52.0.oaa`、`nano-8.4.oaa`，以及对应的 `.sha256`

## Qt 组件缓存

根目录下大量以 git 对象名形式命名的目录，以及 `manifest.json`、`qt611/`，是 Qt 在线安装器的组件缓存（目录里是 `Updates.xml` 和 `repository.txt`）。它们不是 Okra 的软件包仓库。

## 根目录 Git

工作区根有一个 Git 仓库，目前只跟踪 `README.md`。`okrapm/` 和 `okra-linux/` 各自有独立的 Git 历史。文档不要写进这个根仓库；新增或修改说明时，改 `git@github.com:OkraLinux/DOCS.git`。
