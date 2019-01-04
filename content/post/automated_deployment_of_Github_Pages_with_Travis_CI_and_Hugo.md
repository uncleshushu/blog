---
title: "使用 Travis-CI 和 Hugo 来自动化构建 Github Pages"
date: 2019-01-04T22:57:56+08:00
draft: false
isCJKLanguage: true
image: ""
showonlyimage: false
categories: ["Tech"]
keywords: []
tags: ["Tool", "Hugo", "CI"]
weight: 1
---

我的博客采用 [Hugo][] 生成，托管于 [Github Pages][] 。本文将介绍：

1. 如何使用 Hugo 生成静态博客；

1. 如何将博客部署到 Github Pages ；

1. 如何利用 [Travis CI][] 来自动化上述过程。

**Table of contents**

<!-- TOC -->

- [Why Hugo](#why-hugo)
- [Installing](#installing)
- [Quick start](#quick-start)
- [Deploy to Github Pages](#deploy-to-github-pages)
    - [Manual deployment](#manual-deployment)
    - [Automated deployment with Travis CI](#automated-deployment-with-travis-ci)
- [References](#references)

<!-- /TOC -->

## Why Hugo

之前的博客用的是 [Ghost](https://ghost.org/) ，它一般是作为动态博客的服务器，部署在 VPS 上。我不能忍受的是它的内容管理方式，它把原始的 markdown 文件存在 SQLite 之类的数据库里，没法对源文件进行自由编辑和版本管理。它的动态特性对我来说用处也不大。

而 Hugo 作为一个热门的新框架，以目录的方式管理原始 markdown 文件，这一点符合我的需求，而且又没什么明显缺点，可能还比那些经典的框架在某些方面先进，那就决定用它了。

## Installing

参见 [Install Hugo | Hugo](https://gohugo.io/getting-started/installing) 。

对于 Debian/Ubuntu 稳定版的用户，由于 Hugo 目前还没有到达 1.0 版本，仍在快速更新中，所以 `apt` 源中的版本相对较旧，最好去它在 Github 的 Release 页面 [Hugo Releases](https://github.com/gohugoio/hugo/releases) 下载最新的安装包。

## Quick start

请参考 [Quick Start | Hugo][] 和 [Basic Usage | Hugo][] ，官方教程写得很清楚。

对于 Hugo 的默认行为，有下面几个需要注意的地方：

1. 下载并配置主题这一步不可省略，否则无法生成可用的网站（ Hugo 并不会报错），因为 Hugo 没有提供默认的 theme 。

1. 用 `hugo new` 创建的 markdown 文件，默认是 `archetypes/default.md` 的 copy 。开头会有属性配置的 header ，其中有一个叫 `draft` 的 `bool` 属性，代表该文章是否为 draft ，默认是 `true` 。`hugo` 和 `hugo server` 等命令，对该属性为 `true` 的 markdown 文件，不会将其编译为 html 文件，除非加上 `-D` 或 `--buildDrafts` 选项。可以修改 `archetypes/default.md` 或者设定自己的 archetype 来控制 `hugo new` 使用的 markdown 模板，具体参见 [Archetypes | Hugo][] 。

1. 如果已有 `public` 目录， Hugo 只会覆盖其中的文件而不会删除，因此最好先清空 `public` ,再使用 `hugo` 命令重新生成，否则在 `content` 目录中删除的文章仍然会留在 `public` 中。Hugo 官方对此并不想提供类似于 `hugo clean` 的命令，参见 ['hugo clean' · Issue #2389 · gohugoio/hugo · GitHub][] 。

## Deploy to Github Pages

[Github Pages][] 是 Github 提供的免费的静态网站托管服务，分为 user or organization site 和 project site 两种，个人 blog 一般采用前一种。具体操作流程见 [Github Pages][]。运行机制概括如下：

1. `username.github.io` 这个 git repository 的目录结构应该符合静态网站的惯例，即根目录下有 `index.html` ，然后是其他静态资源的文件/目录；

1. Github Pages 的服务会在 `username.github.io` 这个 URL 展示上述 repository 对应的网站；

1. 该服务会自动地随 repository 的内容的更新而更新网站。

Hugo 生成的 `public` 目录就对应于上述 `username.github.io` 的 repository。将它 push 到 `username.github.io` ，就可以更新网站内容。

### Manual deployment

如果要手动部署到 Github Pages ,可以如下操作：

1. 首先，将 `public` 目录初始化为 git repository ：

    ```sh
    cd public
    git init
    git remote add origin https://github.com/username/username.github.io.git
    ```
1. 每次编辑完文章，确定要发布到网站时：

    ```sh
    # delete obsolete files except '.git'
    cd public
    git rm -rf .
    # regenerate website
    hugo
    cd public
    git add .
    git commit -m "added new post"
    git push origin master
    ```

    可以写一个 `deploy.sh` 来自动化上述过程（参考了 [Put it Into a Script - Host on GitHub | Hugo](https://gohugo.io/hosting-and-deployment/hosting-on-github/#put-it-into-a-script) ）：

    ```sh
    #!/bin/bash

    echo -e "\033[0;32mDeploying updates to GitHub...\033[0m"

    cd public
    git rm -rf .

    cd ..
    # Build the project.
    hugo # if using a theme, replace with `hugo -t <YOURTHEME>`

    cd public
    # Add changes to git.
    git add .

    # Commit changes.
    msg="rebuilding site `date`"
    if [ $# -eq 1 ]
    then msg="$1"
    fi
    git commit -m "$msg"

    # Push to Github
    git push origin master

    # Come Back up to the Project Root
    cd ..
    ```

    将 `deploy.sh` 放在 blog 的目录下，然后运行

    ```sh
    ./deploy.sh "Your optional commit message"
    ```
    就可以实现发布了。

上述方法有一个局限性：要想更新 blog ，哪怕只是改一个错字，也需要在安装了 Hugo 的设备上操作。如果将整个 blog 的目录作为一个 git repository 托管在 Github 上，并且将 Hugo 的 build 过程使用在线服务来实现，就可以在 Github 的网页端上完成编辑和发布，而无需使用专门的设备。

### Automated deployment with Travis CI

[Travis CI][] 是配合 Github 使用的持续集成服务。将一个 Github repository 注册到 Travis CI ,有新的 push 时， Travis CI 就会启动一台虚拟机， `git pull` 该 repository ，根据 repository 根目录下的 `.travis.yml` 来执行构建和部署。

要想通过 Travis CI 来实现 blog 的自动的构建和发布，需要将整个 blog 目录托管在 Github 上（将 `public` 加入 `.gitignore` ），然后在 Travis CI 上激活该项目的持续集成，并在该目录下添加 `.travis.yml` （详见 [Travis CI Tutorial - Travis CI][]），内容如下：

```yaml
install:
    # install latest release version of hugo
    - while true; do curl -s https://api.github.com/repos/gohugoio/hugo/releases/latest | grep -E '"browser_download_url":[[:space:]]*".*hugo_[0-9]+\.[0-9]+_Linux-64bit.tar.gz"' | cut -d ':' -f 2- | xargs wget; [ $? -ne 0 ] || break; done
    - tar -xzf hugo*

script:
    - ./hugo

deploy:
  provider: pages
  skip-cleanup: true
  github-token: $GITHUB_TOKEN  # Set in the settings page of your repository, as a secure variable
  email: $GITHUB_EMAIL
  name: $GITHUB_USERNAME
  keep-history: true
  local_dir: public
  repo: username/username.github.io
  target-branch: master
  on:
    branch: master
```

Travis CI 创建的虚拟机会把 blog 的 repository 通过 `git pull` 拉取，然后进入该目录，执行上述文件中的指令。 `install` 部分对应的是下载并解压 Hugo ，其中 `while` 循环的目的是下载失败后重试直到成功，因为我多次测试后发现，下载失败的概率不小，而下载失败会导致构建失败 （在 Job log 中可以看到）。 `script` 部分是构建的脚本。 `deploy` 部分利用了 Travis CI 对 Github Pages 作为 [provider](https://docs.travis-ci.com/user/deployment/#supported-providers) 的支持，这部分的详细配置文档请参考 [GitHub Pages Deployment - Travis CI][] 。

这样，对 blog 的修改 push 到 Github 后，就会触发构建，并自动将生成的 `public` 目录 push 到 `username.github.io` 的 repository 。甚至，在 Github 的网页端编辑 blog 目录的文件并 commit ，就可以更新对应的网站。

## References

1. [Hugo: The world's fastest framework for building websites][Hugo]

    [Hugo]: https://gohugo.io/

1. [Install Hugo | Hugo][]

    [Install Hugo | Hugo]: https://gohugo.io/getting-started/installing

1. [Quick Start | Hugo][]

    [Quick Start | Hugo]: https://gohugo.io/getting-started/quick-start/

1. [Basic Usage | Hugo][]

    [Basic Usage | Hugo]: https://gohugo.io/getting-started/usage/

1. [Archetypes | Hugo][]

    [Archetypes | Hugo]: https://gohugo.io/content-management/archetypes/

1. [Directory Structure | Hugo](https://gohugo.io/getting-started/directory-structure/)

1. ['hugo clean' · Issue #2389 · gohugoio/hugo · GitHub][]

    ['hugo clean' · Issue #2389 · gohugoio/hugo · GitHub]:(https://github.com/gohugoio/hugo/issues/2389)

1. [GitHub User or Organization Pages - Host on GitHub | Hugo](https://gohugo.io/hosting-and-deployment/hosting-on-github/https://gohugo.io/hosting-and-deployment/hosting-on-github/#github-user-or-organization-pages)

1. [Github Pages][]

    [Github Pages]: https://pages.github.com/

1. [Travis CI][]

    [Travis CI]: https://travis-ci.com/

1. [Travis CI Tutorial - Travis CI][]

    [Travis CI Tutorial - Travis CI]: https://docs.travis-ci.com/user/tutorial/

1. [GitHub Pages Deployment - Travis CI][]

    [GitHub Pages Deployment - Travis CI]: https://docs.travis-ci.com/user/deployment/pages/
