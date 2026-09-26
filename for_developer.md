# OKRA-Linux C 编码规范

本文给写 C 代码的人用，也给代码评审用。基准是 Linux 内核编码风格。第四、五、六节改掉了内核里对应的习惯；没点名的部分仍然按内核来，包括花括号、空格、函数怎么写、宏怎么写、头文件怎么排、`goto` 怎么收尾、`switch` 怎么排、空行怎么留。

## 1. 谁必须遵守

从本文生效之后，新写的 C 代码按本文来。两种情况算新代码：

- 新建的 `.c`、`.h` 文件。
- 旧文件里新加的函数、类型、宏、全局变量。

已经在仓库里的旧代码保持原样。修旧函数里的缺陷时，沿用那个函数现在的缩进和命名，不要顺手改成新风格。不要为了统一风格去批量重排、重命名或重写旧文件。

本文只管 C。Shell、C++、Python 不在这里规定。

## 2. 设计原则

1. 读代码的人比写代码的人多。名字要能直接读成一句英文。
2. 新代码看起来要像内核代码的排版，再换成项目自己的名字和缩进宽度。
3. 错误路径集中收尾，用 `goto` 按申请的反序释放资源。
4. 旧代码是历史，不是待整改清单。评审只对本次新增的 C 代码使用本文。

## 3. 仍然沿用内核的部分

这些不要在项目里另起一套。细节以内核 `Documentation/process/coding-style.rst` 为准，只把被第四、五、六节改掉的几点除外。

- 函数的左花括号单独占一行。`if`、`else`、`for`、`while`、`do`、`switch` 的左花括号跟在语句同一行。
- `if`、`for`、`while`、`switch` 与左括号之间有一个空格。函数名和左括号之间没有空格。
- 多数二元运算符两边各有一个空格。一元运算符、`->`、`.` 旁边不加空格。
- 指针星号靠名字：`int *Pointer`。
- 不要给结构体和指针做 `typedef`。需要类型时写 `struct DeviceContext`。内核允许的例外（如不透明句柄确实需要隐藏布局）才使用 `typedef`，名字仍按第五节。
- 分配失败、调用失败走 `goto`。标签放在函数末尾，释放顺序与申请顺序相反。标签名称使用 PascalCase，例如 `FreeBuffer:`。
- `switch` 的 `case` 与 `switch` 对齐，不在 `switch` 里面再缩进一层。贯穿到下一个 `case` 时要写注释说明。
- 函数之间空一行。函数内部用空行把申请、工作和收尾分开，不要每行都空。
- 头文件按组排列，组与组之间空一行：先内核或系统头，再本模块头。
- 返回值沿用内核习惯：成功为 `0`，失败为负的 errno。
- 不要在表达式里写多余的括号来“保险”。不要把赋值塞进 `if` 的条件里，除非这是内核里那种已经公认的简写，并且不会让人读错。

## 4. 行宽与缩进

缩进只用真正的 Tab 字符。Tab 宽度按 4 个字符显示和计算。禁止用空格堆出缩进级别。

一行最多 120 个字符。计算时一个 Tab 算 4。超过就在逗号或运算符后换行。续行先用 Tab 进入下一缩进级；如果还要和上一行的某个括号对齐，Tab 后面可以再补空格。这些空格是对齐，不是缩进。

编辑器请把 Tab 显示为 4，并打开“缩进使用 Tab”。不要打开“保存时把 Tab 换成空格”。

### 正确

```c
int OpenDeviceContext(const char *DevicePath, struct DeviceContext **Context)
{
	struct DeviceContext *LocalContext;

	if (!DevicePath || !Context)
		return -EINVAL;

	LocalContext = kzalloc(sizeof(*LocalContext), GFP_KERNEL);
	if (!LocalContext)
		return -ENOMEM;

	return 0;
}
```

### 错误

用空格缩进，或者一行超过 120 个字符仍不换行：

```c
int OpenDeviceContext(const char *DevicePath, struct DeviceContext **Context)
{
    struct DeviceContext *LocalContext;
    if (!DevicePath || !Context) return -EINVAL;
    LocalContext = kzalloc(sizeof(*LocalContext), GFP_KERNEL); /* 这一行若继续拼上很长的日志或参数，就会超过 120 */
    return 0;
}
```

## 5. 命名

项目里新代码的标识符一律使用 PascalCase。范围包括：函数、全局变量、局部变量、静态变量、结构体、联合体、枚举、枚举成员、`typedef`、宏，以及 `goto` 标签。

用完整的英文单词。不要自己发明缩写。`Id`、`Ctx` 这种已经通用的短写可以用。`Buf`、`Tmp`、`Dev`、`Cfg`、`Ptr`、`Len` 这种临时缩写不要用，写成 `Buffer`、`Temporary`、`Device`、`Config`、`Pointer`、`Length`。

禁止：

- 小驼峰 `camelCase`
- 蛇形 `snake_case`
- 宏使用全大写加下划线，例如 `BUFFER_LIMIT`

文件名仍按内核习惯使用小写，不把文件名改成 PascalCase。

### 正确

```c
#define BufferLimit 120

enum DeviceState {
	DeviceStateIdle,
	DeviceStateBusy,
};

struct DeviceContext {
	unsigned int DeviceId;
	enum DeviceState State;
	char *DevicePath;
};

static int NextContextId;

int ResetDeviceContext(struct DeviceContext *Context)
{
	if (!Context)
		return -EINVAL;

	Context->State = DeviceStateIdle;
	return 0;
}
```

### 错误

```c
#define BUFFER_LIMIT 120

enum device_state {
	DEVICE_STATE_IDLE,
	deviceStateBusy,
};

struct device_context {
	unsigned int deviceId;
	char *dev_path;
};

static int next_id;

int reset_device_ctx(struct device_context *ctx)
{
	ctx->deviceId = 0;
	return 0;
}
```

上面的错误同时包含了全大写下划线宏、蛇形、小驼峰和自造缩写。新代码里任何一种都不要出现。

## 6. 注释

块注释、文件头注释、说明多行意图的注释，仍按内核习惯使用 `/* */`。新代码的单行注释可以使用 `//`。不要用 `//` 写跨很多行的说明。

每个对外函数在定义上方写文档注释，格式固定为：

```c
/**
 * ResetDeviceContext() - 把设备上下文恢复到空闲状态。
 * @Context: 要恢复的上下文。调用者保证它来自成功的打开操作。
 *
 * Return: 成功返回 0。Context 为空时返回 -EINVAL。
 */
```

三样都要有：函数名后的一句功能说明、每个参数一行 `@名字:`、`Return:` 说明成功和失败分别是什么。只在本文件使用的 `static` 函数不必写这套头注释；一两句 `//` 或内核式块注释就够。

### 正确

```c
/**
 * ReleaseDeviceContext() - 释放上下文及其路径缓冲区。
 * @Context: 可以为空。为空时什么都不做。
 *
 * Return: 无。
 */
void ReleaseDeviceContext(struct DeviceContext *Context)
{
	if (!Context)
		return;

	kfree(Context->DevicePath);
	kfree(Context);
}
```

### 错误

对外函数没有文档注释，或者用 `//` 代替参数和返回值说明：

```c
// release the context
void ReleaseDeviceContext(struct DeviceContext *Context)
{
	kfree(Context);
}
```

## 7. 错误处理示例

新增函数按内核方式集中收尾。标签用 PascalCase。

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

在每个失败点重复释放，或者标签使用蛇形名字：

```c
int CreateDeviceContext(const char *device_path, struct DeviceContext **context)
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

资源一多，这种重复释放就会漏。新代码用一个出口收尾。

## 8. 评审时查什么

只看本次 diff 里的新 C 代码。旧行的风格问题不作为本次评审的修改要求。

- 缩进是 Tab，显示宽度按 4。没有用空格充当缩进。
- 新行不超过 120 字符。
- 新标识符是 PascalCase，能读成完整英文单词。
- 没有 camelCase、snake_case，也没有全大写下划线宏。
- 新的对外函数有 `/** */`，里面有功能、参数和 `Return:`。
- 花括号、空格、`switch`、`goto` 收尾仍然像内核。
- diff 没有把无关的旧函数重新排版或重命名。
