---
title: 使用github-action自动部署hexo博客，并配置免费cdn加速访问
date: 2021-02-14 17:02:47
categories:
- 随笔
tags:
- 随笔
---

## 安装 hexo

1. 按照官方文档创建一个博客 [https://hexo.io/zh-cn/docs/](https://hexo.io/zh-cn/docs/)

2. 创建两个仓库，一个名为 `blog_name` 的仓库用来放 hexo 博客原始文件，一个名为 `your_github_name.github.io` 的仓库用来放生成好的静态站点

3. 修改 `_config.yml` 文件的 `deploy` 配置

```yml
deploy:
  type: 'git'
  repo: 'git@github.com:your_github_name/your_github_name.github.io.git'
  branch: 'gh-pages'
```

4. 在本地执行 `hexo deploy` 后就会自动在本地生成静态文件并推送到 `your_github_name.github.io` 这个仓库，然后在该仓库的设置中开启 `GitHub Pages`，随后访问 [http://your_github_name.github.io](http://your_github_name.github.io) 就能看到你的博客

## 编写 github action

***github action*** 是 github 官方的 ci 工具，可以利用它实现推送 `blog_name` 仓库时，自动借助 github 的机器远程执行 `hexo deploy` 来部署博客，这样只需要写完博客推送一下就行

在远程机器上要推送，需要配置一下 `deploy key`，但是为了防止直接写在配置文件中被别人获取，可以填写在 `blog_name` 仓库的 `设置=>Secrets` 中，Key 为 `ACTION_DEPLOY_KEY`，Value 为 `~/.ssh/id_rsa` 中的内容

在 `blog_name` 仓库页 `Actions` 页面新建一个 workflow 文件，填入下面的内容

```yml
name: Deploy Blog

on: [push] # 当有新push时运行

jobs:
  build: # 一项叫做build的任务

    runs-on: ubuntu-latest # 在最新版的Ubuntu系统下运行
    
    steps:
    - name: Checkout # 将仓库内master分支的内容下载到工作目录
      uses: actions/checkout@v1 # 脚本来自 https://github.com/actions/checkout
      
    - name: Use Node.js 10.x # 配置Node环境
      uses: actions/setup-node@v1 # 配置脚本来自 https://github.com/actions/setup-node
      with:
        node-version: "10.x"
    
    - name: Setup Hexo env
      env:
        ACTION_DEPLOY_KEY: ${{ secrets.ACTION_DEPLOY_KEY }}
      run: |
        # set up private key for deploy
        mkdir -p ~/.ssh/
        echo "$ACTION_DEPLOY_KEY" | tr -d '\r' > ~/.ssh/id_rsa # 配置秘钥
        chmod 600 ~/.ssh/id_rsa
        ssh-keyscan github.com >> ~/.ssh/known_hosts
        # set git infomation
        git config --global user.name 'your_name' # 换成你自己的邮箱和名字
        git config --global user.email 'your_name@email.com'
        # install dependencies
        npm i -g hexo-cli # 安装hexo
        npm i
  
    - name: Deploy
      run: |
        # publish
        hexo generate
        # 执行部署程序
        hexo deploy 
```

## 配置 cdn 和自定义域名

完成上面两步就能实现推送博客仓库时，自动部署静态站点的工作

访问博客过程中发现 github 的速度较慢，于是借助免费 cdn jsdelivr 来加速一些静态资源，github 仓库中的文件天然就能通过 jsdelivr链接访问到，比如你的 `your_github_name.github.io` 仓库中的 `css/style.css` 文件对应的 jsdelivr 链接就是 `https://cdn.jsdelivr.net/gh/your_github_name/your_github_name.github.io@master/css/style.css`

利用上述 cdn 原理，可以在 `hexo generate` 后通过替换 html 代码中资源链接的方式来使用 cdn，比如我要替换 js 的路径，使用 sed 命令将所有 html 文件中 `/src="/js` 开头的资源文件前面替换加上 jsdelivr 的路径

```sh
$ find . -type f -name *.html | \
xargs sed -i 's/src="\/js\//src="https:\/\/cdn.jsdelivr.net\/gh\/your_github_name/your_github_name.github.io@master\/js\//g'
```

同理可以替换 css 等其他资源，将相关代码填在 actions 中 `hexo generate` 和 `hexo deploy` 之间，在 action 执行的时候会完成替换

另外，如果想要配置自定义域名，需要在域名中添加一条 cname 的解析指向 your_github_name.github.io 域名，并在站点 source 目录中添加一个 CNAME 文件并填上你的域名，最后在 `GitHub Pages` 设置中填上并开启自定义域名
