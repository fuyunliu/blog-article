---
title: asyncio 笔记
date: 2018-11-11 15:02:28
categories:
- 笔记
tags:
- Python3
---

<!-- toc -->

<!-- markdownlint-disable MD033 -->

<blockquote><p>并发是指一次处理多件事。
并行是指一次做多件事。
二者不同，但是有联系。
一个关于结构，一个关于执行。
并发用于制定方案，用来解决可能（但未必）并行的问题。</p>
<p align="right">——Rob Pike
Go 语言的创造者之一</p></blockquote>

<!-- more -->

---

# 异步版 `hello-world`

```python
import asyncio


async def main():
    print('hello')
    await asyncio.sleep(.1)
    print('world')

# python3.7 提供
asyncio.run(main())

# main 函数是个协程，直接运行不会执行操作
# main() --> <coroutine object main at 0x109be6d48>
```

# 运行协程的三种方式

- `asyncio.run()`
- 使用 `await` 关键字

```python
import asyncio
import time


async def say_after(delay, what):
    await asyncio.sleep(delay)
    print(what)


async def main():
    print(f"started at {time.strftime('%X')}")

    await say_after(1, 'hello')
    await say_after(2, 'world')
    print(f"finished at {time.strftime('%X')}")


asyncio.run(main())
```

- 使用 `asyncio.create_task()` 创建一个 `Task` 对象

```python
# 直接对上面的 main 函数进行修改
async def main():
    t1 = asyncio.create_task(say_after(1, 'hello'))
    t2 = asyncio.create_task(say_after(2, 'world'))

    print(f"started at {time.strftime('%X')}")

    await t1
    await t2

    print(f"finished at {time.strftime('%X')}")
```

# Awaitable 对象

`awaitable` 对象是指可以在 `await` 表达式中使用的对象。

`coroutines`, `Tasks` 和 `Futures` 是 `awaitable` 对象。

- 协程 `coroutines`

```python
import asyncio


async def nested():
    return 42


async def main():

    # 这中方式不会执行 nested 函数
    nested()

    # await
    print(await nested())


asyncio.run(main())
```

- 任务 `Tasks`

```python
import asyncio


async def nested():
    return 42


async def main():
    t = asyncio.create_task(nested())
    await t


asyncio.run(main())
```

- 期物 `Futures`

# 并发执行 Tasks

使用 `asyncio.gather` 并发执行 `Tasks`

```python
import asyncio


async def factorial(name, number):
    f = 1
    for i in range(2, number + 1):
        print(f"Task {name}: Compute factorial({i})...")
        await asyncio.sleep(1)
        f *= i
    print(f"Task {name}: factorial({number}) = {f}")


async def main():
    await asyncio.gather(
        factorial('A', 2),
        factorial('B', 3),
        factorial('C', 4),
    )


asyncio.run(main())
```

# 线程和协程的对比

```python
# 线程版以动画形式显示文本旋转指针
import threading
import itertools
import time
import sys


class Signal:
    go = True


def spin(msg, signal):
    write, flush = sys.stdout.write, sys.stdout.flush
    for char in itertools.cycle('|/-\\'):
        status = char + ' ' + msg
        write(status)
        flush()
        write('\x08' * len(status))
        time.sleep(.1)
        if not signal.go:
            break
    write(' ' * len(status) + '\x08' * len(status))


def slow_funtion():
    time.sleep(3)
    return 42


def supervisor():
    signal = Signal()
    spinner = threading.Thread(
        target=spin,
        args=('thinking!', signal))
    print('spinner object: ', spinner)
    spinner.start()
    result = slow_funtion()
    signal.go = False
    spinner.join()
    return result


def main():
    result = supervisor()
    print('Answer: ', result)


if __name__ == '__main__':
    main()
```

```python
# 协程版以动画形式显示文本旋转指针
import asyncio
import itertools
import sys


async def spin(msg):
    write, flush = sys.stdout.write, sys.stdout.flush
    for char in itertools.cycle('|/-\\'):
        status = char + ' ' + msg
        write(status)
        flush()
        write('\x08' * len(status))
        try:
            await asyncio.sleep(.1)
        except asyncio.CancelledError:
            break
    write(' ' * len(status) + '\x08' * len(status))


async def slow_funtion():
    await asyncio.sleep(3)
    return 42


async def supervisor():
    spinner = asyncio.create_task(spin('thinking!'))
    print('spinner object: ', spinner)
    result = await slow_funtion()
    spinner.cancel()
    return result


# 一般执行方式
loop = asyncio.get_event_loop()
result = loop.run_until_complete(supervisor())
loop.close()
print('Answer: ', result)


# python3.7 执行方式
asyncio.run(supervisor())
```

- Task 对象像是实现协作式多任务的库（如 `gevent`）中的绿色线程（`green thread`）。
- Task 对象用于驱动协程，Thread 对象用于调用可调用对象。
- Task 对象不由自己手动实例化，而是由 `asyncio.create_task` 方法获取。
- 获取的 Task 对象已经排定了运行时间，而 Thread 实例需要调用 `start` 方法运行。
- 异步版 `slow_funtion` 是协程，由 `await` （就是 `yield from`）驱动。
- 终止线程需要借助外部变量 `go`,终止 Task 可以调用 `Task.cancel()` 实例方法，在协程内部抛出 `CancelledError` 异常，协程内部也可以捕获这个异常，处理终止请求。
