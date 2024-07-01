---
layout: post
title: Gitlab CI/CD 介绍
categories: [Git]
description: 持续集成，持续交付，持续部署。
keywords: Git, CI/CD
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
topmost: true
---

## 前言

在项目迭代过程，可能有一个专门负责 CI/CD 的人员，但当想做一些静态代码检查，依赖检查，图片大小检查等事情的时候，就自己需要了解 CI/CD，编写特定 Pipeline Job。本文将做一些 CI/CD 基本介绍，看完后能够在  `.gitlab-ci.yml` 中配置需要的 Job 就行，所以这篇文章适合未接触过，或者刚想入手Gitlab CI/CD 的人。

本文不算原创，内容来源自于官网 [GitLab CI/CD](https://docs.gitlab.com/ee/ci/) 和自己的理解，以及部分项目经验。

## 基础概念

### CI/CD

CI/CD 是一种持续开发软件的方法，可以**不断的进行构建、测试和部署代码迭代更改**。这种迭代有助于减少基于错误或失败的版本进行开发新代码的可能性。使用这种方法，从新代码开发到部署，可以减少人工干预甚至不用干预。

达到持续的方法主要是：**持续集成，持续交付，持续部署**。

CI（Continuous Integration）：持续集成，也就是当每一次更改的代码被推送到远程分支后，可以创建一组脚本来自动地构建和测试这些更改，确保这些更改可以通过一些基本的准则，减少引入错误的机会。

CD :

- Continuous Delivery：持续交付，在持续集成的基础上更进一步，当每一次更改的代码落库后，不仅会构建和测试，也会进行部署，但是部署需要人工干预，手动的有目的进行部署。
- Continuous Deployment：持续部署，持续集成之外的另一个步骤，类似于持续交付。不同之处在于，它不是手动部署应用程序，而是将其设置为自动部署。不需要人为干预。

### Gitlab CI/CD

Gitlab CI/CD 也就是 Gitlab 提供了上面的 CI/CD 能力，可以进行持续集成，持续交付和持续部署。

Gitlab CI/CD 适用于通用的开发工作流程。

当将本地 commits 推送到在 Gitlab 上的远程分支上，就会触发项目的 CI/CD pipeline：

自动运行（串行或并行）脚本：

- 构建和测试应用程序；
- 在应用程序中查看修改，检查是否和本地运行一样。

当达到预期以后：

- Review  和 Approve 更改的代码；

- 合并分支，然后 GitLab CI/CD会自动地将更改部署到生产环境中。
  
  在CI/CD 过程中，如果遇到失败，可以回滚修改的代码：

![p1.png](/images/posts/2022-02-24-Gitlab-CICD-Intro/p1.png)

上面 Gitlab CI/CD 工作流程图，展示了主要的步骤。在实际项目中，CI 主要是提交或合并代码的时候触发，负责一些基本规则的检查，如果检查遇到失败，那么回滚或修改代码后再提交或合并，降低代码风险；CD 主要手动的触发，在CI的基础上，还负责功能检查，如果功能符合验收标准，那么就可以交付或部署。

如果更进一步看这个工作流程，可以看到 GitLab 在 DevOps 生命周期的每个阶段提供的功能：

![p2.png](/images/posts/2022-02-24-Gitlab-CICD-Intro/p2.png)

#### DevOps

DevOps，就是 Development 和 Operations 两个词的组合。

它在维基百科定义是：

> DevOps是一组过程、方法与系统的统称，用于促进开发、技术运营和质量保障（QA）部门之间的沟通、协作与整合。

首先，软件从开发到交付过程需要经历的阶段为：规划、编码、构建、测试、发布、部署和维护。那么 DevOps 就是让开发，测试，运维人员在这个过程中能够个更好的沟通、协作与整合。

通常的软件开发过程：

![p3.png](/images/posts/2022-02-24-Gitlab-CICD-Intro/p3.png)

这种方式就是开发人员负责规划，编码，构建，然后交给测试人员负责测试，然后交给运维人员负责发布，部署，维护。这个过程是顺序进行的，一个阶段完成之后，再进入下一个阶段，这种方式符合瀑布式或敏捷开发。

 DevOps 软件开发过程：

![p4.jpeg](/images/posts/2022-02-24-Gitlab-CICD-Intro/p4.jpeg)

DevOps 主要是 开发和运维人员相互更了解，更紧密的合作，不再有隔阂，让软件从开发到部署的过程中，开发人员的生产力可以提高，运维人员的可靠性也可以增强。

### Pipelines

Pipeline 是  CI/CD 重要组成之一。

Pipeline 也就是流水线，包括：

- Jobs：也就是任务，定义了该做什么，比如：编译和测试代码；
- Stages：也就是阶段，定义什么时候执行 Jobs，比如：在编译代码的阶段之后进入运行测试的阶段。

Job 是由 Runner 来执行，如果有足够多的并发 Runner，同一个 Stage 的 Job 可以并行执行。

如果同一个 Stage 中的所有 Job 都执行成功，Pipeline 就会进入下一个 Stage；如果一个 Stage 中的 任何一个 Job之 执行失败，Pipeline 就不会进入下一个 Stage，提前结束。

通常，Pipeline 是自动执行的，一旦创建就不需要干预。但是，有时也可以手动与 Pipeline 交互。

一般 Pipeline 包含四个 Stage（阶段），按照以下顺序执行：

- 一个 build（构建） 阶段，包含一个 compile 的 job；
- 一个 test（测试）阶段，包含两个 test1 和 test2 的 job；
- 一个 staging（预发） 阶段，包含一个 deploy to stage 的 job；
- 一个 production（生产）阶段，包含一个 deploy-to-prod 的 job。

### Jobs

Pipeline 的配置从 Job 开始，Job 是 `.gitlab-ci.yml` 文件的最重要基本元素。

Job 其实就是任务，是 GitLab CI 系统中可以独立控制并运行的最小单位：

- 用约束来定义，说明在什么条件下应该执行这些约束；
- 可以有任意名称，但是至少包含 script 元素；
- 对定义多少没有限制。

在  `.gitlab-ci.yml` 文件中定义 Job：

```
job1:
  script: "execute-script-for-job1"

job2:
  script: "execute-script-for-job2"
```

定义 Job 的名字不能使用下面预定好的关键字名称：

- `image`
- `services`
- `stages`
- `types`
- `before_script`
- `after_script`
- `variables`
- `cache`
- `include`
- `true`
- `false`
- `nil`

### Variables

CI/CD 提供了预定义好的环境变量，比如：




| Variable           | Description                                                                  |
| ------------------ | ---------------------------------------------------------------------------- |
| CI_COMMIT_REF_NAME | 用于构建项目的分支或tag名称                                                              |
| CI_COMMIT_REF_SLUG | 先将$CI_COMMIT_REF_NAME的值转换成小写，最大不能超过63个字节，然后把除了0-9和a-z的其他字符转换成-。在URLs和域名名称中使用 |
| CI_COMMIT_SHA      | commit的版本号                                                                   |
| CI_JOB_STAGE       | .gitlab-ci.yml中定义的stage的名称                                                   |
| CI_PROJECT_ID      | GitLab CI在内部使用的当前项目的唯一ID                                                     |
| ...                | ...                                                                          |

这些环境变量可以在 `.gitlab-ci.yml` 文件中使用，特别是在自定义的脚本中使用。

```
test_variable:
  stage: test
  script:
    - echo "$CI_JOB_STAGE"
```

CI/CD 也支持自定义环境变量，在`.gitlab-ci.yml` 文件中声明即可：

```
variables:
  TEST_VAR: "All jobs can use this variable's value"

job1:
  variables:
    TEST_VAR_JOB: "Only job1 can use this variable's value"
  script:
    - echo "$TEST_VAR" and "$TEST_VAR_JOB"
```

在上面中，`TEST_VAR` 相当于是全局变量，在 `.gitlab-ci.yml` 文件中的定义的其它 Job 也可以使用；`TEST_VAR_JOB` 相当于是局部变量，只有 `job1` （当前Job）可以使用。

### Cache and artifacts

cache 是 Job 下载并保存的一个或多个文件。使用相同 cahce 的后续 Job 不必再次下载文件，因此执行速度更快。

cache：

- 使用 `cache` 来定义每个 Job 的缓存；
- 后续的 Pipeline 可以使用缓存；
- 如果依赖相同的，同一个 Pipeline 的后续 Job 可以使用缓存；
- 不同的项目不能共用缓存。

artifacts：

- 定义每个 Job 的产物；
- 同一个 Pipeline 的后面 Stage 中 的后续 Job 可以使用前面 Job 的产物；
- 不同的项目不能共享产物； 
- 默认情况下，产物在30天后过期。可以自定义过期时间；
- 如果启用了“保留最新产物”，则最新产物不会过期；
- 使用依赖项来控制哪些 Job 获取 产物。

cache 和 artifacts 的区别：

- 对依赖项使用 cache，比如从网络上下载的包。缓存存储在安装 GitLab Runner的地方，如果启用了分布式缓存，则将其安装并上传载到S3；
- **使用 artifacts 在 stage 之间传递中间构建结果。artifacts 由 Job 生成，存储在GitLab中，可以下载**；
- artifacts 和 cache 都定义了它们相对于项目目录的路径，并且不能链接到项目目录之外的文件。

每个分支中的 Job 使用相同的缓存，可以使用 key：$CI_COMMIT_REF_SLUG:

```
cache:
  key: $CI_COMMIT_REF_SLUG
```

所有分支和所有 Job 之间共享缓存，使用相同的 key 就行：

```
cache:
  key: one-key-to-rule-them-all
```

在执行静态代码扫描 Job 的时候，定义产物：

```
lint and tests:
  tags:
    - apk
    - android
  stage: analyze
  script:
    - ./gradlew --build-cache --no-daemon testDebugUnitTest lintDebug
  artifacts:
    name: "$CI_JOB_STAGE}_reports_${CI_PROJECT_NAME}_$CI_COMMIT_REF_SLUG"
    when: on_failure
    expire_in: 3 days
    paths:
      - "*/build/reports"
  only:
    - branches
  except:
    - master
```

- name：产物名称；
- when：on_failure 表示 Job 执行失败；
- expire_in：超时时间3天；
- paths：将路径 `*/build/reports` 下的文件添加到产物中。

### `.gitlab-ci.yml`

`.gitlab-ci.yml` 文件存放在项目仓库的根目录下，由 Gitlab Runner 来执行。在这个文件中，定义好 CI/CD 相关的配置，配置中主要包括 Pipeline 执行相关的 一系列 Stage ， Stage 中包含 一系列 Job，Job 中包含一系列 Script。 

```
stages:
  - build
  - test

build-code-job:
  stage: build
  script:
    - echo "Check the ruby version, then build some Ruby project files:"
    - ruby -v
    - rake

test-code-job1:
  stage: test
  script:
    - echo "If the files are built successfully, test some files with one command:"
    - rake test1

test-code-job2:
  stage: test
  script:
    - echo "If the files are built successfully, test other files with a different command:"
    - rake test2
```

关于 `.gitlab-ci.yml` 文件 的编写，使用的是[YAML](https://www.runoob.com/w3cnote/yaml-intro.html)，因为它适用于表达数据结构和各种配置文件。

可以下载 AndroidStudio 提供的 YAML 插件，来编辑 `.gitlab-ci.yml` 文件。

#### Keywords

Gitlab CI/CD  pipeline 的配置包含：

1. 使用全局 Keyword 来配置 pipeline 行为；
2. 使用 Job 特有 Keyword 来配置 Job；

配置 pipeline 行为的 全局 Keyword 有：

| Keyword                                                    | 描述                               |
| ---------------------------------------------------------- | -------------------------------- |
| [default](https://docs.gitlab.com/ee/ci/yaml/#default)     | 定义Job 关键字的自定义默认值                 |
| [include](https://docs.gitlab.com/ee/ci/yaml/#include)     | 定义导入其它的 `*.yml` 文件来配置，本地和远程文件都可以 |
| [stages](https://docs.gitlab.com/ee/ci/yaml/#stages)       | 定义pipeline 中 stage 的名称和顺序        |
| [variables](https://docs.gitlab.com/ee/ci/yaml/#variables) | 定义 pipeline 中 所有 job 可以使用的全局变量   |
| [workflow](https://docs.gitlab.com/ee/ci/yaml/#workflow)   | 控制可以运行的 pipeline 类型              |




配置 Job 使用的 Job Keyword 有：




| Keyword                                                                      | 描述                                                                                                |
| ---------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| [after_script](https://docs.gitlab.com/ee/ci/yaml/#after_script)             | 定义Job 完成后执行的 script，执行失败的 Job 也可以执行 script                                                        |
| [allow_failure](https://docs.gitlab.com/ee/ci/yaml/#allow_failure)           | 定义是否允许 Job 失败，如是是 true，则 当前 Job 执行失败后，不会导致 pipeline 失败，会继续往后执行，否则 pipeline 终止                     |
| [artifacts](https://docs.gitlab.com/ee/ci/yaml/#artifacts)                   | 定义 Job 产物，也就是 Job 执行完成后（成功和失败都行），将哪些文件或目录添加到产物中或者构建结果中                                            |
| [before_script](https://docs.gitlab.com/ee/ci/yaml/#before_script)           | 定义Job 开始前执行的 script                                                                               |
| [cache](https://docs.gitlab.com/ee/ci/yaml/#cache)                           | 定义 Job 之前缓存的文件或目录                                                                                 |
| [coverage](https://docs.gitlab.com/ee/ci/yaml/#coverage)                     | 给定 Job 的代码覆盖率设置                                                                                   |
| [dast_configuration](https://docs.gitlab.com/ee/ci/yaml/#dast_configuration) | 在 Job 级别上使用DAST配置文件中的配置                                                                           |
| [dependencies](https://docs.gitlab.com/ee/ci/yaml/#dependencies)             | 控制 Job 产物的下载行为，也就是从哪些 Job 获取产物；如果不定义，则前几个 stage 的所有产物都会传递给每个 Job                                  |
| [environment](https://docs.gitlab.com/ee/ci/yaml/#environment)               | Job 部署到的环境的名称                                                                                     |
| [except](https://docs.gitlab.com/ee/ci/yaml/#except)                         | 使用 only 和 except 控制 Job 何时被添加到 pipeline，except 控制 Job 何时不运行；可以使用新 Keyword:rules 来替代 only 和 except |
| [only](https://docs.gitlab.com/ee/ci/yaml/#only)                             | 使用 only 和 except 控制 Job 何时被添加到 pipeline，except 控制 Job 何时运行；可以使用新 Keyword:rules 来替代 only 和 except  |
| [rules](https://docs.gitlab.com/ee/ci/yaml/#rules)                           | 定义在 pipeline 中包含和不包含 Job 的规则，rules 与 only 和 except 的作用一样，所以不能同时出现                                 |
| [extends](https://docs.gitlab.com/ee/ci/yaml/#extends)                       | 扩展某个 Job                                                                                          |
| [image](https://docs.gitlab.com/ee/ci/yaml/#image)                           | 指定 Job 在其中运行的 Docker image                                                                        |
| [inherit](https://docs.gitlab.com/ee/ci/yaml/#inherit)                       | 定义继承的全局设置的默认变量                                                                                    |
| [interruptible](https://docs.gitlab.com/ee/ci/yaml/#interruptible)           | 定义在 Job 执行完成前启动了新 pipeline 的时候，是否需要取消 Job，ture 取消，否则不取消                                           |
| [needs](https://docs.gitlab.com/ee/ci/yaml/#needs)                           | 该关键字，可以让 Job 不需要按照 Stage 顺序来执行，可以不等待，按照需要来执行                                                      |
| [pages](https://docs.gitlab.com/ee/ci/yaml/#pages)                           | 定义一个 将静态内容上传到 GitLab 的 GitLab pages Job，然后会将内容作为网址发布                                              |
| [parallel](https://docs.gitlab.com/ee/ci/yaml/#parallel)                     | 单个 pipeline 中并以并行的方式运行 Job 多次                                                                     |
| [release](https://docs.gitlab.com/ee/ci/yaml/#release)                       | 让 Runner 生成 release                                                                               |
| [resource_group](https://docs.gitlab.com/ee/ci/yaml/#resource_group)         | 限制 Job 并发运行                                                                                       |
| [retry](https://docs.gitlab.com/ee/ci/yaml/#retry)                           | Job 执行失败，重试的次数，默认为0                                                                               |
| [script](https://docs.gitlab.com/ee/ci/yaml/#script)                         | Job 执行的脚本                                                                                         |
| [secrets](https://docs.gitlab.com/ee/ci/yaml/#secrets)                       | 指定 CI/CD secrets                                                                                  |
| [services](https://docs.gitlab.com/ee/ci/yaml/#services)                     | 指定额外的 Docker image 来运行 script                                                                     |
| [stage](https://docs.gitlab.com/ee/ci/yaml/#stage)                           | 定义 Job 所处的 stage                                                                                  |
| [tags](https://docs.gitlab.com/ee/ci/yaml/#tags)                             | 通过 tag 来选择 Runner 运行，Runner 建立的时候会创建 tag 列表                                                       |
| [timeout](https://docs.gitlab.com/ee/ci/yaml/#timeout)                       | Job 执行的超时时间                                                                                       |
| [trigger](https://docs.gitlab.com/ee/ci/yaml/#trigger)                       | 定义下游 pipeline 触发器                                                                                 |
| [variables](https://docs.gitlab.com/ee/ci/yaml/#variables)                   | 定义 Job 可以使用的局部变量                                                                                  |
| [when ](https://docs.gitlab.com/ee/ci/yaml/#variables)                       | 何时运行 Job 的条件                                                                                      |

在 全局 Keyword 中的 `default` 可以 用来配置所有 Job 需要执行通用配置，它支持的 Job 配置有：

- `after_script`
- `artifacts`
- `before_script`
- `cache`
- `image`
- `interruptible`
- `retry`
- `services`
- `tags`
- `timeout`

例子：

```
default:
  image: ruby:3.0

rspec:
  script: bundle exec rspec

rspec 2.7:
  image: ruby:2.7
  script: bundle exec rspec
```

在上面，rspec 使用的 `image` 是 `default` 中的 ruby:3.0，而 rspec 2.7 使用的  `image`  它自己定义的 `ruby:2.7`。

CI/CD. 的基础概念到这里介绍就完成了，下面再介绍一下在项目中具体的使用。

## 实例

在一个Android 项目中 的 `.gitlab-ci.yml` 文件的简单的配置：

```
#定义所有Job的通用配置
default:
  image: openjdk:8-jdk
  cache:
    key: "$CI_PROJECT_ID-$CI_COMMIT_REF_SLUG"
    paths:
      - ".gradle/"
      - "build/intermediates/"
      - "*/build/intermediates"
      - "*/build/generated"
      - "*/build/kotlin"
      - "*/build/tmp"
  before_script:
    - test -z "${ANDROID_HOME}" -a -d "/data/gitlab-runner/android-sdk-linux" && export ANDROID_HOME=/data/gitlab-runner/android-sdk-linux
    - test -z "${ANDROID_HOME}" && echo "no ANDROID_HOME" && exit 1 || echo "ANDROID_HOME=$ANDROID_HOME"
    - chmod +x ./gradlew
#定义阶段
stages:
  - build
  - analyze
#定义全局变量
variables:
  GIT_SUBMODULE_STRATEGY: recursive
#编译release包
compile release sources:
  tags:
    - apk
    - android
  stage: build
  only:
    refs:
      - triggers
  except:
    - master
  script:
    - ./gradlew  :app:assembleRelease --no-daemon --no-build-cache -Pshrinker.enabled=true -PandResGuard.enabled=true  -Photfix.enabled=true -Pandroid.enableD8.desugaring=false -Pflutter.doctor=1
  allow_failure: false
#lint静态代码检查
lint and tests:
  tags:
    - apk
    - android
  stage: analyze
  script:
    - ./gradlew --build-cache --no-daemon testDebugUnitTest lintDebug
  artifacts:
    name: "$CI_JOB_STAGE}_reports_${CI_PROJECT_NAME}_$CI_COMMIT_REF_SLUG"
    when: on_failure
    expire_in: 3 days
    paths:
      - "*/build/reports"
  only:
    - branches
  except:
    - master
```

### 定义所有Job的通用配置

```
default:
  image: openjdk:8-jdk
  cache:
    key: "$CI_PROJECT_ID-$CI_COMMIT_REF_SLUG"
    paths:
      - ".gradle/"
      - "build/intermediates/"
      - "*/build/intermediates"
      - "*/build/generated"
      - "*/build/kotlin"
      - "*/build/tmp"
  before_script:
    - test -z "${ANDROID_HOME}" -a -d "/data/gitlab-runner/android-sdk-linux" && export ANDROID_HOME=/data/gitlab-runner/android-sdk-linux
    - test -z "${ANDROID_HOME}" && echo "no ANDROID_HOME" && exit 1 || echo "ANDROID_HOME=$ANDROID_HOME"
    - chmod +x ./gradlew
```

在全局 Keyword 中的 `default` 配置了 `image，cache，before_script`。其中 cache 配置了：同一个项目的每个分支中的 Job 使用相同的缓存，需要缓存的文件是 gradle 和 build 目录下的文件；before_script 配置了：每个 Job 都可以使用 `./gradlew` 命令。

### 定义全局变量

```
variables:
  GIT_SUBMODULE_STRATEGY: recursive
```

定义了全局变量 `GIT_SUBMODULE_STRATEGY`，这个变量的主要作用是在 Gitlab CI/CD 中 拉取 submodules，具体可以查看上一篇文档：[Gitlab CI 拉取 submodules](https://blog.csdn.net/wangjiang_qianmo/article/details/122691224)。

### 定义阶段

```
stages:
  - build
  - analyze
```

stages 中定义了 pipeline 的两个阶段：build 和 analyze，执行完 build 阶段 再进入执行 analyze 阶段。

![p5.jpeg](/images/posts/2022-02-24-Gitlab-CICD-Intro/p5.png)

### 定义 Job

build 阶段：

```
compile release sources:
  tags:
    - apk
    - android
  stage: build
  only:
    refs:
      - triggers
  except:
    - master
  script:
    - ./gradlew  :app:assembleRelease --no-daemon --no-build-cache -Pshrinker.enabled=true -PandResGuard.enabled=true  -Photfix.enabled=true -Pandroid.enableD8.desugaring=false -Pflutter.doctor=1
  allow_failure: false
```

compile release sources 是 Job 名称 ；tags 指定了 运行 Job 的 runner，这个runner 是在注册的时候确定的；stage 指定的是 build 阶段；only 定义表示的是 限制 Job 运行的条件，这里使用的是 `triggers` ，它的意思是通过 `trigger token` ，使用 `pipeline triggers API`来触发 pipeline ，一般就是在项目外远程触发 pipeline，当前`处于 CD 阶段`；except 定义表示的是 限制 Job 不运行的条件，也就是在 master 分之不运行；script 表示该 Job 执行的脚本，也就是编译 release 包；allow_failure 设置false， 表示 此 Job 失败，整个 pipeline 就结束。

analyze 阶段：

```
lint and tests:
  tags:
    - apk
    - android
  stage: analyze
  script:
    - ./gradlew --build-cache --no-daemon testDebugUnitTest lintDebug
  artifacts:
    name: "$CI_JOB_STAGE}_reports_${CI_PROJECT_NAME}_$CI_COMMIT_REF_SLUG"
    when: on_failure
    expire_in: 3 days
    paths:
      - "*/build/reports"
  only:
    - branches
  except:
    - master
```

与 `compile release sources` 差不多，only 和 except 设置的条件表示，除了 master 分之，在任何开发分之上提交或合并代码就会触发 pipeline，`处于 CI 阶段`；artifacts 指定了产物，name：产物名字（阶段名字_reports_项目名字_分之名字）， when：表示执行 Job 失败的时候；expire_in：产物超时时间为3天，paths： 将`*/build/reports` 目录下的文件添加到产物。

**在这里需要注意的是 CI 阶段的 Jobs 主要是一些 规则性检查的 Job，如静态代码检查，依赖检查，图片大小检查；CD 阶段的 Jobs 包含 CI 阶段的 Jobs，还执行正常流程规定的 Job，像上面的 `./gradlew  :app:assembleRelease`。**
