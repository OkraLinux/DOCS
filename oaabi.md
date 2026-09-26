# OAABI

OAABI（Okra Application Binary Interface）是 Okra 用户态的二进制合同，当前版本是 1，机器名是 `OAABI1`，架构是 `x86_64`。

新建的动态库、插件，以及 C、C++、Rust、Python 之间要穿过模块边界的调用，都按本文。源文件里的缩进和命名仍按 [编码规范](for_developer.md)。本文只规定进了 ELF 之后、别的模块看得到的那一层。

已经编进系统里的旧符号保持原样。第 13 节列出现在还不算 OAABI 的接口。

## 1. 范围

OAABI 规定这些事：

- ELF 对象的类别、字节序、机器类型和动态链接器
- 穿过动态库或插件边界的类型、寄存器、栈和对齐
- 导出符号的拼写、可见性和版本
- 错误怎么返回
- OAA 包怎样声明自己属于这一合同

OAABI 不重写这些已经存在的合同：

- Linux `x86_64` 系统调用。用户程序继续用内核自己的系统调用约定。
- 同一个动态库内部的 C++ 类布局。那是编译器的 Itanium C++ ABI，出了这个动态库就不再成立。
- 内核模块 `okrapm_*`。那是内核内部接口。
- Shell 脚本、OPSIS、Python 源码。它们调用可执行文件，不共享函数签名。
- systemd 的 D-Bus 接口。

一个进程里只装 OAABI 1 的动态库。以后若有 OAABI 2，两代动态库不放进同一个进程。

## 2. 身份

当前 Okra sysroot（`OKRALINUX/lib64/libc.so.6`）已经是下面这个对象。新的可执行文件和动态库与它对齐。

| 项目 | 值 |
|---|---|
| 机器名 | `OAABI1` |
| OAA `architecture` | `x86_64` |
| OAA `abi` | `OAABI1`。只含数据、没有 ELF 的包可以不写 |
| 目标三元组 | `x86_64-okra-linux-gnu` |
| ELF 类别 | `ELFCLASS64` |
| 字节序 | 小端，`ELFDATA2LSB` |
| 机器 | `EM_X86_64`（62） |
| 链接后的 OS/ABI | `ELFOSABI_GNU`（3），ABI 版本 0。未链接的 `.o` 可以仍是 `ELFOSABI_NONE` |
| 动态链接器 | `/lib64/ld-linux-x86-64.so.2` |
| C 库 | 这份 sysroot 里的 glibc。`NT_GNU_ABI_TAG` 记的是 Linux 7.2.0 |
| 指令集下限 | x86-64-baseline |

x86-64-baseline 是：CMOV、CMPXCHG8B、x87、FXSR、MMX、syscall、SSE、SSE2。glibc 上的 GNU property 把 `ISA needed` 标成 baseline；对象里可以出现 v2、v3、v4 指令，但那些路径必须在 CPUID 之后才进入。新包的 `ISA needed` 保持 baseline。不要把 AVX、AVX2 或 AVX-512 写成加载这个文件的前提。

C 库必须是 Okra 自己的 glibc。Fedora 的 `libmount.so.1` 这类外来库和这份 libc 不是一套，不能 `dlopen`，也不能写进 `DT_NEEDED`。

## 3. 引用的外部约定

调用约定采用 System V AMD64 psABI，数据模型是 LP64。OAABI 在它上面加第 4 节以后的限制，让 C 以外的语言能按同一张表生成绑定。

系统调用单独记在这里，避免和函数调用混用：

| | 函数调用 | 系统调用 |
|---|---|---|
| 号或返回值 | 整数返回值在 `RAX` | 调用号在 `RAX`，返回值也在 `RAX` |
| 参数 | `RDI` `RSI` `RDX` `RCX` `R8` `R9` | `RDI` `RSI` `RDX` `R10` `R8` `R9` |
| 被调用方破坏 | 见第 5 节 | `RCX` 和 `R11` |

内核把失败写成 `RAX` 里的负 errno。OAABI 函数同样用负 errno 做返回值，这样绑定层不必再读线程局部的 `errno`。

## 4. 数据模型

`char` 是有符号的。空指针的位型是全 0。宽字符 `wchar_t` 是 4 字节。下面的宽度和对齐是穿过边界时的宽度。

| 类型 | 宽度 | 对齐 |
|---|---|---|
| `char`、`_Bool` | 1 | 1 |
| `short` | 2 | 2 |
| `int`、`int32_t` | 4 | 4 |
| `long`、`long long`、`int64_t`、指针、`size_t`、`ptrdiff_t` | 8 | 8 |
| `float` | 4 | 4 |
| `double` | 8 | 8 |
| `long double` | 16（x87 80 位，占 16 字节） | 16 |

`long double` 只允许留在单个模块内部。OAABI 签名里用 `float` 或 `double`。文件偏移写 `int64_t`，不写 `off_t`，这样宽度写在签名上。

导出的状态码用 `int32_t` 常量。不用 C 的 `enum` 做导出类型，避免编译器把枚举收成不同宽度。

共享布局的结构体：

- 成员按声明顺序排列，对齐取成员对齐的最大值。
- 不使用位域。
- 为了对齐空出来的字节写成显式成员，名字 `Padding0`、`Padding1`。
- 尾部填充算进 `sizeof`。
- 布局一旦发布就冻结。要加字段就新建一个结构体和对应的新符号。

能藏在一个动态库里的对象用不完全类型。调用方只有指针，布局由创建它的那个库独占。

## 5. 调用

整数和指针参数从左到右使用 `RDI`、`RSI`、`RDX`、`RCX`、`R8`、`R9`。浮点参数使用 `XMM0` 到 `XMM7`。整数返回值在 `RAX`。`float` 和 `double` 的返回值在 `XMM0`。

第 7 个整数参数起放入栈，从右往左，每个参数占 8 字节。`call` 执行时 `RSP` 是 16 的倍数。函数入口处 `RSP+8` 是 16 的倍数，因为返回地址占了 8 字节。

被调用方必须保存 `RBX`、`RBP`、`R12`、`R13`、`R14`、`R15`。方向标志在进入和返回时都是 0。不调用别的函数、也不改栈指针的叶子函数可以使用 `RSP` 下方 128 字节的红区。

聚合类型一律用指针传递和返回。调用方分配对象，把指针放进整数寄存器。这样绑定层没有隐藏的返回指针，也没有按 psABI 分类寄存器的步骤。

新的 OAABI 函数不是可变参数函数。`printf` 属于 libc，不属于 Okra 库的导出面。

## 6. 可以穿过边界的类型

可以出现在导出签名里的类型：

- `void`
- 第 4 节里除 `long double` 以外的整数、`_Bool`、`float`、`double`、指针
- 指向不完全结构体的指针
- 由上述类型组成、并且按第 5 节用指针传递的结构体
- 符合本文的函数指针

字符串是以空字符结尾的 UTF-8，类型写 `const char *`。逻辑内容里不再嵌空字符。

所有权写进该函数的文档注释，只用这三词：

- `Borrowed`：指针只在这次调用期间有效。
- `Transferred`：被调用方接手，并由它销毁。
- `CallerFrees`：被调用方用 Okra glibc 的 `malloc` 分配，调用方用同一份 libc 的 `free` 释放。

有自己构造函数的对象提供成对的创建和销毁函数，不把裸指针交给对方 `free`。

禁止出现在导出签名里：

- C++ 类、引用、`std::string`、`std::vector`、虚函数、异常
- Rust 的 `String`、`Vec`、trait 对象、`Result`
- `PyObject *`
- `FILE *`。要传文件就传文件描述符 `int`
- 位域、`long double`、C 的 `enum`

一个动态库里 `malloc` 得到的内存，可以由另一个动态库 `free`，因为进程里只有这一份 Okra glibc。别的分配器的指针由那个分配器自己的释放函数回收。

## 7. 错误

导出函数二选一，写进文档注释：

- 成功返回 `0`，失败返回负的 errno。
- 成功返回字节数或个数（大于等于 `0`），失败返回负的 errno。

C++ 异常、Rust panic、Python 异常停在边界里面，由包装函数收成负 errno。边界上不抛出、不展开到另一种语言的栈。

函数可以同时设置 `errno`，但调用方只看返回值。

这些函数默认不是异步信号安全的。只有文档注释写了可以在信号处理函数里调用的，才可以。

## 8. 符号和动态库

导出使用 C 链接。C++ 写 `extern "C"`。Rust 写 `extern "C"`。动态符号表里不出现 C++ 或 Rust 的名字改编。

新符号是 PascalCase，并带库自己的前缀。本文的合同符号用前缀 `Oaabi`。一个符号的参数和返回值发布之后不改。要改行为就增加新符号。

可见性默认隐藏，只把合同符号标成默认可见。新的动态库和可执行文件使用位置无关代码。可执行文件是 PIE。链接时打开 `RELRO` 和立即绑定（`-Wl,-z,relro,-z,now`）。

`DT_SONAME` 写成 `libName.so.1`。这里的 `1` 跟着该库的不兼容变更走，和 OAABI 的版本号分开。删掉已有符号、或改变已有符号的含义时，才增加这个主版本。

使用版本脚本时，版本标签写 `OAABI_1`。这是链接器的版本标签，不是 C 标识符。只增加符号时留在 `OAABI_1`。旧符号留在原标签里，继续按原语义工作。

`DT_NEEDED` 只指向 Okra sysroot 里的库。

新的插件加载器使用 `dlopen` 标志 `RTLD_NOW | RTLD_LOCAL`。

## 9. 插件入口

新插件导出一个数据符号 `OaabiPlugin`。布局在 OAABI 1 里冻结。

| 偏移 | 宽度 | 成员 | 含义 |
|---|---|---|---|
| 0 | 4 | `Version` | 必须是 `1` |
| 4 | 4 | `Reserved` | 必须是 `0` |
| 8 | 8 | `Init` | 成功返回 `0`，失败返回负 errno |
| 16 | 8 | `Fini` | 释放 `Init` 拿到的资源。返回类型是 `void` |

结构体大小 24，对齐 8。加载器只调用 `Version == 1` 的入口。其他版本直接拒绝，不调用 `Init`。

规范头文件是下面这一份。它是头文件合同，OAABI 1 不要求单独的 `liboaabi.so`。

```c
#ifndef OaabiHeader
#define OaabiHeader

#ifdef __cplusplus
extern "C" {
#endif

#define OaabiVersion 1

#include <stdint.h>

/**
 * struct OaabiPlugin - 插件入口，布局在 OAABI 1 冻结。
 * @Version: 必须是 OaabiVersion。
 * @Reserved: 必须是 0。
 * @Init: 装入插件时调用。
 * @Fini: 卸载插件时调用。
 */
struct OaabiPlugin {
	uint32_t Version;
	uint32_t Reserved;
	int (*Init)(void);
	void (*Fini)(void);
};

/**
 * OaabiGetVersion() - 返回本头文件的 OAABI 主版本。
 *
 * Return: 恒为 1。
 */
static inline int OaabiGetVersion(void)
{
	return OaabiVersion;
}

#ifdef __cplusplus
}
#endif

#endif
```

安装路径是 `/usr/include/oaabi/oaabi.h`。某个库自己的 C 合同放在 `/usr/include/<库名>/`，规则与此相同：只有 C 类型，每个导出函数有文档注释。

插件示例：

```c
/**
 * PluginInit() - 注册本插件提供的操作。
 *
 * Return: 成功返回 0。
 */
static int PluginInit(void)
{
	return 0;
}

static void PluginFini(void)
{
}

const struct OaabiPlugin OaabiPlugin = {
	.Version = OaabiVersion,
	.Reserved = 0,
	.Init = PluginInit,
	.Fini = PluginFini,
};
```

下面这份导出不能当 OAABI 插件。它把 C++ 类指针放进了动态符号，任何一次 Lunar 重编译都可能改变布局：

```c++
extern "C" bool lunar_plugin_init(okrapm::ExtensionApi *Api);
```

## 10. 语言怎么接上

同一个动态库内部可以自由使用该语言的类型和错误机制。穿出动态库的那一层改成第 6 节和第 7 节。

C++：包装函数用 `extern "C"`。包装函数内部接住全部异常，返回负 errno。不把 `std::mutex` 交给另一个动态库；跨库的锁用 glibc 的 `pthread_mutex_t`。

Rust：导出函数是 `extern "C"`，名字按 PascalCase 手写，不用 Rust 默认的蛇形导出。`catch_unwind` 把 panic 收成负 errno。共享结构体使用 `repr(C)`，并且优先改成不透明指针。

Python：`PyObject *` 留在 CPython 扩展模块里。Okra 函数由 ctypes 或一层很薄的 C 扩展去调。`argtypes` 和 `restype` 与 C 签名一致。不把别的发行版的 `libpython` 装进 Okra 进程。

Shell：进程退出码 `0` 表示成功，大于 `0` 表示失败。退出码不是负 errno。脚本和 OAABI 函数之间隔着一次进程启动。

## 11. 包

含有 ELF 的 OAA 在 `meta.yaml` 里写：

```yaml
architecture: x86_64
abi: OAABI1
```

当前 Lunar 的解析器不认识 `abi`，会忽略这一行，所以旧工具仍能安装这种包。认识 OAABI 的检查器在 `abi` 存在且不是 `OAABI1` 时拒绝安装；在这台机器上，`architecture` 不是 `x86_64` 时也拒绝安装。

包内可执行文件的解释器是 `/lib64/ld-linux-x86-64.so.2`。动态库带第 8 节的 `DT_SONAME`。

## 12. 演进

OAABI 1 的宽度、对齐、寄存器和 `struct OaabiPlugin` 不改。

只增加新符号时，机器名仍是 `OAABI1`，版本标签仍是 `OAABI_1`。

改变已有宽度、寄存器用法或某个旧符号的含义时，发布新的机器名。进程不混装两个机器名。

旧符号在该主版本的寿命里保持可调用。

## 13. 现在还不是 OAABI 的接口

这些接口继续按它们今天的样子工作。新代码不要照着它们再导出一份。

| 接口 | 现在的合同 | 新代码 |
|---|---|---|
| Lunar `lunar_plugin_init(okrapm::ExtensionApi *)` | C++ 类指针。加载标志是 `RTLD_NOW \| RTLD_GLOBAL`，符号名是 `lunar_plugin_init` | 新插件导出 `OaabiPlugin`。现有加载器在迁到第 9 节之前保持不动 |
| 内核 `okrapm_install` 等 | 内核内部，GPL | 不作为用户态导出 |
| `/sbin/okra-init`、安装器脚本 | 可执行文件和 shell | 不增加跨库函数 |
| GRUB、Limine | 引导 | 不属于 OAABI |

## 14. 评审时查什么

只查本次新增的导出面。

- 新的动态符号是 C 链接、PascalCase，签名里只有第 6 节允许的类型。
- 聚合类型通过指针传递。
- 失败是负 errno，文档注释写了所有权和返回值。
- 动态库隐藏了其余符号，`DT_SONAME` 和 `DT_NEEDED` 指向 Okra 自己的库。
- 插件导出的是 `struct OaabiPlugin`，`Version` 为 1，`Reserved` 为 0。
- 含 ELF 的包写了 `architecture: x86_64` 和 `abi: OAABI1`。
- 没有把第 13 节的旧符号改名或改签名。
