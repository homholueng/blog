---
title: Chapter 4. 文本和字节序列
tags:
  - Python
categories:
  - Fluent Python
date: 2019-03-17 22:36:12
---


## 了解编码问题

### 如何找出字节序列的编码

如果我们不知道一个文件使用的是什么编码格式，可以使用统一字符编码侦测包 Chardet 来帮助我们判断文件的编码：

```shell
$ chardetect 04-text-byte.asciidoc
04-text-byte.asciidoc: utf-8 with confidence 0.99
```

## 处理文本文件

处理文本你文件的最佳实践是 “Unicode Sandwich”。即：要尽早的把输入（例如读取文本文件时）的字节序列解码成字符串，而三明治中的 “肉片” 是程序的业务逻辑，在这里只能处理字符串对象。最后，要尽可能晚的把字符串编码成字节序列。

{% asset_img unicode_sandwich.png [Unicode 三明治] %}

同时，在进行文本文件的读取和写入时，最好显式地指定编解码方式，不要依赖当前环境的默认值：

```python
>>> open('cafe.txt', 'w', encoding='utf-8').write('café')
>>> open('cafe.txt', encoding='utf-8').read()
café
```

## 为了正确比较二规范化 Unicode 字符串

因为 Unicode 有组合字符（变音符号和附加到前一个字符上的记号，打印时作为一个整体），所以字符串比较起来很复杂。

例如，`“café”` 这个词可以使用两种方式构成，分别有 4 个和 5 个码位，但是结果完全一样：

```python
In [26]: s1 = 'café'

In [27]: s2 = 'cafe\u0301'

In [28]: s1, s2
Out[28]: ('café', 'café')

In [29]: len(s1), len(s2)
Out[29]: (4, 5)

In [30]: s1 == s2
Out[30]: False
```

在 Unicode 标准中，`'é'` 和 `'e\u0301'` 这样的序列叫“标准等价物”（canonical equivalent），应用程序应该把它们视作相同的字符。但是，Python 看到的是不同的码位序列，因此判定二者不相等。

这个问题的解决方案是使用 `unicodedata.normalize` 函数提供的 Unicode 规范化。这个函数的第一个参数是这 4 个字符串中的一个：
- `'NFC'`：使用最少的码位构成等价的字符串。
- `'NFD'`：把组合字符分解成基字符和单独的组合字符。
- `'NFKC'`：使用最少的码位构成等价的字符串，对兼容字符有影响。
- `'NFKD'`：把组合字符分解成基字符和单独的组合字符，对兼容字符有影响。


```python
In [18]: from unicodedata import normalize

In [19]: s1 = 'café'

In [20]: s2 = 'cafe\u0301'

In [21]: len(s1), len(s2)
Out[21]: (4, 5)

In [22]: len(normalize('NFC', s1)), len(normalize('NFC', s2))
Out[22]: (4, 4)

In [23]: len(normalize('NFD', s1)), len(normalize('NFD', s2))
Out[23]: (5, 5)

In [24]: normalize('NFC', s1) == normalize('NFC', s2)
Out[24]: True

In [25]: normalize('NFD', s1) == normalize('NFD', s2)
Out[25]: True
```

西方键盘通常能输出组合字符，因此用户输入的文本默认是 NFC 形式。不过，安全起见，保存文本之前，最好使用 `normalize('NFC', user_text)` 清洗字符串。

### 兼容字符

虽然 Unicode 的目标是为各个字符提供“规范的”码位，但是为了兼容现有的标准，有些字符会出现多次。例如，虽然希腊字母表中有 `“μ”` 这个字母（码位是 U+03BC，GREEK SMALL LETTER MU），但是 Unicode 还是加入了微符号 `'µ'`（U+00B5），以便与 latin1 相互 转换。因此，微符号是一个“兼容字符”。

在 NFKC 和 NFKD 形式中，各个兼容字符会被替换成一个或多个“兼容分解”字符。二分之一 `'½'（U+00BD）`经过兼容分解后得到的是三个字符序列 `'1/2'`；微符号 `'µ'（U+00B5）`经过兼容分解后得到的是小写字母 `'μ'（U+03BC）`。

```python
In [31]: from unicodedata import normalize, name

In [32]: half = '½'

In [33]: normalize('NFKC', half)
Out[33]: '1⁄2'

In [34]: four_squared = '4²'

In [36]: normalize('NFKC', four_squared)
Out[36]: '42'
```

从上面的例子中能够看出，NFKC 或 NFKD 可能会损失或曲解信息，在使用这两种规范化形式时要格外注意。

### 大小写折叠

大小写折叠其实就是把所有文本变成小写，再做些其他转换。这个功能由 `str.casefold()` 方法（Python 3.3 新增）支持。

自 Python 3.4 起，`str.casefold()` 和 `str.lower()` 得到不同结果的有 116 个码位。Unicode 6.3 命名了 110 122 个字符，这只占 0.11%。

### 规范化文本匹配实用函数

下面两个工具函数能够帮助我们对 Unicode 字符进行比较：

```python
from unicodedata import normalize

def nfc_equal(str1, str2): 
    return normalize('NFC', str1) == normalize('NFC', str2)

def fold_equal(str1, str2): 
    return (normalize('NFC', str1).casefold() == normalize('NFC', str2).casefold())


In [38]: s1 = 'café'

In [39]: s2 = 'cafe\u0301'

In [40]: s1 == s2
Out[40]: False

In [41]: nfc_equal(s1, s2)
Out[41]: True

In [42]: nfc_equal('A', 'a')
Out[42]: False

In [43]: s3 = 'Straße'

In [44]: s4 = 'strasse'

In [46]: s3 == s4
Out[46]: False

In [47]: nfc_equal(s3, s4)
Out[47]: False

In [48]: fold_equal(s3, s4)
Out[48]: True
```

### 去除变音符号

```python
import unicodedata
import string

def shave_marks(txt):
    norm_txt = unicodedata.normalize('NFD', txt)
    shaved = ''.join(c for c in norm_txt if not unicodedata.combining(c))
    return unicodedata.normalize('NFC', shaved)


In [56]: order = '“Herr Voß: • ½ cup of OEtker™ caffè latte • bowl of açaí.”'

In [57]: shave_marks(order)
Out[57]: '“Herr Voß: • ½ cup of OEtker™ caffe latte • bowl of acai.”'
```

## Unicode 文本排序

在 Python 中，非 ASCII 文本的标准排序方式是使用 `locale.strxfrm` 函数，根据 locale 模块的文档 （https://docs.python.org/3/library/locale.html?highlight=strxfrm#locale.strxfrm)，这个函数会“把字符串转换成适合所在区域进行比较的形式”。

但是上述的排序方式需要更改当前环境下的全局区域设置，而且，还需要操作系统支持这一功能，所以，推荐使用 `PyUCA` 库来进行 Unicode 字符的排序：

```python
In [1]: import pyuca

In [2]: coll = pyuca.Collator()

In [3]: fruits = ['caju', 'atemoia', 'cajá', 'açaí', 'acerola']

In [4]: sorted_fruits = sorted(fruits, key=coll.sort_key)

In [5]: sorted_fruits
Out[5]: ['açaí', 'acerola', 'atemoia', 'cajá', 'caju']
```


