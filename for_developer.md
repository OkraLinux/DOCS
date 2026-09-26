# OKRA-Linux 编码规范

本文给写代码的人用，也给代码评审用。已经在仓库里的旧代码保持原样。修旧函数时沿用那个函数现在的缩进和命名。不要为了统一风格去批量重排、重命名或重写旧文件。

本文只约束新代码：新建的源文件，以及旧文件里新加的函数、类型、宏和全局变量。

## 1. 语言和基准

| 语言 | 仓库里的位置 | 新代码的基准 | 被下面共同规则改掉的部分 |
|---|---|---|---|
| C | 内核模块等 | Linux 内核编码风格 | 缩进宽度、行宽、命名、单行注释 |
| C++ | Lunar、OAA、Kanina | 与 C 相同的排版；资源用 RAII | 同上，另加 C++ 类型和错误处理 |
| Python | 软件源、OAA 的 Python 工具 | PEP 8 的语句和空行 | 缩进改 Tab，命名改 PascalCase，行宽改 120 |
| Rust | 目前没有，以后新写时适用 | Rust 语法和 `rustfmt` 的花括号 | 缩进改 Tab，命名改 PascalCase，行宽改 120 |
| Shell | 安装器、Live CD 构建 | 本仓库现有构建脚本的控制结构 | 缩进改 Tab，新函数和新变量改 PascalCase |

关键字、标准库、系统调用和协议要求的名字保持语言原来的拼写。例如 `if`、`std::vector`、`Vec`、`__init__`、Rust trait 规定的 `fmt`、Qt 虚函数原来的名字、环境变量 `PATH`。这些不是项目自己起的标识符。

文件名保持小写，不改成 PascalCase。

## 2. 所有语言共同遵守

1. 缩进只用真正的 Tab。Tab 宽度按 4 个字符计算。禁止用空格堆出缩进级别。续行如果要对齐括号，Tab 后面可以再补空格，那些空格只用于对齐。
2. 一行最多 120 个字符。一个 Tab 算 4。超长就在逗号或运算符后面换行。
3. 项目自己起的标识符一律 PascalCase。范围：函数、方法、变量、常量、类型、枚举成员、宏、`goto` 标签。
4. 用完整英文单词。可以保留的短写只有已经通用的，例如 `Id`、`Ctx`。不要写 `Buf`、`Tmp`、`Dev`、`Cfg`、`Ptr`、`Len`、`Cnt`、`Num`。
5. 禁止小驼峰 `camelCase`，禁止蛇形 `snake_case`，禁止全大写加下划线，例如 `BUFFER_LIMIT`。常量写 `BufferLimit`。
6. 对外函数要有文档注释，里面有三样：一句功能说明、每个参数、返回值或失败时发生什么。各语言的注释记号见后面各节。只在本文件使用的内部函数不必写这套头注释。

编辑器把 Tab 显示为 4，并使用 Tab 缩进。不要在保存时把 Tab 换成空格。

## 3. C

没在第 2 节改掉的部分，按内核 `Documentation/process/coding-style.rst`。

- 函数的左花括号单独占一行。`if`、`else`、`for`、`while`、`do`、`switch` 的左花括号跟在语句同一行。
- 关键字和左括号之间一个空格。函数名和左括号之间没有空格。
- 指针星号靠名字：`int *Pointer`。
- 不要给结构体和指针做 `typedef`。写 `struct DeviceContext`。
- 失败返回负的 errno，成功返回 `0`。
- 多步分配失败时用 `goto` 收到函数末尾，按申请的反序释放。标签用 PascalCase，例如 `FreeContext:`。
- `case` 与 `switch` 对齐。需要贯穿到下一个 `case` 时写注释。
- 函数之间空一行。头文件按组排列，组之间空一行。
- 块注释用 `/* */`。单行注释可以用 `//`。

对外函数使用：

```c
/**
 * ResetDeviceContext() - 把设备上下文恢复到空闲状态。
 * @Context: 要恢复的上下文。调用者保证它来自成功的打开操作。
 *
 * Return: 成功返回 0。Context 为空时返回 -EINVAL。
 */
```

### 正确

```c
/**
 * CreateDeviceContext() - 分配上下文并复制设备路径。
 * @DevicePath: 以空字符结尾的设备路径。
 * @Context: 成功时写入新上下文。
 *
 * Return: 成功返回 0。参数非法返回 -EINVAL，内存不足返回 -ENOMEM。
 */
int CreateDeviceContext(const char *DevicePath, struct DeviceContext **Context)
{
	struct DeviceContext *LocalContext;
	char *PathCopy;

	if (!DevicePath || !Context)
		return -EINVAL;

	LocalContext = kzalloc(sizeof(*LocalContext), GFP_KERNEL);
	if (!LocalContext)
		return -ENOMEM;

	PathCopy = kstrdup(DevicePath, GFP_KERNEL);
	if (!PathCopy)
		goto FreeContext;

	LocalContext->DevicePath = PathCopy;
	LocalContext->State = DeviceStateIdle;
	*Context = LocalContext;
	return 0;

FreeContext:
	kfree(LocalContext);
	return -ENOMEM;
}
```

### 错误

空格缩进、蛇形名字、失败路径重复释放：

```c
int create_device_context(const char *device_path, struct DeviceContext **context)
{
    struct DeviceContext *local;

    local = kzalloc(sizeof(*local), GFP_KERNEL);
    if (!local)
        return -ENOMEM;
    local->DevicePath = kstrdup(device_path, GFP_KERNEL);
    if (!local->DevicePath) {
        kfree(local);
        return -ENOMEM;
    }
    *context = local;
    return 0;
}
```

## 4. C++

排版和 C 相同：Tab、120 列、PascalCase、内核式花括号和空格。头文件仍按组排列。

另外：

- 类型写 `class DeviceContext` 或 `struct DeviceContext`，不要用 `typedef` 包一层。需要别名时写 `using DeviceId = std::uint32_t;`。
- 命名空间用 PascalCase，例如 `namespace OkraPackage`。
- 成员变量同样是 PascalCase，不加 `m_` 前缀。
- 能由析构函数释放的资源用 RAII，不手写 `goto`。必须中途退出、又还没装进对象里的资源，仍用 PascalCase 标签的 `goto`，顺序与 C 相同。
- 对外函数的文档注释和 C 用同一套 `/** */`。
- 对接 C API 时，只在边界做转换。项目自己的新名字仍然是 PascalCase。

### 正确

```cpp
/**
 * CopyPayload() - 把包内文件复制到安装根。
 * @Payload: 含有 files 或 rootfs 的目录。
 * @InstallRoot: 复制目标的根目录。
 *
 * Return: 成功返回 true。读目录或复制失败返回 false，并删掉本次已写出的文件。
 */
bool CopyPayload(const std::filesystem::path &Payload,
	const std::filesystem::path &InstallRoot)
{
	std::error_code Error;
	std::filesystem::recursive_directory_iterator It(Payload, Error);

	if (Error)
		return false;

	// 复制每个普通文件。目录只负责被创建出来。
	return true;
}
```

### 错误

```cpp
namespace okra_package {
bool copyPayload(const std::filesystem::path &payload_dir)
{
    return true;
}
}
```

## 5. Python

语句写法沿用 PEP 8：运算符两边空格、顶层函数之间空两行、导入分组。下面三项改掉 PEP 8：

- 缩进是 Tab，不是 4 个空格。
- 行宽 120，不是 79。
- 项目自己的函数、变量、类、常量用 PascalCase，不是 `snake_case`，常量也不是 `ALL_CAPS`。

语言规定的双下划线方法保持原名，例如 `__init__`、`__enter__`。

对外函数用文档字符串，字段和 C 的文档注释一样：

```python
def LoadRepositoryIndex(Root):
	"""LoadRepositoryIndex() - 从目录读取软件源索引。
	@Root: 放有 index.yaml 的目录。
	Return: 解析后的文档。文件不存在时抛出 RepositoryError。
	"""
```

用异常表示失败，不要学 C 返回负 errno。类型标注里的名字同样用 PascalCase。

### 正确

```python
class RepositoryError(Exception):
	pass


def LoadRepositoryIndex(Root):
	"""LoadRepositoryIndex() - 从目录读取软件源索引。
	@Root: 放有 index.yaml 的目录。
	Return: 解析后的文档。文件不存在时抛出 RepositoryError。
	"""
	IndexPath = Root / "index.yaml"
	if not IndexPath.is_file():
		raise RepositoryError(IndexPath)
	return IndexPath.read_text(encoding="utf-8")
```

### 错误

```python
BUFFER_LIMIT = 120

def load_repository_index(root):
    index_path = root / "index.yaml"
    return index_path.read_text()
```

## 6. Rust

仓库里还没有 Rust。一旦新增，语法和所有权按 Rust 通常写法：函数的左花括号跟在签名同一行，错误用 `Result` 和 `?`，不写 `goto`。

改掉 `rustfmt` 默认值的只有三条：Tab 缩进、行宽 120、项目标识符 PascalCase。因此函数和常量也是 PascalCase，而不是 `snake_case` 或 `SCREAMING_SNAKE_CASE`。不要为了通过默认 `rustfmt` 把名字改回去。

trait 要求的方法名保持 trait 的拼写，例如 `fmt`、`drop`。

对外函数用 `///`，字段顺序与 C 的文档注释相同：

```rust
/// OpenDevice() - 打开设备节点并读出标识。
/// @DevicePath: 设备节点路径。
/// Return: 成功时返回标识。打不开时返回 Io 错误。
```

### 正确

```rust
pub struct DeviceContext {
	pub DeviceId: u32,
	pub DevicePath: String,
}

/// OpenDevice() - 打开设备节点并读出标识。
/// @DevicePath: 设备节点路径。
/// Return: 成功时返回上下文。打不开时返回 Io 错误。
pub fn OpenDevice(DevicePath: &str) -> std::io::Result<DeviceContext> {
	let _File = std::fs::File::open(DevicePath)?;
	Ok(DeviceContext {
		DeviceId: 0,
		DevicePath: DevicePath.to_string(),
	})
}
```

### 错误

```rust
const BUFFER_LIMIT: usize = 120;

pub fn open_device(device_path: &str) -> std::io::Result<()> {
    let _file = std::fs::File::open(device_path)?;
    Ok(())
}
```

## 7. Shell

安装器和构建脚本继续用 bash 或 POSIX sh 里该文件已经选用的那一种。不要把一个 POSIX 脚本改成依赖 bash 数组，除非这个文件本来就是 bash。

新函数和新变量用 PascalCase。环境变量和工具自己规定的名字保持原样，例如 `PATH`、`HOME`、`CFLAGS`。

控制结构跟现有构建脚本：`then` 可以跟在 `if` 同一行。同一个新文件里只选一种摆法。

对外函数上面写注释，字段与 C 相同，记号用 `#`：

```bash
# MountRoot() - 把根分区挂到目标目录。
# @RootDevice: 块设备路径。
# @MountPoint: 挂载点。目录必须已经存在。
# Return: 0 表示成功，非 0 表示 mount 失败。
```

### 正确

```bash
# MountRoot() - 把根分区挂到目标目录。
# @RootDevice: 块设备路径。
# @MountPoint: 挂载点。目录必须已经存在。
# Return: 0 表示成功，非 0 表示 mount 失败。
MountRoot() {
	local RootDevice="$1"
	local MountPoint="$2"

	if [ -z "$RootDevice" ] || [ -z "$MountPoint" ]; then
		return 1
	fi
	mount "$RootDevice" "$MountPoint"
}
```

### 错误

```bash
mount_root() {
    local root_device="$1"
    mount "$root_device" "$2"
}
```

## 8. 以后可能出现的其他语言

还没有单独章节的语言，新代码先遵守第 2 节，其余语法遵守该语言自己的通用写法。

- CMake：我们写的函数名和变量名用 PascalCase。`CMAKE_` 开头的内建变量不要改写。
- 汇编：指令和寄存器保持汇编器的写法。新标签用 PascalCase。注释用该汇编器已经在用的记号。

## 9. 评审时查什么

只看本次 diff 里的新代码。旧行的风格问题不要求在这次修改。

- 缩进是 Tab，宽度按 4。没有用空格充当缩进。
- 新行不超过 120 字符。
- 新标识符是 PascalCase 的完整英文单词。语言强制的名字除外。
- 没有 camelCase、snake_case，也没有项目自己的全大写下划线常量。
- 新的对外函数有该语言规定的文档注释，里面有功能、参数和返回值。
- C 和 C++ 的花括号、空格、`switch` 仍然像内核。
- diff 没有把无关的旧函数重新排版或重命名。

## 10. 二进制边界

新建的动态库、插件和跨语言导出遵守 [OAABI](oaabi.md)。同一个动态库内部的 C++ 或 Rust 类型不由那份标准重写。已经导出的旧符号保持原样，包括 Lunar 的 `lunar_plugin_init`。
