# OKRA-Linux C 代码生成约束

生成新的 C 代码时，每一行都遵守本文。未列出的排版、花括号、空格、宏的写法、头文件分组、`goto` 收尾、`switch`、空行，全部按 Linux 内核编码风格。第 2 节的五条覆盖内核中的同名习惯。

## 1. 范围

- 只对即将写出的新 C 代码生效。
- 新建文件：整个文件遵守本文。
- 旧文件：只对新增的函数、类型、宏、全局变量遵守本文。
- 禁止修改、重排、重命名已有旧代码来对齐本文。
- 修复旧函数时，保持该函数现有缩进和命名。不要把旧的 snake_case 改成 PascalCase。
- 不要改 Shell、C++、Python，除非用户明确要求改那些语言。

## 2. 硬性规则

1. 缩进只用 Tab 字符。禁止用空格做缩进。Tab 宽度 = 4。行宽上限 = 120，一个 Tab 计 4。超长则在逗号或运算符后换行。续行的缩进级用 Tab；括号对齐用的空格只能出现在这些 Tab 之后。
2. 新标识符全部 PascalCase。包括函数、全局变量、局部变量、静态变量、结构体、联合体、枚举、枚举成员、typedef、宏、goto 标签。
3. 名字用完整英文单词。禁止自造缩写。允许的短写只有已经通用的形式，例如 `Id`、`Ctx`。禁止 `Buf`、`Tmp`、`Dev`、`Cfg`、`Ptr`、`Len`、`Cnt`、`Num` 这类缩写。
4. 禁止 camelCase。禁止 snake_case。禁止全大写加下划线的宏，例如 `BUFFER_LIMIT`。宏写 `BufferLimit`。
5. 单行注释可以用 `//`。多行说明和文件头用 `/* */`。每个非 static 的对外函数定义上方必须有且仅使用下面这种文档注释：

```
/**
 * FunctionName() - 一句话说明功能。
 * @ParameterName: 该参数的含义与约束。
 *
 * Return: 成功时的值，以及失败时的值。
 */
```

每个参数一行。必须有 `Return:`。`static` 函数不写这套头注释。

## 3. 必须保持的内核规则

- 函数左花括号单独成行。`if` / `else` / `for` / `while` / `switch` 的左花括号留在行尾。
- 关键字和 `(` 之间一个空格。函数名和 `(` 之间无空格。
- 指针星号靠名字：`struct DeviceContext *Context`。
- 不给结构体和指针做 typedef。写 `struct DeviceContext`。
- 失败返回负 errno，成功返回 `0`，除非所在子系统已有别的约定。
- 多步分配失败时 `goto` 到函数末尾，按申请的反序释放。标签 PascalCase，例如 `FreeContext:`。
- `case` 与 `switch` 对齐。需要贯穿时加注释。
- 函数之间空一行。
- 头文件分组，组间空一行。
- 源文件名保持小写，不使用 PascalCase 文件名。

## 4. 输出前自检

生成结束前逐条确认：

- 没有把空格当作缩进。
- 没有一行的显示宽度超过 120。
- 新名字里没有下划线，也不是全大写。
- 新的对外函数都有第 2 节第 5 条规定的 `/** */`。
- 没有改动 diff 范围之外的旧代码。
- 没有为了风格去格式化整个旧文件。

## 5. 新函数模板

新函数按此骨架输出。缩进是 Tab。

```c
/**
 * CreateDeviceContext() - Allocate a context and copy the device path.
 * @DevicePath: NUL-terminated device path.
 * @Context: Receives the new context on success.
 *
 * Return: 0 on success. -EINVAL if an argument is NULL. -ENOMEM if allocation fails.
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

文档注释的说明文字可以用英文或中文，结构不能变。标识符必须是 PascalCase 英文。
