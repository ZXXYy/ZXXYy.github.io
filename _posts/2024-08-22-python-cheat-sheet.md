---
title:      "Python Cheat Sheet"
date:       2024-08-22 09:00:00
author:     zxy
math: true
categories: ["Coding", "Python"]
tags: ["Python"]
post: true
---

这篇博客主要记录一下，在实际使用Python处理数据的过程中，会涉及到的一些常用Python代码。

### 1. 使用json读取/写入文件
主要涉及4个函数

- `json.load()`

  - 读取一个json文件，把文件中的内容转化成为python object (load from file)

  - ```python
    import json
    with open('data.json', 'r') as json_file:
    	python_object = json.load(json_file)
    ```

- `json.loads()`

  - 解析一个字符串，把字符串转化成为python object，可以解析dict, list, int, bool, float, null等

    loads = load string

  - ```python
    import json
    json_string = '{"name": "John", "age": 30, "city": "New York"}'
    python_dict = json.loads(json_string)
    ```

- `json.dump()`

  - 序列化python object到一个文件

  - ```python
    data = {"name": "Alice", "age": 30}
    with open('data.json', 'w') as file:
        json.dump(data, file, indent=4)
    ```

- `json.dumps()`

  - 序列化python object到json formatted string

  - ```python
    import json
    data = {"name": "Alice", "age": 30}
    json_string = json.dumps(data, indent=4)
    print(json_string)
    ```

使用`json.loads()`读取一个文件中的多个dict

```json
// data.json
{"name": "Alice", "age": 30}
{"name": "Bob", "age": 31}
{"name": "Cindy", "age": 33}
```

```python
with open('data.json', 'r') as file:
    for line in file:
        data = json.loads(line)
        print(data)
```

### 2.  `print` 保留小数，设置格式

```python
value = 12.34567
width = 10
decimal = 2
print(f"{value:<{width}.{decimal}f}")  # Left-aligns within 10 characters, 2 decimal places
```

- 左对齐:`{:<}` 

- 右对齐：`{:>}`
- 居中：`{:^}`
- 默认使用空格当占位符，` {:0<}`表示用0充当占位符

### 3. 正则表达式

#### 元字符

```
. ^ $ * + ? { } [ ] \ | ( )
```

有一些元字符不会使解析引擎在字符串中前进一个字符，他们不占用任何字符，只是表示字符串的位置信息，被称为*零宽度断言*

| 元字符 | 含义                                                         | 零宽度断言 | 例子                                         | 可匹配字符                            |
| ------ | ------------------------------------------------------------ | ---------- | -------------------------------------------- | ------------------------------------- |
| `.`    | 除\n外的所有字符                                             | 否         | -                                            | -                                     |
| `[ ]`  | 表示匹配的字符的一个集合                                     | 否         | `[abc]`                                      | 匹配 `a`、`b`、`c` 之中的任意一个字符 |
| `|`    | "or”运算符,优先级很低，一般和`[]`一起使用                    | 是         | `[Crow | Servo]`                             | 匹配字符串`Crow`/`Servo`              |
| `\`    | 后面可以跟各种字符来表示各种特殊序列;或者用于转义元字符      | -          |                                              |                                       |
|        | \d 等价于字符类 `[0-9]`                                      | 否         | -                                            | 匹配任何十进制数字                    |
|        | \D 等价于字符类 `[^0-9]`                                     | 否         | -                                            | 匹配任何非数字字符                    |
|        | \s 等价于字符类 `[ \t\n\r\f\v]`                              | 否         | -                                            | 匹配任何空白字符                      |
|        | \S 等价于字符类 `[^ \t\n\r\f\v]`                             | 否         | -                                            | 匹配任何非空白字符                    |
|        | \w 等价于字符类 `[a-zA-Z0-9_]`                               | 否         | -                                            | 匹配任何字母与数字字符                |
|        | \W 等价于字符类 `[^a-zA-Z0-9_]`                              | 否         | -                                            | 匹配任何非字母与数字字符              |
|        | \b 仅在单词的开头或结尾处匹配,当单词包含在另一个单词中时将不会匹配 | 是         | `re.search(r'\bclass\b', 'no class at all')` | 可以匹配到class                       |
|        |                                                              |            | `re.search(r'\bclass\b', 'one subclass is')` | 不可以匹配到class                     |

 表示固定点标记的元字符`^ $`

| 元字符 | 含义                                                 | 零宽度断言 | 例子                                          | 可匹配字符                                                   |
| ------ | ---------------------------------------------------- | ---------- | --------------------------------------------- | ------------------------------------------------------------ |
| `^`    | 和`[]`一起使用，表示取反，来匹配字符类中未列出的字符 | /          | `[^0-9]`                                      | 不是数字的单个字符                                           |
|        | 作为固定点标记，表示行/字符串的开头                  | 是         | `^[a-zA-Z][a-zA-Z0-9_]{4,15}$`                | 合法账号，长度在5-16个字符之间，只能用字母数字下划线，且第一个位置必须为字母 |
|        |                                                      |            | `re.search('^From', 'From Here to Eternity')` | 这个正则表达式可以匹配到From                                 |
|        |                                                      |            | `re.search('^From', 'Reciting From Memory')`  | 该字符串不以From开头，这个正则表达式匹配不到From             |
| `$`    | 作为固定点标记，表示行/字符串的结尾                  | 是         | `re.search('}$', '{block} '`                  | 匹配不到`}`（不在行/字符串的结尾）                           |
|        |                                                      |            | `re.search('}$', '{block}\n')`                | 可以匹配到`}`（在行的结尾）                                  |

和重复相关的元字符 `* + ? { }`

| 字符        | 含义                                                         | 例子                                 | 可匹配字符                              |
| ----------- | ------------------------------------------------------------ | ------------------------------------ | --------------------------------------- |
| `*`         | 表示匹配0次或多次，等价于`{0,}`                              | `ca*t`                               | `ct`, `cat`, `caat`...                  |
| `+`         | 表示匹配1次或多次，等价于`{1,}`                              | `ca+t`                               | `cat`, `caat`...                        |
| `?`         | 表示匹配0次或1次，等价于`{0,1}`                              | `home-?brew`                         | `homebrew`, `home-brew`                 |
|             | 与其他重复字符一起使用，表示lazy matching，匹配尽可能少的字符 | `re.search('<.*?>','<python>perl>')` | 匹配到`<python>`而不是`'<python>perl>'` |
| `{m,n}`     | 必须至少重复 *m* 次，至多重复 *n* 次                         | `a/{1,3}b`                           | `'a/b'`, `'a//b'` 和 `'a///b'`          |
| `{,n} {m,}` | 缺失 *m* 会解释为最少重复 0 次 ，缺失 *n* 则解释为最多重复无限次。 |                                      |                                         |
| `{m}`       | 与前一项完全匹配 *m* 次                                      | `a/{2}b`                             | `a//b`                                  |

组 group

| 符              | 含义                                                         | 例子                                      | 可匹配字符                                                   |
| --------------- | ------------------------------------------------------------ | ----------------------------------------- | ------------------------------------------------------------ |
| `()`            | 将所包含的表达式合为一组,并且可以使用限定符例如 `*`, `+`, `?`, 或 `{m,n}` 来重复一个分组的内容 | `(ab)*`                                   | 匹配 `ab` 的零次或多次重复。                                 |
| `(?P<name>...)` | 命名组，用于给分组命名                                       | `p=re.compile('(?P<first>\d)-(\d)-(\d)')` | `p.search('1-2-3').group('first')`<br />输出`1`              |
| `(?:...)`       | 表示匹配该模式，但不捕获该分组                               | `(?:\d+)`                                 |                                                              |
| `\1`            | 分组的反向引用                                               | `r'([ab])\1'`                             | 当分组`([ab])`内的`a`或`b`匹配成功后，将开始匹配`\1`，`\1`将匹配前面分组成功的字符。因此该正则表达式将匹配`aa`或`bb`。 |
| `(?P=Y)`        | Match the named group Y                                      |                                           |                                                              |

```python
# 带分组名称的反向引用
In [27]:
s = '12,56,89,123,56,98, 12'
p = re.compile(r'\b(?P<name>\d+)\b.*\b(?P=name)\b')
m = p.search(s)
m.group(1)

Out[27]:
'12'
```

#### 使用raw string来处理反斜杠灾难

- 正则表达式使用反斜杠字符 (`'\'`) 来表示特殊形式或允许使用特殊字符而不调用它们的特殊含义
- Python 在字符串文字中也使用(`\`)表示转义

由于这两个规则，如果要在正则中匹配一个`\section`就需要写很多反斜杠，这会导致大量重复的反斜杠，并使得生成的字符串难以理解。

| 字符            | 阶段                                                         |
| :-------------- | :----------------------------------------------------------- |
| `\section`      | 被匹配的字符串                                               |
| `\\section`     | 为 [`re.compile()`](https://docs.python.org/zh-cn/3/library/re.html#re.compile) 转义的反斜杠 |
| `"\\\\section"` | 为字符串字面转义的反斜杠                                     |

因此，需要使用 Python 的原始字符串表示法来表示正则表达式；反斜杠不以任何特殊的方式处理前缀为 `'r'` 的字符串

| 正则    | 含义                                   |
| ------- | -------------------------------------- |
| `r"\n"` | 一个包含 `'\'` 和 `'n'` 的双字符字符串 |
| `"\\n"` | 一个包含 `'\'` 和 `'n'` 的双字符字符串 |
| `"\n"`  | 一个包含换行符的单字符字符             |

`"\\w+\\s+\\1"`和`r"\w+\s+\1"`是等价的，也就是说加了raw string后只需要考虑正则表达式内部的转义就可以了

#### 正则表达式常用函数

| 方法 / 属性  | 目的                                                         | 返回值                                                       |
| :----------- | :----------------------------------------------------------- | ------------------------------------------------------------ |
| `match()`    | 确定正则是否从字符串的开头匹配。                             | `None`/一个 [匹配对象](https://docs.python.org/zh-cn/3/library/re.html#match-objects) 实例将被返回，包含匹配相关的信息：起始和终结位置、匹配的子串以及其它。 |
| `search()`   | 扫描字符串，查找此正则匹配的任何位置，找到第一个匹配         | `None`/一个 [匹配对象](https://docs.python.org/zh-cn/3/library/re.html#match-objects) 实例将被返回，包含匹配相关的信息：起始和终结位置、匹配的子串以及其它。 |
| `findall()`  | 找到正则匹配的所有子字符串，并将它们作为列表返回。           | findall()只取得所有匹配字符串，返回包含所有匹配字符串的列表，不关心匹配字符串在原字符串中的各项信息。 |
| `finditer()` | 找到正则匹配的所有子字符串，并将它们返回为一个 [iterator](https://docs.python.org/zh-cn/3/glossary.html#term-iterator)。 |                                                              |
| `split()`    | 用正则表达式切分字符串，返回一个list                         |                                                              |

检查匹配对象实例也有几个方法和属性

| 方法 / 属性 | 目的                                 |
| :---------- | :----------------------------------- |
| `group()`   | 返回正则匹配的字符串                 |
| `start()`   | 返回匹配的开始位置                   |
| `end()`     | 返回匹配的结束位置                   |
| `span()`    | 返回包含匹配 (start, end) 位置的元组 |
| `groups()`  | 用groups()函数取出匹配的所有分组     |

由于 [`match()`](https://docs.python.org/zh-cn/3/library/re.html#re.Pattern.match) 方法只检查正则是否在字符串的开头匹配，所以 `start()` 将始终为零。 但是，模式的 [`search()`](https://docs.python.org/zh-cn/3/library/re.html#re.Pattern.search) 方法会扫描字符串，因此在这种情况下匹配可能不会从零开始。

```python
import re
m = re.search(r'\d{3,4}-?\d{8}', '010-66677788,02166697788, 0451-22882828')
print(m.group()) # [Output]: '010-66677788', search总是返回第一个成功匹配
print(m.span())  # [Output]: (0, 12)

ms = re.finditer(r'\d{3,4}-?\d{8}', '010-66677788,02166697788, 0451-22882828')
for m in ms:
    print(m.group()) 
# Output:
# 010-66677788
# 02166697788
# 0451-22882828

p=re.compile('(\d)-(\d)-(\d)')
print(p.search('1-2-3').group())  # [Output]:'1-2-3'
print(p.search('1-2-3').groups()) # [Output]: ('1', '2', '3')

p=re.compile('(?P<first>\d)-(\d)-(\d)')
print(p.search('1-2-3').group('first')) # [Output]: '1'

words = re.split(r'[,-]', '010-66677788,02166697788,0451-22882828')
print(words) # [Output]: ['010', '66677788', '02166697788', '0451', '22882828']

```

以下两种使用方式只是在性能上有一些区别，功能上没有区别。`re.compile`会事先编译好正则表达式，在多次循环中访问这个正则时，在性能上会更好

- `p = re.compile(<regular expression>)` +  `p.search('<target_string>')`

- `re.search('<regular expression>', '<target_string>')`

### 4. `argparse`

`argparse`中有两种类型的参数，都通过`add_argument()`这个函数添加

- positional arguments： 在命令行中的相对位置决定了这个参数的用途， `path`
- optional arguments： options, flags, or switches，不是必要的，`long` 

最基本的用法：

```python
import argparse
parser = argparse.ArgumentParser()

parser.add_argument("path")
parser.add_argument("-l", "--long", action="store_true"）
                    
args = parse.parse_args()
```

---

通过添加`add_argument_group`的方式来让help的输出更加好看，对CLI的输入方式不产生影响，**注意这个不是subcommand**：

```python
parser = argparse.ArgumentParser(
    prog="ls",
    description="List the content of a directory",
    epilog="Thanks for using %(prog)s! :)",
)

general = parser.add_argument_group("general output")
general.add_argument("path")

detailed = parser.add_argument_group("detailed output")
detailed.add_argument("-l", "--long", action="store_true")

args = parser.parse_args()
```

---

`argparse`有辅助缩写的功能，当option的名字很长的时候，可以直接用缩写，默认开启，可以使用`argparse.ArgumentParser(allow_abbrev=False)`在初始化时关闭

```python
parser.add_argument("--argument-with-a-long-name")
```

```shell
# 以下三条指令是等价的
$ python abbreviate.py --argument-with-a-long-name 42
$ python abbreviate.py --argument 42
$ python abbreviate.py --a 42
```

---

`add_argument()`这个函数中对option设置不同的`action` 参数

```python
import argparse
parser = argparse.ArgumentParser()
parser.add_argument(
    "--name", action="store"
)  # Equivalent to parser.add_argument("--name")
parser.add_argument("--is-valid", action="store_true")
parser.add_argument("--is-invalid", action="store_false")
parser.add_argument("--item", action="append")
parser.add_argument(
    "--version", action="version", version="%(prog)s 0.1.0"
)
```

```shell
python actions.py `
   --name Python `
   --pi `
   --is-valid `
   --is-invalid `
   --item 1 --item 2 --item 3 
---------------------------------
Namespace(
    name='Python',
    is_valid=True,
    is_invalid=False,
    item=['1', '2', '3']
)
```

---

一个参数有多个输入使用`nargs`，对optional 和positional arguments都适用

```python
import argparse
parser = argparse.ArgumentParser()
parser.add_argument("files", nargs="+")
args = parser.parse_args()
print(args)
```

```shell
$ python files.py hello.txt realpython.md README.md
# files=['hello.txt', 'realpython.md', 'README.md']
```

---

设置参数的指定输入

```python
parser.add_argument("--size", choices=["S", "M", "L", "XL"], default="M")
```

---

设置互斥的组，`-v`和`-s`不能在命令行中同时存在

```python
parser = argparse.ArgumentParser()
group = parser.add_mutually_exclusive_group(required=True)
group.add_argument("-v", "--verbose", action="store_true")
group.add_argument("-s", "--silent", action="store_true")
```

---

使用`add_subparser()`来添加子命令**subcommand**

```python
global_parser = argparse.ArgumentParser(prog="calc")
subparsers = global_parser.add_subparsers(
    title="subcommands", help="arithmetic operations"
)

arg_template = {
    "dest": "operands",
    "type": float,
    "nargs": 2,
    "metavar": "OPERAND",
    "help": "a numeric value",
}

add_parser = subparsers.add_parser("add", help="add two numbers a and b")
add_parser.add_argument(**arg_template)
add_parser.set_defaults(func=add)

sub_parser = subparsers.add_parser("sub", help="subtract two numbers a and b")
sub_parser.add_argument(**arg_template)
sub_parser.set_defaults(func=sub)

mul_parser = subparsers.add_parser("mul", help="multiply two numbers a and b")
mul_parser.add_argument(**arg_template)
mul_parser.set_defaults(func=mul)

div_parser = subparsers.add_parser("div", help="divide two numbers a and b")
div_parser.add_argument(**arg_template)
div_parser.set_defaults(func=div)

args = global_parser.parse_args()

print(args.func(*args.operands))
```

### 5. Unit test in Python

```python
# project/code/my_calculations.py
class Calculations:
    def __init__(self, a, b):
        self.a = a
        self.b = b

    def get_sum(self):
        return self.a + self.b

    def get_difference(self):
        return self.a - self.b

```

```python
# project/test.py
import unittest
from code.my_calculations import Calculations

class TestCalculations(unittest.TestCase):
		# 需要测试的case需要以test开头，让unittest框架识别
    def test_sum(self):
        calculation = Calculations(8, 2)
        self.assertEqual(calculation.get_sum(), 10, 'The sum is wrong.')

if __name__ == '__main__':
    unittest.main()
```

```shell
python -m unittest
python -m unittest -v project.test # 指定单元测试的文件
```



### Reference

1. [python正则表达式快速基础教程](https://notebook.community/liupengyuan/python_tutorial/chapter3/python%E6%AD%A3%E5%88%99%E8%A1%A8%E8%BE%BE%E5%BC%8F)
2. [正则表达式](https://pythonhowto.readthedocs.io/zh-cn/latest/regular.html)

3. [正则表达式指南](https://docs.python.org/zh-cn/3/howto/regex.html)
4. [real python argparse](https://realpython.com/command-line-interfaces-python-argparse/#creating-command-line-interfaces-with-pythons-argparse)
5. [real python unitest](https://realpython.com/python-unittest/)
6. [A Beginner’s Guide to Unit Tests in Python (2023)](https://www.dataquest.io/blog/unit-tests-python/)
