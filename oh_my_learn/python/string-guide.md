# Python 字符串与 `strip()` 常用操作

> `strip()` 用于删除字符串首尾的空白或指定字符；Python 字符串不可变，大多数字符串方法都会返回新字符串。

## 一、`strip()` 是什么

不传参数时，`strip()` 删除字符串开头和结尾的空白字符：

```python
text = "  hello world  "
result = text.strip()

print(result)  # hello world
```

它不会删除字符串中间的空格：

```python
text = "  hello world  "

print(text.strip())  # hello world
```

`hello` 和 `world` 中间的空格仍然存在。

默认清理的空白包括普通空格、换行符 `\n`、制表符 `\t`、回车符 `\r` 等：

```python
text = "\n\t hello \r\n"

print(text.strip())  # hello
```

## 二、`strip()`、`lstrip()` 和 `rstrip()`

```python
text = "  hello  "

print(text.strip())   # "hello"
print(text.lstrip())  # "hello  "
print(text.rstrip())  # "  hello"
```

| 方法 | 作用 |
| --- | --- |
| `strip()` | 清理左右两侧 |
| `lstrip()` | 只清理左侧 |
| `rstrip()` | 只清理右侧 |

读取文本行时，经常只删除行尾换行符：

```python
line = "    Python\n"
line = line.rstrip("\n")

print(repr(line))  # '    Python'
```

使用 `rstrip("\n")` 可以保留行首缩进。

## 三、指定要删除的字符

```python
text = "###hello###"

print(text.strip("#"))  # hello
```

也可以传入多个字符：

```python
text = "xyyxhelloxxy"

print(text.strip("xy"))  # hello
```

这里最容易误解：

> `strip("xy")` 把参数当成字符集合，不是完整子串。

它会从左右两侧不断删除 `x` 或 `y`，直到遇到不属于这个集合的字符。

## 四、删除前后缀不要使用 `strip()`

错误示例：

```python
filename = "happy.py"

print(filename.rstrip(".py"))  # 可能得到 ha
```

`rstrip(".py")` 表示删除末尾所有属于 `.`、`p`、`y` 集合的字符，并不是删除完整后缀 `.py`。

正确写法：

```python
filename = "happy.py"

print(filename.removesuffix(".py"))  # happy
```

删除前缀：

```python
url = "https://example.com"

print(url.removeprefix("https://"))  # example.com
```

记忆方式：

```text
strip(chars)           删除首尾字符集合
removeprefix(prefix)   删除完整前缀
removesuffix(suffix)   删除完整后缀
```

## 五、字符串是不可变对象

调用字符串方法不会修改原字符串：

```python
text = "  hello  "

text.strip()

print(repr(text))  # '  hello  '
```

需要接收返回值：

```python
text = text.strip()

print(text)  # hello
```

也不能直接修改某个字符：

```python
text = "Python"

# text[0] = "J"  # TypeError
```

需要创建新字符串：

```python
text = "Python"
text = "J" + text[1:]

print(text)  # Jython
```

## 六、创建字符串

单引号和双引号没有本质区别：

```python
name = "Li Hao"
city = 'Hangzhou'
```

字符串中包含单引号时，可以在外侧使用双引号：

```python
message = "I'm learning Python"
```

多行字符串使用三个引号：

```python
message = """
第一行
第二行
第三行
"""
```

三个引号也经常用于文档字符串：

```python
def add(left: int, right: int) -> int:
    """计算两个整数的和。"""
    return left + right
```

## 七、转义字符和原始字符串

常见转义字符：

| 写法 | 含义 |
| --- | --- |
| `\n` | 换行 |
| `\t` | 制表符 |
| `\r` | 回车 |
| `\\` | 一个反斜杠 |
| `\"` | 双引号 |
| `\'` | 单引号 |

```python
text = "第一行\n第二行"

print(text)
```

原始字符串在前面添加 `r`，反斜杠通常不再作为转义符处理：

```python
path = r"C:\Users\demo"
pattern = r"\d+"

print(path)
print(pattern)
```

它常用于 Windows 路径和正则表达式。原始字符串不能以单个反斜杠结束。

## 八、长度、下标和切片

### 长度

```python
text = "Python"

print(len(text))  # 6
```

中文同样按 Unicode 字符计算：

```python
text = "你好"

print(len(text))                  # 2
print(len(text.encode("utf-8")))  # 6 个字节
```

### 下标

```python
text = "Python"

print(text[0])   # P
print(text[1])   # y
print(text[-1])  # n
```

下标从 0 开始，负数从末尾开始。下标越界会抛出 `IndexError`。

### 切片

```text
字符串[开始位置:结束位置:步长]
```

开始位置包含，结束位置不包含：

```python
text = "Python"

print(text[0:3])   # Pyt
print(text[:3])    # Pyt
print(text[3:])    # hon
print(text[-3:])   # hon
print(text[::2])   # Pto
print(text[::-1])  # nohtyP
```

切片范围超出字符串长度时通常不会报错。

## 九、字符串格式化

现代 Python 最常用 f-string：

```python
name = "Li Hao"
age = 18

message = f"姓名：{name}，年龄：{age}"

print(message)
```

可以直接计算表达式：

```python
left = 10
right = 20

print(f"结果：{left + right}")
```

常见格式：

```python
price = 12.34567
rate = 0.856
number = 7

print(f"{price:.2f}")  # 12.35
print(f"{rate:.1%}")   # 85.6%
print(f"{number:03d}") # 007
```

## 十、拼接和重复

少量字符串可以用 `+`：

```python
first_name = "Li"
last_name = "Hao"

full_name = first_name + " " + last_name
```

重复字符串使用 `*`：

```python
line = "-" * 20
```

连接多个字符串更适合 `join()`：

```python
languages = ["Python", "Java", "Go"]

result = ", ".join(languages)

print(result)  # Python, Java, Go
```

调用者是分隔符：

```text
"分隔符".join(字符串序列)
```

元素必须是字符串：

```python
numbers = [1, 2, 3]

result = ",".join(str(number) for number in numbers)

print(result)  # 1,2,3
```

## 十一、查找和判断包含

判断是否包含子串：

```python
text = "hello Python"

print("Python" in text)    # True
print("Java" not in text)  # True
```

查找位置：

```python
print(text.find("Python"))  # 6
print(text.find("Java"))    # -1
```

`index()` 找不到时会抛出 `ValueError`：

```python
print(text.index("Python"))  # 6
```

对照：

```text
find()    找不到返回 -1
index()   找不到抛出 ValueError
in        只判断是否包含，通常最直观
```

统计出现次数：

```python
text = "banana"

print(text.count("a"))  # 3
```

## 十二、判断开头和结尾

```python
filename = "report.pdf"

print(filename.startswith("report"))  # True
print(filename.endswith(".pdf"))      # True
```

判断多个后缀：

```python
filename = "photo.png"

if filename.endswith((".jpg", ".jpeg", ".png")):
    print("这是图片")
```

## 十三、替换字符串

```python
text = "I like Java"

result = text.replace("Java", "Python")

print(result)  # I like Python
```

限制替换次数：

```python
text = "a-a-a"

print(text.replace("a", "b", 2))  # b-b-a
```

`replace()` 同样返回新字符串，不会修改原字符串。

## 十四、大小写转换

```python
text = "hello PYTHON"

print(text.lower())       # hello python
print(text.upper())       # HELLO PYTHON
print(text.capitalize())  # Hello python
print(text.title())       # Hello Python
print(text.swapcase())    # HELLO python
```

不区分大小写比较时，可以使用 `casefold()`：

```python
left = "Python"
right = "PYTHON"

print(left.casefold() == right.casefold())  # True
```

## 十五、分割字符串

```python
text = "Python,Java,Go"

languages = text.split(",")

print(languages)
```

限制分割次数：

```python
text = "key=value=other"

print(text.split("=", 1))
# ['key', 'value=other']
```

不传参数时，按连续空白分割：

```python
text = "  Python   Java \n Go  "

print(text.split())
# ['Python', 'Java', 'Go']
```

从右侧分割：

```python
filename = "archive.tar.gz"

print(filename.rsplit(".", 1))
# ['archive.tar', 'gz']
```

按行分割：

```python
text = "第一行\n第二行\n第三行"

print(text.splitlines())
```

## 十六、内容判断方法

```python
print("123".isdigit())          # True
print("abc".isalpha())          # True
print("abc123".isalnum())       # True
print("   ".isspace())          # True
print("python".islower())       # True
print("PYTHON".isupper())       # True
print("hello world".istitle())  # False
```

需要注意：

```python
print("-123".isdigit())  # False
print("12.3".isdigit())  # False
```

负号和小数点都不是数字字符。判断能否转换成整数，更可靠的方式是实际转换：

```python
def is_integer(text: str) -> bool:
    try:
        int(text)
        return True
    except ValueError:
        return False
```

## 十七、对齐和填充

```python
text = "Python"

print(text.ljust(10, "-"))   # Python----
print(text.rjust(10, "-"))   # ----Python
print(text.center(10, "-"))  # --Python--
print("42".zfill(5))         # 00042
```

## 十八、`str` 和 `bytes`

Python 的 `str` 表示 Unicode 文本，`bytes` 表示二进制数据。

编码：

```python
text = "你好"
data = text.encode("utf-8")

print(data)
```

解码：

```python
decoded = data.decode("utf-8")

print(decoded)
```

关系：

```text
str ──encode──→ bytes
str ←─decode─── bytes
```

用户可读文本通常使用 `str`；文件协议、网络传输和加密接口经常处理 `bytes`。

## 十九、常见处理组合

### 清理并校验用户输入

```python
name = input("请输入姓名：").strip()

if not name:
    print("姓名不能为空")
```

用户只输入空格时，`strip()` 得到空字符串 `""`，而空字符串在布尔判断中为 `False`。

### 处理逗号分隔内容

```python
text = " Python, Java, Go "

languages = [
    item.strip()
    for item in text.split(",")
    if item.strip()
]

print(languages)
# ['Python', 'Java', 'Go']
```

执行过程：

```text
按逗号 split
      ↓
逐项 strip
      ↓
过滤清理后的空字符串
      ↓
得到干净列表
```

### 解析简单键值对

```python
text = "username=lihao"

key, value = text.split("=", 1)

print(key)
print(value)
```

限制只分割一次，可以让 value 中继续包含等号。

### 处理真实文件路径

真实路径和扩展名更推荐 `pathlib`：

```python
from pathlib import Path


path = Path("archive.tar.gz")

print(path.name)    # archive.tar.gz
print(path.stem)    # archive.tar
print(path.suffix)  # .gz
```

路径解析比手动 `split()` 或 `rstrip()` 更可靠。

## 二十、Java 开发者对照

| Java | Python |
| --- | --- |
| `text.trim()` / `strip()` | `text.strip()` |
| `text.substring(a, b)` | `text[a:b]` |
| `text.contains(x)` | `x in text` |
| `text.startsWith(x)` | `text.startswith(x)` |
| `text.endsWith(x)` | `text.endswith(x)` |
| `String.join(",", items)` | `",".join(items)` |
| `text.split(...)` | `text.split(...)`，但规则不完全相同 |
| `String.format()` | f-string |
| `StringBuilder` | 多字符串连接常用 `join()` 或列表收集后连接 |

Python 切片的结束下标同样不包含在结果中，这一点与 Java `substring(begin, end)` 相似。

## 最常用方法速查

| 需求 | 写法 |
| --- | --- |
| 清理首尾空白 | `text.strip()` |
| 清理行尾换行 | `text.rstrip("\n")` |
| 分割 | `text.split(",")` |
| 连接 | `",".join(items)` |
| 替换 | `text.replace(old, new)` |
| 判断包含 | `part in text` |
| 判断开头 | `text.startswith(prefix)` |
| 判断结尾 | `text.endswith(suffix)` |
| 删除完整前缀 | `text.removeprefix(prefix)` |
| 删除完整后缀 | `text.removesuffix(suffix)` |
| 格式化 | `f"name={name}"` |
| 获取长度 | `len(text)` |
| 反转 | `text[::-1]` |
| 按行分割 | `text.splitlines()` |

## 常见错误

### 把 `strip()` 当成子串删除

```python
"happy.py".rstrip(".py")
```

参数是字符集合。删除完整后缀应使用 `removesuffix()`。

### 忘记接收返回值

```python
text.strip()
```

不会修改 `text`，应该写成：

```python
text = text.strip()
```

### 把字节长度当成字符长度

```python
len(text)
```

得到字符数量；编码后的字节长度需要：

```python
len(text.encode("utf-8"))
```

### 用 `isdigit()` 判断所有数字格式

负数、小数通常不能通过 `isdigit()`，应尝试使用 `int()` 或 `float()` 转换并处理 `ValueError`。

### 用字符串方法手工解析路径

跨平台路径优先使用 `pathlib.Path`。

## 小练习

1. 将 `"  Python, Java, , Go \n"` 转换为不含空元素的语言列表。
2. 删除 `"https://example.com"` 的完整 `https://` 前缀。
3. 从 `"archive.tar.gz"` 中取得 `.gz`，再思考如何取得 `.tar.gz`。
4. 判断 `" -12 "` 清理后能否转换为整数。
5. 使用 f-string 输出价格，保留两位小数。
6. 比较 `rstrip(".py")` 和 `removesuffix(".py")` 对 `"happy.py"` 的不同结果。

## 总结

初学阶段优先掌握：

```text
strip
split
join
replace
startswith / endswith
removeprefix / removesuffix
in
切片
f-string
```

最后牢记：

```text
strip("abc") 删除的是首尾字符集合，不是完整子串
字符串是不可变对象，字符串方法通常返回新字符串
```
