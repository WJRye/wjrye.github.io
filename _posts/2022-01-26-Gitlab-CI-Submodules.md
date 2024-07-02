---
layout: post
title: Gitlab CI 拉取 submodules
categories: [Git]
description: 在 Gitlab 上正确的使用 submobules。
keywords: Git, Gitlab, CI
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

## 前言

在项目开发中，有时需要使用另外一个项目（第三方或独立项目），这时可以通过 [Git 工具 - 子模块](https://git-scm.com/book/zh/v2/Git-%E5%B7%A5%E5%85%B7-%E5%AD%90%E6%A8%A1%E5%9D%97) 来管理。当在本地添加好子模块（`git submodule add <project gitlab url>`）并推送到 gitlab 上，我们期望 gitlab CI 能够自动的去 clone 或 pull 对应的 依赖项目并正常构建。但是，这时我们可能会遇到：

```
fatal: could not read Username for 'https://gitserver.com/ ': No such device or address
```

等问题，本篇文章将叙述如何 在 Gitlab 上正确的使用 submobules。

## 第一步，配置 `.gitmodules` 文件

### 方式一：官网提供

在 Gitlab CI/CD 使用 submodules，官网提供的方式：[Using Git submodules with GitLab CI/CD](https://docs.gitlab.com/ee/ci/git_submodules.html)，大概说了两点：

1. 如果主项目和子项目（submodule）在同一个 gitlab server 下，比如，主项目地址：`https://gitlab.com/android/android-main`，子项目地址：`https://gitlab.com/android/android-sub` 或 `https://gitlab.com/android/common-project/android-sub1`，那么在`.gitmodules`文件可以使用 **相对 url 地址** 配置方式：

```
[submodule "android-sub"]
  path = android-sub
  url = ../../android-sub.git
```

2. 如果主项目和子项目（submodule）**不**在同一个 gitlab server 下，在`.gitmodules`文件可以使用 **绝对 url 地址** 配置方式：

```
[submodule "android-sub"]
  path = android-sub
  url = https://gitserver.com/group/android-sub.git
```

### 方式二：实际项目需要（推荐）

对于官网提供的方式，不管是主项目和子项目（submodule）在或不在同一个 gitlab server 下，子项目开放的成员（members）权限通常是收敛的，一个项目组中，可能只有部分人有 read 和 write 权限，而其他人只有 read 权限。

这时，就需要使用 Gitlab 提供的 [Deploy tokens](https://docs.gitlab.com/ee/user/project/deploy_tokens/) 来配置`.gitmodules` 文件。

关于 Deploy tokens 有两点：

1. 由有项目权限 **Maintainer** 或 **Owner**的成员负责 Deploy tokens 的管理（创建和删除） ；
2. Deploy tokens 允许对项目 clone 或 push 或 pull，但是无需 账号和密码。

#### [创建和配置 Deploy tokens](https://docs.gitlab.com/ee/user/project/deploy_tokens/#creating-a-deploy-token)

进入子项目，左侧菜单栏，打开settings/repository/deploy_tokens：
![在这里插入图片描述](/images/posts/2022-01-26-Gitlab-CI-Submodules/p1.png)
创建完成后，复制 **username** 和 **deploy_token** 来配置 `.gitmodules` 文件：

```
[submodule "android-sub"]
 path = android-sub
 url = https://<username>:<deploy_token>@gitserver.com/android/android-sub.git
```

使用这种方式，可以避免遇到 Gitlab CI/CD 权限问题。

## 第二步，配置 `.gitlab-ci.yml` 文件

在 Gitlab jobs 使用 submodules，官网提供的方式：[Use Git submodules in CI/CD jobs](https://docs.gitlab.com/ee/ci/git_submodules.html#use-git-submodules-in-cicd-jobs)，大概就是说：

1. 确保子项目 和 主项目 在 同一个 gitlab server 上；
2. 在 `.gitlab-ci.yml` 文件 配置 **GIT_SUBMODULE_STRATEGY：normal 或 recursive**：

```
variables:
  GIT_SUBMODULE_STRATEGY: recursive
```

配置 `GIT_SUBMODULE_STRATEGY：normal` 相当于执行：

```
# 将新的URL更新到文件.git/config
git submodule sync
# 从新 URL 更新子模块
git submodule update --init
```

配置 `GIT_SUBMODULE_STRATEGY：recursive `相当于执行：

```
# 将新的 URL 复制到本地配置中
git submodule sync --recursive
# 从新 URL 更新子模块
git submodule update --init --recursive
```

## 第三步，确保权限

在上面两个步骤中，需要确保执行上面命令的项目成员，在主项目和子项目中都有项目的 **Maintainer** 或 **Owner** 权限。

## 遇到的问题

### 1.fatal: repository  not found

如果 配置 `.gitmodules` 文件使用的是 **相对 url 地址** 配置方式，Gitlab CI 遇到

```
Synchronizing submodule url for 'android-sub '
Entering 'android-sub'
Entering 'android-sub'
HEAD is now at 5b2e725 Feat. ci submodule test
remote: The project you were looking for could not be found.
fatal: repository 'https://gitlab-ci-token:xxxxxxxxxxxxxxxxxxxx@gitlab/android-sub.git/' not found
Unable to fetch in submodule path 'android-sub'
stdin: is not a tty
ERROR: Job failed: exit status 1
```

这种是子项目和主项目的 **相对 url 地址** 不正确，比如，一个公司有两个项目：`https://gitlab.com/android/project1/android-main`，`https://gitlab.com/android/project2/android-main`，现在有一个公用子项目：`https://gitlab.com/android/common-project/android-sub1`，那么使用在主项目中 引入 公用子项目 的时候，使用的 **相对 url 地址**应该是：

```
[submodule "android-sub"]
  path = android-sub1
  url = ../../../android-sub.git
```

### 2.构建失败，找不到子项目的相关文件（类）

集成 submodule 后，第一次在 Gitlab 构建项目，遇到 找不到子项目的相关文件（类）导致构建失败。查看 job 控制台日志显示：

```
HEAD is now at 691bc88845 Feat. ci 拉取submodule 打包
From https://gitlab.com/android/android-main
 + 691bc88845...246c8cfd6f feature/2.3.0-wangjiang -> origin/feature/2.3.0-wangjiang  (forced update)
Checking out 246c8cfd as feature/2.3.0-wangjiang...
Updating/initializing submodules recursively...
Synchronizing submodule url for 'android-sub'
Cloning into '/home/gitlab-runner/builds/aWHpgegt/0/studio/android/android-sub'...
Submodule path 'android-sub': checked out 'd26c5afc724dde0b78a64c45b3333900b32a6cf9'
```

此时，只是表示 已经 clone 子项目到主项目，但是 clone 后的主项目中只含有子项目的根目录文件夹，文件夹里面没有下载任何 子项目（submodule）的文件。

解决此问题，可以在主项目编写一个脚本（算是双重校验）去执行命令：`git submodule update --init --recursive`，比如（Android 项目脚本）：

```
    def repositoryPath = "android-sub"
    def repositoryFile = file(repositoryPath)
    def childFiles = repositoryFile.listFiles()
    if (!repositoryFile.exists() || childFiles == null || childFiles.length == 0) {
        def cmd = 'git submodule update --init --recursive'
        exec {
            ExecSpec execSpec ->
                executable 'bash'
                args '-c', cmd
        }
    }
```

另外，构建成功情况下，job 控制台日志应该显示：

```
HEAD is now at 0b26b59181 Feat. ci 拉取submodule 打包 test
Checking out 0b26b591 as feature/2。3.0-wangjiang...
Updating/initializing submodules recursively...
Synchronizing submodule url for 'android-sub'
Entering 'android-sub'
Entering 'android-sub'
HEAD is now at c8914ee Feat. Android todo list
```

表示此时已经执行了`GIT_SUBMODULE_STRATEGY：recursive`。

### 3.权限问题

在 Gitlab 执行 pipeline ，读取子项目（submodule），遇到：

```
fatal: could not read Username for 'https://gitserver.com/': No such device or address
fatal: clone of 'https://gitserver.com/android/android-sub.git' into submodule path '/home/gitlab-runner/builds/hF55UpTC/0/studio/android/android-sub' failed
Failed to clone 'android-sub'. Retry scheduled
Cloning into '/home/gitlab-runner/builds/hF55UpTC/0/studio/android/android-sub''...
fatal: could not read Username for 'https://gitserver.com/'': No such device or address
fatal: clone of 'https://gitserver.com/android/android-sub.giit' into submodule path '/home/gitlab-runner/builds/hF55UpTC/0/studio/android/android-sub' failed
Failed to clone 'android-sub' a second time, aborting
stdin: is not a tty
ERROR: Job failed: exit status 1
```

这是项目权限问题，把执行主项目的权限给对应的子项目也添加上，权限至少是 **Maintainer** 或 **Owner** 。或者 使用 **第一步，配置 `.gitmodules` 文件 中的 方式二** 对 `.gitmodules` 文件 进行配置。
