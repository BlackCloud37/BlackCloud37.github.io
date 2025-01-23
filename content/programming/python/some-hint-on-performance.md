---
title: Some notes on python performance
date: 2025-01-22
taxonomy:
    categories:
        - python
        - tuning
draft: true
---

记录一下这两天写 python 时遇到的一些性能问题和解决。大部分不能算优化，只是 latency numbers you should know。大部分测试基于 python 3.8。如果有错误还请发邮件指正。

### isinstance 和 type is

如果希望检查实例是否 exactly 是某个类型 A，而不包括 A 的子类，用 `type(a) is A`。

如果希望检查实例是否是 A 或者 A 的子类，用 `isinstance(a, A)`。

除了语义上的区别，二者的性能也不同，如果检查的类型*继承链比较复杂*，且结果为 False 时，`isinstance` 的性能会差很多，否则差不多。
- 在我的 case 中，被检查的类型从 `pydantic.BaseModel` 派生多级。`isinstance` 为 False 时，比 `type is` 慢 ~10 倍。
    ```python
    from pydantic import BaseModel
    import timeit

    class A(BaseModel):
        a: int
        pass

    class B(A):
        b: int
        pass

    class C1(B):
        c: int
        pass

    class C2(B):
        c: int
        pass

    class C3(B):
        c: int
        pass

    c1 = C1(a=1, b=2, c=3)
    c2 = C2(a=1, b=2, c=3)
    print("isinstance true ", \
        timeit.timeit(lambda: isinstance(c1, C1), number=1000000))
    print("type is    true ", \
        timeit.timeit(lambda: type(c1) is C1, number=1000000))
    print("isinstance false", \
        timeit.timeit(lambda: isinstance(c1, C2), number=1000000))
    print("type is    false", \
        timeit.timeit(lambda: type(c1) is C2, number=1000000))    
    ```

    结果是
    ```
    isinstance true  0.05928762990515679
    type is    true  0.08189487305935472
    isinstance false 0.6317731359740719
    type is    false 0.08188665006309748
    ```
