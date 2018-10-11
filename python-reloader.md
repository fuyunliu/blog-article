---
title: python 脚本自动重载
date: 2018-03-09 17:51:08
categories:
- 笔记
tags:
- Python3
---

Django 和 Flask 应用开启 debug 模式之后都能检测代码的变化然后自动重载，于是去找实现代码，发现 Flask 是用的 werkzeug 库里面的功能，而 Django 的不好用于自己写的脚本，因为和 Django 应用结合了。

<!-- more -->

<!-- toc -->

下面是 `werkzeug` 中的 `_reloader` 模块中的 `run_with_reloader` 函数
```python
def run_with_reloader(main_func, extra_files=None, interval=1,
                      reloader_type='auto'):
    """Run the given function in an independent python interpreter."""
    import signal
    reloader = reloader_loops[reloader_type](extra_files, interval)
    signal.signal(signal.SIGTERM, lambda *args: sys.exit(0))
    try:
        if os.environ.get('WERKZEUG_RUN_MAIN') == 'true':
            t = threading.Thread(target=main_func, args=())
            t.setDaemon(True)
            t.start()
            reloader.run()
        else:
            sys.exit(reloader.restart_with_reloader())
    except KeyboardInterrupt:
        pass

```

用法如下
```python
import time


def main():
    while True:
        print("hello, world!")
        time.sleep(2)


if __name__ == '__main__':
    run_with_reloader(main)

```

但是执行函数不能传参，修改 `run_with_reloader` 如下
```python
def run_with_reloader(main_func, args=(), kwargs=None,
                      extra_files=None, interval=1,
                      reloader_type='auto'):
    """Run the given function in an independent python interpreter."""
    import os
    import sys
    import signal
    import threading
    from werkzeug._reloader import reloader_loops
    reloader = reloader_loops[reloader_type](extra_files, interval)
    signal.signal(signal.SIGTERM, lambda *args: sys.exit(0))
    try:
        if os.environ.get('WERKZEUG_RUN_MAIN') == 'true':
            t = threading.Thread(target=main_func, args=args, kwargs=kwargs)
            t.setDaemon(True)
            t.start()
            reloader.run()
        else:
            sys.exit(reloader.restart_with_reloader())
    except KeyboardInterrupt:
        pass

```

然后就能传参了
```python
import time


def main(name, age):
    while True:
        print("hello, ", name, '!')
        time.sleep(2)
        print(age)


if __name__ == '__main__':
    run_with_reloader(main, args=('foo', 20))

```
