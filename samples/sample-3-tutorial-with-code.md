# 用生成器处理大文件

## 为什么要用生成器

读取一个 10GB 的日志文件时，`open(f).readlines()` 会把整个文件读进内存，直接把内存打满。生成器（generator）的做法是每次只产出一条数据，读一条处理一条，内存占用始终等于一条记录的大小。

生成器和普通函数的区别在于 `yield`。函数执行到 `yield` 时会把值交出去并暂停在原地，下一次调用再从暂停处继续。这意味着函数的局部变量状态被保留，但函数栈不会被反复重建。

## 基本写法

```python
def read_lines(path):
    with open(path, encoding="utf-8") as f:
        for line in f:
            yield line.strip()

for line in read_lines("access.log"):
    handle(line)
```

注意这里没有调用 `list()`。一旦包一层 `list(read_lines(...))`，生成器会被一次性求值，省内存的效果完全消失，这是最常见的误用。

## 生成器表达式

不想专门写一个函数时，用圆括号的生成器表达式：

```python
total = sum(int(x) for x in read_lines("nums.txt"))
```

它等价于 Generator 版本，但不用 `def`。两者的取舍是：逻辑超过一行就写成函数，一行以内用表达式，否则可读性会迅速变差。

## 管道式串联

生成器可以像管道一样串起来，每一级都是惰性的：

```python
lines = read_lines("access.log")
errors = (l for l in lines if "ERROR" in l)
ips = (l.split()[0] for l in errors)
for ip in ips:
    count(ip)
```

这段代码在 `for` 循环真正执行前一行都没读。三个生成器串接时，每一步过滤都在上一步产出单个元素时立刻发生，不需要中间列表。

## 两个坑

第一，生成器是一次性的。迭代完之后再 iterate 同一个生成器对象，什么都得不到。需要多次遍历时，要么重新调用生成函数，要么用 `itertools.tee()`。

第二，`yield` 写在有装饰器或异常捕获的复杂函数里时，暂停点会让调试变得困难。遇到这种情况宁可拆成两个简单生成器，也不要在一个 200 行的函数里塞五个 `yield`。
