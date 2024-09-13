---
layout: post
title: Python 封装 detekt 和 pmd 命令- 增量静态代码检查
categories: [Python]
description: 利用 Python 封装 detekt 和 pmd cli，切入项目静态代码检查，让静态代码检查更加符合现实项目的日常开发。
keywords: Python, Detekt, PMD
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

## 引言

上一篇文章介绍了[Python 封装 git 命令]({{site.url}}/2024/01/23/Python-Git-Intro/)，用于**查看某个版本某个作者的所有提交更改**和**查看某个提交第一次出现的 release 分支**。如果能直接获取某个版本某个作者的所有提交更改文件，那么将这些更改文件交给静态代码检查工具，就可以实现每个迭代版本的增量静态代码检查。

在之前的另外一篇文章 [Android 静态代码检查]({{site.url}}/2022/09/08/Android-Code-Analyze/)中，介绍了通过在 gradle 中集成 lint、 detekt、pmd 工具，以及结合在 gradle 中执行 git 命令获取修改文件，以实现增量代码检查。但是在实行的过程中，并不顺利，或者说最后直接中断了。因为项目涉及多仓库多模块多人开发，多仓库导致只能集成在主仓库的 gradle 中，但是如果在主仓库中实行静态代码检查，不同模块不同业务团队会有不同的需求。

对于项目开发来说，静态代码检查是有必要的，能防止项目出现低端错误问题。本篇文章将介绍利用 Python 封装 detekt 和 pmd cli，切入项目静态代码检查，让静态代码检查更加符合现实项目的日常开发。

**Github 项目地址：[在 Android 开发中使用 Python 脚本](https://github.com/WJRye/android-script)**

## 增量静态代码检查

在日常的 Android 项目开发中，如果每个 release 版本都去做全量静态代码检查，是不切实际的。因为很多项目都存在历史遗留问题，全量静态代码检查，会增加开发人员的工作负担，所以增量静态代码检查才可能实行。

首先，假设已经看过文章：[Python 封装 git 命令]({{site.url}}/2024/01/23/Python-Git-Intro/)，了解如何获取 某个版本某个作者的所有提交文件，提交文件包含修改，删除、添加、重命名文件等。也看过文章： [Android 静态代码检查]({{site.url}}/2022/09/08/Android-Code-Analyze/)，了解 detekt 对 kotlin 语言静态代码检查，以及 pmd 对 java 语言静态代码检查 和 pmd cpd 对 java 和 kotlin 语言做重复代码检查。


### CLI 介绍

cli 其实就是命令行工具，detekt 和 pmd 都提供了 cli ，那么利用 python ，可以轻松的操作 detekt 和 pmd 的 cli 相关命令。

#### detekt cli 介绍

detekt 工具主要是对 kotlin 语言做静态代码检查。使用命令行运行 detekt 官方文档介绍：[detekt doc](https://detekt.dev/docs/gettingstarted/cli/)。首先，在 detekt 的 github [releases](https://github.com/detekt/detekt/releases) 中下载目前最新的版本：1.23.4 - 2023-11-26 中的 detekt-cli-1.23.4-all.jar。下载解压后，即可以在命令行使用：

```bash
./detekt-cli-1.23.4/bin/detekt-cli --help 

选项：
--all-rules 激活所有可用（甚至是不稳定的）规则。 默认值：false 
--auto-correct, -ac 允许规则自动纠正代码（如果它们支持）。默认规则集不支持自动纠正，不会更改用户代码库中的任何行。但是可以编写自定义规则以支持自动纠正。添加了 '--plugins' 的额外的 'formatting' 规则集支持它，并需要此标志。 默认值：false 
--base-path, -bp 指定一个目录作为基本路径。目前它影响格式化报告中的所有文件路径。控制台输出和 txt 报告中的文件路径不受影响，仍然保持为绝对路径。 
--baseline, -b 如果传入了一个基线 xml 文件，则仅在控制台中打印不在基线中的新代码异味。 
--build-upon-default-config 预先配置 detekt 为您提供了一堆规则和一些默认设置。允许额外提供的配置覆盖默认设置。 默认值：false 
--classpath, -cp 实验性功能：用于查找用户类文件和依赖 jar 文件的路径。用于类型解析。 
--config, -c 配置文件的路径（path/to/config.yml）。可以使用 ',' 或 ';' 作为分隔符指定多个配置文件。 
--config-resource, -cr detekt 类路径上配置资源的路径（path/to/config.yml）。 
--create-baseline, -cb 将当前分析结果视为未来 detekt 运行的异味基线。 默认值：false 
--debug 打印有关配置和扩展的额外信息。 默认值：false 
--disable-default-rulesets, -dd 禁用默认的规则集。 默认值：false 
--excludes, -ex 描述要从分析中排除的路径的通配符模式。 
--generate-config, -gc 导出默认配置。路径可以用 --config 选项指定（默认路径：default-detekt-config.yml） 默认值：false 
--help, -h 显示用法。 
--includes, -in 描述要包含在分析中的路径的通配符模式。与 'excludes' 模式组合使用很有用。 
--input, -i 要分析的输入路径。多个路径用逗号分隔。如果未指定，则使用当前工作目录。
--jdk-home 实验性功能：使用自定义 JDK 主目录包含到类路径中。 
--jvm-target 实验性功能：生成的 JVM 字节码的目标版本，该字节码在编译期生成并现在用于类型解析（1.8、9、10，...，20） 默认值：1.8 
--language-version 实验性功能：Kotlin 语言版本 X.Y 的兼容性模式，对于所有后来发布的语言特性报告错误 可能的值：[1.0, 1.1, 1.2, 1.3, 1.4, 1.5, 1.6, 1.7, 1.8, 1.9, 2.0, 2.1] 
--max-issues 仅当找到的问题数量不超过指定的问题数量时才返回退出码 0。 
--parallel 启用并发编译和分析源文件。在启用此标志之前，请先进行一些基准测试。根据启发式法，从 Kotlin 代码的 2000 行开始，性能有所提升。 默认值：false 
--plugins, -p 由 ',' 或 ';' 分隔的插件 jar 文件的额外路径。 
--report, -r 为给定的 'report-id' 生成报告，并将其存储在给定的 'path' 上。条目应包括：[report-id:path]。可用的 'report-id' 值为：'txt'、'xml'、'html'、'md'、'sarif'。它们也可以结合使用，例如 '-r txt:reports/detekt.txt -r xml:reports/detekt.xml' 
--version 打印 detekt CLI 版本。 默认值：false
```

对于这些命令选项，可以首先使用选项 --generate-config 生成检查规则配置文件： detekt.yml 

```bash
./detekt-cli-1.23.4/bin/detekt-cli --generate-config

Successfully copied default config to /Users/wangjiang/detekt.yml
```
detekt.yml 文件内容：

```bash
build:
  maxIssues: 0
  excludeCorrectable: false
  weights:
    # complexity: 2
    # LongParameterList: 1
    # style: 1
    # comments: 1

config:
  validation: true
  warningsAsErrors: false
  checkExhaustiveness: false
  # when writing own rules with new properties, exclude the property path e.g.: 'my_rule_set,.*>.*>[my_property]'
  excludes: ''

processors:
  active: true
  exclude:
    - 'DetektProgressListener'
  # - 'KtFileCountProcessor'
  # - 'PackageCountProcessor'
  # - 'ClassCountProcessor'
  # - 'FunctionCountProcessor'
  # - 'PropertyCountProcessor'
  # - 'ProjectComplexityProcessor'
  # - 'ProjectCognitiveComplexityProcessor'
  # - 'ProjectLLOCProcessor'
  # - 'ProjectCLOCProcessor'
  # - 'ProjectLOCProcessor'
  # - 'ProjectSLOCProcessor'
  # - 'LicenseHeaderLoaderExtension'

console-reports:
  active: true
  exclude:
     - 'ProjectStatisticsReport'
     - 'ComplexityReport'
     - 'NotificationReport'
     - 'FindingsReport'
     - 'FileBasedFindingsReport'
  #  - 'LiteFindingsReport'

output-reports:
  active: true
  exclude:
  # - 'TxtOutputReport'
  # - 'XmlOutputReport'
  # - 'HtmlOutputReport'
  # - 'MdOutputReport'
  # - 'SarifOutputReport'
 
//......省略  
```
当然，也可以直接从 github 上下载 [default-detekt-config.yml](https://github.com/detekt/detekt/blob/main/detekt-core/src/main/resources/default-detekt-config.yml)。

detekt 的规则有：

 - Comments Rule Set：代码注释和文档相关规则
 - Complexity Rule Set：代码复杂性相关规则
 - Coroutines Rule Set：代码协程问题相关的规则
 - Empty-blocks Rule Set：代码空代码块相关规则
 - Exceptions Rule Set：代码抛出和处理异常相关规则
 - Formatting Rule Set：这个规则集为ktlint实现的规则提供了包装器 - https://ktlint.github.io/。
 - Libraries Rule Set：库暴露 API 相关规则
 - Naming Rule Set：代码中命名相关的规则
 - Performance Rule Set：代码性能相关问题规则
 - Potential-bugs Rule Set：代码潜在性 bug 规则
 - Style Rule Set：代码风格相关规则

对于 detekt 静态代码检查，只需要有规则配置 -c ，输入文件列表 -i ， 输出结果 -r 就可以：

```bash
使用 detekt-config.yml 配置的规则，检查 file_path1, file_path2 指向的文件，并把结果输出到 reports/detekt.html 

./detekt-cli-1.23.4/bin/detekt-cli -c detekt-config.yml -i file_path1,file_path2 -r html:reports/detekt.html
```

#### pmd cli 介绍

pmd 工具主要是对 java 语言做静态代码检查。使用命令行运行pmd官方文档介绍：[pmd doc](https://docs.pmd-code.org/latest/pmd_userdocs_installation.html)。首先，在 pmd 的 github [releases](https://github.com/pmd/pmd/releases) 中下载目前最新的版本：30-September-2023 - 7.0.0-rc4 中的 pmd-dist-7.0.0-rc4-bin.zip
。下载解压后，即可以在命令行使用：

```bash
./pmd-bin-7.0.0-rc4/bin/pmd --help
```
选项：

- **`check`**: 标准源代码分析器。用于执行代码分析并查找潜在的问题或违规。

- **`cpd`**: 复制/粘贴检测器（Copy/Paste Detector）。用于查找重复的代码块。

- **`designer`**: PMD 规则可视化设计器。提供一个可视化工具，用于设计和配置 PMD 规则。

- **`cpd-gui`**: 复制/粘贴检测器的图形用户界面。提供一个图形界面，用于运行复制/粘贴检测。

- **`ast-dump`**: 实验性命令，用于输出解析源代码后的抽象语法树（AST）。

工具还提供了一些退出码，用于指示执行的结果：
 - 0: 成功分析，未发现违规。
 - 1: 执行期间发生意外错误。
 - 2: 使用错误，请参考命令帮助。
 - 4: 成功分析，至少发现一个违规。

pmd 工具主要有两个功能：对 java 语言做静态代码检查和 对 java 或 kotlin语言做重复代码块检查（也支持其它语言，如 python, c/c++，swift等）。

```bash
./pmd-bin-7.0.0-rc4/bin/pmd check --help
```
选项：

- **`<inputPaths>`**: 要分析的源文件或包含源文件的目录的路径。等效于使用 `--dir` 选项。

- **`--aux-classpath=<auxClasspath>`**: 指定源代码使用的库的类路径。用于解析 Java 源文件中的类型。

- **`-b, --benchmark`**: 基准模式，完成后输出基准报告，默认输出到 System.err。

- **`--cache=<cacheLocation>`**: 指定增量分析的缓存文件位置。应包括文件名（而不仅仅是父目录）。如果文件不存在，将在第一次运行时创建。

- **`-d, --dir=<inputPaths>`**: 要分析的源文件或包含源文件的目录的路径。支持 Zip 和 Jar 文件，如果直接指定它们（在探索目录时找到的归档文件不会递归扩展）。

- **`-D, -v, --debug, --verbose`**: 调试模式。

- **`-e, --encoding=<encoding>`**: 指定源代码文件的字符集编码，默认为 UTF-8。

- **`-f, --format=<format>`**: 报告格式。支持多种格式，如 codeclimate、csv、html、json 等。还可以提供自定义 Renderer 的完全限定名称。

- **`--file-list=<fileListPath>`**: 包含要分析的文件列表的文件路径。必须提供 --dir、--file-list 或 --uri 之一。

- **`--force-language=<forceLanguage>`**: 强制使用给定语言进行所有输入文件的语言分析。禁用按文件名自动选择语言。

- **`-h, --help`**: 显示帮助消息并退出。

- **`--ignore-list=<ignoreListPath>`**: 包含要从分析中排除的文件列表的文件路径。

- **`--minimum-priority=<minimumPriority>`**: 规则优先级阈值，低于此配置的规则将不会被使用。

- **`--no-cache`**: 明确禁用增量分析。

- **`--[no-]fail-on-violation`**: 默认情况下，如果发现违规，PMD 将以状态 4 退出。使用此选项禁用该行为。

- **`--[no-]progress`**: 启用/禁用实时分析进度的进度条指示器。

- **`--no-ruleset-compatibility`**: 禁用规则集兼容性过滤器，该过滤器默认情况下处于活动状态并尝试自动“修复”具有旧规则名称的旧规则集文件。

- **`-P, --property=<String=String>`**: 定义报告格式的属性的键值对。

- **`-r, --report-file=<reportFile>`**: 报告输出的文件路径。如果未指定此选项，报告将渲染到标准输出。

- **`-R, --rulesets=<rulesets>`**: 规则集 xml 文件的路径。路径可以引用应用程序类路径上的资源，是本地文件系统路径，也可以是 URL。

- **`--show-suppressed`**: 报告应显示已抑制的规则违规。

- **`--suppress-marker=<suppressMarker>`**: 指定 PMD 应忽略的字符串。

- **`-t, --threads=<threads>`**: 设置 PMD 使用的线程数，默认为 1。

- **`-u, --uri=<uri>`**: 用于源的数据库 URI。必须提供 --dir、--file-list 或 --uri 之一。

- **`--use-version=<languageVersion>`**: 解析源代码时 PMD 应使用的语言版本。

- **`-z, --relativize-paths-with=<relativizePathsWith>`**: 相对于其目录渲染在报告中的路径。此选项

对于这些命令选项，首先需要确定选项 --rulesets 规则配置文件，对于 pmd 规则有：

 - Best Practices：代码最佳实践相关规则
 - Code Style：代码风格相关规则
 - Design：代码设计相关规则
 - Documentation：代码文档相关规则
 - Error Prone：代码易出错相关规则 
 - Multithreading：代码多线程问题相关规则
 - Performance：代码性能问题相关规则
 - Security：代码潜在安全漏洞相关规则

定义 rulesets.xml 规则配置文件：

```xml
<?xml version="1.0"?>
<ruleset xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    name="Custom Rules"
    xmlns="http://pmd.sourceforge.net/ruleset/2.0.0"
    xsi:schemaLocation="http://pmd.sourceforge.net/ruleset/2.0.0 https://pmd.sourceforge.io/ruleset_2_0_0.xsd">

    <description>
        Custom ruleset for Android application
    </description>

    <!-- rule des :https://pmd.github.io/pmd-6.41.0/pmd_rules_java.html -->
    <exclude-pattern>.*/R.java</exclude-pattern>
    <exclude-pattern>.*/gen/.*</exclude-pattern>
    <!-- Your rules will come here -->
    <!--     代码风格 -->
    <rule ref="category/java/codestyle.xml" />
    <!--     代码注释和文档 -->
    <rule ref="category/java/documentation.xml" />
    <!--     代码设计 -->
    <rule ref="category/java/design.xml" />
    <!--     最佳实践 -->
    <rule ref="category/java/bestpractices.xml" />
    <!--     容易出现运行时错误 -->
    <rule ref="category/java/errorprone.xml" />
    <!--    多线程时问题-->
    <rule ref="category/java/multithreading.xml" />
    <!--    需要优化代码的地方-->
    <rule ref="category/java/performance.xml" />
    <!--    安全问题-->
    <rule ref="category/java/security.xml" />
</ruleset>
```
对于 pmd 静态代码检查，只需要有规则配置 -R ，输入文件列表 -d， 输出结果 -r ，输出结果报告文档格式 -f ，语言设置 --force-language，失败继续执行 --no-fail-on-violation 就可以：

```bash
使用 rulesets.xml 配置的规则，检查 file_path1, file_path2 指向的 java 文件，并把结果输出到 reports/detekt.html 

./pmd-bin-7.0.0-rc4/bin/pmd check -R rulesets.xml -d file_path1,file_path2 -r reports/pmd.html -f html --force-language=java  --no-fail-on-violation
```

### python 执行 detekt 和 pmd cli

新建立一个文件夹存放下载的 detekt-cli-1.23.4-all.jar 和 pmd-dist-7.0.0-rc4-bin.zip，目录结构：

```bash
python-lint-cli
  detect
    detekt-cli-1.23.4-all
    detekt.yml
  pmd
    pmd-dist-7.0.0-rc4
    rulesets.xml
```
假设文件夹命名为：python-lint-cli，python 封装脚本命名为：increment_detect.py，并把 python-lint-cli 和 increment_detect.py 放在同一个目录下。在 pytho中 获取 python-lint-cli 中 detekt 和 pmd 相关配置为：

```python
def get_pmd_cli_path():
    # or pmd.bat
    return get_cli_path('pmd')


def get_pmd_config_path():
    return get_cli_path('rulesets.xml')


def get_detekt_cli_path():
    # or detekt-cli.bat
    return get_cli_path('detekt-cli')


def get_detekt_config_path():
    return get_cli_path('detekt.yml')


def get_cli_path(target):
    """
      获取 pmd或detekt cli 路径
      :return: pmd或detekt cli 路径
      """
    # 获取当前执行的 Python 脚本文件的路径
    script_path = os.path.abspath(__file__)
    # 获取该文件所在的目录路径
    py_project_path = os.path.dirname(script_path)
    py_lint_cli_path = f"{py_project_path}{os.path.sep}python-lint-cli"
    for root, dirs, files in os.walk(py_lint_cli_path):
        for file in files:
            pmd_cli_path = os.path.join(root, file)
            if os.path.basename(pmd_cli_path) == target:
                return pmd_cli_path
    return None
```


第一步：同步目标分支

```python
def sync_branch(branch):
    """
    同步分支到最新代码
    :param branch: 分支名称
    :return: 检查结果
    """
    result = run_git_command(
        ['git', 'fetch', 'origin', branch])
    if result is None:
        return None
    result = run_git_command(
        ['git', 'checkout', branch])
    if result is None:
        return None
    result = run_git_command(
        ['git', 'pull', 'origin', branch])
    return result


def check_branch(target_branch, current_branch):
    """
    检查分支
    :param target_branch: 要比对的分支
    :param current_branch: 当前分支
    :return: 检查分支结果，True表示成功，否则失败
    """
    if sync_branch(target_branch) is None:
        print(f"Sync branch: {target_branch} Failed")
        return False
    if sync_branch(current_branch) is None:
        print(f"Sync branch: {current_branch} Failed")
        return False
    return True
```

第二步：比较 current_branch 和 target_branch，获取提交的文件相对路径列表

```python
def get_commit_file_path_set(target_branch, current_branch, author):
    """
    比对 branch，获取提交的文件相对路径列表
    :param target_branch: 要比对的分支
    :param current_branch: 当前分支
    :param author: git user.name
    :return: 提交的文件相对路径列表
    """
    try:
        if (target_branch == 'master' or target_branch.startswith('release')) and not current_branch.startswith(
                'release'):
            branch_command = f"{target_branch}..{current_branch}"
        else:
            branch_command = f"{current_branch}...{target_branch}"

        command = ['git', 'log', branch_command, '--author=' + author,
                   '--name-status', '--oneline']
        process = subprocess.Popen(command, stdout=subprocess.PIPE, stderr=subprocess.STDOUT, text=True)
        file_path_set = set()
        while True:
            output = process.stdout.readline()
            if output == '' and process.poll() is not None:
                process.kill()
                break
            if output:
                text = output.strip().replace("\t", "")
                if text.startswith('M') or text.startswith('A') or text.startswith('D'):
                    file_path = text[1:]
                    file_path_set.add(file_path)
                else:
                    # 记录重命名文件，需要移除
                    if output.strip().startswith('R'):
                        rename_file_path = output.strip().split('\t')[1]
                        file_path_set.discard(rename_file_path)
        if len(file_path_set) == 0:
            print(f"{' '.join(command)}: No commit files")
            return None
        filter_file_path_set = set()
        for file_path in file_path_set:
            file_in_current_branch = run_git_command(['git', 'ls-tree', current_branch, file_path])
            if file_in_current_branch.find('blob') >= 0:
                filter_file_path_set.add(file_path)
        return filter_file_path_set
    except subprocess.CalledProcessError as e:
        print(f"Error executing command: {e}")
        return None

```
第三步：获取提交的 java 和 kotlin 文件全路径，并以,分割

```python
def get_commit_file_full_path_list(project_path, file_path_list):
    """
    获取提交的文件的全路径列表，并以,分割
    :param project_path: 项目路径
    :param file_path_list: 提交的文件的相对路径列表
    :return: kotlin_file_list 表示 kotlin 文件的全路径列表，以 , 分割；java_file_list 表示 java 文件的全路径列表，以 , 分割
    """
    kotlin_file_list = []
    java_file_list = []
    for index in range((len(file_path_list))):
        if file_path_list[index].endswith('.kt'):
            kotlin_file_list.append(project_path + os.path.sep + file_path_list[index])
        if file_path_list[index].endswith('.java'):
            java_file_list.append(project_path + os.path.sep + file_path_list[index])
    return ','.join(kotlin_file_list), ','.join(java_file_list)
```
第四步：将 kotlin 文件交给 detekt 处理，将 java 文件交给 pmd 处理

- dekekt：`./detekt-cli-1.23.4/bin/detekt-cli -c detekt-config.yml -i file_path1,file_path2 -r html:reports/detekt.html`
- pmd：`./pmd-bin-7.0.0-rc4/bin/pmd check -R rulesets.xml -d file_path1,file_path2 -r reports/pmd.html -f html --force-language=java`

```
def make_report(project_path, kotlin_file_list, java_file_list):
    result = []
    result.append(make_detekt_report(project_path, kotlin_file_list))
    result.append(make_pmd_report(project_path, "java", java_file_list))
    return result


def make_detekt_report(project_path, kotlin_file_list):
    """
    生成 detekt html 文件报告
    :param project_path: 项目路径
    :param kotlin_file_list: kotlin 文件路径列表
    :return: detekt 静态代码分析结果报告地址
    """
    if len(kotlin_file_list) == 0:
        return ""
    report_dir = f"{project_path}{os.path.sep}build{os.path.sep}reports"
    if not os.path.exists(report_dir):
        os.makedirs(report_dir)
    report_path = f"{report_dir}{os.path.sep}detekt.html"
    if os.path.exists(report_path):
        os.remove(report_path)
    detekt_cli_path = get_detekt_cli_path()
    detekt_cli_arg_config = f"-c {get_detekt_config_path()}"
    detekt_cli_arg_report = f"-r html:{report_path}"
    detekt_cli_arg_input = f"-i {kotlin_file_list}"

    args = f"{detekt_cli_path} {detekt_cli_arg_config} {detekt_cli_arg_report} {detekt_cli_arg_input}"
    try:
        subprocess.run(args, shell=True, stdout=subprocess.PIPE, text=True)
    except subprocess.CalledProcessError:
        pass
    return report_path


def make_pmd_report(project_path, language, file_list):
    """
    生成 pmd html 文件报告
    :param project_path: 项目路径
    :param language: java 或 kotlin
    :param file_list: java 或 kotlin 文件路径列表
    :return: pmd 静态代码分析结果报告地址
    """
    if len(file_list) == 0:
        return ""
    report_dir = f"{project_path}{os.path.sep}build{os.path.sep}reports"
    if not os.path.exists(report_dir):
        os.makedirs(report_dir)
    report_path = f"{report_dir}{os.path.sep}pmd-{language}.html"
    if os.path.exists(report_path):
        os.remove(report_path)
    pmd_cli_path = f"{get_pmd_cli_path()} check"
    pmd_cli_arg_rule = f"-R {get_pmd_config_path()}"
    pmd_cli_arg_format = f"-f html"
    pmd_cli_arg_language = f"--force-language {language}"
    pmd_cli_arg_report = f"-r {report_path}"
    pmd_cli_arg_input = f"-d {file_list}"

    args = f"{pmd_cli_path} {pmd_cli_arg_rule} {pmd_cli_arg_format} {pmd_cli_arg_language} {pmd_cli_arg_report} {pmd_cli_arg_input}"
    try:
        subprocess.run(args, shell=True, stdout=subprocess.PIPE, text=True)
    except subprocess.CalledProcessError:
        pass
    return report_path
```
第五步：将上面步骤结合在一起

- 将 python 脚本与 python-lint-cli 放在一起


```python
if __name__ == "__main__":
    root_project_path = ''
    current_branch = ''
    target_branch = ''

    args = sys.argv[1:]
    if len(args) > 0:
        root_project_path = args[0]
    if len(args) > 1:
        current_branch = args[1]
    if len(args) > 2:
        target_branch = args[2]

    # 获取提交作者名字 user.name
    author = get_git_user()
    if author is None:
        exit(1)

    os.chdir(root_project_path)
    print(f"\ncd project: {root_project_path}")
    # 第一步：同步目标分支
    if not check_branch(target_branch, current_branch):
        exit(1)
    # 第二步：比较 current_branch 和 target_branch，获取提交的文件相对路径列表
    commit_file_path_set = get_commit_file_path_set(target_branch, current_branch, author)
    if commit_file_path_set is None or len(commit_file_path_set) == 0:
        exit(0)
    # 第三步：获取提交的文件全路径，并以,分割
    kotlin_file_list, java_file_list = get_commit_file_full_path_list(root_project_path, list(commit_file_path_set))
    # 第四步：将提交的文件全路径传给 detekt 和 pmd cli，生成 对应的 java 和 kotlin 静态代码检查 html 报告
    report_path_list = make_report(root_project_path, kotlin_file_list, java_file_list)
    for report_path in report_path_list:
        if len(report_path) == 0:
            continue
        print(f"Report File Path: {report_path}")
        open_file(report_path)
```
在命令行执行脚本：

```bash
python3 incrememt_lint.py /Users/wangjiang/Public/software/android-workplace/Demo release/7.63.0 release/7.62.0
```
输出结果，这里比较私密，就不展示了。

## copy-paste 重复代码检查

pmd 提供了检查粘贴复制代码-重复代码工具 cpd，这在  [Android 静态代码检查]({{site.url}}/2022/09/08/Android-Code-Analyze/)中也介绍过。

### cpd cli介绍

cpd 命令：
```bash
./pmd-bin-7.0.0-rc4/bin/pmd cpd --help
```
选项：
- **`--dir`**：指定要分析的源代码文件或包含源代码文件的目录。
- **`-D`、`-v`、`--debug`、`--verbose`**：调试模式，打印详细的调试信息。
- **`-e`、`--encoding`**：指定源代码文件的字符集编码，默认为UTF-8。
- **`--exclude`**：排除分析的文件。
- **`-f`、`--format`**：指定报告的格式，可以是**csv、text、xml**等。
- **`--file-list`**：指定包含要分析的文件列表的文件路径。
- **`-h`、`--help`**：显示帮助信息。
- **`--ignore-annotations`**：在比较文本时忽略语言注释。
- **`--ignore-identifiers`**：在比较文本时忽略类、方法、变量、常量等的名称。
- **`--ignore-literal-sequences`**：忽略文本中的文字序列。
- **`--ignore-literals`**：在比较文本时忽略数字和字符串等的字面值。
- **`--ignore-sequences`**：忽略标识符和文字序列。
- **`--ignore-usings`**：在C#中忽略使用指令。
- **`-l`、`--language`**：指定源代码的编程语言。
- **`--minimum-tokens`**：报告为重复的最小令牌长度。
- **`--[no-]fail-on-violation`**：默认情况下，如果发现违规行为，PMD将以状态4退出。使用`--no-fail-on-violation`禁用此选项，以便以状态0退出，仅写入报告。
- **`--no-skip-blocks`**：不跳过用`--skip-blocks-pattern`标记的代码块。
- **`--non-recursive`**：不扫描子目录。
- **`--skip-blocks-pattern`**：指定要跳过的代码块的模式。
- **`--skip-duplicate-files`**：忽略相同名称和长度的文件的多个副本。
- **`--skip-lexical-errors`**：跳过由于无效字符而无法进行标记化的文件，而不是报告错误。
- **`-u`、`--uri`**：源代码的数据库URI。
- **`-z`、`--relativize-paths-with`**：在报告中渲染相对路径的路径。

对于 pmd 检查重复代码，只需要有输入文件列表 --dir ，输出结果报告文档格式 -f ，语言设置 --language，token 设置 --minimum-tokens，失败继续执行 --no-fail-on-violation 就可以：
```bash
检查 file_path1, file_path2 指向的 java 文件是否有重复代码，并把文本结果

./pmd-bin-7.0.0-rc4/bin/pmd cpd --dir=file_path1,file_path2 -f text --language=java  --minimum-tokens=java --no-fail-on-violation
```
由于输出报告的格式只有 csv、 text、xml，所以还需使用 python 将结果重新输出到 html 中。

### python 执行 cpd

新定义封装脚本 find_duplicated_code.py，并将其与 python-lint-cli 放在一起，期望执行命令：

```bash
python3 find_duplicated_code.py project_path current_branch(可选)
```

第一步：同步分支，与上面一样

第二步：获取项目路径下的 java 和 kotlin 文件

```python
def get_file_path_dict(project_path, languages, exclude_file_dirs):
    """
    获取项目src文件夹下的java文件路径列表，以,分割
    :param project_path: 项目路径
    :param languages: 语言
    :param exclude_file_dirs: 不包含的目录
    :return: 文件路径列表
    """
    file_path_dict = {}
    for language in languages.keys():
        file_path_dict[language] = []
    for root, dirs, files in os.walk(project_path):
        for file in files:
            file_path = os.path.join(root, file)
            exclude = False
            for exclude_file_dir in exclude_file_dirs:
                if file_path.find(exclude_file_dir) > 0:
                    exclude = True
                    break
            if exclude:
                continue
            for language, file_suffix in languages.items():
                if file_path.endswith(file_suffix):
                    file_path_dict[language].append(file_path)
                    break

    return file_path_dict
    
# 第二步：获取文件
default_language = {'java': '.java', 'kotlin': '.kt', 'python': '.py', 'swift': '.swift'}
exclude_file_dirs = ['venv', 'build', 'gen']
file_path_dict = get_file_path_dict(root_project_path, default_language, exclude_file_dirs)
```

第三步：将 java 文件交给 pmd 处理
- `./pmd-bin-7.0.0-rc4/bin/pmd cpd --dir=file_path1,file_path2 -f text --language=java  --minimum-tokens=java --no-fail-on-violation`

```python
def make_cpd_report(project_path, pmd_cli_path, file_list, language):
    """
    生成重复代码报告
    :param project_path: 项目路径
    :param pmd_cli_path: detekt 和 pmd cli 路径
    :param file_list: java 文件路径地址
    :return: html 报告地址
    """
    build_dir = f"{project_path}{os.path.sep}build"
    if os.path.exists(build_dir) and os.path.isfile(build_dir):
        build_dir = f"{project_path}{os.path.sep}build-py"
    report_dir = f"{build_dir}{os.path.sep}reports"
    if not os.path.exists(report_dir):
        os.makedirs(report_dir)
    report_path = f"{report_dir}{os.path.sep}pmd-cpd-{language}.html"
    if os.path.exists(report_path):
        os.remove(report_path)
    cpd_cli_arg_cpd = f"{pmd_cli_path} cpd"
    cpd_cli_arg_token = "--minimum-tokens=120"
    cpd_cli_arg_dir = f"--dir={file_list}"
    cpd_cli_arg_language = f"--language={language}"
    cpd_cli_arg_format = "-f text"
    cpd_cli_arg_no_fail = "--no-fail-on-violation"
    cpd_cli_arg_encoding = "--encoding=UTF-8"
    cpd_cli_arg_ignore_annotations = "--ignore-annotations"
    cpd_cli_arg_skip_lexical_errors = " --skip-lexical-errors"
    args = f"{cpd_cli_arg_cpd} {cpd_cli_arg_token} {cpd_cli_arg_dir} {cpd_cli_arg_language} {cpd_cli_arg_format} {cpd_cli_arg_no_fail} {cpd_cli_arg_encoding} {cpd_cli_arg_ignore_annotations} {cpd_cli_arg_skip_lexical_errors}"
    try:
        output = subprocess.run(args, shell=True, stdout=subprocess.PIPE, text=True)
        lines = output.stdout.split('\n')
        content = ""
        count = 1
        for line in lines:
            if line.startswith('='):
                count += 1
                content += "<hr>"
                continue
            content += line + "</br>"
        if count > 1:
            count += 1
        title = f"{project_path} Duplicate code: {count} found"
        html_content = f"""
           <!DOCTYPE html>
           <html lang="en">
           <head>
               <meta charset="UTF-8">
               <style>
                   body {{
                       font-family: 'Arial', sans-serif;
                       background-color: #272822;
                       color: #f8f8f2;
                       margin: 20px;
                   }}
                   pre {{
                       white-space: pre-wrap;
                       font-size: 14px;
                       line-height: 1.5;
                       background-color: #1e1e1e;
                       padding: 20px;
                       border: 1px solid #333;
                       border-radius: 5px;
                       overflow-x: auto;
                   }}
                   .header {{
                       color: #66d9ef;
                   }}
                   .bordered-div {{
                       border: 1px solid #000;
                       padding: 10px;
                   }}
               </style>
           </head>
           <body>
               <h1>{title}</h1>
               <pre>
                  {content}
               </pre>
           </body>
           </html>
           """
        with open(report_path, 'w') as html_file:
            html_file.write(html_content)
            html_file.close()
    except subprocess.CalledProcessError:
        pass
    return report_path
```
第四步：将上面步骤结合在一起

- 将 python 脚本与 python-lint-cli 放在一起

```python
if __name__ == "__main__":
    root_project_path = ''
    current_branch = ''

    args = sys.argv[1:]
    if len(args) > 0:
        root_project_path = args[0]
    if len(args) > 1:
        current_branch = args[1]

    # 第一步：同步分支
    os.chdir(root_project_path)
    if len(current_branch) > 0 and not check_branch(current_branch):
        exit(1)
    # 第二步：获取文件
    default_language = {'java': '.java', 'kotlin': '.kt', 'python': '.py', 'swift': '.swift'}
    exclude_file_dirs = ['venv', 'build', 'gen']
    file_path_dict = get_file_path_dict(root_project_path, default_language, exclude_file_dirs)
    if len(file_path_dict) == 0:
        exit(0)
    for language, file_path_list in file_path_dict.items():
        if len(file_path_list) == 0:
            continue
        report_path = make_cpd_report(root_project_path, file_path_list, language)
        print(f"Report File Path: {report_path}")
        open_file(report_path)
```
在命令行执行脚本：

```bash
python3 find_duplicated_code.py /Users/wangjiang/Public/software/android-workplace/Demo release/7.63.0(可选)
```
输出结果，示例（以本项目[在 Android 开发中使用 Python 脚本](https://github.com/WJRye/android-script)为例，python 重复代码检查）：

![pmd-cpd-python.png]({{site.url}}/images/posts/2024-02-19-Python-Code-Analyze/p1.jpeg)

## 总结
利用 python 封装 git 命令，获取某个版本某个作者的所有提交更改文件，再利用 python 封装 detekt 和 pmd cli 命令，然后对更改文件做静态代码检查，这就实现了增量静态代码检查。python 封装的静态代码检查脚本 比使用 gradle 集成静态代码检查plugin更具有弹性，它能脱离多仓库多模块多业务开发的项目环境限制，对大项目下的小业务团队也可以轻松自由的执行。

