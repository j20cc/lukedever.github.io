---
title: centos安装并使用无头浏览器puppeteer
date: 2021-02-22 17:02:47
categories:
- 随笔
tags:
- 随笔
---

puppeteer 在做复杂的爬虫时特别好用，centos 安装时坑比较多，本文记录一下安装过程，以及一些使用技巧

## 安装

使用 `npm install puppeteer` 在 centos 上安装依赖，官方文档说会自动下载 chromium 浏览器，如果没有安装，可以使用 `node node_modules/puppeteer/install.js` 来安装

继续使用时可能报依赖缺失错误，根据官方文档可以安装一下下面这些依赖

```sh
$ sudo yum install -y alsa-lib.x86_64 atk.x86_64 cups-libs.x86_64 gtk3.x86_64 \
 ipa-gothic-fonts libXcomposite.x86_64 libXcursor.x86_64 libXdamage.x86_64 libXext.x86_64 \
  libXi.x86_64 libXrandr.x86_64 libXScrnSaver.x86_64 libXtst.x86_64 pango.x86_64 \
   xorg-x11-fonts-100dpi xorg-x11-fonts-75dpi xorg-x11-fonts-cyrillic xorg-x11-fonts-misc \
    xorg-x11-fonts-Type1 xorg-x11-utils
$ sudo yum update nss -y
```

继续使用时确实可以启动浏览器，但是截图发现字体有缺失，安装一下应该可以正常使用了

```sh
sudo yum groupinstall "fonts" -y
```

## 使用技巧

1. 防止被检测

登录谷歌时，我发现谷歌能检测到使用不安全的浏览器不给登录，这时候可以使用 `puppeteer-extra` 库搭配 `puppeteer-extra-plugin-stealth` 插件来防止被谷歌检测到，使用很简单

```js
// 使用 npm 安装 puppeteer-extra, puppeteer-extra-plugin-stealth
const puppeteer = require('puppeteer-extra')
const StealthPlugin = require('puppeteer-extra-plugin-stealth')
puppeteer.use(StealthPlugin())
// your puppeterr code ...
```

2. 等待页面跳转完成

有的页面需要跳转，需要等待完全跳转完才继续进行下一步操作时可以用 `page.waitForNavigation` 完成

```js
const navigationPromise = page.waitForNavigation();
await navigationPromise;
// 多次使用 await navigationPromise;
```

3. 直到完全打开一个页面

使用 `page.goto(url, options)` 打开一个页面时，可以通过 options 对象配置超时时间 timeout(单位毫秒)，waitUntil 配置什么时候算打开完成，选项有

- load - 页面的load事件触发时
- domcontentloaded - 页面的 DOMContentLoaded 事件触发时
- networkidle0 - 不再有网络连接时触发（至少500毫秒后）
- networkidle2 - 只有2个网络连接时触发（至少500毫秒后）

4. 页面内执行 js 方法

有时候想在页面执行一段 js 方法，比如模拟下拉，可以使用 `page.evaluate` 方法，代码如下

```js
// 传入 num 用来限制滚动多少秒
page.evaluate(num => {
    let times = 0;
    const distance = 100;
    const timer = setInterval(() => {
        //滚动条向下滚动 distance
        window.scrollBy(0, distance);
        times += 1
        if (times >= num) {
            clearInterval(timer)
        }
    }, 1000);
}, num)
```

未完待续~
