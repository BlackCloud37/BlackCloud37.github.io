---
title: "c++ pointer to member"
date: 2025-01-02
taxonomies:
  categories: ["programming-language"]
  tags: ["cpp"]
---

## 背景
在做侵入式容器时，容器通常需要访问对象的成员。例如，侵入式链表需要将链表节点嵌入到对象中，这时用 pointer to member 来实现，可以避免硬编码成员名：

```c++
struct Node {
    Node* next;
    Node* prev;
};

struct Foo {
    Node node1;
    Node node2;
};

template <auto PtrToMember>
class List {
// access via
// Node node = foo->*PtrToMember;
};

List<&Foo::node1> list1;
List<&Foo::node2> list2;
```

## 推导 Pointer to *Who's* Member

由于容器往往需要知道储存对象的类型，例如上面的 Foo，最简单的做法是也放到模板参数里

```c++
template <auto PtrToMember, typename T>
class List {};

List<&Foo::node1, Foo> list1;
```

但既然 `PtrToMember` 已经包含了 `Foo`，其实可以自动推导出 `T`，参考 [Inferring type and class when passing a pointer to data member as a non-type template argument](https://stackoverflow.com/questions/65375734/inferring-type-and-class-when-passing-a-pointer-to-data-member-as-a-non-type-tem)

```c++
template <typename T> struct type_from_member;

template <typename Cls,typename M>
struct type_from_member<M Cls::*> {
    using class_type = Cls;
    using member_type = M;
};

template <auto PtrToMember, typename T = type_from_member<decltype(PtrToMember)>::class_type>
class List {
};

List<&Foo::node1 /* T=Foo */> list1;
```

可以看到，上述模板特化中，`PtrToMember` 的类型是 `M Cls::*`，即 `Node Foo::*`。`Node Foo::*` 是一个明确的 `typename`：
```c++
Node Foo::* ptr = &Foo::node1;

Foo foo;
Node& node = foo.*ptr;
```

完整的定义可以参考 [Pointer declaration](https://en.cppreference.com/w/cpp/language/pointer) 的 Pointers to members 一节

## 继承

在实践中，`Foo` 可能会涉及继承，此时会发现自动推导失效了：

```c++
struct Base {
    Node node;
};

struct Derived : Base {};

List<&Derived::node /* T=Base */> list; /* 此处推导的结果是 T=Base, 而非期望的 Derived,
                                         * 即使我们写了 &Derived::node */
```

原因还是 `&Derived::node` 的类型是 `Node Base::*`，而不是 `Node Derived::*`
```c++
static_assert(std::is_same_v<decltype(&Derived::node), Node Base::*>); // Pass
// static_assert(std::is_same_v<decltype(&Derived::node), Node Derived::*>); // Fail
```

另外，基类的 `PtrToMember` 可以隐式转换成派生类的 `PtrToMember`
> Pointer to data member of an accessible unambiguous non-virtual base class can be implicitly converted to pointer to the same data member of a derived class
```c++
Node Base::* ptr = &Base::node;
Node Derived::* ptr2 = ptr; // OK
```

反向的转换则需要 `static_cast`，且如果用基类没有该 member，但试图用这个 ptr 访问基类的 member，是 undefined behavior
```c++
struct Base {};
struct Derived : Base { Node node; };
Node Derived::* derived_ptr = &Derived::node;
Node Base::* base_ptr = static_cast<Node Base::*>(derived_ptr); // OK

Derived derived;
derived.*derived_ptr; // OK

Base base;
base.*base_ptr; // UB
```
