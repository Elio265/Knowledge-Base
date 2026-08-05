---
tags: [python, basics, interview]
created: 2026-08-04
updated: 2026-08-04
status: 学习中
---

# Python 语言基础

## 一句话理解

没系统学过不等于不能上：后端面试只挑高频考点考，目标不是语法大全，而是「常用语法能写对、经典坑能讲清、项目里的 Python 能接住追问」。

## 基础学习路线

按顺序过一遍，每项能独立写出小例子：

1. 语法与数据类型：变量、数字/字符串/列表/字典/集合/元组，切片、推导式、解包。
2. 控制流与函数：条件/循环、函数参数（位置、关键字、默认值、`*args/**kwargs`）、作用域与 `global/nonlocal`。
3. 异常处理：`try/except/finally`、异常链、自定义异常。
4. 迭代器与生成器：`__iter__/__next__`、`yield`、`yield from`。
5. 装饰器：函数装饰器、带参装饰器、`functools.wraps`。
6. 类与面向对象：`__init__/__new__`、`__slots__`、属性、类方法/静态方法、`super()`、MRO。
7. 上下文管理器：`with`、`__enter__/__exit__`、`contextlib.contextmanager`。
8. 常用标准库：`os`、`sys`、`pathlib`、`json`、`re`、`datetime`、`logging`、`collections`、`typing`。
9. 工程基础：`venv`/`pip`、`requirements.txt`/`pyproject.toml`、模块与包、`if __name__ == "__main__"`。

## 高频面试考点（必须会）

| 考点 | 一句话答案 |
|---|---|
| 可变默认参数 | `def f(x, lst=[])` 的默认列表跨调用共享，应改为 `None` 再初始化 |
| 浅拷贝 vs 深拷贝 | `copy.copy` 复制外层、内层仍共享；`copy.deepcopy` 递归复制全部可变对象；`list[:]` 是浅拷贝 |
| `==` vs `is` | `==` 比较值，`is` 比较对象身份；CPython 缓存小整数 -5~256，短字符串编译期常量会驻留 |
| 装饰器本质 | 接受函数返回新函数的语法糖，`@wraps` 保留原函数元信息 |
| 生成器 vs 列表 | 惰性求值、节省内存；只能迭代一次 |
| `with` 的作用 | 保证 `__exit__` 一定执行，用于资源释放 |
| finally 语义 | 无论 `return` 还是异常被捕获，`finally` 必执行；只有 `os._exit()`/进程被杀才会跳过 |
| 类属性 vs 实例属性 | 类体里定义的可变对象所有实例共享；`self.x = []` 才是实例私有 |
| 三种方法 | 第一参数 `self`/`cls`/无，分别对应实例方法/类方法/静态方法 |
| 深拷贝的坑 | 循环引用、不可拷贝对象、性能开销 |
| 推导式作用域 | Python 3 推导式有独立作用域，不污染外层变量 |

## 类与面向对象（2026-08-05 首学）

### 类属性 vs 实例属性

```python
class Dog:
    tricks = []        # 类属性：定义时创建一次，所有实例共享

d1 = Dog()
d2 = Dog()
d1.tricks.append("roll over")
print(d2.tricks)       # ['roll over'] —— 共享同一个列表
```

改成 `self.tricks = []`（在 `__init__` 里）后，每个实例创建时各自 new 一个列表，互不影响。这和「可变默认参数」是同一个根源：**定义在类体/默认参数里的可变对象，生命周期只有一份**。

### 三种方法

| 方法 | 第一参数 | 能访问什么 | 典型场景 |
|---|---|---|---|
| 实例方法 | `self` | 实例属性 + 类属性 | 绝大多数业务方法 |
| `@classmethod` | `cls` | 类属性、可改类状态 | 工厂方法、替代构造（`cls.from_dict(data)`） |
| `@staticmethod` | 无 | 都不访问 | 逻辑上属于类、但不需要类/实例状态的工具函数 |

记忆：看第一参数——`self` 是实例方法，`cls` 是类方法，没有就是静态方法。

类方法内访问类属性必须写 `cls.count`：类体不是方法的作用域，裸写 `count` 会 `NameError`，必须通过 `cls`（或类名）取。

### `__new__` 与 `__init__`

- `__new__(cls, ...)` 先执行，负责**创建**实例（返回实例对象），是隐式静态方法。
- `__init__(self, ...)` 后执行，负责**初始化**（给实例填属性）。
- 正常写业务代码不碰 `__new__`；需要用的场景：单例、继承不可变类型（`int`/`str`/`tuple` 子类）、实例缓存/对象池。
- 一句话：**`__new__` 造对象，`__init__` 填属性。**

## 上下文管理器（2026-08-05 首学）

核心一句话：**`with` 保证「进入时准备、离开时清理」，无论中间是否异常都必清理。**

```python
class File:
    def __enter__(self):
        print("enter")
        return self
    def __exit__(self, exc_type, exc_val, exc_tb):
        print("exit")

with File() as f:
    print("body")
# 输出顺序：enter -> body -> exit
```

- `__enter__` 在进入 with 时执行，返回值赋给 `as f`；`__exit__` 在离开时执行（正常结束或抛异常都会）。
- body 抛异常时 `__exit__` 仍会执行（和 finally 同源）；三个参数是 `exc_type` / `exc_val` / `exc_tb`，没有异常时都是 None。
- `@contextmanager` 把生成器变成上下文管理器：`yield` 之前是「进入时」的代码，`yield` 之后是「离开时」的代码；body 的异常会在 `yield` 处抛出，所以标准写法是 `try: yield finally: 收尾`。
- `__exit__` 返回 `True` 表示「异常已处理」，会吞掉异常不再抛；返回 `False`/`None` 则继续传播。默认不要返回 `True`。
- 适用场景：文件、锁、数据库连接/事务、socket——一切需要保证释放资源的场景。

`with my_ctx():` 的完整执行过程（和生成器同一套暂停/恢复机制）：

```python
@contextmanager
def safe_ctx():
    print("enter")    # ① with 开始时执行
    try:
        yield         # ② 暂停点：body 在这里执行，结束后从这里继续
    finally:
        print("exit") # ③ with 结束时执行（异常也保证执行）

with safe_ctx():
    print("body")     # 这段 body 在 yield 暂停期间运行
```

1. `with` 调用 `safe_ctx()` 得到生成器（函数体还没跑）。
2. 进入时驱动生成器跑到 `yield`：打印 enter，然后暂停。
3. body 在暂停期间执行。
4. 离开时恢复生成器：执行 `yield` 后面的代码；body 的异常也会在 `yield` 处抛出来，被 `finally` 接住。

## 常用标准库与工程基础（2026-08-05 首学）

### `if __name__ == "__main__":`

直接运行脚本时（`python script.py`），Python 把 `__name__` 设为 `"__main__"`；被别的模块 import 时，`__name__` 是模块名。所以这行守卫的意思是：**只有直接运行才执行，被 import 时不执行**，避免导入模块时触发副作用（启动服务、跑 main、打印）。

### venv 与 requirements.txt

- `venv`：每个项目独立的 Python 环境，自带 site-packages，项目间依赖互不污染，也不动系统 Python。
- `requirements.txt`：锁定的依赖清单（如 `fastapi==0.115.0`），`pip install -r requirements.txt` 即可复现环境，CI/部署都用它。
- `pyproject.toml`：现代的项目元数据与构建规范（PEP 621），比 requirements.txt 更完整。

### logging vs print

`print` 只能往 stdout 输出，没有级别、时间戳、过滤和落盘能力。`logging` 提供：

- 级别（DEBUG/INFO/WARNING/ERROR）与过滤；
- 时间戳 + 自定义格式（formatter）；
- 多 handler：控制台、文件、轮转；
- 线程安全、生产可配置（basicConfig/dictConfig）；
- 第三方库也用 logging，可以统一捕获它们的日志。

生产代码用 `logging`，`print` 只用于临时调试。

### json 序列化

- `json.dumps(data)`：Python 对象 -> JSON 字符串（序列化），如 `json.dumps({"name": "eds"})`。
- `json.loads(s)`：JSON 字符串 -> Python 对象（反序列化）。
- FastAPI 的请求/响应体、配置文件读写都是这套机制。
- 中文默认被转成 `\uXXXX`，需要保留原文时用 `json.dumps(data, ensure_ascii=False)`。

## 异常处理（2026-08-05 首学）

核心一句话：**`finally` 无论如何都会执行——即使 `return` 了、即使异常被捕获，它是 Python 里「必定收尾」的保证。**

四段执行顺序：

1. `try`：先执行主体。
2. 抛异常 -> 按顺序匹配 `except`（具体类型优先，`except Exception` 兜底业务异常）。
3. 没抛异常 -> 执行 `else`（可选的「成功路径」）。
4. `finally`：无论上面哪条路，最后必执行（`return` 前也会先跑 finally）。

```python
def f():
    try:
        return "try"
    finally:
        print("finally")   # 先打印 finally，函数才真正返回 "try"

print(f())   # 输出：finally 换行 try
```

确认练习（已通过）：`try: raise ValueError("boom") finally: print("finally")` 会先打印 `finally`，随后异常继续向上抛（打印 traceback），**finally 只收尾、不吞异常**。

`finally` 不执行的情况（面试追问）：`os._exit()`、进程被 kill、断电、死循环卡死——都是「进程直接没了」的场景；正常控制流下 finally 必执行。注意 `sys.exit()` 抛的是 `SystemExit`，finally 仍会执行。

裸 `except:` 的坑：会连 `KeyboardInterrupt`/`SystemExit` 一起吞掉，掩盖真实错误，排障困难。正确做法：捕获具体异常类型，或至少 `except Exception`；要么处理，要么 `raise` 转抛，不静默吞掉。

自定义异常：当调用方需要区分你的业务错误时定义（如 `class PermissionDenied(Exception)`），网关层据此做统一错误码转换（对应 [[SangBridge AI统一接入与管理控制台]] 的错误转换职责）。

## 装饰器（2026-08-04 首学）

核心一句话：**函数也是对象，可以当参数传、也可以被返回；装饰器就是「接收函数、返回新函数」的函数。**

```python
@timer
def work():
    time.sleep(0.1)

# 上面的写法完全等价于：
work = timer(work)
```

计时装饰器标准写法：

```python
import time

def timer(func):
    def wrapper(*args, **kwargs):
        start = time.time()
        result = func(*args, **kwargs)   # 调用原函数
        print(f"elapsed: {time.time() - start:.4f}s")
        return result
    return wrapper
```

通用模板（before/after 型）：

```python
def my_decorator(func):
    def wrapper(*args, **kwargs):
        print("before")
        result = func(*args, **kwargs)
        print("after")
        return result
    return wrapper
```

三个必会考点：

1. `@timer` 只是语法糖，等价于 `work = timer(work)`；之后调用 `work()` 实际执行的是 `wrapper`。
2. 不用 `functools.wraps(func)` 时，`wrapper` 会顶替原函数的 `__name__`/`__doc__`，调试和文档工具看到错误名字；`@wraps` 把原函数元信息复制回来。
3. 多个装饰器叠加：**定义时从下往上、调用时从上往下**。`@deco1 @deco2` 等价于 `f = deco1(deco2(f))`；调用 `f()` 先进 `deco1` 的 wrapper，再进 `deco2` 的 wrapper，最后到原函数。

## 生成器与迭代器（2026-08-04 首学）

核心一句话：**生成器是惰性的迭代器——边算边吐，不一次性生成全部；调用函数不执行，`next` 才执行到 `yield` 暂停。**

三个概念分层：

- 可迭代对象（iterable）：能 for 循环的东西，如 list、dict、str；`iter(x)` 可以拿到它的迭代器。
- 迭代器（iterator）：有 `__next__`，一次吐一个值，吐完抛 `StopIteration`；列表本身不是迭代器。
- 生成器（generator）：用 `yield` 写的函数，或生成器表达式 `(x*x for x in range(5))`；它就是一种迭代器，且惰性求值。

```python
gen = (x * x for x in range(5))   # 生成器对象，不是列表
lst = [x * x for x in range(5)]   # 一次性算好全部元素

# len(gen) -> TypeError：生成器不知道后面还有多少，只能一个一个 next
```

`yield` 的执行时机（面试必考）：

```python
def counter():
    print("start")
    yield 1        # 第一次 next 到这里暂停
    print("middle")
    yield 2        # 第二次 next 到这里再暂停

c = counter()      # 只创建生成器，函数体一行都没执行
next(c)            # 打印 "start"；返回 1
next(c)            # 打印 "middle"；返回 2
```

三个必会考点：

1. `len()` 对生成器不可用；内存优势：列表一次性存全部，生成器一次只占一个值的空间。
2. 生成器只能单向迭代一次，吐完就没了；想重来必须重新调用函数创建新生成器。
3. `yield` 暂停并保留函数状态，`return` 结束函数；生成器内部也能 `return` 但通常用于结束。

填空练习（已通过）：

```python
def countdown(n):
    while n > 0:
        yield n   # 返回 n 并暂停
        n -= 1
```

## 推导式与解包（2026-08-05 补充）

- 推导式（`[x for x in range(3)]`）在 Python 3 有**独立作用域**，不污染外层变量：`x = "outer"; [x for x in range(3)]; print(x)` 仍打印 `"outer"`（Python 2 会泄漏）。
- 星号解包：`a, b, *rest = [1, 2, 3, 4, 5]` -> `a = 1`，`b = 2`，`rest = [3, 4, 5]`；`*` 收集剩余部分，位置任意。

## 容易踩坑的地方

- 用 `is` 比较字符串/数值而不是 `==`。
- 遍历时修改列表/字典。
- 异常吞掉不记录上下文；`except Exception` 过滤掉真实错误。
- 模块级可变全局状态：list/dict 被多处修改。
- 依赖不锁版本导致环境不一致；要维护 requirements/锁文件。

## 我的薄弱点

本轮已确认掌握：可变默认参数跨调用共享、`b = a` 是引用、`c = a[:]` 是浅拷贝、`is` 比较身份而 `==` 比较值。

- 整数缓存边界记错：`x = 256; y = 256; x is y` 应为 **True**（CPython 缓存 -5~256），之前误判为 False；257 超出缓存，通常为 False。
- 字符串驻留只覆盖编译期常量（如 `"hello"`）；运行时拼接/构造的字符串一般不复用同一对象，不能依赖 `is` 比较字符串。（2026-08-04 已确认掌握：`"he" + part` 运行时拼接产生新对象，`is` 为 False）
- 深浅拷贝（2026-08-04 已确认掌握）：浅拷贝外层独立、内层共享——`a[0].append(99)` 会影响浅拷贝 b，而 `a.append([5,6])` 不影响；选型判断是「防止污染 vs 有意共享」。
- 待补：深拷贝的代价与风险——递归复制全部对象的性能开销；循环引用（`copy.deepcopy` 用 memo 表处理，手写递归会死循环）；文件句柄/锁/连接等不可拷贝对象；复杂对象需自定义 `__deepcopy__`。
- 装饰器此前完全未接触，2026-08-04 首学；首轮手写练习仍缺失关键细节：`def` 语法、`*args/**kwargs` 收集参数、`func(*args, **kwargs)` 调用原函数、`return wrapper`，需再练一次到能独立写出通用模板。
- 生成器与迭代器（2026-08-04 首学，填空练习已通过）：掌握 `yield n` 暂停交值、调用函数不执行、`next` 驱动执行；仍需巩固：生成器表达式、`len` 不可用、只能迭代一次的口语化表达。
- 异常处理（2026-08-05 首学）：`finally` 必执行语义此前误判（以为匹配到异常后 finally 不执行），已通过确认练习纠正（未捕获异常时 finally 仍执行且不吞异常）；`else` 分支、裸 `except:` 与自定义异常仍待巩固。
- 类与面向对象（2026-08-05 首学）：类属性共享、`__new__`/`__init__` 分工已掌握；classmethod 填空练习暴露薄弱点——类方法内访问类属性必须写 `cls.count`，裸写 `count` 会 `NameError`；`@staticmethod` 的使用场景仍需口头巩固。
- 上下文管理器（2026-08-05 首学）：`__enter__/__exit__` 顺序、异常时 `__exit__` 仍执行、`yield` 位置、返回 `True` 吞异常等均未接触，待填空练习巩固。
- 基础阶段收尾（2026-08-05）：生成器「调用不执行、停在 yield」与「finally 先于 return 输出」已通过回忆检查；装饰器等价写法（`@timer` == `work = timer(work)`、必须 `return wrapper`）、推导式作用域、星号解包仍需巩固。
- 复习口诀：值比较一律用 `==`；`is` 只用于 `None`、单例或确实需要身份判断的场景。

## 面试高频问题

1. 可变默认参数为什么是坑？怎么改？
2. 浅拷贝和深拷贝的区别？`list[:]` 拷贝了什么？
3. 装饰器怎么实现？带参数的装饰器呢？
4. 生成器和迭代器有什么区别？生成器有什么好处？
5. 说一下 `with` 的实现原理。
6. 你在项目里写过哪些 Python？主要负责什么？（对照 04-Project/ 回答）

## 关联知识

- [[Python并发与asyncio]]
- [[FastAPI与ASGI后端框架]]
- [[Kerberos域接入与用户映射管控]]
- [[SangBridge AI统一接入与管理控制台]]
