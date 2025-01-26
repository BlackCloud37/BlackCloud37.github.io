---
title: "python gc 与循环引用"
date: 2024-09-11
taxonomies:
  categories: ["python"]
  tags: ["bug"]
---

在优化日志落盘的任务里，为了避免消费日志队列过慢导致爆队列，其中一个优化是快速消费队列并全部缓存到 deque 里，再慢慢落盘

但测试时发现，虽然理论上 `deque` 的 `append` 是 O(1) 的，但实际表现是 append 时不时会变慢(体现为消费队列并 `append` 变慢导致队列爆掉)，且随着 `deque` 变大，对应的延迟会变高

这个问题表现类似 [is there a way to circumvent python list append becoming progressively slower](https://stackoverflow.com/questions/2473783/is-there-a-way-to-circumvent-python-list-append-becoming-progressively-slower)，所以猜测可能也是 `gc` 导致的，禁用 `gc` 后确实不会变慢了，但随后发现引入了内存泄露，即虽然 `deque` 里缓存的元素被消费完了，但进程内存用量不会下降

由于 `gc` 只被用于回收有循环引用的对象，所以猜测是项目里的某些对象有循环引用，定位到了如下代码：

```python
class Foo:
    @property
    def bar(self):
        try:
            return self._bar
        except AttributeError:
            self._bar = self.c_bar.contents
            return self._bar
```

其中 `c_bar` 是一个 `ctypes.POINTER`，在命中 `AttributeError` 时的逻辑会使 `self._bar` 引用 `self.c_bar`，产生循环引用，改掉这里后内存泄露消失
