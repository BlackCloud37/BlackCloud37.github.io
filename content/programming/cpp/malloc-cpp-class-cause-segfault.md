---
title: "malloc c++ class 而不调用构造函数引发的 segfault"
date: 2023-01-29
taxonomies:
  categories: ["cpp"]
---

## 背景
原本有一个纯 C 的库，其某结构体内保存了一个函数指针作为 callback
```c
struct Foo {
    int (*output)(...);
};
```

为了在 C++ 中方便地使用 lambda 等，我自作主张地将其改成了
```c++
struct Foo {
    std::function<int(...)> output;
};
```

起初工作得很好，`std::function` 在替代函数指针方面非常方便。直到将代码放到另一套环境中时发现会随机 segfault，gdb 调试定位到了该回调相关的部分。一定是遇到了 Undefined Behavior 了

## 排查
出现问题的代码片段位置不尽相同，但是总是出现在 **给该结构体的 output 成员赋值** 上，如
```c++
// when creating a empty Foo instance, give it's member a default value
foo->output = std::function<int(...)>();
```

segfault 的 backtrace 最终总是
```
#0  0x0000000000404ce4 in std::_Function_base::~_Function_base (this=0x7fffffffc3c0, __in_chrg=<optimized out>)
#1  0x000000000040ca7e in std::function<int (...)>::~function()
#2  0x00000000004180df in std::function<int(...)>::operator=<::<lambda(...)> >::<lambda(...)> >(struct {...} &&) 
```

即调用 `std::function` 的 `operator=` 赋值时，调用了某个 `std::function` 的析构函数，然后析构函数触发了 segfault。根据赋值的位置，应当是将新的 output 赋值到成员变量时，原本成员变量的 output 需要析构

然后想到因为是纯 C 的库，申请 `Foo` 的位置均使用 `malloc`，`malloc` 默认不会构造结构体及其 member 的构造函数，因此得到的 `foo->output` 的位置上也是未初始化的、随意一段内存，却被当成了一个合法的 `std::function`。在这段随意的内存上试图调用 `std::function` 的函数则成为了 UB，导致了 segfault

因此解决方式也很简单，在 `malloc` 之后手动 placement new 一下结构体（或者是单独初始化 output）即可
```c++
Foo* foo = (Foo*)malloc(sizeof(Foo));
new (&(foo->output)) std::function<int(...)>();
```
