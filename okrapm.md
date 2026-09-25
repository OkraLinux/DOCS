# OkraPM

OkraPM 是 Okra Rolling Linux 的用户态软件包管理工具链。当前入口是 C++17 的 Lunar；OAA 打包脚本是一组 shell 工具。更早的单文件管理器 `okpm` 仍留在 `lunar/okpm.c`，新的安装、升级和仓库操作走 Lunar。

本文是 [Okra 文档总仓库](README.md) 的一部分。总仓库地址是 `git@github.com:OkraLinux/DOCS.git`。

## 代码布局

```text
okrapm/
├── lib/okrapmlib/     C++ 核心库
├── src/lunar/         lunar 命令
├── oaatools/          oaa 命令和打包脚本
├── opsis/             OPSIS 安装脚本运行时
├── repo-server/       本地 HTTP 软件源
├── scripts/           把工具链打成 .oaa 的脚本
├── repo/              本地仓库数据
└── lunar/okpm.c       早期单文件包管理器
```

核心库 `LunarCore` 管这些部分：对象模型、仓库、依赖解析、事务、系统状态、快照和扩展接口。数据目录默认是 `/var/lib/lunar`。

## 构建 Lunar

依赖 CMake 3.16 或更高版本、C++17 编译器，以及系统里的 `tar`。

```bash
cmake -S . -B build
cmake --build build -j2
ctest --test-dir build --output-on-failure
```

可执行文件在 `build/src/lunar/lunar`。

开发和测试时可以换数据目录，避免写到系统路径：

```bash
LUNAR_DATA_DIR=/tmp/lunar lunar list
lunar --root /tmp/lunar status
```

非 root 安装默认写到数据目录下的 `rootfs/`。也可以指定安装根：

```bash
LUNAR_INSTALL_ROOT=/tmp/rootfs lunar install package.oaa
```

数据目录里主要有：

- `system.db`：已安装对象和系统状态
- `repos/`：仓库数据
- `cache/downloads/`：下载缓存
- `snapshots/`：系统快照
- `rootfs/`：非 root 时的安装根

## lunar 命令

```text
lunar [选项] <命令> [参数...]

选项：
  -r, --root <dir>          指定状态和数据目录

安装与升级：
  install <ref...>          安装包、组（#group）或 .oaa
  install-group <name>      安装一组对象
  download <ref...>         只下载，不安装
  remove <ref...>           删除
  purge <ref...>            删除并清掉配置
  update [ref...]           滚动更新整个系统或指定对象
  upgradle <target...>      系统基线升级
  sync [target...]          同步仓库或系统对象
  plan <command> <ref...>   只预览事务，不执行

流水线：
  pipe '<expr>'             对象流，例如 find "gnu.*" | where outdated | update

包文件：
  build <dir> [out_file]    从目录构建 .oaa
  verify <file>             校验 .oaa
  artifact inspect <file>   查看元数据和文件列表
  artifact verify <file>    校验校验和与归档
  artifact extract <f> <d>  解包到目录

查询：
  search <query>            按关键字搜索
  find <pattern>            按 glob 查找，例如 "gnu.*"
  list                      已安装对象
  status                    系统状态摘要
  info <ref>                对象详情
  members <#group>          展开组里的成员

状态：
  transaction list          最近的事务
  transaction show <id>     事务详情
  snapshot list             快照列表
  snapshot create [desc]    创建快照
  rollback [snapshot_id]    回滚到快照

仓库：
  repo list
  repo add <name> <url> [type]
  repo remove <name>
  repo enable <name>
  repo disable <name>

扩展：
  ext list
  ext load <path.so>
  ext run <name> [args...]
```

`upgrade` 是常见写法；当前正式的系统基线命令是 `upgradle`。目标参数可以写 `-`，从标准输入按行读入。

预览而不执行：

```bash
lunar plan install package-name
lunar plan remove package-name
lunar plan update
```

### 对象引用

Lunar 里软件包、组、系统组件和包文件都是对象。引用写成 `命名空间.名字`，可以带版本：

| 写法 | 含义 |
|---|---|
| `GNU.gcc` | 命名空间 `GNU`、名字 `gcc` 的包 |
| `GNU.gcc@16.2.1` | 指定版本。版本解析失败时，这条引用无效 |
| `gcc` | 没有点号时，整段当作名字，命名空间为空 |
| `#kde.kde-desktop` | 组。成员记在这个对象的依赖列表里 |
| `okra.systemkernel` | 命名空间是 `okra` 且名字以 `system` 开头时，类型是系统对象 |
| `./app.oaa`、`pkg.okra` | 本地包文件。名字取文件名去掉后缀，命名空间记为 `local` |
| `gnu.*` | 通配。`*` 可以出现在名字里，用于 `find` 和管道 |

标准全名是 `命名空间.名字`，带版本时是 `命名空间.名字@版本`。对象状态有五种：仓库里可用、已安装、已安装但仓库有更新、缺失、已安装但依赖断了。

没有配置仓库时，Lunar 会在数据目录里建一个名为 `main` 的本地仓库。仓库配置写在 `repos/repos.conf`，一行一个，字段用 `|` 分开：

```text
name|local|url|
name|remote|http://10.0.2.2:8765|1
```

最后一列是启用标记。`0`、`false`、`disabled` 表示禁用，空着表示启用。

### 事务和安装

改系统状态的操作都进事务。状态依次是：待处理、依赖已解析、计划已生成、已校验、提交中、已提交。失败时从提交中进入失败，再标成已回滚。

提交前会做两件事：触发扩展钩子，并自动建一份系统快照，说明文字是 `Auto snapshot before txn <id>`。快照记的是当时已安装对象的列表，文件在 `snapshots/snapshot_<id>.dat`，索引是 `snapshots/snapshots.idx`。`lunar rollback` 按这份列表生成删除和安装操作，再把系统库恢复到该快照。

安装一个对象时：

1. 本地 `.oaa` / `.okra` 的来源记成 `local:<绝对路径>`。旁边有 `.sha256` 时，第一列必须等于 `sha256sum` 的结果；没有 sidecar 则跳过这一步。
2. 仓库里的对象按下面的文件名依次查找，命中缓存就不再下载：
   - `命名空间.名字@版本.oaa`，例如 `GNU.gcc@16.2.1.oaa`
   - `名字-版本.oaa`
   - `名字-版本.okra`
   - `名字.oaa`
3. 远程下载依次试这三个 URL：`/artifacts/<文件名>`、`/<文件名>`、`/packages/<文件名>`。缓存目录是该仓库目录下的 `artifacts/`。
4. 解包到临时目录。里面有 `files/` 就用它做 payload，否则用 `rootfs/`。两边都没有则这次安装失败。
5. 安装根默认是 `/`。设了 `LUNAR_INSTALL_ROOT` 就用那个目录。没设、而且 `/` 对当前用户不可写时，改写到数据目录的 `rootfs/`。
6. 复制时会换掉目标上已有的同名文件。复制中途失败，则删掉本事务已经写入的文件，事务标成回滚，`system.db` 不更新。
7. 全部操作成功后才把已安装对象写入 `system.db`，并增加系统状态号。

删除和 purge 从系统库去掉对象记录。当前提交路径不调用包内的 `scripts/pre-install`。包内脚本由 OAA 打包阶段或 OPSIS 安装脚本自己执行。

启动时会注册三个内置扩展：`lunar-core`、`lunar-oaa`（`.oaa`）、`okrapm`（`.okra`，兼容早期 okpm）。额外的 `.so` 从数据目录的 `extensions/` 和 `/usr/lib/lunar/extensions` 加载。

### 管道

`lunar pipe` 把对象流按 `|` 拆开。第一段不是数据源时，默认源是已安装对象。

```text
数据源    find <pattern> | search <query> | list | installed | groups
过滤      where outdated
          where installed
          where repository=<名>    或 repo=
          where namespace=<名>     或 ns=
          where type=package|group|system|artifact
          where name=<通配>
转换      sort name|version|size
          limit <n>          默认 10
          unique
          expand             把组展开成成员
终端      inspect | count | install | update | remove | plan
```

例子：

```bash
lunar pipe 'find "GNU.*" | where outdated | inspect'
lunar pipe 'list | where namespace=GNU | sort name | count'
```

## OAA 包

OAA（Okra Application Artifact）是 tar 归档。`oaa` 是 `oaatools` 的安装名，脚本本身不用编译：

```bash
make -C oaatools
sudo make -C oaatools install
```

安装后命令在 `/usr/bin`，库脚本在 `/usr/lib/okrapm`。也可以直接运行 `oaatools/oaatools`。

```text
oaa new <dir>
oaa build <dir> -o package.oaa
oaa inspect package.oaa
oaa list package.oaa
oaa extract package.oaa <destination>
oaa sha256 package.oaa
oaa verify package.oaa
oaa install package.oaa
oaa remove package-name
```

`oaa install` 和 `oaa remove` 会调用 `lunar`。Lunar 不在 `PATH` 里时：

```bash
LUNAR_BIN=/path/to/lunar oaa install package.oaa
```

`oaa verify` 按这个顺序检查：

1. 旁边的 `<文件>.sha256`。有则第一列必须等于当前文件的 SHA256，不一致就失败。没有则只打印当前哈希，继续。
2. 归档里 `meta.yaml` 的 `checksum`。有则和当前哈希比较；不一致只告警，不因此失败。
3. `tar -t` 能否读出成员。依次试 zstd、gzip、xz、再试 tar 自己判断。都失败则归档不可用。

`oaa-build` 在打包前执行 `scripts/pre-build`，打包后执行 `scripts/post-build`。钩子返回非零只告警，不中断。压缩优先 `tar --zstd`；当前 tar 不支持 zstd 时改用 gzip。产物默认叫 `<name>-<version>.oaa`，并写出 sidecar：

```text
<sha256>  <文件名>
```

打包时排除 `.git`、正在生成的 `.oaa` 和对应的 `.oaa.sig`。骨架里的 `.oaaignore` 目前不会被 `oaa-build` 读取。

Lunar 自带的 `lunar build` 走 C++ `ArtifactBuilder`，同样先跑 `scripts/pre-build`，再打 tar。压缩由选项指定，默认 zstd，失败后依次试 gzip 和未压缩 tar；也可以指定 xz。输出路径为空时，文件名同样是 `<name>-<version>.oaa`。这条路径不写 `.sha256` sidecar。

### 目录和 meta.yaml

`oaa new` 生成的骨架：

```text
package/
├── meta.yaml
├── .oaaignore          列出 *.oaa、*.oaa.sig、.git/
├── rootfs/
│   ├── usr/bin/<name>  一个会打印版本的占位脚本
│   ├── usr/lib/
│   └── etc/
└── scripts/
    ├── pre-build
    ├── post-build
    ├── pre-install
    └── post-install
```

可选参数：`--name`、`--namespace`（默认 `app`）、`--version`（默认 `1.0.0`）、`--desc`、`--arch`（默认 `x86_64`）。目录已经存在时拒绝创建。

Lunar 安装时，payload 二选一：

- `files/`：工具链打包脚本使用这个名字，内容按根文件系统排布，例如 `files/usr/bin/make`
- `rootfs/`：`oaa new` 使用这个名字

两边同时存在时用 `files/`。

`meta.yaml` 由一个很小的键值解析器读取，不是完整 YAML。字段：

```yaml
name: gcc
namespace: GNU          # 也可写成 ns
version: 16.2.1
description: "GNU Compiler Collection"
architecture: x86_64    # 也可写成 arch
maintainer: "OkraLinux Team <maintainer@okralinux.cn>"
installed_size: 220567536   # 也可写成 size，单位是字节
checksum: <sha256>          # 也可写成 sha256
dependencies:               # 也可写成 deps；值是命名空间.名字
  - GNU.make
files:
  - /usr/bin/gcc
```

`name` 不能空。没写 `namespace` 时 Lunar 默认是 `app`。列表用 `- ` 开头。`#` 后面是注释。已附带的元数据例子在 `oaatools/packages/nano-8.4` 和 `oaatools/packages/fastfetch-2.52.0`。

传统 `.okra` 包用同一套 Lunar 入口安装：

```bash
lunar install package.okra
```

## OPSIS

OPSIS（OkraLinux Package Standard Installation Script）是给安装脚本用的 shell 函数库，文件是 `opsis/opsis-runtime.sh`，版本 1.0.0，按 POSIX sh / BusyBox ash 编写。它不提供独立命令，Lunar 提交事务时也不会 source 它。包的安装脚本在开头引入：

```sh
. /path/to/opsis-runtime.sh
```

引入时就会检查安装根是否存在，并创建记录目录。

| 变量 | 默认 | 作用 |
|---|---|---|
| `OPSIS_SYSROOT` | `/` | 安装根。目录不存在则脚本直接退出 |
| `OPSIS_ALLOW_NONROOT` | 未设置 | 设为 `1` 时，`opsis_check_user` 允许非 root |
| `OPSIS_DB_DIR` | `$OPSIS_SYSROOT/var/lib/okrapm/db` | 包记录目录。这和 Lunar 的 `/var/lib/lunar` 是两套状态 |

函数：

| 函数 | 行为 |
|---|---|
| `opsis_log_info` / `opsis_log_warn` / `opsis_log_error` / `opsis_log_step` | 带 `[OPSIS:INFO]` 等前缀的日志。警告和错误走标准错误 |
| `opsis_die` | 打失败日志并 `exit 1` |
| `opsis_check_user` | 当前用户不是 root，且没开 `OPSIS_ALLOW_NONROOT=1`，则退出 |
| `opsis_check_disk_space <KB>` | 用 `df -k` 看安装根的可用空间，不够则退出。参数为空则跳过 |
| `opsis_install_file <src> <dst> [mode] [owner]` | 拷到 `$OPSIS_SYSROOT` 下的绝对路径。mode 默认 `0755`，owner 默认 `0:0`。`chmod` / `chown` 失败不中断 |
| `opsis_install_dir <src> [dst前缀]` | 把源目录下的每一项拷进安装根。源目录不存在则什么都不做 |
| `opsis_update_ldconfig` | 安装根里有 `ldconfig` 时，先试 `chroot` 执行，再试 `ldconfig -r` |
| `opsis_update_systemd_units` | 安装根里已有 `/run/systemd/system` 且本机有 `systemctl` 时执行 `daemon-reload` |
| `opsis_record_installed <名> <版本> <manifest>` | 写入 `$OPSIS_DB_DIR/<名>/version`、`installed_time`（UTC）和 `manifest` |

一个安装脚本可以这样写：

```sh
#!/bin/sh
. ./opsis-runtime.sh

opsis_check_user
opsis_check_disk_space 10240
opsis_install_dir ./files /
opsis_install_file ./files/usr/bin/make /usr/bin/make 0755 0:0
opsis_update_ldconfig
opsis_record_installed make 4.4.1 ./meta.yaml
```

## 本地软件源

`repo-server/server.py` 是一个多线程 HTTP 服务。它扫 `artifacts/` 里的 `.oaa` 和 `.okra`，从每个归档抽出 `meta.yaml`，用 `---` 拼成 `index.yaml`，再把归档本身按静态文件提供出去。

```bash
python3 repo-server/server.py
python3 repo-server/server.py --root okrapm/repo --bind 0.0.0.0 --port 8765
```

`--root` 默认是 `okrapm/repo`。启动时会创建 `artifacts/` 并先写一次索引。

| 请求 | 行为 |
|---|---|
| `/`、`/index.html` | 重新生成索引，返回 HTML 包列表和给客户机用的 `lunar` 命令 |
| `/index.yaml`、`/packages.idx` | 重新生成索引并返回正文。Lunar 同步时认这两个名字，也认 `index.db` |
| `/artifacts/<文件名>` | 下载归档。这是 Lunar `fetch_artifact` 的第一候选路径 |

索引里一个包是一段 `meta.yaml`。抽不出元数据的归档会变成一行 `# skipped <文件名>: <原因>`，不进入对象列表。当前 `okrapm/repo/index.yaml` 里有两段：`GNU.gcc` 16.2.1（依赖 `GNU.make`）和 `GNU.make` 4.4.1。

客户机，包括 QEMU 用户网络里的虚拟机（宿主机在客户机侧是 `10.0.2.2`）：

```bash
lunar repo add okra http://10.0.2.2:8765 remote
lunar sync okra
lunar install GNU.gcc
```

`repo add` 可以多带一个类型参数。URL 以 `http://`、`https://` 或 `file://` 开头时默认是 `remote`，其他路径默认是 `local`。远程仓库同步时依次下载 `<url>/index.yaml`、`<url>/index.db`、`<url>/packages.idx`。正文里出现 `name:` 或 `packages:` 时按 YAML 段解析，段与段之间用 `---` 分开；否则按系统库那种一行一个对象来解析。解析出的对象写入该仓库缓存里的 `index.db`。

`lunar sync` 不带参数时同步全部已启用仓库。安装 `GNU.gcc` 时会先解析依赖，再按 `GNU.gcc@16.2.1.oaa` 这个文件名到 `/artifacts/` 下载。

`scripts/package-gnu-toolchain.sh` 用来灌满这个仓库：

- 在 `OKRALINUX` sysroot（默认是构建盘上的 `OKRALINUX/`）里编译 GNU make，默认版本 `4.4.1`，安装前缀 `/usr`。
- payload 放在 `files/`，元数据命名空间是 `GNU`，打成 gzip 的 `repo/artifacts/GNU.make@<版本>.oaa`，并写 `.sha256`。
- 把已有的 `gcc-16.2.1.oaa` 解开，换上 `GNU.gcc` 的 `meta.yaml`（依赖 `GNU.make`），再打成 `repo/artifacts/GNU.gcc@16.2.1.oaa`。
- 调用 `server.write_index()` 刷新 `repo/index.yaml`。

覆盖变量：`OKRALINUX`、`REPO`、`GCC_OAA`、`MAKE_VER`、`JOBS`。make 的源码压缩包缓存在 `okra-linux/live-build/src/`，没有就从 `https://ftp.gnu.org/gnu/make/` 下载。
