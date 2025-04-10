---
title: "C++ generator & ranges"
date: 2025-04-10
taxonomies:
  categories: ["cpp"]
---

## 背景

考虑以下需求：
- 需要对一个序列中，满足一定要求的前 N 个元素做某种操作
- 元素的筛选逻辑和具体操作是解耦的，操作者不知道有哪些元素、如何遍历、如何筛选

用 Python 实现的话，可以用 generator：

```python
def elems_generator():
    elems = list(range(10)) # all elements
    for elem in elems:      # how to iterate
        if elem % 2 == 0:   # filter logic
            yield elem

def process():
    N = 3                           # how many to process

    done = 0
    for elem in elems_generator():
        if elem % 3 == 0:           # may also need to filter
            print(f"{done} {elem}") # do something with the elem
            done += 1
        if done >= N:
            break
    print(f"{done=}")
```

c++ 中 naive 的写法是先将所有符合要求的元素 collect 成一个 vector 之类的再传给 process，但假设符合要求的元素很多，但内部只需要用到几个，这样做就显得有点 heavy。核心是想做到 lazy-evaluation。

由于 c++ 没有原生的 generator 语义，记录下收集到的实现方法。

## 1. Lambda Generator

```cpp
#include <iostream>
#include <vector>

template <typename Generator>
void process(Generator&& generator)
{
    constexpr int N = 3;

    int done = 0;
    int elem;
    while ((elem = generator()) != -1)
    {
        if (elem % 3 == 0)
        {
            std::cout << elem << std::endl;
            if (++done >= N)
            {
                break;
            }
        }
    }
}

int main()
{
    std::vector<int> elems = {0, 1, 2, 3, 4, 5, 6, 7, 8, 9};
    auto generator = [&elems, i = 0]() mutable -> int
    {
        while (i < elems.size())
        {
            const auto e = elems[i++];
            if (e % 2 == 0)
                return e;
        }
        return -1;
    };

    process(generator);
}
```

## 2. Custom Iter Action

```cpp
#include <iostream>
#include <vector>

enum class IterAction {
    Continue,
    Break,
};

template <typename ForEachElem>
void process(ForEachElem&& for_each_elem)
{
    constexpr int N = 3;

    int done = 0;
    for_each_elem([&](auto&& elem)
    {
        if (elem % 3 == 0)
        {
            std::cout << elem << std::endl;
            if (++done >= N)
            {
                return IterAction::Break;
            }
        }
        return IterAction::Continue;
    });
}

int main()
{
    std::vector<int> elems = {0, 1, 2, 3, 4, 5, 6, 7, 8, 9};
    const auto for_each_elem = [&elems](auto&& process_elem)
    {
        for (const auto& elem : elems)
        {
            if (IterAction::Break == process_elem(elem))
            {
                break;
            }
        }
    };

    process(for_each_elem);
}
```

## 3. Ranges

```cpp
#include <iostream>
#include <vector>
#include <ranges>

template <typename Range>
void process(Range&& range)
{
    constexpr int N = 3;

    for (auto&& elem : range 
        | std::views::filter([](auto&& elem) { return elem % 3 == 0; })
        | std::views::take(N))
    {
        std::cout << elem << std::endl;
    }
}

int main()
{
    std::vector<int> elems = {0, 1, 2, 3, 4, 5, 6, 7, 8, 9};
    process(elems | std::views::filter([](auto&& elem) { return elem % 2 == 0; }));
}
```
