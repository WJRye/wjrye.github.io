---
layout: post
title: Python 封装 adb 命令
categories: [Python]
description: 使用 Python 脚本封装常用的 adb 命令。
keywords: Python, adb
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
topmost: false
---

## 引言

在日常的 Android 项目开发中，我们通常会使用 adb 命令来获取连接设备的内存、屏幕、CPU等信息，也会使用 gradle 命令来获取项目构建相关的 projects、tasks、dependencies等信息，还会使用 git 命令来获取代码 commit、log、diff 等信息。这些信息的获取，每次都在command 中输入相关命令进行操作（有时命令记不住，还需要查询一下），重复的操作让人感到厌倦和疲乏。现在，可以尝试使用 python 来简化这一部分工作，将常用的执行命令封装到 python 脚本中，每次想要获取某个信息时，就直接执行相关 python 脚本。这样就可以省去开发中的细碎工作。（将脚本一次写好，使用到退休:)）



github 项目地址：[在 Android 开发中 使用 Python 脚本](https://github.com/WJRye/android-script)

---

## python 准备

使用 Python 脚本封装常用的 adb、gradle、git 命令，首先需要对 python 有一定的基本了解。

### 学习 python 基础

如果对 python 的 基础概念不是很了解，可以通过[Python 基础教程](https://www.runoob.com/python/python-tutorial.html)快速学习。只需掌握 python 基本的语法、数据结构、函数即可。

### 使用 subprocess 模块

`subprocess` 模块是 Python 中执行外部命令的标准库。通过该模块，可以执行 adb、gradle、git 等命令，以及获取命令输出结果。

```python
import subprocess

result = subprocess.run(['adb', 'devices'], capture_output=True, text=True, check=True)
print(result.stdout)
```

将执行命令的操作简单地封装成一个方法：

```python
def run_command(command):
    try:
        result = subprocess.run(command, check=True, text=True, capture_output=True)
        return result.stdout
    except subprocess.CalledProcessError as e:
        print(f"Error executing command: {e}")
        return None

result = run_command(['adb', 'devices'])
print(result)
```

如果要实时打印执行命令的每一行结果，可以使用 `subprocess.Popen` 方法：

```python
def run_command_process(command):
    try:
        process = subprocess.Popen(command, stdout=subprocess.PIPE, stderr=subprocess.STDOUT, text=True)
        while True:
            output = process.stdout.readline()
            if output == '' and process.poll() is not None:
                print("stop")
                break
            if output:
                print(output.strip())

        return process.returncode
    except subprocess.CalledProcessError as e:
        print(f"Error executing command: {e}")
        return None
```

### 切换工作目录路径

使用 `os` 模块中的 `chdir` 函数可以切换当前工作目录的路径：

```python
import os

os.chdir(current_project_path)
```

---

## 执行 adb 命令

在日常的 Android 项目开发中，通常使用 adb 命令来获取屏幕、设备、应用程序等信息。

在之前的一篇文章：[adb常用命令]({{site.url}}/2018/12/04/Adb-Command-Intro/)中介绍了 adb 命令相关的基本信息，现在利用 python 对这些操作命令做一些简化或组合。

首先定义一个执行 adb 命令的基础方法：

```python
def run_adb_command(adb_path='adb', command=None):
    """
    :param adb_path: adb 工具路径，如果在系统环境中配置了adb路径，则不需要传递，否则需要
    :param command: 实际相关命令
    :return: 执行命令结果
    """
    if command is None:
        command = []
    try:
        command.insert(0, adb_path)
        return subprocess.run(command, check=True, text=True, capture_output=True, encoding='utf-8')
    except subprocess.CalledProcessError as e:
        print(f"Error executing command: {e}")
        return None
```

### 操作组合-截屏

截屏算是 Android 开发中的一个频繁操作。在通常情况下，截屏操作步骤为：在设备上点击截屏 &rarr; 找到设备上存储的截屏文件地址 &rarr; 使用 adb pull 将截屏文件保存到电脑上 &rarr; 在电脑上打开截屏文件（或者使用 Android Studio 的 Logcat 面板截屏）。如果使用 pythoh，那么就可以将这几个步骤组合，然后写成一个 pythoh 脚本。

第一步：截屏并保存到设备

- 获取设备外部存储根路径命令：`adb shell echo '$EXTERNAL_STORAGE'`

- 截屏命令：`adb shell screencap -p screen_cap_file_path`，screen_cap_file_path为截屏文件地址，该地址和名称可以自定义

- 创建文件目录：`adb shell mkdir -p file_dir`，file_dir 为文件目录
  
  ```python
  def screen_cap(adb_path='adb'):
   """
   截屏并保存到设备外部存储卡 /Pictures/Screenshots 目录下，截屏文件名为规则为Screenshot_年月日_时分秒.png，例如：Screenshot_20220502_175425.png
   :param adb_path: adb 工具路径
   :return: 截屏文件路径和名称
   """
   current_time = time.strftime("%Y%m%d_%H%M%S", time.localtime())
   screen_cap_name = 'Screenshot_' + current_time + '.png'
   # 获取手机外部存储目录
   external_result = run_adb_command(adb_path, ['shell', "'echo'", '$EXTERNAL_STORAGE'])
   external_dir = external_result.stdout.strip()
   android_file_separator = '/'
   screen_cap_file_dir = external_dir + android_file_separator + 'Pictures' + android_file_separator + 'Screenshots'
   # 创建文件夹 adb shell mkdir -p file_dir
   run_adb_command(adb_path, ['shell', 'mkdir', '-p', screen_cap_file_dir])
   screen_cap_file_path = screen_cap_file_dir + android_file_separator + screen_cap_name
   screen_cap_result = run_adb_command(adb_path, ['shell', 'screencap', '-p', screen_cap_file_path])
   if screen_cap_result.returncode == 0:
       return [screen_cap_file_path, screen_cap_name]
   else:
       print(screen_cap_result.stderr)
       return ["", ""]
  ```

第二步：保存截屏文件到电脑

- 保存截屏文件到电脑命令：`adb pull src_file_path des_file_path`，src_file_path 为截屏文件在设备上的地址，des_file_path  为截屏文件保存到电脑上的地址。

```python
def pull_screen_cap(src_file_path, des_file_path, adb_path='adb'):
    """
    将截屏文件保存到电脑上
    :param src_file_path: 设备截屏文件地址
    :param des_file_path: 电脑上文件地址
    :param adb_path: adb 工具路径
    :return: ture表示成功，否则失败
    """
    pull_screen_cap_result = run_adb_command(adb_path, ['pull', src_file_path, des_file_path])
    return pull_screen_cap_result.returncode == 0


def get_desktop_path(screen_cap_name):
    """
    获取保存到电脑上的截屏文件路径
    :param screen_cap_name: 截屏文件名称
    :return: 电脑上桌面中的截屏文件路径
    """
    desktop_path = os.path.join(os.path.expanduser("~"), "Desktop")
    return desktop_path + os.path.sep + screen_cap_name
```

第三步：在电脑上打开截屏文件

- windows 打开文件命令：`start file_path`
- mac 打开文件命令：`open file_path`

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

将上面步骤组合在一起：

```python
if __name__ == '__main__':
    screen_cap_file_path, screen_cap_name = screen_cap()
    if len(screen_cap_file_path) > 0:
        des_file_path = get_desktop_path(screen_cap_name)
        result = pull_screen_cap(screen_cap_file_path, des_file_path)
        if result:
            print(f"Screenshot path: {des_file_path}")
            open_file(des_file_path)
        else:
            print("Failed to take a screenshot")
```

将上面的代码放到 screencap.py 中，在需要截屏的时候，执行 `python3 screencap.py` 脚本，就可以自动截屏并在电脑上打开截屏文件。

### 操作简化-基础信息

#### 获取设备上的三方应用程序的 Main Activity（启动 Activity）信息

如果直接使用 adb 命令，获取设备上的三方应用程序的 Main Activity（启动 Activity）信息，是非常繁琐的。利用 python，可以将操作简化，而且将信息格式化。

第一步：获取三方应用程序包名

- 获取设备上安装的三方应用程序列表命令：`adb shell pm list packages -3 -f`
- 利用正则表达式找到应用程序包名：`r"base.apk=(\S+)"`

```python
def get_third_party_app_activities(adb_path='adb'):
    """
    获取设备上的三方应用程序的 main activity
    :param adb_path: adb 工具的路径
    :return:三方应用程序的 main activity 信息
    """
    result = run_adb_command(adb_path, ['shell', 'pm', 'list', 'packages', '-3', '-f'])
    third_party_app_main_activities = []
    if result is not None:
        # 使用正则表达式提取包名
        package_names = re.findall(r"base.apk=(\S+)", result)
        print(package_names)
        # 获取每个应用程序的主活动
        for package_name in package_names:
            activity = get_main_activity(package_name)
            if activity:
                third_party_app_main_activities.append(f"Package: {package_name}, Main Activity: {activity}")
    return '\n'.join(third_party_app_main_activities)
```

第二步：通过应用程序包名获取 Main Activity

- 获取应用程序 Main Activity 命令：`adb shell dumpsys package package_name | grep -A 1 MAIN`

```python
def get_main_activity(package_name, adb_path='adb'):
    """
    根据包名获取 main activity
    :param package_name: 应用程序包名
    :param adb_path: adb 工具的路径
    :return: main activity 信息，例如：com.tencent.mm/.ui.LauncherUI
    """
    try:
        result = run_adb_command(adb_path,
                                  ['shell', 'dumpsys', 'package', package_name, '|', 'grep', '-A', '1', 'MAIN'])
        if result is not None:
            # 使用正则表达式提取 main activity
            text = result.strip()
            end_flag = 'filter'
            end_index = text.index(end_flag)
            android_file_separator = '/'
            start_flag = package_name + android_file_separator
            start_index = text.index(start_flag, 0, end_index)
            main_activity = text[start_index:end_index]
            if main_activity.startswith(package_name):
                return main_activity
            else:
                print("Error extracting main activity: index-" + package_name)
                return None
        else:
            print("Error extracting main activity: code-" + package_name)
            return None
    except AttributeError:
        print("Error extracting main activity: caught-" + package_name)
```

执行上面代码可以看到：

```python
if __name__ == "__main__":
    print(get_third_party_app_activities())

输出结果：
Package: com.tencent.mm, Main Activity: com.tencent.mm/.ui.LauncherUI 
Package: com.sangfor.vpn.client.phone, Main Activity: com.sangfor.vpn.client.phone/.WelcomeActivity 
Package: com.huawei.cloud, Main Activity: com.huawei.cloud/.wi.WIActivity 
Package: com.meishe.myvideoapp, Main Activity: com.meishe.myvideoapp/com.meishe.myvideo.activity.MainActivity 
Package: com.screeclibinvoke, Main Activity: com.screeclibinvoke/.component.activity.AppStartActivity 
Package: com.tencent.wework, Main Activity: com.tencent.wework/.launch.LaunchSplashActivity 
```

#### 获取设备上当前应用程序的当前Activity

获取设备上当前应用程序的当前Activity，对于快速定位代码是非常有帮助的。利用 python 可以轻松搞定：

```python
def get_top_activity(adb_path='adb'):
    return _run_adb_command(adb_path,
                            ['shell', 'dumpsys', 'activity', 'top', '|', 'grep', 'ACTIVITY', '|', 'tail', '-n', '1'])
```

执行结果：

```python
ACTIVITY com.tencent.wework/.login.controller.LoginWxAuthActivity 8e93087 pid=22406
```

### 小结

上面简单介绍了使用 python 执行 adb 命令获取截屏、三方应用程序Main Activity、当前应用程序当前Activity信息，可以看到在 python 中，将相关 adb 命令操作组合或简化后输出，能够快速的得到关键信息，这就可以省去了部分琐碎时间。其它信息的获取，例如：

- 获取设备基础信息：adb shell getprop

- 获取设备屏幕信息：adb shell wm size 或 adb shell wm density

- 获取设备cpu信息：adb shell cat /proc/cpuinfo

- 获取设备memory信息：adb shell cat /proc/meminfo

- 其它
  
  也可以使用 python，这里就不再将每一个列举。

---

**下一篇，将介绍 Python 封装 gradle 命令。**
