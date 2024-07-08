---
layout: post
title: Python 封装 gradle 命令
categories: [Python]
description: 使用 Python 封装常用的 gradle 命令。
keywords: Python,Gradle
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
topmost: false
---

 上一篇文章介绍了[Python 封装 adb 命令]({{site.url}}/2024/01/16/Python-Adb-Intro/)，这一篇将介绍 Python 封装 gradle 命令，并生成可视化文档报告。

github 项目地址：[在 Android 开发中 使用 Python 脚本](https://github.com/WJRye/android-script)

## 执行 gradle 命令

在日常的 Android 项目开发中，通常使用 gradle 命令来获取项目 tasks、projects、depdencies等信息，或执行自定义的 task。

在之前的一篇文章：[使用 Gradle 命令了解项目构建信息]({{site.url}}/2023/11/16/Gradle-Intro/)中介绍了使用 gradle 命令的基础操作，现在利用 python 对这些命令的输出结果，进行信息过滤或生成可视化的 html 文档报告。

首先定义一个执行 gradle 命令的基础方法：

```python
def run_gradle_command(command):
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

### 操作简化-基础信息

如果在 Android 项目控制台中，直接使用 gradle 命令执行 `Task :app:dependencyInsight` （输出项目中特定依赖项的详细信息），不仅等待时间长（项目越大时间越长），而且控制台还会输出很多冗余信息。利用 python，可以将执行结果信息简化。

第一步：执行 gradle task

```python
def execute_task_dependency_insight(dependency):
    """
    输出项目中特定依赖项的详细信息
    :param dependency: 依赖项，例如：io.reactivex:rxjava'
    :return: task_name: 任务名称，task_result：执行的任务结果
    """
    task_name = ':app:dependencyInsight'
    task_result = run_gradle_command(
        ['./gradlew', task_name, '--configuration', 'releaseRuntimeClasspath', '--dependency',
         dependency])
    return task_name, task_result
```

第二步：保存结果到文件中

- 在 gradle 命令执行 task 的输出结果中，从 `> Task :app:dependencyInsight` 行开始才是该 task 的关键信息。

```python
def save_result_to_file(file_dir, file_name, task_name, task_result):
    """
    保存结果到文件
    :param file_dir: 文件目录
    :param file_name: 文件名称
    :param task_name: gradle task 名称
    :param task_result: 内容¬
    :return: 文件地址
    """
    # 从 > Task 行开始截取执行结果
    start_str = "> Task " + task_name
    content = task_result[task_result.rfind(start_str):]
    if len(file_name) > 0:
        des_file_name = file_name + '.txt'
    else:
        des_file_name = task_name[1:].strip().replace(':', '-') + ".txt"
    file_path = file_dir + "/" + des_file_name
    # 如果目录不存在，先创建一个目录
    if not os.path.exists(file_dir):
        os.makedirs(file_dir)
    with open(file_path, 'w') as f:
        f.write(content)
        f.close()
    return file_path
```

第三步：将结果文件用电脑打开

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

第四步：将上面步骤组合在一起执行

- 输出项目中特定依赖项的详细信息：`./gradlew :app:dependencyInsight --configuration releaseRuntimeClassPath --dependency io.reactivex:rxjava`

```python
if __name__ == "__main__":
    android_project_path = '/Users/wangjiang/Public/software/android-workplace/Demo'
    # 切换到安卓项目工作目录
    os.chdir(android_project_path)
    # 结果输出文件地址
    report_file_dir = android_project_path + '/build/reports'
    #  输出项目中 io.reactivex:rxjava 依赖项的详细信息
    task_name, task_result = execute_task_dependency_insight('io.reactivex:rxjava')
    # 结果输出问文件名称
    file_name = task_name[1:].replace(':', '-')
    # 结果文件地址
    file_path = _save_result_to_file(report_file_dir, file_name, task_name, task_result)
    open_file(file_path)
```

执行上面的 python 脚本后，电脑会自动打开 `> Task :app:dependencyInsight` 的输出结果`项目目录/build/reports/app-dependencyInsight.txt`：

```python
> Task :app:dependencyInsight
io.reactivex:rxjava:1.3.0
   variant "runtime" [
      org.gradle.status                                               = release (not requested)
      org.gradle.usage                                                = java-runtime
      org.gradle.libraryelements                                      = jar (not requested)
      org.gradle.category                                             = library (not requested)

      Requested attributes not found in the selected variant:
         com.android.build.api.attributes.BuildTypeAttr                  = release
         com.android.build.api.attributes.ProductFlavor:type             = release
         org.gradle.jvm.environment                                      = android
         com.android.build.api.attributes.AgpVersionAttr                 = 7.2.2
         org.jetbrains.kotlin.platform.type                              = androidJvm
   ]
   Selection reasons:
      - By conflict resolution : between versions 1.3.0 and 1.1.6

//.....省略
```

假设已经将上面的代码封装成了一个 dependency_insight.py，那么执行 `python3 dependency_insight.py`时，可以将安卓项目路径传递传递进去：

```python
if __name__ == "__main__":
    args = sys.argv[1:]
    android_project_path = args[0]

python3 dependency_insight.py android_project_path
```

### 操作简化- so 依赖信息

在之前的一篇文章：[使用 Gradle 命令了解项目构建信息]({{site.url}}/2023/11/16/Gradle-Intro/)介绍了通过监听`Task:app:mergeDebugNativeLibs`来获取本项目、子项目、三方库的 so 依赖信息，但是在控制台输出的 so 依赖信息不是很直观，如果利用python，可以输出一个 so 依赖信息的 html 文档报告。

#### 封装 mergeDebugNativeLibs 脚本

- 将监听 `Task:app:mergeDebugNativeLibs` 的 groovy 脚本封装到一个单独文件 `native_lib.gradle`中 

- 将 `Task:app:mergeDebugNativeLibs` 的输出结果用 json 格式保存到文件，文件路径需要在后面的 python 脚本中指定

- 把文件`native_lib.gradle`放到 app 的 build.gralde 文件所在同级目录下，并在项目 build.gralde 中添加依赖： `apply from: "./native_libs.gradle"`

- 在命令行执行 `./gradlew app:mergeDebugNativeLibs`，输出项目 so 依赖信息 json 结果
  
  native_lib.gradle 脚本：

```groovy
class NativeLibInfo {
    // native lib 名称
    String lib_name
    // so 依赖的项目相对路径信息列表
    List<String> so_relative_path_list

    NativeLibInfo(String lib_name, List<String> so_relative_path_list) {
        this.lib_name = lib_name
        this.so_relative_path_list = so_relative_path_list
    }
}

class ReportInfo {
    //本项目、子项目、三方库
    String name
    //依赖 so 信息列表
    List<NativeLibInfo> info

    ReportInfo(String name, List<NativeLibInfo> info) {
        this.name = name
        this.info = info
    }
}

project.afterEvaluate {
    project.android.applicationVariants.all { variant ->
        //获取构建变体的名称
        logger.lifecycle("${variant.name}")
        def variantName = variant.name
        def name = String.valueOf(variantName.charAt(0)).toUpperCase() + variantName.substring(1)
        def mergeNativeLibsTask = project.tasks.findByName("merge${name}NativeLibs")
        if (mergeNativeLibsTask != null) {
            mergeNativeLibsTask.doLast { task ->
                def result = new ArrayList<ReportInfo>()
                //当前项目相关的 so 文件列表
                def projectNativeResult = new ReportInfo("project native libs", getProjectSoInfo(false, task.projectNativeLibs.getFiles()))
                result.add(projectNativeResult)
                //子项目相关的 so 文件列表
                def subProjectNativeResult = new ReportInfo("sub project native libs", getProjectSoInfo(false, task.subProjectNativeLibs.getFiles()))
                result.add(subProjectNativeResult)
                //三方库相关的 so 文件列表
                def externalProjectNativeResult = new ReportInfo("external project native libs", getProjectSoInfo(true, task.externalLibNativeLibs.getFiles()))
                result.add(externalProjectNativeResult)

                def fileDir = new File(project.buildDir.path + "/reports/so")
                if (!fileDir.exists()) {
                    fileDir.mkdirs()
                }
                saveSoInfoReport(fileDir, result)
            }
        }
    }
}
/**
 * 保存项目依赖的 so 信息列表到 json 文件中
 * @param savePath 保存路径目录
 * @param result 项目依赖的 so 信息列表
 */
def saveSoInfoReport(File saveDir, ArrayList<ReportInfo> result) {
    def reportFile = new File(saveDir, "native_libs.json")
    def jsonBuilder = new groovy.json.JsonBuilder()
    jsonBuilder result.collect { reportInfo ->
        [
                name: reportInfo.name,
                info: reportInfo.info.collect { nativeLibInfo ->
                    [
                            lib_name             : nativeLibInfo.lib_name,
                            so_relative_path_list: nativeLibInfo.so_relative_path_list
                    ]
                }
        ]
    }
    def jsonResult = jsonBuilder.toPrettyString()
    def writer = new BufferedWriter(new FileWriter(reportFile))
    writer.write(jsonResult)
    writer.flush()
    writer.close()
    project.logger.info("Project Native Libs Json Report:" + reportFile.path + '\n')
}

/**
 * 获取项目依赖的 native lib 中的 so 信息列表
 * @param isExternal 是否是三方库
 * @param fileSet so 路径列表
 * @return so 信息列表
 */
def getProjectSoInfo(boolean isExternal, Set<File> fileSet) {
    def projectNames = new HashSet<String>()
    def nativeLibsInfoList = new ArrayList<NativeLibInfo>()
    def rootProjectPath = project.rootProject.projectDir.path
    def buildName = project.rootProject.buildDir.name
    fileSet.forEach { file ->
        def projectName
        if (!isExternal) {
            projectName = file.path.substring(rootProjectPath.length() + 1, file.path.indexOf(buildName) - 1)
        } else {
            projectName = file.parentFile.name
        }
        projectNames.add(projectName)
        def childFiles = file.listFiles().toList()
        def soRelativePathList = new ArrayList<>()
        while (childFiles.size() > 0) {
            def childFile = childFiles.remove(0)
            if (childFile.isDirectory()) {
                childFiles.addAll(childFile.listFiles())
            } else {
                soRelativePathList.add(childFile.path.substring(file.path.length() + 1))
            }
        }
        nativeLibsInfoList.add(new NativeLibInfo(projectName, soRelativePathList))
    }
    return nativeLibsInfoList
}
```

在命令行执行`./gradlew app:mergeDebugNativeLibs`，输出结果例如：

```bash
[
    {
        "name": "project native libs",
        "info": [

        ]
    },
    {
        "name": "sub project native libs",
        "info": [

        ]
    },
    {
        "name": "external project native libs",
        "info": [
            {
                "lib_name": "dynamicview-core-6a43f8cfb73faca6dfc0926d8d08a6d8_44bfd0faea8a1c23a4e56bba23a0f3a191d06c00-debug",
                "so_relative_path_list": [
                    "armeabi-v7a/libSapling.so",
                    "armeabi-v7a/libc++_shared.so",
                    "x86/libSapling.so",
                    "x86/libc++_shared.so",
                    "arm64-v8a/libSapling.so",
                    "arm64-v8a/libc++_shared.so",
                    "x86_64/libSapling.so",
                    "x86_64/libc++_shared.so"
                ]
            },
            {
                "lib_name": "matrix-io-canary-2.0.1",
                "so_relative_path_list": [
                    "armeabi-v7a/libio-canary.so",
                    "x86/libio-canary.so",
                    "arm64-v8a/libio-canary.so",
                    "x86_64/libio-canary.so"
                ]
            },
            {
                "lib_name": "matrix-fd-2.0.2",
                "so_relative_path_list": [
                    "armeabi-v7a/libmatrix-fd.so",
                    "arm64-v8a/libmatrix-fd.so"
                ]
            },
            {
                "lib_name": "matrix-hooks-2.0.2",
                "so_relative_path_list": [
                    "include/ThreadPool.h",
                    "include/ReentrantPrevention.h",
                    "include/SoLoadMonitor.h",
                    "include/Log.h",
                    "include/Macros.h",
                    "include/HookCommon.h",
                    "include/Maps.h",
                    "include/ScopedCleaner.h",
                    "include/JNICommon.h",
                    "include/ProfileRecord.h",
                    "armeabi-v7a/libmatrix-memguard.so",
                    "armeabi-v7a/libmatrix-hookcommon.so",
                    "armeabi-v7a/libmatrix-memoryhook.so",
                    "armeabi-v7a/libmatrix-pthreadhook.so",
                    "armeabi-v7a/libc++_shared.so",
                    "arm64-v8a/libmatrix-memguard.so",
                    "arm64-v8a/libmatrix-hookcommon.so",
                    "arm64-v8a/libmatrix-memoryhook.so",
                    "arm64-v8a/libmatrix-pthreadhook.so",
                    "arm64-v8a/libc++_shared.so",
                    "include/struct/lock_free_array_queue.h",
                    "include/struct/lock_free_queue.h",
                    "include/struct/splay_map.h",
                    "include/struct/buffer_source.h"
                ]
            }
        ]
    }
]
```

#### 定义 python 脚本，输出 html 文档报告

第一步

- 定义上面 groovy 脚本中的数据模型，用于 json 反序列化

```python
class NativeLibInfo:
    def __init__(self, lib_name, so_relative_path_list):
        """
        :param lib_name: native lib 名称
        :param so_relative_path_list: so 依赖的项目相对路径信息列表
        """
        self.lib_name = lib_name
        self.so_relative_path_list = so_relative_path_list


class ReportInfo:
    def __init__(self, name, info):
        """
        :param name: 本项目、子项目、三方库
        :param info: 依赖 so 信息列表
        """
        self.name = name
        self.info = info
```

第二步

- 执行 `./gradlew app:mergeDebugNativeLibs`

```python
def execute_task():
    """
    执行 task
    :return: 是否成功，true表示成功，否则失败
    """
    task_name = ':app:mergeDebugNativeLibs'
    result = run_gradle_command(['./gradlew', task_name])
    return result is None
```

第三步

- 反序列化 json

```python
def deserialize_json(json_path):
    """
    :param json_path: json 路径
    :return: ReportInfo List
    """
    if not os.path.exists(json_path):
        print(f"FileNotExists:{json_path}")
        return None
    with (open(json_path, 'r') as file):
        data = json.load(file)
        # json 反序列化
        return [ReportInfo(name=report_data.get('name'), info=[
            NativeLibInfo(**native_lib_info) for native_lib_info in report_data.get('info', [])
        ]) for report_data in data]
```

第四步

- 将反序列化 json 后的数据转化为表格数据

```python
def _custom_group_key(key=''):
    """
    :param key:如 armeabi-v7a/xx.so 或 arm64-v8a/xx.so等等
    :return: armeabi-v7a 或 arm64-v8a
    """
    return key[:key.index('/')]

def format_data(data, project_name):
    """
     将data格式化为表格
    :param data: json 结果地址
    :return:表格数据和表格标题
    """
    # html 中表格数据
    table_data = {}
    max_len = 0
    for report_info in data:
        for native_lib_info in report_info.info:
            # 对结果分组
            grouped_so_relative_path_list = [list(group) for key, group in
                                             groupby(native_lib_info.so_relative_path_list, _custom_group_key)]
            new_so_relative_path_list = []
            for item in grouped_so_relative_path_list:
                new_so_relative_path_list.append(', '.join(item))
            print(native_lib_info.lib_name + ": " + str(new_so_relative_path_list))
            native_lib_info.so_relative_path_list = new_so_relative_path_list
            so_relative_path_list_size = len(native_lib_info.so_relative_path_list)
            if so_relative_path_list_size > max_len:
                max_len = so_relative_path_list_size
    # native lib 数量
    num_libs = 0
    for report_info in data:
        num_libs = num_libs + len(report_info.info)
        for native_lib_info in report_info.info:
            while len(native_lib_info.so_relative_path_list) < max_len:
                native_lib_info.so_relative_path_list.append('')
            table_data[native_lib_info.lib_name] = native_lib_info.so_relative_path_list

    title = f"{project_name} native libs: {num_libs}"
    return title, table_data
```

第四步

- 利用 pandas 库生成 html 文档报告

```python
def generate_html(title, table_data, output_path):
    """
    生成 html 文档报告
    :param title: html 文档标题
    :param table_data: native libs info
    :param output_path: html 文档报告文件地址
    """
    table = pd.DataFrame.from_dict(data=table_data).set_index(list(table_data.keys())).transpose()
    styled_html = """
      <style>
        table {
          width: 50%;
          border-collapse: collapse;
          margin-top: 10px;
        }
        th, td {
          border: 1px solid black;
          padding: 8px;
          text-align: left;
        }
      </style>
      """ + table.to_html()
    if os.path.exists(output_path):
        os.remove(output_path)
    html_content = f"<H1>{title}</H1>\n{styled_html}"
    with open(output_path, 'w') as f:
        f.write(html_content)
    print(output_path)
    open_file(output_path)
```

第五步：将上面步骤组合在一起执行

注意： json_path 的定义，就是执行 ./graldew :app:mergeDebugNativeLibs 完后的 json 结果存储地址

```python
if __name__ == "__main__":
    android_project_path = '/Users/wangjiang/Public/software/android-workplace/Demo'
    # 切换到安卓项目工作目录
    os.chdir(android_project_path)
    # 结果输出目录
    report_file_dir = android_project_path + '/build/reports/so'
    if not os.path.exists(report_file_dir):
        os.makedirs(report_file_dir)
    # 结果输出地址
    report_path = report_file_dir + '/native_libs.html'
    # 执行 ./graldew mergeDebugNativeLibs 后的 json 结果存储地址，与 native_lib.gradle 中的对应
    json_path = '/Users/wangjiang/Public/software/android-workplace/Demo/build/reports/so/native_libs.json'
    # 项目名称，方便在输出结果html中显示
    project_name = 'My Project'
    isSuccess = execute_task()
    if isSuccess:
        data = deserialize_json(json_path)
        if data is None:
            print("Deserialize json failed: " + json_path)
        else:
            title, table_data = format_data(data, project_name)
            generate_html(title, table_data, report_path)
    else:
        print("Execute task failed")
```

输出结果例如 `file:///Users/wangjiang/Public/software/android-workplace/Demo/build/reports/so/native_libs.html`：
![loading-ag-312](/images/posts/2024-01-19-Python-Gradle-Info/p1.jpeg)

假设已经将上面的代码封装成了一个 native_libs.py，那么执行 `python3 native_libs.py`时，可以将安卓项目路径传递传递进去：

```python
if __name__ == "__main__":
    args = sys.argv[1:]
    android_project_path = args[0]
    json_path = android_project_path+'/build/reports/so/native_libs.json'

python3 native_libs.py android_project_path
```

#### 将 python 脚本放到 CI/CD 中执行

在项目 CI/CD build 阶段完成后，在 analyze 阶段新增一个 `job: so dependency` 用于分析项目依赖的 so 信息，例如：

```bash
so dependency:
  tags:
    - apk
    - android
  stage: analyze
  script:
    - ./gradlew :app:mergeDebugNativeLibs
  after_script:
    - python3.9 native_lib.py
  artifacts:
    name: "$CI_JOB_STAGE}_reports_${CI_PROJECT_NAME}_$CI_COMMIT_REF_SLUG"
    when: on_success
    expire_in: 3 days
    paths:
      - "*/build/reports"
  only:
    - branches
  except:
    - master
```

在项目每次跑完 pipiline 后，就会生成一个项目 so 依赖信息报告：native_lib.html ，方便每个开发人员下载查看。

### 小结

使用 python 执行相关 gradle 命令，主要是简化输出信息-直接获取关键信息，或生成可视化报告，或是放在 CI/CD 中执行的 script。可以用 python 封装的常用 gradle 命令有：

- 输出项目中特定依赖项的详细信息：`./gradlew :app:dependencyInsight`
- 输出项目依赖信息：`./gradlew :app:dependencies --configuration releaseRuntimeClasspath`
- 输出项目项目信息：`./graldew projects`
- 输出项目 so 信息：`./gradlew :app;mergeDebugNativeLibs`
- 其它

如果感兴趣，可以自定义更多 gradle 命令的  python 脚本。

---

**下一篇将介绍 Python 封装 git 命令**

**后续会将完整代码放到 github。**
