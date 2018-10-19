---
title: Flask v0.1 源码阅读
date: 2018-09-27 17:17:36
categories:
- 笔记
tags:
- Flask
---

[flask v0.1](https://github.com/pallets/flask/tree/0.1) 是第一个发布的版本，单文件版，v0.4 是 flask 的最后一个单文件版本，文章中 flask 的源码有修改，因为依赖包有更新。

<!-- more -->

<!-- toc -->

#### 导包部分
```python
# python2.5 版本加入 with 语句，低于 2.5 需要引入，高于则忽略。
from __future__ import with_statement

import os
import sys

# 没有用到
from threading import local

# jinja2 模板引擎
from jinja2 import Environment, PackageLoader, FileSystemLoader

# Flask 的 Request 和 Response 继承自 werkzeug 的 Request 和 Response
from werkzeug.wrappers import Request as RequestBase, Response as ResponseBase

# 最后几行 _request_ctx_stack, current_app, request, session, g 用到。
from werkzeug.local import LocalStack, LocalProxy

# 用于测试请求上下文，在 Flask 类的方法 test_request_context 中调用，已失效。
# 最新版 test_request_context 方法调用 flask.testing 的 make_test_environ_builder，最终调用 werkzeug.test 的 EnvironBuilder。
from werkzeug import create_environ

from werkzeug.utils import cached_property
from werkzeug.wsgi import SharedDataMiddleware

# 路由
from werkzeug.routing import Map, Rule

# 错误处理
from werkzeug.exceptions import HTTPException, InternalServerError

# flask 自带的 session 用到
from werkzeug.contrib.securecookie import SecureCookie

# 没有用到，作为对外接口
from werkzeug import abort, redirect
from jinja2 import Markup, escape

# 用于获取应用程序根目录
try:
    import pkg_resources
    pkg_resources.resource_stream
except (ImportError, AttributeError):
    pkg_resources = None

```

#### `Request` 和 `Response`
`flask` 的 `Request` 和 `Response` 继承自 `werkzeug` 的 `Request` 和 `Response`
如果你想要自定义 `Request` 和 `Response`，你可以继承这两个类，然后指定 `Flask` 的 `request_class` 和 `response_class`
```python
class Request(RequestBase):
    """请求类"""

    def __init__(self, environ):
        RequestBase.__init__(self, environ)  # WSGI 环境
        self.endpoint = None  # 视图函数的键名
        self.view_args = None  # 视图函数的参数


class Response(ResponseBase):
    """响应类"""

    default_mimetype = 'text/html'


class _RequestGlobals(object):
    pass


class _RequestContext(object):
    """请求上下文"""

    def __init__(self, app, environ):
        self.app = app
        self.url_adapter = app.url_map.bind_to_environ(environ)
        self.request = app.request_class(environ)
        self.session = app.open_session(self.request)  # 带上下文的 session
        self.g = _RequestGlobals()  # 带上下文的 g
        self.flashes = None

    def __enter__(self):
        _request_ctx_stack.push(self)

    def __exit__(self, exc_type, exc_value, tb):
        if tb is None or not self.app.debug:
            _request_ctx_stack.pop()
```

#### 几个有用的函数
```python
def url_for(endpoint, **values):
    """函数跳转
    endpoint: Flask 类有个 view_functions 字典，存储的就是 endpoint 和 视图函数的映射关系，默认 endpoint 是视图函数的名字
    values: 路由传过来的参数
    """
    return _request_ctx_stack.top.url_adapter.build(endpoint, values)


def flash(message):
    """
    消息闪现，存储在 session 中，是个列表
    """
    session['_flashes'] = (session.get('_flashes', [])) + [message]


def get_flashed_messages():
    """
    把 session 中存储的消息全部 pop 出来并返回
    """
    flashes = _request_ctx_stack.top.flashes
    if flashes is None:
        _request_ctx_stack.top.flashes = flashes = \
            session.pop('_flashes', [])
    return flashes


def render_template(template_name, **context):
    """
    从文件渲染模板
    """
    current_app.update_template_context(context)
    return current_app.jinja_env.get_template(template_name).render(context)


def render_template_string(source, **context):
    """
    从字符串渲染模板
    """
    current_app.update_template_context(context)
    return current_app.jinja_env.from_string(source).render(context)


def _default_template_ctx_processor():
    """
    模板处理，使得在所有模板中可以使用 request, session 和 g 三个全局变量
    """
    reqctx = _request_ctx_stack.top
    return dict(
        request=reqctx.request,
        session=reqctx.session,
        g=reqctx.g
    )


def _get_package_path(name):
    """根据名字获取模块的路径"""
    try:
        return os.path.abspath(os.path.dirname(sys.modules[name].__file__))
    except (KeyError, AttributeError):
        return os.getcwd()

```

#### `Flask` 类
```python
class Flask(object):
    """
    Flask 类实现了一个 WSGI 应用，只要将包或者模块的名字传递给它
    一旦创建的时候，它将首先注册视图函数、路由映射、模板配置等等几个重要的对象
    传入包的名字是用于解决应用内部资源的引用，具体请查看 open_resource 函数
    一般情况下，你只需要这样创建：
        from flask import Flask
        app = Flask(__name__)
    """

    # 请求类
    request_class = Request

    # 响应类
    response_class = Response

    # 静态文件路径，设置为 None 可以禁用
    static_path = '/static'

    # 密钥，用于 cookies 签名验证
    secret_key = None

    # 基于 cookie 的 session 的名字
    session_cookie_name = 'session'

    # 默认会直接传给 jinja2 的 options 参数值
    jinja_options = dict(
        autoescape=True,
        extensions=['jinja2.ext.autoescape', 'jinja2.ext.with_']
    )

    def __init__(self, package_name):
        # 调试模式开关，设置 True 以打开调试模式
        # 在调试模式下，应用程序出错会有特殊的错误页面以供调试
        # 并且服务会监控文件的变化，文件发生变化会重载服务
        self.debug = False

        # 包或者模块的名字，一旦设置好了就不要改动
        self.package_name = package_name

        # 应用程序顶级目录
        self.root_path = _get_package_path(self.package_name)

        # 包含所有注册好的视图函数字典，键是函数的名字，也用于生成 URL
        # 值就是函数本身，可以用 route 装饰器注册一个函数
        self.view_functions = {}

        # 所有注册好的错误处理函数，键是错误代码，值是处理函数
        # 可以用 errorhandler 注册一个错误处理函数
        self.error_handlers = {}

        # 预处理函数，每次请求之前会执行，用 before_request 装饰器注册
        self.before_request_funcs = []

        # 后处理函数，每次请求完成以后执行，函数会截获响应并且改变它
        # 用 after_request 装饰器注册
        self.after_request_funcs = []

        # 模板上下文处理器，默认有一个处理函数 _default_template_ctx_processor
        # 默认的函数功能是向模板上下文添加三个对象 request, session, g
        # 每个函数执行不需要参数，返回值为字典，用于填充模板上下文
        self.template_context_processors = [_default_template_ctx_processor]

        # 路由映射，在 werkzeug.routing.Map
        self.url_map = Map()

        # 架起一个静态资源服务，一般用于开发环境，生产环境用 nginx
        if self.static_path is not None:
            self.url_map.add(Rule(self.static_path + '/<filename>',
                                  build_only=True, endpoint='static'))
            if pkg_resources is not None:
                target = (self.package_name, 'static')
            else:
                target = os.path.join(self.root_path, 'static')
            self.wsgi_app = SharedDataMiddleware(self.wsgi_app, {
                self.static_path: target
            })

        # jinja2 模板配置，包括模板目录和默认开启的功能
        self.jinja_env = Environment(loader=self.create_jinja_loader(),
                                     **self.jinja_options)
        # 这是两个模板能用到的函数
        # url_for 用于根据 endpoint 获取 URL
        # get_flashed_messages 用于获取消息闪现
        self.jinja_env.globals.update(
            url_for=url_for,
            get_flashed_messages=get_flashed_messages
        )

    def create_jinja_loader(self):
        """
        加载模板目录，默认目录为 templates
        """
        if pkg_resources is None:
            return FileSystemLoader(os.path.join(self.root_path, 'templates'))
        return PackageLoader(self.package_name)

    def update_template_context(self, context):
        """
        为模板上下文注入几个常用的变量，比如 request, session, g
        context 为填充模板上下文的字典
        """
        reqctx = _request_ctx_stack.top
        for func in self.template_context_processors:
            context.update(func())

    def run(self, host='localhost', port=5000, **options):
        """
        运行开发服务器
        """
        from werkzeug import run_simple
        if 'debug' in options:
            self.debug = options.pop('debug')
        options.setdefault('use_reloader', self.debug)
        options.setdefault('use_debugger', self.debug)
        return run_simple(host, port, self, **options)

    def test_client(self):
        """
        为应用程序创建一个测试客户端
        """
        from werkzeug import Client
        return Client(self, self.response_class, use_cookies=True)

    def open_resource(self, resource):
        """
        动态加载模块
        """
        if pkg_resources is None:
            return open(os.path.join(self.root_path, resource), 'rb')
        return pkg_resources.resource_stream(self.package_name, resource)

    def open_session(self, request):
        """
        创建一个 session，secret_key 必须设置
        基于 werkzeug.contrib.securecookie.SecureCookie
        """
        key = self.secret_key
        if key is not None:
            return SecureCookie.load_cookie(request, self.session_cookie_name,
                                            secret_key=key)

    def save_session(self, session, response):
        """
        保存 session
        """
        if session is not None:
            session.save_cookie(response, self.session_cookie_name)

    def add_url_rule(self, rule, endpoint, **options):
        """
        创建 URL 和视图函数的映射规则，等同于 route 装饰器
        只是 add_url_rule 并没有为视图函数注册一个 endpoint
        这一步也就是向 view_functions 字典添加 endpoint: view_func 键值对
        以下：
            @app.route('/')
            def index():
                pass
        等同于：
            def index():
                pass
            app.add_url_rule('index', '/')
            app.view_functions['index'] = index
        options: 参数选项详见 werkzeug.routing.Rule
        """
        options['endpoint'] = endpoint
        options.setdefault('methods', ('GET',))
        self.url_map.add(Rule(rule, **options))

    def route(self, rule, **options):
        """
        为给定的 URL 注册一个视图函数
        用法：
            @app.route('/')
            def index():
                return 'Hello World'

        变量部分可以用尖括号(``/user/<username>``)指定，默认接受任何不带斜杆的字符串
        变量也可以指定一个转换器，以指定类型的参数：
        =========== ===========================================
        int         整数
        float       浮点数
        path        路径
        =========== ===========================================

        值得注意的是 Flask 如何处理结尾的斜杆，核心思路是保证每个 URL 唯一：
            1、如果配置了一个带结尾斜杆的 URL，用户请求不带结尾斜杆的这个 URL，则跳转到带结尾斜杆的页面。
            2、如果配置了一个不带结尾斜杆的 URL，用户请求带结尾斜杆的这个 URL，则触发404错误。
        这和 web 服务器处理静态资源 static 的逻辑是一样的

        参数：
        rule: URL 字符串
        methods: 允许的请求方法，是个列表，默认只接受 GET 请求和隐式的 HEAD
        subdomain: 指定子域名
        strict_slashes: 上述对结尾斜杆处理的开关
        options: 参数选项详见 `werkzeug.routing.Rule`
        """
        def decorator(f):
            self.add_url_rule(rule, f.__name__, **options)
            self.view_functions[f.__name__] = f
            return f
        return decorator

    def errorhandler(self, code):
        """
        注册一个错误码处理函数
        用法：
            @app.errorhandler(404)
            def page_not_found():
                return 'This page does not exist', 404
        等同于：
            def page_not_found():
                return 'This page does not exist', 404
            app.error_handlers[404] = page_not_found
        参数：
            code: 错误码
        """
        def decorator(f):
            self.error_handlers[code] = f
            return f
        return decorator

    def before_request(self, f):
        """注册一个预处理函数"""
        self.before_request_funcs.append(f)
        return f

    def after_request(self, f):
        """注册一个后处理函数"""
        self.after_request_funcs.append(f)
        return f

    def context_processor(self, f):
        """注册一个模板上下文处理函数"""
        self.template_context_processors.append(f)
        return f

    def match_request(self):
        """
        根据当前请求的路由去和url_map匹配，拿到 enpoint 和 view_args
        endpoint: 端点，是 view_functions 中对应视图函数的key
        view_args: 视图函数的参数
        """
        rv = _request_ctx_stack.top.url_adapter.match()
        request.endpoint, request.view_args = rv
        return rv

    def dispatch_request(self):
        """
        首先调用上面的 match_request 方法拿到 endpoint 和 view_args
        根据 endpoint 可以从 view_functions 找到对应的视图函数
        再传入视图函数的参数 view_args，并返回结果，这个结果只是函数的返回值
        并没有包装成响应类 response_class，可以调用 make_response 方法生成响应
        如果函数执行失败，则根据错误码调用对应的错误处理函数
        """
        try:
            endpoint, values = self.match_request()
            return self.view_functions[endpoint](**values)
        except HTTPException, e:
            handler = self.error_handlers.get(e.code)
            if handler is None:
                return e
            return handler(e)
        except Exception, e:
            handler = self.error_handlers.get(500)
            if self.debug or handler is None:
                raise
            return handler(e)

    def make_response(self, rv):
        """
        将视图函数的返回值包装成一个真实的响应类 response_class
        函数的返回值支持以下几种类型：
        ======================= ===========================================
        response_class:         响应类本身，原样返回
        str:                    字符串，创建相应类并返回
        unicode:                unicode 编码，utf-8编码后创建相应类并返回
        tuple:                  元组，解包元组传入参数创建响应类并返回
        a WSGI function:        WSGI 函数
        ======================= ===========================================
        参数：
        rv: 视图函数的返回值
        """
        if isinstance(rv, self.response_class):
            return rv

        # basestring 是 str 和 unicode 的超类，只支持 python2
        if isinstance(rv, basestring):
            return self.response_class(rv)

        if isinstance(rv, tuple):
            return self.response_class(*rv)

        return self.response_class.force_type(rv, request.environ)

    def preprocess_request(self):
        """
        在分发请求之前执行所有的预处理函数，如果预处理函数有返回值不为 None
        则返回结果并中断其余的预处理
        """
        for func in self.before_request_funcs:
            rv = func()
            if rv is not None:
                return rv

    def process_response(self, response):
        """
        依次传入响应并执行所有的后处理函数，返回新的响应
        """
        session = _request_ctx_stack.top.session
        if session is not None:
            # 保存 `session`
            self.save_session(session, response)
        for handler in self.after_request_funcs:
            response = handler(response)
        return response

    def wsgi_app(self, environ, start_response):
        """
        WSGI 应用，可以用中间件包装：
            app.wsgi_app = MyMiddleware(app.wsgi_app)
        参数：
        environ: WSGI 环境，是一个字典，包含了所有请求的信息
        start_response: 回调函数
        """
        with self.request_context(environ):
            rv = self.preprocess_request()
            if rv is None:
                rv = self.dispatch_request()
            response = self.make_response(rv)
            response = self.process_response(response)
            return response(environ, start_response)

    def request_context(self, environ):
        """
        通过给定的 WSGI 环境创建一个请求上下文，并把它绑定到当前上下文中
        必须通过 with 语句使用，因为 request 对象只在请求上下文
        也就是 with 语句块中起作用
        用法如下：
            with app.request_context(environ):
                do_something_with(request)
        参数：
        environ: WSGI 环境
        """
        return _RequestContext(self, environ)

    def test_request_context(self, *args, **kwargs):
        """
        测试请求上下文，参数详见 werkzeug.create_environ
        """
        return self.request_context(create_environ(*args, **kwargs))

    def __call__(self, environ, start_response):
        """调用 wsgi_app 方法"""
        return self.wsgi_app(environ, start_response)
```

#### 全局变量
```python
_request_ctx_stack = LocalStack()
current_app = LocalProxy(lambda: _request_ctx_stack.top.app)
request = LocalProxy(lambda: _request_ctx_stack.top.request)
session = LocalProxy(lambda: _request_ctx_stack.top.session)
g = LocalProxy(lambda: _request_ctx_stack.top.g)
```

#### `werkzeug` 的 `Local`, `LocalStack` 和 `LocalProxy`
`Flask` 中有两个上下文（`Context `）：应用上下文（`App Context`）、请求上下文（`Request Context`）
上下文就是函数运行时所需要的外部变量，当我们运行一个简单的求和函数 `sum` 是不需要外部变量的
而像 `Flask` 中的视图函数运行需要知道当前的请求的路由、表单和请求方法等等一些信息，
`Django` 和 `Tornado` 把视图函数所需要的外部信息封装成一个对象 `Request`，并把这个对象当作参数传给视图函数，
无论视图函数有没有用到，所以编写 `Django` 的视图函数会到处都见到一个 `request` 参数，
而 `Flask` 则使用了类似 `Thread Local` 的对象，它可以隔离多线程/协程之间的状态，使得多线程/协程像操作一个全局变量一样操作各自的状态而互不影响
原理就是使用当前的线程ID作为命名空间，保存多份字典，每个线程只获取各自线程ID对应的字典
`Flask` 并没有使用 `Python` 标准库的 `Thread Local`，而是用了 `werkzeug` 实现的 `Local`

`Local` 和 `threading.local` 相似，但是 `Local` 在 `Greenlet` 可用的情况下优先使用 `getcurrent` 获取当前线程ID，用以支持 `Gevent` 调度
`Local` 还有一个 `__release_local__` 方法，用以释放当前线程存储的状态信息。

`LocalStack` 是基于 `Local` 实现的栈结构，可以入栈（`push`）、出栈（`pop`）和获取栈顶对象（`top`）

`LocalProxy` 是作为 `Local` 的一个代理模式，它接受一个 `callable` 对象，注意参数不是 `Local` 实例，这个 `callable` 对象返回的结果才是 `Local` 实例
所有对于 `LocalProxy` 对象的操作都会转发到对应的 `Local` 上


```python
# 优先使用 Greenlet 的线程ID
try:
    from greenlet import getcurrent as get_ident
except ImportError:
    try:
        from thread import get_ident
    except ImportError:
        from _thread import get_ident


class Local(object):
    # 固定属性
    __slots__ = ('__storage__', '__ident_func__')

    def __init__(self):
        # 数据存储，是一个字典
        object.__setattr__(self, '__storage__', {})

        # 获取当前线程ID的方法
        object.__setattr__(self, '__ident_func__', get_ident)

    def __iter__(self):
        """以生成器的方式返回字典的所有元素"""
        return iter(self.__storage__.items())

    def __call__(self, proxy):
        """创建一个 LocalProxy"""
        return LocalProxy(self, proxy)

    def __release_local__(self):
        """清空当前线程/协程所保存的数据"""
        self.__storage__.pop(self.__ident_func__(), None)

    def __getattr__(self, name):
        """属性访问"""
        try:
            return self.__storage__[self.__ident_func__()][name]
        except KeyError:
            raise AttributeError(name)

    def __setattr__(self, name, value):
        """属性设置"""
        ident = self.__ident_func__()
        storage = self.__storage__
        try:
            storage[ident][name] = value
        except KeyError:
            storage[ident] = {name: value}

    def __delattr__(self, name):
        """属性删除"""
        try:
            del self.__storage__[self.__ident_func__()][name]
        except KeyError:
            raise AttributeError(name)


class LocalStack(object):

    """
    Local 的栈结构实现
    """

    def __init__(self):
        # Local 实例
        self._local = Local()

    def __release_local__(self):
        # 释放当前线程的数据
        self._local.__release_local__()

    def _get__ident_func__(self):
        # 返回获取当前线程ID的函数
        return self._local.__ident_func__

    def _set__ident_func__(self, value):
        # 
        object.__setattr__(self._local, '__ident_func__', value)
    __ident_func__ = property(_get__ident_func__, _set__ident_func__)
    del _get__ident_func__, _set__ident_func__

    def __call__(self):
        """返回一个 LocalProxy 对象，该对象始终指向 LocalStack 实例的栈顶元素"""
        def _lookup():
            rv = self.top
            if rv is None:
                raise RuntimeError('object unbound')
            return rv
        return LocalProxy(_lookup)

    def push(self, obj):
        """入栈"""
        rv = getattr(self._local, 'stack', None)
        if rv is None:
            self._local.stack = rv = []
        rv.append(obj)
        return rv

    def pop(self):
        """出栈"""
        stack = getattr(self._local, 'stack', None)
        if stack is None:
            return None
        elif len(stack) == 1:
            release_local(self._local)
            return stack[-1]
        else:
            return stack.pop()

    @property
    def top(self):
        """获取栈顶元素"""
        try:
            return self._local.stack[-1]
        except (AttributeError, IndexError):
            return None


@implements_bool
class LocalProxy(object):

    """
    Local 的代理模式实现，所有对 LocalProxy 对象的操作，包括属性访问、方法调用和二元操作
    都会转发到那个 callable 参数返回的 Local 对象上
    """
    __slots__ = ('__local', '__dict__', '__name__', '__wrapped__')

    def __init__(self, local, name=None):
        # 把 callable 参数绑定到 __local 属性上
        object.__setattr__(self, '_LocalProxy__local', local)
        # 代理名字
        object.__setattr__(self, '__name__', name)
        if callable(local) and not hasattr(local, '__release_local__'):
            # 注意这里的参数 local 仅仅是一个 callable 对象
            # 该对象执行返回的结果才是 Local 实例
            object.__setattr__(self, '__wrapped__', local)

    def _get_current_object(self):
        """获取当前 Local 实例"""
        if not hasattr(self.__local, '__release_local__'):
            return self.__local()
        try:
            return getattr(self.__local, self.__name__)
        except AttributeError:
            raise RuntimeError('no object bound to %s' % self.__name__)

    @property
    def __dict__(self):
        try:
            return self._get_current_object().__dict__
        except RuntimeError:
            raise AttributeError('__dict__')

    def __repr__(self):
        try:
            obj = self._get_current_object()
        except RuntimeError:
            return '<%s unbound>' % self.__class__.__name__
        return repr(obj)

    def __bool__(self):
        try:
            return bool(self._get_current_object())
        except RuntimeError:
            return False

    def __unicode__(self):
        try:
            return unicode(self._get_current_object())  # noqa
        except RuntimeError:
            return repr(self)

    def __dir__(self):
        try:
            return dir(self._get_current_object())
        except RuntimeError:
            return []

    def __getattr__(self, name):
        if name == '__members__':
            return dir(self._get_current_object())
        return getattr(self._get_current_object(), name)

    def __setitem__(self, key, value):
        self._get_current_object()[key] = value

    def __delitem__(self, key):
        del self._get_current_object()[key]

    # 下面的源码省略，LocalProxy 重写了所有的魔法方法
```
