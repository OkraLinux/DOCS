# Okra 文档

`git@github.com:OkraLinux/DOCS.git` 是 Okra 全部文档的总仓库。发行版、包管理器和构建工作区的说明都收在这里。各代码仓库里的 README 只保留该仓库的构建入口；完整说明以本仓库为准。

## 文档

| 文档 | 内容 |
|---|---|
| [OkraPM](okrapm.md) | Lunar、OAA、OPSIS 和本地软件源 |
| [OkraLinux](okra-linux.md) | LiveCD 构建、引导和安装器 |
| [构建工作区](workspace.md) | 这块构建盘上的源码树、sysroot 和镜像 |

## 获取

```bash
git clone git@github.com:OkraLinux/DOCS.git
```

## 组成部分

Okra 目前由两块代码组成：

- **OkraPM**：Okra Rolling Linux 的用户态包管理工具链。命令行入口是 `lunar`，包格式是 `.oaa` 和传统 `.okra`。
- **OkraLinux**：LiveCD 发行版构建。用 Limine 引导，SquashFS 加 OverlayFS，systemd 作为 PID 1。

二者在构建盘上相邻放置。LiveCD 流水线默认从 OkraLinux 仓库的上一级目录读取内核、rootfs 和 Limine。
