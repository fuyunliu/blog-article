---
title: Centos 安装 PhantomJS
date: 2018-11-21 16:10:23
categories:
- 笔记
tags:
- Python3
- CentOS
---

`PhantomJS` 已经不再开发了，`seleniumn` 也警告使用 `PhantomJS` 是过时的，推荐使用 `headless` 版的 `Chrome` 或者 `Firefox`，但是有时候需要用到，够用就行，而且在 `Linux` 下安装也相对简单。

<!-- more -->

<!-- toc -->

## 安装 `fontconfig` 依赖

```sh
yum install -y fontconfig freetype freetype-devel fontconfig-devel libstdc++
```

## 下载 `PhantomJS` 并解压

```sh
# 安装到此目录
cd /usr/local

# 下载
wget https://bitbucket.org/ariya/phantomjs/downloads/phantomjs-2.1.1-linux-x86_64.tar.bz2

# 解压
tar -jxvf phantomjs-2.1.1-linux-x86_64.tar.bz2

# 重命名
mv phantomjs-2.1.1-linux-x86_64 phantomjs

# 添加软链接
ln -s /usr/local/phantomjs/bin/phantomjs /usr/bin/phantomjs

# 验证
phantomjs --version
```

## 用 `selenium` 驱动 `PhantomJS`

```python
from selenium import webdriver
from selenium.webdriver.common.desired_capabilities import DesiredCapabilities

# 更改浏览器头
dcap = dict(DesiredCapabilities.PHANTOMJS)
dcap["phantomjs.page.settings.userAgent"] = "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/70.0.3538.102 Safari/537.36"
driver = webdriver.PhantomJS(desired_capabilities=dcap)
driver.set_page_load_timeout(10)
driver.set_script_timeout(10)
driver.get("https://www.baidu.com")
```

## 推荐使用的 `Chrome` 用法

```python
from selenium import webdriver
from selenium.webdriver.chrome.options import Options

# 无界面浏览器
options = Options()
options.add_argument('headless')
options.add_argument('disable-gpu')
options.add_argument('window-size=1200x600')

# 禁用 javascript
prefs = {'profile.managed_default_content_settings.javascript': 2}
options.add_experimental_option("prefs", prefs)

# 禁止弹出式窗口
prefs = {"profile.default_content_setting_values.notifications": 2}
options.add_experimental_option("prefs", prefs)

# 禁用图片
prefs = {'profile.managed_default_content_settings.images': 2}
options.add_experimental_option("prefs", prefs)

driver = webdriver.Chrome(chrome_options=options)

# 执行JS
driver.execute_script('window.scrollTo(0, 0)')  # scroll to top
driver.execute_script('window.scrollTo(0, document.body.scrollHeight)')  # end
```
