---
layout: post
title: Python 封装 Git 命令
categories: [Python]
description: Python 封装 Git 命令，查看某个版本某个作者的所有提交更改，或查看某个提交第一次出现的版本。
keywords: Python, Git
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

## 执行 git 命令

在日常的 Android 项目开发中，一般只会使用到： `git add, git commit, git push, git pull, git rebase, git merge, git diff`等常规命令。但是使用 git 命令，还可以做一些特别的事情，比如查看某个版本某个作者的所有提交更改，方便自己或其他人进行 code review；比如查看某个提交第一次出现的版本，方便排查问题。下面将介绍使用 python 封装 git 命令。

### 查看某个版本某个作者的所有提交更改

在某个版本迭代中，不管是单人还是多人开发，如果想在 mr 之前 或 之后，或者 release 之前 或 之后，随时查看自己本次版本迭代中的所有提交更改（随时对自己编写的代码进行自我 code review），现只能使用 git 命令：git log branch1...branch2 --author=wangjiang --name-status --oneline 等进行简单查看，而且较麻烦。我们期望有一个工具，能够展示自己当前分支提交的所有代码更改内容。现利用 python 可以实现这个工具。

虽然创建的 mr 也能查看自己在本版本迭代中提交的所有代码更改，但是在本次迭代中提交了多个 mr 或者过了很久也想查看自己在某个版本中的更改时，用 mr 查看就很不方便。

#### git 命令介绍

要比对两个分支（branch）的提交历史，可以使用 git log 命令并指定不同的分支名称。

##### 比对两个具体的分支

```bash
git log branch1..branch2
```

该命令只显示 branch2 相对于 branch1 的提交历史，也就是：

- 显示在 branch2 中而不在 branch1 中的提交历史
- 只会显示 branch2 中相对于 branch1 的新增提交
- 不包括 branch1 中相对于 branch2 的新增提交

这个命令用于比对开发分支与拉出开发分支的主分支，比如从 master 分支 拉出 feature/7.63.0-wangjiang 分支，那么使用：
`git log master..feature/7.63.0-wangjiang --author=wangjiang --name-status --oneline` 可以查看自己在  feature/7.63.0-wangjiang 分支上的所有提交记录。

##### 显示共同的祖先以及两个分支的不同

```bash
git log branch1...branch2
```

显示两个分支的共同祖先以及它们之间的不同，也就是：

- 显示两个分支之间的差异，包括它们各自相对于共同祖先的所有提交
- 显示两个分支的共同祖先以及它们之间的不同
- 如果两个分支有共同的提交，... 语法将显示两个分支最新的共同提交之后的提交

这个命令用于比对两个 release 分支，比如当前要发布的版本分支 release/7.63.0，上一个发布的版本分支 release/7.62.0，那么使用：
`git log release/7.63.0...release/7.62.0 --author=wangjiang --name-status --oneline` 可以查看自己在 release/7.63.0 分支上的**所有提交记录，包含本次迭代提交的所有 feature**。

如果上面比对开发分支与拉出开发分支的主分支使用 ... ：`git log master...feature/7.63.0-wangjiang --author=wangjiang --name-status --oneline` ，那么其它 feature 分支合并到 master 的提交记录，也会显示。

另外，对于 `git log branch1..branch2 ` 或 `git log branch1...branch2` 添加 `--name-status --oneline` 会显示：

```bash
d7a42a90c feat:--story=1004796 --user=王江 Python封装 git 命令需求 
M       music/src/main/java/com/music/upper/module/fragment/MusicAlbumPickerFragment.kt
A       music/src/main/res/drawable-xxhdpi/ic_draft.png
M       music/src/main/res/layout/music_album_choose_container_fragment.xml
D       music/src/main/res/drawable-xxhdpi/ic_save.png
R098   music/src/main/java/com/music/upper/module/fragment/MusicVideoPickerFragment.kt music/src/main/java/com/music/upper/module/fragment/MusicVideoListPickerFragment.kt
```

其中 M 表示修改文件，A 表示添加文件，D 表示删除文件，R098 表示重命名文件。

##### 检查文件是否存在

上面提交记录里会有修改、添加、删除、重命名文件，那么需要查看某个分支上某个文件是否存在。

```bash
git ls-tree branch file-path
```

该命令这将列出 branch 分支上指定路径的文件信息。如果文件存在，将显示相关信息；如果文件不存在，则命令不会有输出。

##### 显示指定分支上指定文件的详细更改信息

```bash
git show branch:file-path
```

这个命令会显示指定分支上指定文件的详细更改信息，包括修改的内容。如果文件存在，将显示文件内容信息；如果文件不存在，则命令会输出错误信息。

---

了解了上面的 git 命令后，使用 python 将这些命令组合，并输出自己当前分支提交的所有代码更改内容的 html 文档报告。

期望执行的 python 脚本命令：

```bash
 python3 diff_branch.py android_project_path current_branch target_branch

 例如：
 python3 diff_branch.py /Users/wangjiang/Public/software/android-workplace/Demo release/7.63.0 release/7.62.0
 python3 diff_branch.py /Users/wangjiang/Public/software/android-workplace/Demo feature/wangjiang master
```

首先定义一个执行 git 命令的基础方法：：

```python
def run_git_command(command):
    """
    :param command: 实际相关命令
    :return: 执行命令结果
    """
    try:
        result = subprocess.run(command, check=True, text=True, capture_output=True, encoding='utf-8')
        return result.stdout
    except subprocess.CalledProcessError as e:
        print(f"Error executing command: {e}")
        return None
```

第一步：同步目标分支

- 获取远程分支：`git fetch origin branch`
- 切换到分支：`git checkout branch`
- 更新分支：`git pull origin branch`

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
    :param current_branch: 当前分支
    :param target_branch: 要比对的分支
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

第二步：获取自己的 git 账号名称

- 获取 git 账号名称：`git config --get user.name`

```python
def get_git_user():
    """
    获取自己的 git user name
    :return: git 账户名称
    """
    return run_git_command(['git', 'config', '--get', 'user.name']).strip()
```

第三步：比较 current_branch 和 target_branch，获取提交的文件列表

- 比对分支：git log branch1..branch2 或 git log branch1...branch2

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
        # 如果当前开发分支与master或release分支比对，使用 git log master..feature
        if (target_branch == 'master' or target_branch.startswith('release')) and not current_branch.startswith(
                'release'):
            branch_command = f"{target_branch}..{current_branch}"
        else:
            # 否则都是用 git log branch1...branch2
            branch_command = f"{current_branch}...{target_branch}"

        command = ['git', 'log', branch_command, f"--author={author}",
                   '--name-status', '--oneline']
        process = subprocess.Popen(command, stdout=subprocess.PIPE, stderr=subprocess.STDOUT, text=True)
        file_path_list = set()
        rename_file_path_list = set()
        while True:
            output = process.stdout.readline()
            if output == '' and process.poll() is not None:
                process.kill()
                break
            if output:
                text = output.strip().replace("\t", "")
                if text.startswith('M') or text.startswith('A') or text.startswith('D'):
                    file_path = text[1:]
                    file_path_list.add(file_path)
                else:
                    # 重命名文件
                    if output.strip().startswith('R'):
                        rename_file_path = output.strip().split('\t')[1]
                        rename_file_path_list.add(rename_file_path)
        if len(file_path_list) == 0:
            print(f"{' '.join(command)}: No commit files")
            return None
        for rename_file_path in rename_file_path_list:
            file_path_list.remove(rename_file_path)
        return file_path_list
    except subprocess.CalledProcessError as e:
        print(f"Error executing command: {e}")
        return None
```

第四步：根据文件列表获取文件内容

- 查看分支上是否有该文件：`git ls-tree branch-name file-path`
- 显示分支上该文件内容：`git show branch:file-path`

```python
def get_commit_content(commit_file_path_set, target_branch, current_branch):
    """
    获取提交的内容
    :param commit_file_path_set: 提交的文件相对路径列表
    :param target_branch: 要比对的分支
    :param current_branch: 当前分支
    :return: 要比对的分支内容，当前分支内容
    """
    target_content_lines = []
    current_content_lines = []
    for file_path in commit_file_path_set:
        try:
            file_in_target_branch = run_git_command(['git', 'ls-tree', target_branch, file_path])
            if file_in_target_branch.find('blob') >= 0:
                target_content = run_git_command(
                    ['git', 'show', target_branch + ":" + file_path])
                if target_content is not None:
                    target_content_lines += target_content.splitlines()
        except UnicodeDecodeError as e:
            target_content_lines += [file_path + '\n']
        try:
            file_in_current_branch = run_git_command(['git', 'ls-tree', current_branch, file_path])
            if file_in_current_branch.find('blob') >= 0:
                current_content = run_git_command(
                    ['git', 'show', current_branch + ":" + file_path])
                if current_content is not None:
                    current_content_lines += current_content.splitlines()
        except UnicodeDecodeError as e:
            current_content_lines += [file_path + '\n']
    return target_content_lines, current_content_lines
```

第五步：生成 html 报告文件

- 生成 html 报告文件：difflib.HtmlDiff

```python
def make_html_file(project_path, target_branch_content, current_branch_content, target_branch, current_branch, author):
    """
    生成 html 文件报告
    :param project_path: 项目路径
    :param target_branch_content: 要比对的分支内容
    :param current_branch_content:  当前分支内容
    :param target_branch: 要比对的分支
    :param current_branch: 当前分支
    :param author: git user.name
    :return: html 文件报告路径
    """
    html_report_dir = f"{project_path}{os.path.sep}build{os.path.sep}reports{os.path.sep}diff{os.path.sep}{author}"
    if not os.path.exists(html_report_dir):
        os.makedirs(html_report_dir)
    html_file_path = f"{html_report_dir}{os.path.sep}{current_branch.replace('/', '_')}-diff-{target_branch.replace('/', '_')}.html"
    d = difflib.HtmlDiff(wrapcolumn=120)
    diff_html = d.make_file(target_branch_content, current_branch_content, target_branch, current_branch, context=True)
    if os.path.exists(html_file_path):
        os.remove(html_file_path)
    with open(html_file_path, 'w', encoding='utf-8') as html_file:
        html_file.write(diff_html)
        html_file.close()
    print(f"{project_path} Html Report Path: {html_file_path}")
    return html_file_path
```

第六步：在浏览器中打开 html 文档报告

```python
def open_file(file_path):
    """
    在电脑上打开截屏文件
    :param file_path: 电脑上的截屏文件地址
    """
    system = platform.system().lower()
    if system == "darwin":  # macOS
        subprocess.run(["open", file_path])
    elif system == "linux":  # Linux
        subprocess.run(["xdg-open", file_path])
    elif system == "windows":  # Windows
        subprocess.run(["start", file_path], shell=True)
    else:
        print("Unsupported operating system.")
```

第七步：将上面步骤组合在一起执行

```python
if __name__ == "__main__":
    args = sys.argv[1:]
    if len(args) > 0:
        project_path = args[0]
    if len(args) > 1:
        current_branch = args[1]
    if len(args) > 2:
        target_branch = args[2]

    os.chdir(project_path)
    # 第一步：同步目标分支
    if not check_branch(target_branch, current_branch):
        exit(1)
    # 第二步：获取自己的git账户名称
    author = get_git_user()
    if author is None:
        exit(1)
    # 第三步：比较 current_branch 和 target_branch，获取提交的文件列表
    commit_file_path_set = get_commit_file_path_set(target_branch, current_branch, author)
    if commit_file_path_set is None or len(commit_file_path_set) == 0:
        exit(0)
    # 第四步：根据文件列表获取文件内容
    last_branch_content, current_branch_content = get_commit_content(commit_file_path_set, target_branch,
                                                                     current_branch)
    # 第五步：生成 html 报告文件
    report_html_file_path = make_html_file(project_path, last_branch_content, current_branch_content, target_branch,
                                           current_branch, author)
    # 第六步：打开 html 报告文件
    open_file(report_html_file_path)
```

将上面的代码封装成 diff_branch.py，在命令行执行：

```python
python3 diff_branch.py /Users/wangjiang/Public/software/android-workplace/Demo  release/7.63.0 release/7.62.0
```

输出结果，这里比较私密，就不展示了。

### 查看某个提交第一次出现的 release 分支

在日常的 Android 项目开发中，如果想排查问题，或查看 feature 在哪个版本上线的，那么查看某个 commit 第一次出现的 release 分支，能够辅助你得到更多有用的信息。

第一步：查找包含 commit id 的所有分支名称

- 查找包含 commit id 的所有分支：`git branch --contains commit-id -all`

```python
def find_commit(commit_id):
    """
    查找包含 commit id 的所有分支名称
    :param commit_id: commit id 值
    :return: 分支列表
    """
    result = run_git_command(
        ['git', 'branch', '--contains', commit_id, '--all'])
    if result is not None:
        return result.splitlines()
    return None
```

第二步：找到 commit id 第一次出现的 release 分支

```python
def compare_versions(version1, version2):
    """
    比较版本号
    :param version1: 7.63.0
    :param version2: 7.64.0
    :return: 如果 version1<version2，返回-1；如果version1>version2，返回1；如果version1=version2，返回0
    """
    parts1 = list(map(int, version1.split('.')))
    parts2 = list(map(int, version2.split('.')))

    length = max(len(parts1), len(parts2))

    for i in range(length):
        part1 = parts1[i] if i < len(parts1) else 0
        part2 = parts2[i] if i < len(parts2) else 0

        if part1 < part2:
            return -1
        elif part1 > part2:
            return 1

    return 0


def find_min_release_branch(branch_list):
    """
    筛选出版本最低的 release branch，也就是找到 commit id 第一次出现的 release branch
    :param branch_list: 分支列表
    :return: 版本最低的 release branch
    """
    min_version_name = None
    min_branch = None
    release_prefix = 'remotes/origin/release/'
    for branch in branch_list:
        index = branch.find(release_prefix)
        if index >= 0:
            version_name = branch[index + len(release_prefix):]
            if min_version_name is None:
                min_version_name = version_name
                min_branch = branch
            else:
                if compare_versions(min_version_name, version_name) > 0:
                    min_version_name = version_name
                    min_branch = branch

    return min_branch.strip()
```

 第三步：获取 commit 信息

- 获取 commit 信息：`git show commit-id`

```python
def get_commit_info(commit_id):
    """
    获取提交的信息
    :param commit_id: commit id值
    :return: commit 信息，包含文件更改信息
    """
    return run_git_command(
        ['git', 'show', commit_id])
```

第四步：生成 html 文档报告

```python
def make_html_file(project_path, commit_id, title, content):
    """
    生成 html 文件报告
    :param project_path: 项目路径
    :param commit_id: commit id值
    :param title: html 文档标题
    :param content: html 文档内容
    :return: html 文件报告路径
    """
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
    html_report_dir = f"{project_path}{os.path.sep}build{os.path.sep}reports{os.path.sep}diff{os.path.sep}commit_id"
    if not os.path.exists(html_report_dir):
        os.mkdir(html_report_dir)
    html_file_path = f"{html_report_dir}{os.path.sep}{commit_id}.html"
    if os.path.exists(html_file_path):
        # 如果文件存在，删除文件
        os.remove(html_file_path)
    with open(html_file_path, 'w') as html_file:
        html_file.write(html_content)
        html_file.close()
    print(f"Html Report Path: {html_file_path}")
    return html_file_path
```

第五步：在浏览器中打开 html 文档报告

```python
def open_file(file_path):
    """
    在电脑上打开截屏文件
    :param file_path: 电脑上的截屏文件地址
    """
    system = platform.system().lower()
    if system == "darwin":  # macOS
        subprocess.run(["open", file_path])
    elif system == "linux":  # Linux
        subprocess.run(["xdg-open", file_path])
    elif system == "windows":  # Windows
        subprocess.run(["start", file_path], shell=True)
    else:
        print("Unsupported operating system.")
```

第六步：将上面步骤组合在一起执行

```python
if __name__ == "__main__":
    args = sys.argv[1:]
    if len(args) > 0 and os.path.exists(args[0]):
        project_path = args[0]
    if len(args) > 1 :
        commit_id = args[1]

    os.chdir(project_path)
    # 第一步：查找包含 commit id 的所有分支名称
    branch_list = find_commit(commit_id)
    # 第二步：找到 commit id 第一次出现的 release 分支
    min_release_branch = find_min_release_branch(branch_list)
    title = f"<p>Project: {project_path}</p>The commit id: {commit_id} first appears in the release branch: {min_release_branch}"
    # 第三步：获取 commit 信息
    content = get_commit_info(commit_id)
    # 第四步：生成 html 文档报告
    html_file_path = make_html_file(project_path, commit_id, title, content)
    # 第五步：打开 html 文档报告
    open_file(html_file_path)
```

将上面的代码封装成 find_commit.py，在命令行执行：

```python
python3 find_commit.py /Users/wangjiang/Public/software/android-workplace/Demo 00b9d42d70
```

输出结果例如：

![截屏2024-01-23 10.03.26.png](/images/posts/2024-01-23-Python-Git-Intro/p1.jpeg)

### 小结

使用 python 执行相关 git 命令，主要是生成可视化的 html 文档报告。比对分支操作，在开发 feature 合并到主分支前，可以查看自己当前分支提交的所有代码更改内容；在本迭代版本 release 前，可以反复查看自己的所有更改，进行代码 double check，防止出现线上 bug。在排查问题或者代码回溯中，可以快速找到 commit id 第一次出现的 release 版本，得到有用关键信息。总之，利用 python 组合 git 命令，可以在开发中做很多意想不到的事情。

*另外，没有提供上面的完整的代码，但是依照步骤去做，就可以完全实现。*

---

写在最后，**使用 python 不止可以封装 adb, gradle, git 命令，还可以做 json 比对，代码静态分析(利用detekt,pmd等的cli)，下线或升级某个库查看库在项目中的代码分布情况，业务和技术指标可视化报告，查看pb文件，用户日志定制化分析等**。学习 python，对于日常 Android 开发，非常有用，能帮助省去很多琐碎时间。

**下一篇将介绍 Python 封装 detekt 和 pmd 命令，做增量代码检查**

**后续会将完整代码放到 github。**
