---
title: "Fraked by mutable default values"
date: 2025-01-21
taxonomies:
  categories: ["python"]
  tags: ["bug", "dataclass"]
---

**[Mutable default values](https://docs.python.org/3/library/dataclasses.html#mutable-default-values)**

在写一个 python server 时，因为需要定义一个 `Connection` 类，用于保存一个连接的 `socket` 和 `recvbuf`，于是写出了如下代码：

```python
import socket
from dataclasses import dataclass

@dataclass
class Connection:
    socket: socket.socket
    recvbuf: bytearray = bytearray()
```

之后就遇到了一些难以定位的 bug，比如来自 Client A 的连接上好像收到了本应来自 Client B 的数据。

发现是 default member value 会被保存在 class attributes 里，recvbuf 被所有 Connection 实例共享。

一般写非 dataclass 的代码时，默认值都会在 `__init__` 里设置，写 dataclass 时习惯了这种在 class attribute 里设默认值的方式，才遇到这个问题。

正确写法是：

```python
from dataclasses import field, dataclass

@dataclass
class Connection:
    socket: socket.socket
    recvbuf: bytearray = field(default_factory=bytearray)
```

{{ hr() }}

dataclass 其实会检查这种 case，即默认值是否是 mutable 的，比如[文档](https://docs.python.org/3/library/dataclasses.html#mutable-default-values)指出：

```python
@dataclass
class D:
    x: list = []      # This code raises ValueError
```

为什么 bytearray 就不会报错呢。

测试发现，我本地的 python 3.13.1 对 bytearray 和 list 均会报错
```
ValueError: mutable default <class 'bytearray'> for field x is not allowed: use default_factory
```

但服务器上的 python 3.8 只会对 list 报错

去看 dataclass 的实现，3.8 的逻辑是：
```python
# For real fields, disallow mutable defaults for known types.
if f._field_type is _FIELD and isinstance(f.default, (list, dict, set)):
    raise ValueError(f'mutable default {type(f.default)} for field '
                        f'{f.name} is not allowed: use default_factory')
```

3.13
```python
# For real fields, disallow mutable defaults.  Use unhashable as a proxy
# indicator for mutability.  Read the __hash__ attribute from the class,
# not the instance.
if f._field_type is _FIELD and f.default.__class__.__hash__ is None:
    raise ValueError(f'mutable default {type(f.default)} for field '
                        f'{f.name} is not allowed: use default_factory')
```

新版本的检查更充分一些
