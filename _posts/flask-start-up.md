---
title: 几种启动 Flask 应用的方式
date: 2018-03-12 10:15:58
categories:
- 笔记
tags:
- Flask
---

记录几种启动 Flask 应用的方式

<!-- more -->

<!-- toc -->

首先写一个简单的 `index.py`
```python
# -*- coding: utf-8 -*-

from flask import Flask

app = Flask(__name__)


@app.route('/', methods=['GET'])
def hello():
    return 'hello, world!'

```

#### 最简单的启动方式
```python
if __name__ == '__main__':
    app.run()
```
这只能用于开发模式，可以设置`debug=True`开启调试模式，并且这是单线程同步的。


#### 用tornado驱动flask
写一个`server.py`，并将上面`index.py`中的启动代码去掉，终端运行`python server.py`。
```python
# -*- coding: utf-8 -*-

from tornado.wsgi import WSGIContainer
from tornado.httpserver import HTTPServer
from tornado.ioloop import IOLoop
from index import app
http_server = HTTPServer(WSGIContainer(app))
http_server.listen(5000)  # flask默认的端口
IOLoop.instance().start()

```

这也是同步的，同一时间只能处理一个请求，可以写个简单的接口测试一下。

```python
# -*- coding: utf-8 -*-

import time
from flask import Flask

app = Flask(__name__)


@app.route('/', methods=['GET'])
def hello():

    return 'hello, world!'


@app.route('/test', methods=['GET'])
def test():
    for n in range(10):
        print(n)
        time.sleep(2)

    return 'hello'

```
用postman同时发起5个请求，后台按顺序打印0-9，5个请求是一个一个执行的。


#### 用twisted驱动flask
这个可以同时处理多个请求。
```python
# -*- coding: utf-8 -*-

import time
from flask import Flask

app = Flask(__name__)


@app.route('/', methods=['GET'])
def hello():

    return 'hello, world!'


@app.route('/test', methods=['GET'])
def test():
    for n in range(10):
        print(n)
        time.sleep(2)

    return 'hello'


if __name__ == "__main__":
    reactor_args = {}

    def run_twisted_wsgi():
        from twisted.internet import reactor
        from twisted.web.server import Site
        from twisted.web.wsgi import WSGIResource

        resource = WSGIResource(reactor, reactor.getThreadPool(), app)
        site = Site(resource)
        reactor.listenTCP(5000, site)
        reactor.run(**reactor_args)

    if app.debug:
        # Disable twisted signal handlers in development only.
        reactor_args['installSignalHandlers'] = 0
        # Turn on auto reload.
        import werkzeug.serving
        run_twisted_wsgi = werkzeug.serving.run_with_reloader(run_twisted_wsgi)

    run_twisted_wsgi()


if __name__ == '__main__':
    app.run()

```
