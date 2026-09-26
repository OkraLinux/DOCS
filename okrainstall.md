# OKRAINSTALL

OKRAINSTALL 是 OkraLinux 的三阶段 TUI 安装程序，**跨两次重启**完成。源码在 `okra-linux/OKRAINSTALL/`。

本文是 [Okra 文档总仓库](README.md) 的一部分。

## 流程

```text
LiveCD 启动【阶段1】
    ↓
分区格式化 → 复制 Base-OS 到硬盘 → 将安装脚本+阶段标记放入硬盘 → 重启（第1次）
    ↓
从硬盘启动最小 Base-OS【阶段2】
    ↓
读取标记，执行脚本：本机配置、尝试加载驱动、跑钩子 → 修改阶段标记 → 重启（第2次）
    ↓
硬盘启动 Final-OS【阶段3】
    ↓
用户配置（账号/时区），清理安装残留 → 删除标记文件，安装完成
```

| 阶段 | 运行环境 | 标记 | 主要动作 |
|---|---|---|---|
| 1 | LiveCD | 写入 `phase=2` | 分区/格式化、rsync Base-OS、植入 OKRAINSTALL、GRUB、重启 |
| 2 | 硬盘 Base-OS | 改为 `phase=3` | machine-id/locale、驱动、hooks、重启。不写软件源，不装包 |
| 3 | 硬盘 Final-OS | **删除** 标记 | 主机名/用户/时区、清安装器与残留。不做 lunar 更新 |

## 阶段标记

| 路径 | 含义 |
|---|---|
| `/var/lib/okrainstall/phase` | 下一阶段编号：`2` 或 `3` |
| `/var/lib/okrainstall/state.env` | 磁盘 UUID 等跨阶段状态 |
| `/var/lib/okrainstall/install.log` | 安装日志（阶段 3 清理时删除） |

有 `phase` 文件时，硬盘上的 `okrainstall-continue.service` 在 tty1 启动并执行 `okrainstall --continue`。

## 布局

```text
OKRAINSTALL/
├── bin/okrainstall
├── lib/
│   ├── common.sh          # 标记协议、磁盘枚举
│   ├── ui.sh              # TUI（dialog/whiptail/纯 bash）
│   ├── stage1-live.sh
│   ├── stage2-base.sh
│   └── stage3-final.sh
├── libexec/okrainstall-stage1
├── systemd/
│   ├── okrainstall-live.service       # LiveCD：ConditionPathExists=/run/live/rootfs
│   └── okrainstall-continue.service   # 硬盘：ConditionPathExists=…/phase
├── stage2-packages.list               # 阶段 2 用 lunar 安装的包名列表
├── hooks/                             # （安装后）stage2.d / stage3.d 可执行钩子
└── install-into-rootfs.sh
```

扩展点：在目标系统的 `/usr/lib/okrainstall/hooks/stage2.d/`、`stage3.d/` 放入可执行脚本。`stage2-packages.list` 还在树里，当前阶段 2 不读取它。发行版还没有软件源。

界面写到 `/dev/tty1`，不写 `/dev/tty`。串口上的 `okrainstall: tty1 …` 只说明画面已经提交。整条启动链、GRUB、`okra-init` 和「为什么不链接 Fedora libmount」见 [手册](handbook.md)。

## LiveCD 集成

`build-rootfs` 默认 `INSTALL_OKRAINSTALL=1`，启用 `okrainstall-live.service`。阶段 1 会把 continue 单元启用到硬盘，并关掉 live 单元。

## 与 Kanina

Kanina 仍为可选 Qt 路径（`INSTALL_KANINA=1`）。默认安装流程走 OKRAINSTALL。
