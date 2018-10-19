---
title: selenium 在 ubuntu 服务器上的使用
date: 2018-08-13 10:31:07
categories:
- 笔记
tags:
- Python3
---

# 安装 chrome

```sh
wget -q -O - https://dl-ssl.google.com/linux/linux_signing_key.pub | sudo apt-key add -
echo 'deb [arch=amd64] http://dl.google.com/linux/chrome/deb/ stable main' | sudo tee /etc/apt/sources.list.d/google-chrome.list
sudo apt-get update
sudo apt-get install google-chrome-stable
```

<!-- more -->

<!-- toc -->

# 安装 chromedriver

```sh
wget -N https://chromedriver.storage.googleapis.com/2.41/chromedriver_linux64.zip
unzip chromedriver_linux64.zip
chmod +x chromedriver
cp chromedriver /usr/bin/
```

# 安装 Xvfb

```sh
sudo apt-get -y install xvfb gtk2-engines-pixbuf
sudo apt-get -y install xfonts-cyrillic xfonts-100dpi xfonts-75dpi xfonts-base xfonts-scalable
# 截图功能，可选
sudo apt-get -y install imagemagick x11-apps
Xvfb -ac :99 -screen 0 1280x1024x16 & export DISPLAY=:99
```

# 测试脚本

```python
from selenium import webdriver


chrome_options = webdriver.ChromeOptions()
chrome_options.add_argument('--headless')
chrome_options.add_argument('--disable-gpu')
driver = webdriver.Chrome(chrome_options=chrome_options,executable_path='/usr/bin/chromedriver')
driver.get("https://www.baidu.com")
print(driver.title)
driver.quit()
```

# 遇到的问题

`ubuntu server 18.04` 虽然内置 `python3` 版本，但是没有 `pip`
在 `/etc/apt/sources.list` 添加下列源

```sh
deb http://cn.archive.ubuntu.com/ubuntu bionic main multiverse restricted universe
deb http://cn.archive.ubuntu.com/ubuntu bionic-updates main multiverse restricted universe
deb http://cn.archive.ubuntu.com/ubuntu bionic-security main multiverse restricted universe
deb http://cn.archive.ubuntu.com/ubuntu bionic-proposed main multiverse restricted universe
```

```sh
sudo apt-get update
sudo apt-get install python3-pip
```

# 再用 pip 安装 selenium

```sh
pip3 install selenium
```
