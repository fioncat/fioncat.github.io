---
layout:       post
title:        "Python list作为全局属性问题"
author:       "Fioncat"
header-style: text
catalog:      true
tags:
    - Python
---

以下两种写法：

```python
class Numbers:
    nums = []
class Numbers:
    nums = None
    def __init__(self):
        self.nums = []
```

看上去是一样的，实际上是不一样的。

注意，python中的list是一种对象，它是一种实体。具名变量`nums`仅仅是指向它的一个引用。

在写法1中，`nums`指向的`list`仅在全局被初始化了一次。随后在创建`Numbers`对象时，会为每个对象创建一个新的`nums`引用指向这个全局`list`。

在写法2中，每次创建一个新的对象，都会创建一个新的`list`实体，并使用`nums`指向它。

总结一下：

- 写法1中，每个`Numbers`对象的`nums`引用指向同一个`list`。使用某个对象的`nums`来改变列表，会影响到其它对象。说得不准确点，就是所有`Numbers`对象共享一个`nums`列表。
- 写法2中，每个`Numbers`对象持有自己的`nums`引用和自己的`list`，这些`list`实体之间是没有关系的，互不影响的。

执行以下代码：

```python
nums1 = Numbers()
nums1.nums.append(1)
nums1.nums.append(2)

nums2 = Numbers()
nums2.nums.append(3)
nums2.nums.append(4)

print(f'{nums1.nums}')
print(f'{nums2.nums}')
```

写法1将会输出：

```text
[1, 2, 3, 4]
[1, 2, 3, 4]
```

写法2输出：

```text
[1, 2]
[3, 4]
```

一定一定要记住，`list`是对象，所有对`list`的赋值都是用一个新的引用去引用它，并不会增加新的`list`实体。只有调用`list`的`copy()`方法，才会创建一个新的`list`实体。