---
layout: post
title: ADB 常用命令
categories: [SHELL]
description: ADB 常用命令
keywords: ADB, SHELL
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

## adb 

### 调试设备

```
adb kill-server //关闭adb服务
adb start-server //打开adb服务
adb devices//获取连接的设备
adb pull {手机地址} {电脑存储文件地址} //从手机取出文件
adb push {电脑存储文件地址} {手机地址} //往手机中添加文件
adb install {your package name}//安装包
adb uninstall {your package name} //卸载包

adb shell settings get secure android_id //获得手机id
```

### 查看文件

1.查看手机磁盘文件
```
第一步：adb shell 
第二步：ls
第三部：cd /mnt/sdcard/
```
2.查看手机应用程序包存储的文件

```
第一步：adb shell 
第二步：run-as {应用程序包名}
```
例子：

```
adb shell
1|HWCOL:/ $ run-as com.wangjiang.example
HWCOL:/data/data/com.wangjiang.example $ ls
app_DUHOME app_crashrecord app_textures cache      databases lib      shared_prefs 
app_bugly  app_tbs         app_webview  code_cache files     lib-main 
```
可以看到应用程序相关的数据库文件databases，偏好参数保存文件shared_prefs等。

打开shared_prefs：

```
HWCOL:/data/data/com.kuaikan.comic $ cd shared_prefs
HWCOL:/data/data/com.kuaikan.comic/shared_prefs $ ls
Alvin2.xml                                              ContextData.xml
```
再打开某个xml文件：cat 名字.xml

```
cat Alvin2.xml
<?xml version='1.0' encoding='utf-8' standalone='yes' ?>
<map>
    <long name="lastPostTime" value="1544444280" />
    <string name="hbuid">0EBC4DD2D399B13B0AD911DBA40A9C90</string>
    <long name="ntycfg_check" value="1544444280" />
</map>
```

---

## adb shell service

### 查看服务列表

```
adb shell service list
```
### 查看服务是否存在

```
adb shell service check 服务名
```
--- 

## adb shell wm 

### 获取手机分辨率

```
adb shell wm size
Physical size: 1080x2280
```


### 获取手机物理密度

```
adb shell wm density
Physical density: 480
```
---

## adb shell getprop

### 获取手机产品信息

```
adb shell getprop | grep product
[ro.build.product]: [PAAM00]
[ro.commonsoft.product]: [device1]
[ro.product.authentication]: [2018CP0021]
[ro.product.board]: [sdm660]
[ro.product.brand]: [OPPO]
[ro.product.cpu.abi]: [arm64-v8a]
[ro.product.cpu.abilist]: [arm64-v8a,armeabi-v7a,armeabi]
[ro.product.cpu.abilist32]: [armeabi-v7a,armeabi]
[ro.product.cpu.abilist64]: [arm64-v8a]
[ro.product.device]: [PAAM00]
[ro.product.first_api_level]: [27]
[ro.product.locale]: [zh-CN]//语言地区环境
[ro.product.manufacturer]: [OPPO]//制造商
[ro.product.model]: [PAAM00]/
[ro.product.name]: [PAAM00]//名称
[ro.product.oem_dm]: [1]
[ro.product.sar]: [1.8]
[ro.vendor.product.brand]: [OPPO]/品牌
[ro.vendor.product.device]: [PAAM00]//设备类型
[ro.vendor.product.manufacturer]: [OPPO]
[ro.vendor.product.model]: [PAAM00]
[ro.vendor.product.name]: [PAAM00]

```
### 获取手机配置信息

```
adb shell getprop | grep build
[ro.bootimage.build.date]: [Wed Oct 31 23:21:12 CST 2018]
[ro.bootimage.build.date.utc]: [1540999272]
[ro.bootimage.build.fingerprint]: [OPPO/PAAM00/PAAM00:8.1.0/OPM1.171019.011/1527522509:user/release-keys]
[ro.build.characteristics]: [nosdcard]
[ro.build.date]: [Wed Oct 31 23:21:12 CST 2018]
[ro.build.date.Ymd]: [180930]
[ro.build.date.YmdHM]: [201810312324]
[ro.build.date.utc]: [1540999272]
[ro.build.date.ymd]: [181031]
[ro.build.description]: [sdm660_64-user 8.1.0 OPM1.171019.011 eng.root.20181031.232112 dev-keys]
[ro.build.display.full_id]: [PAAM00_11_A.21_181031]
[ro.build.display.id]: [PAAM00_11_A.21_180930]
[ro.build.fingerprint]: [OPPO/PAAM00/PAAM00:8.1.0/OPM1.171019.011/1527522509:user/release-keys]
[ro.build.flavor]: [sdm660_64-user]
[ro.build.host]: [ubuntu-121-237]
[ro.build.id]: [OPM1.171019.011]
[ro.build.kernel.id]: [4.4.78-G201809302305]
[ro.build.master.date]: [201802192014]
[ro.build.product]: [PAAM00]
[ro.build.release_type]: [true]
[ro.build.shutdown_timeout]: [0]
[ro.build.soft.daily.version]: [false]
[ro.build.soft.majorversion]: []
[ro.build.soft.version]: [A.21]
[ro.build.tags]: [dev-keys]
[ro.build.type]: [user]
[ro.build.user]: [root]
[ro.build.version.all_codenames]: [REL]
[ro.build.version.base_os]: [OPPO/PAAM00/PAAM00:8.1.0/OPM1.171019.011/1524759600:user/release-keys]
[ro.build.version.codename]: [REL]
[ro.build.version.incremental]: [eng.root.20181031.232112]
[ro.build.version.opporom]: [V5.0]
[ro.build.version.ota]: [PAAM00_11.A.21_0212_201809302305]
[ro.build.version.preview_sdk]: [0]
[ro.build.version.release]: [8.1.0]//系统版本
[ro.build.version.sdk]: [27]//系统SDK版本
[ro.build.version.security_patch]: [2018-05-05]
[ro.vendor.build.date]: [Wed Oct 31 23:21:12 CST 2018]
[ro.vendor.build.date.utc]: [1540999272]
[ro.vendor.build.fingerprint]: [OPPO/PAAM00/PAAM00:8.1.0/OPM1.171019.011/1527522509:user/release-keys]
[sys.build.display.full_id]: [PAAM00_11_A.21_181031_0ba5fb34]
[sys.build.display.id]: [PAAM00_11_A.21_180930_0ba5fb34]
```

### 获取手机虚拟机信息

```
adb shell getprop | grep heap
[dalvik.vm.heapgrowthlimit]: [384m]//应用程序的最大内存限制
[dalvik.vm.heapmaxfree]: [16m]//GC相关
[dalvik.vm.heapminfree]: [4m]//GC相关
[dalvik.vm.heapsize]: [512m]//单个java虚拟机的最大内存限制
[dalvik.vm.heapstartsize]: [16m]//应用程序启动后分配的初始内存
[dalvik.vm.heaptargetutilization]: [0.75]//GC相关

```

### 获取更多信息

```
adb shell getprop
```

---

## adb shell  cat  

### 查看手机内存信息

```
adb shell  cat /proc/meminfo
MemTotal:        5877848 kB//总内存
MemFree:          235400 kB//剩余内存
MemAvailable:    3525256 kB
Buffers:            9196 kB//用来给文件做缓冲大小
Cached:          3256192 kB
SwapCached:        12868 kB
Active:          2856164 kB
Inactive:        1407448 kB
Active(anon):     583772 kB
Inactive(anon):   418524 kB
Active(file):    2272392 kB
Inactive(file):   988924 kB
Unevictable:        3404 kB
Mlocked:             256 kB
SwapTotal:       2293756 kB
SwapFree:        1682000 kB
Dirty:               132 kB
Writeback:             0 kB
AnonPages:        997424 kB
Mapped:           573752 kB
Shmem:               924 kB
Slab:             347412 kB
SReclaimable:     149992 kB
SUnreclaim:       197420 kB
KernelStack:       50960 kB
PageTables:        96004 kB
NFS_Unstable:          0 kB
Bounce:                0 kB
WritebackTmp:          0 kB
CommitLimit:     5232680 kB
Committed_AS:   150902944 kB
VmallocTotal:   258867136 kB//可以vmalloc虚拟内存大小
VmallocUsed:           0 kB//已经被使用的虚拟内存大小
VmallocChunk:          0 kB//最大的连续未被使用的vmalloc区域
CmaTotal:         176128 kB
CmaFree:            1052 kB
Oppo2Free:            32 kB
IonTotalUsed:     164908 kB
```
### 查看手机CPU信息

```
adb shell  cat /proc/cpuinfo
Processor       : AArch64 Processor rev 4 (aarch64)
processor       : 0
BogoMIPS        : 38.40
Features        : fp asimd evtstrm aes pmull sha1 sha2 crc32
CPU implementer : 0x51
CPU architecture: 8
CPU variant     : 0xa
CPU part        : 0x801
CPU revision    : 4

processor       : 1
BogoMIPS        : 38.40
Features        : fp asimd evtstrm aes pmull sha1 sha2 crc32
CPU implementer : 0x51
CPU architecture: 8
CPU variant     : 0xa
CPU part        : 0x801
CPU revision    : 4

processor       : 2
BogoMIPS        : 38.40
Features        : fp asimd evtstrm aes pmull sha1 sha2 crc32
CPU implementer : 0x51
CPU architecture: 8
CPU variant     : 0xa
CPU part        : 0x801
CPU revision    : 4

processor       : 3
BogoMIPS        : 38.40
Features        : fp asimd evtstrm aes pmull sha1 sha2 crc32
CPU implementer : 0x51
CPU architecture: 8
CPU variant     : 0xa
CPU part        : 0x801
CPU revision    : 4

processor       : 4
BogoMIPS        : 38.40
Features        : fp asimd evtstrm aes pmull sha1 sha2 crc32
CPU implementer : 0x51
CPU architecture: 8
CPU variant     : 0xa
CPU part        : 0x800
CPU revision    : 2

processor       : 5
BogoMIPS        : 38.40
Features        : fp asimd evtstrm aes pmull sha1 sha2 crc32
CPU implementer : 0x51
CPU architecture: 8
CPU variant     : 0xa
CPU part        : 0x800
CPU revision    : 2

processor       : 6
BogoMIPS        : 38.40
Features        : fp asimd evtstrm aes pmull sha1 sha2 crc32
CPU implementer : 0x51
CPU architecture: 8
CPU variant     : 0xa
CPU part        : 0x800
CPU revision    : 2

processor       : 7
BogoMIPS        : 38.40
Features        : fp asimd evtstrm aes pmull sha1 sha2 crc32
CPU implementer : 0x51
CPU architecture: 8
CPU variant     : 0xa
CPU part        : 0x800
CPU revision    : 2

Hardware        : Qualcomm Technologies, Inc SDM660

```
上面为8核CPU

---

## adb shell  ps

### 查看手机里所有应用程序的进程信息

```
adb shell  ps

USER           PID  PPID     VSZ    RSS WCHAN            ADDR   FRZ  S NAME                       
root             1     0   20488   2068 0                   0   efg  S init
......
```
### 查看应用程序的进程信息

```
adb shell  ps | grep {your package name}
```

---

## adb shell am

### 启动Activity/Broadcast/Service

1.通过Activity名字启动应用程序Activity
```
adb shell am start {your package name} / {your  activity}
```
例如：
```
adb shell am start com.android.settings/com.android.settings.Settings // 打开设置页面
adb shell am start com.android.settings/com.android.settings.SecuritySettings // 打开设置安全页面
adb shell am start com.android.settings/com.android.settings.RadioInfo //打开手机无线信息页面
adb shell am start com.android.settings/com.android.settings.DevelopmentSettings //打开手机开发者选项页面
```
[打开设置更多页面](https://blog.csdn.net/flaming999/article/details/78709396/)：

```
com.android.settings.AccessibilitySettings 辅助功能设置 
com.android.settings.ActivityPicker 选择活动 
com.android.settings.ApnSettings APN设置 
com.android.settings.ApplicationSettings 应用程序设置 
com.android.settings.BandMode 设置GSM/UMTS波段 
com.android.settings.BatteryInfo 电池信息 
com.android.settings.DateTimeSettings 日期和坝上旅游网时间设置 
com.android.settings.DateTimeSettingsSetupWizard 日期和时间设置 
com.android.settings.DevelopmentSettings 开发者设置 
com.android.settings.DeviceAdminSettings 设备管理器 
com.android.settings.DeviceInfoSettings 关于手机 
com.android.settings.Display 显示——设置显示字体大小及预览 
com.android.settings.DisplaySettings 显示设置 
com.android.settings.DockSettings 底座设置 
com.android.settings.IccLockSettings SIM卡锁定设置 
com.android.settings.InstalledAppDetails 语言和键盘设置 
com.android.settings.LanguageSettings 语言和键盘设置 
com.android.settings.LocalePicker 选择手机语言 
com.android.settings.LocalePickerInSetupWizard 选择手机语言 
com.android.settings.ManageApplications 已下载（安装）软件列表 
com.android.settings.MasterClear 恢复出厂设置 
com.android.settings.MediaFormat 格式化手机闪存 
com.android.settings.PhysicalKeyboardSettings 设置键盘 
com.android.settings.PrivacySettings 隐私设置 
com.android.settings.ProxySelector 代理设置 
com.android.settings.RadioInfo 手机信息 
com.android.settings.RunningServices 正在运行的程序（服务） 
com.android.settings.SecuritySettings 位置和安全设置 
com.android.settings.Settings 系统设置 
com.android.settings.SettingsSafetyLegalActivity 安全信息 
com.android.settings.SoundSettings 声音设置 
com.android.settings.TestingSettings 测试——显示手机信息、电池信息、使用情况统计、Wifi information、服务信息 
com.android.settings.TetherSettings 绑定与便携式热点 
com.android.settings.TextToSpeechSettings 文字转语音设置 
com.android.settings.UsageStats 使用情况统计 
com.android.settings.UserDictionarySettings 用户词典 
com.android.settings.VoiceInputOutputSettings 语音输入与输出设置 
com.android.settings.WirelessSettings 无线和网络设置
```
2.通过Intent启动应用程序Activity

```
adb shell am start -a {action} -d {数据}

这里-a表示动作，-d表述传入的数据，还有-t表示传入的类型
```
例如，打开一个网页：

```
 adb shell am start -a android.intent.action.VIEW -d http://www.baidu.com （这里-d表示传入的data）
```
 打开音乐播放器:
```
adb shell am start -a android.intent.action.MUSIC_PLAYER
```
3.发送广播

```
adb shell am broadcast -a {广播动作}
```
4.启动和关闭服务

```
adb shell am startservice {服务名称} //打开服务
adb shell am stopservice {服务名称} //关闭服务
```
5.应用程序启动耗时

```
adb shell am start -W {package name} /{activity name}

wangjiangdeMacBook-Pro:After wangjiang$ adb shell am start -W com.example.wangjiang.after/com.example.wangjiang.MainActivity
Starting: Intent { act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER] cmp=com.example.wangjiang.after/com.example.wangjiang.MainActivity }
Status: ok
LaunchState: COLD（冷启动）
Activity: com.example.wangjiang.after/com.example.wangjiang.MainActivity
TotalTime: 288
WaitTime: 294
Complete
wangjiangdeMacBook-Pro:After wangjiang$ adb shell am start -W com.example.wangjiang.after/com.example.wangjiang.MainActivity
Starting: Intent { act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER] cmp=com.example.wangjiang.after/com.example.wangjiang.MainActivity }
Warning: Activity not started, its current task has been brought to the front
Status: ok
LaunchState: HOT（热启动）
Activity: com.example.wangjiang.after/com.example.wangjiang.MainActivity
TotalTime: 63
WaitTime: 80
Complete
```

 - WaitTime 表示总的耗时，包括前一个应用 Activity pause 的时间和新应用启动的时；
 - ThisTime 表示一连串启动 Activity 的最后一个 Activity 的启动耗时；
 - **TotalTime** 表示新应用启动的耗时，包括新进程的启动和 Activity 的启动，但不包括前一个应用 Activity pause 的耗时。也就是说，**开发者一般只要关心TotalTime 即可**，这个时间才是自己应用真正启动的耗时。

### 查看Activity栈信息

查看所有应用程序的Activity栈信息
```
adb shell am stack list
```
查看某个应用程序的Activity栈信息

```
adb shell am stack list | grep {your package name}
```

### 模拟系统低内存 

```
adb shell am send-trim-memory {pid} {level}
```
level:

 - COMPLETE：该进程接近后台LRU列表的末尾，如果很快找不到更多内存，它将被终止。
 - MODERATE：该进程位于后台LRU列表的中间； 释放内存可以帮助系统在列表中稍后运行其他进程以获得更好的整体性能。
 - BACKGROUND：该进程已进入LRU列表。 这是一个清理资源的好机会，如果用户返回应用程序，可以高效快速地重建这些资源。
 - HIDDEN：该进程一直显示用户界面，并且不再这样做。 此时应该释放具有UI的大量分配，以便更好地管理内存。
 - RUNNING_CRITICAL：该进程不是一个可消耗的后台进程，但是该设备的内存运行极低，并且无法保持任何后台进程运行。 您的运行进程应该释放尽可能多的非关键资源，以允许在其他地方使用该内存。 接下来的事情将在{@link #onLowMemory（）}调用后报告该事件什么都没有可以保留在后台，这种情况可以明显影响用户。
 - RUNNING_LOW：该进程不是可消耗的后台进程，但设备内存不足。 您的运行进程应释放不需要的资源，以允许在其他地方使用该内存。
 - RUNNING_MODERATE：该进程不是可消耗的后台进程，但设备的内存运行速度适中。 您的运行进程可能希望释放一些不需要的资源以供其他地方使用。
 
 例子：
 

```
adb shell am send-trim-memory 10053 RUNNING_LOW
```

### 查看更多信息

```
adb shell am
```

---

## adb shell pm

### 安装包路径信息

```
adb shell pm path --user 0 com.tencent.mm // 腾讯视频路径
package:/data/app/com.tencent.mm-Ks6QB4kkz9mUeGDgfLnrVQ==/base.apk
```

### 安装包信息

1,查看手机上安装的应用程序

```
adb shell pm list packages
package:com.huawei.camera
......
```
2.输出包和包相关联的文件

```
adb shell pm list packages -f
package:/product/region_comm/china/app/ScenePack/ScenePack.apk=com.huawei.scenepack
package:/system/app/HiFolder/HiFolder.apk=com.huawei.hifolder
......
```
3.只输出禁用的包(有可能没有)

```
adb shell pm list packages -d
```
4.只输出启用的包

```
package:com.huawei.scenepack
package:com.huawei.hifolder
package:com.android.cts.priv.ctsshim
package:com.huawei.camera
......
```
5.只输出系统的包

```
adb shell pm list packages -s
```
6.只输出第三方的包

```
adb shell pm list packages -3
```
7.只输出包和安装信息（安装来源）

```
adb shell pm list packages -i
```
8.只输出包和未安装包信息（安装来源）

```
adb shell pm list packages -u
```
9.根据用户id查询用户的空间的所有包，USER_ID代表当前连接设备的顺序，从零开始：

```
adb shell pm list packages --user <USER_ID>
```

```
adb shell pm list packages --user 0
package:com.huawei.scenepack
package:com.huawei.hifolder
package:com.android.cts.priv.ctsshim
package:com.huawei.camera
......
```
10.只输出启用的包

```
adb shell pm list packages -e "ximalaya"
package:com.ximalaya.ting.android
```

### 清除包数据

```
adb shell pm clear {your package name}
```

### 查看更多信息

```
adb shell pm
```
---

## adb shell input

### 模拟输入文本

```
adb shell input text "hello,world"
```
### 模拟键盘输入事件

1.模拟点击返回键：
```
adb shell input keyevent 4
```
2.模拟点击Home键：
```
adb shell input keyevent 3
```
3.查看[android.view.KeyEvent](https://developer.android.com/reference/android/view/KeyEvent)类了解更多的键盘操作。

### 模拟点击事件

```
adb shell input tap 100 100 // 坐标(100,100)
```
### 模拟滑动事件

```
adb shell input swipe 800 100 100 100 //从右往左滑动，坐标从(800,100)到(100,100)
adb shell input swipe 100 100 800 100 //从左往右滑动，坐标从(100,100)到(800,100)
adb shell input swipe 100 800 100 100 //从下往上滑动，坐标从(100,800)到(100,100)
adb shell input swipe 100 100 100 800 //从上往下滑动，坐标从(100,100)到(100,800)
```
### 查看更多信息

```
adb shell input
```

## adb shell dumpsys
---
查看dumpsys相关命令：

```
adb shell dumpsys --help
usage: dumpsys
         To dump all services.
or:
       dumpsys [-t TIMEOUT] [--help | -l | --skip SERVICES | SERVICE [ARGS]]
         --help: shows this help
         -l: only list services, do not dump them
         -t TIMEOUT: TIMEOUT to use in seconds instead of default 10 seconds
         --skip SERVICES: dumps all services but SERVICES (comma-separated list)
         SERVICE [ARGS]: dumps only service SERVICE, optionally passing ARGS to it

```

输出可与dumpsys一起使用的完整系统服务列表：
```
adb shell dumpsys -l
Currently running services:
  BastetService
  Binder.Pged
  DisplayEngineExService
  DisplayEngineService
  DockObserver
  DubaiService
  EmcomManager
  HsmStat
  HwCommBoosterService
  HwFileMonitorService
  Hwmedia.monitor
  IAwareCMSService
  IAwareSdkService
  SelfbuildIRService
  SurfaceFlinger
  accessibility
  account
  activity //activity信息
  alarm
  android.security.keystore
  appops
  appwidget
  aps_service
  attestation_service
  audio
  authentication_service
  autofill
  backup
  battery //电池信息
  batteryproperties
  batterystats //电量信息
  bluetooth_manager
  carrier_config
  chr_service
  clipboard
  com.huawei.bd.BDService
  com.huawei.harassmentinterception.service.HarassmentInterceptionService
  com.huawei.netassistant.service.netassistantservice
  com.huawei.permissionmanager.service.holdservice
  com.huawei.security.IHwKeystoreService
  com.huawei.servicehost
  com.huawei.systemmanager.netassistant.netnotify.policy.NatTrafficNotifyService
  com.huawei.systemmanager.netassistant.netpolicy.NatNetworkPolicyService
  com.huawei.systemmanager.rainbow.service
  com.huawei.vision
  commontime_management
  companiondevice
  connectivity
  connmetrics
  consumer_ir
  content
  contexthub
  country_detector
  cover
  cpuinfo //cpu信息
  dbinfo
  device_identifiers
  device_policy
  deviceidle
  devicestoragemonitor
  diskstats
  display
  dreams
  drm.drmManager
  dropbox
  ethernet
  facerecognition
  faultdetectservice
  fido_authenticator
  fingerprint
  gfxinfo //帧信息
  gpu
  gpuassistant
  graphicsstats
  hardware_properties
  hisi.vrar
  hiview
  hwAlarmService
  hwAntiTheftService
  hwBootanimExService
  hwConnectivityExService
  hwGeneralService
  hwSaveDataService
  hwUsbExService
  hwaftpolicy
  hwext_device_service
  hwfeaturemanager
  hwfm_service
  hwsysresmanager
  hwthermal
  iGraphicsservice
  ihwvsim
  imms
  ims_config
  ims_ut
  input
  input_method
  input_method_secure
  ioinfo
  iphonesubinfo
  isms
  isms_interception
  isub
  jank
  jankshield
  jobscheduler
  launcherapps
  location
  lock_settings
  media.audio_flinger
  media.audio_policy
  media.camera
  media.camera.proxy
  media.drm
  media.extractor
  media.metrics
  media.player
  media.resource_manager
  media.sound_trigger_hw
  media_projection
  media_resource_monitor
  media_router
  media_session
  meminfo /内存信息
  midi
  mount
  multi_task
  netd_listener
  netpolicy
  netstats
  network_management
  network_score
  network_time_update_service
  nfc
  nonhardaccelpkgs
  notification
  otadexopt
  overlay
  package
  package_native
  permission
  pgservice
  phone
  phone_apdu
  phone_huawei
  pinner
  power
  powergenius
  powerprofile
  print
  processinfo
  procstats
  recovery
  restrictions
  rttmanager
  scheduling_policy
  search
  sec_key_att_app_id_provider
  securityserver
  sensorservice
  serial
  servicediscovery
  settings
  shortcut
  simphonebook
  smartcardservice
  soundtrigger
  statusbar
  storaged
  storagestats
  system.hsmcore
  telecom
  telephony.registry
  textservices
  thermalservice
  trust
  tui
  uimode
  updatelock
  usagestats
  usb
  user
  vibrator
  voiceinteraction
  vrmanager
  wallpaper
  webviewupdate
  wifi
  wificond
  wifip2p
  wifiscanner
  window //窗口信息
```

### activity信息

查看设备上所有程序activity的信息：
```
adb shell dumpsys activity activities
```
查看设备上正在运行的activity的信息：
```
adb shell dumpsys activity | grep -i run
```
查看应用程序顶层activity的信息：

```
adb shell dumpsys activity top | grep {your package name}
```
查看更多命令：
```
adb shell dumpsys actiivty -h
Activity manager dump options:
  [-a] [-c] [-p PACKAGE] [-h] [WHAT] ...
  WHAT may be one of:
    a[ctivities]: activity stack state
    r[recents]: recent activities state
    b[roadcasts] [PACKAGE_NAME] [history [-s]]: broadcast state
    broadcast-stats [PACKAGE_NAME]: aggregated broadcast statistics
    i[ntents] [PACKAGE_NAME]: pending intent state
    p[rocesses] [PACKAGE_NAME]: process state
    o[om]: out of memory management
    perm[issions]: URI permission grant state
    prov[iders] [COMP_SPEC ...]: content provider state
    provider [COMP_SPEC]: provider client-side state
    s[ervices] [COMP_SPEC ...]: service state
    as[sociations]: tracked app associations
    settings: currently applied config settings
    service [COMP_SPEC]: service client-side state
    package [PACKAGE_NAME]: all state related to given package
    all: dump all activities
    top: dump the top activity
  WHAT may also be a COMP_SPEC to dump activities.
  COMP_SPEC may be a component name (com.foo/.myApp),
    a partial substring in a component name, a
    hex object identifier.
  -a: include all available server state.
  -c: include client state.
  -p: limit output to given package.
  --checkin: output checkin format, resetting data.
  --C: output checkin format, not resetting data.
  --proto: output dump in protocol buffer format.

```
### 电池和电量信息

获取手机电池信息:

```
adb shell dumpsys battery
Current Battery Service state:
  AC powered: false
  USB powered: true
  Wireless powered: false
  Max charging current: 500000
  Max charging voltage: 4925000
  Charge counter: 139
  status: 2 //电池状态：2为充电状态 ，其他数字为非充电状态 
  health: 2 //电池健康状态：只有数字2表示good
  present: true //电池是否安装在机身
  level: 73 //电量: 百分比
  scale: 100
  voltage: 4058 //电池电压
  temperature: 300 //电池温度，单位是0.1摄氏度
  technology: Li-poly //电池种类
```
 将手机切换为非充电状态：
 
```
adb shell dumpsys battery set status 1
```
改变手机电量：

```
adb shell dumpsys battery set level 100 //电量百分之百
adb shell dumpsys battery set level 1 //电量百分之一
```

获取整个设备的电量消耗信息：
```
 adb shell dumpsys batterystats  | more
```
获取某个应用程序的电量消耗信息：
```
adb shell dumpsys batterystats  {your package name} | more
```

查看更多命令：
```
adb shell dumpsys batterystats -h
Battery stats (batterystats) dump options:
  [--checkin] [--history] [--history-start] [--charged] [-c]
  [--daily] [--reset] [--write] [--new-daily] [--read-daily] [-h] [<package.name>]
  --checkin: generate output for a checkin report; will write (and clear) the
             last old completed stats when they had been reset.
  -c: write the current stats in checkin format.
  --history: show only history data.
  --history-start <num>: show only history data starting at given time offset.
  --charged: only output data since last charged.
  --daily: only output full daily data.
  --reset: reset the stats, clearing all current data.
  --write: force write current collected stats to disk.
  --new-daily: immediately create and write new daily stats record.
  --read-daily: read-load last written daily stats.
  <package.name>: optional name of package to filter output by.
  -h: print this help text.
Battery stats (batterystats) commands:
  enable|disable <option>
    Enable or disable a running option.  Option state is not saved across boots.
    Options are:
      full-history: include additional detailed events in battery history:
          wake_lock_in, alarms and proc events
      no-auto-reset: don't automatically reset stats when unplugged
      pretend-screen-off: pretend the screen is off, even if screen state changes
```
输出自设备上次充电以来指定应用程序的电池使用情况统计信息：

```
adb shell dumpsys batterystats --charged {your package name}
```

查看电池电量相关的更多操作：[Inspect battery diagnostics](https://developer.android.com/studio/command-line/dumpsys)。

### cpu信息

```
adb shell dumpsys cpuinfo
```
### 帧信息（UI性能信息）

使用gfxinfo收集指定包名称的UI性能数据：

```
adb shell dumpsys gfxinfo {your package name}
```
从最近的帧中手机信息：

```
adb shell dumpsys gfxinfo {your package name} framestats
```
查看帧信息相关的更多操作：[gfxinfo](https://developer.android.com/studio/command-line/dumpsys)。

### 内存信息

[查看某个应用程序的内存信息](https://developer.android.com/studio/profile/investigate-ram?hl=zh-cn)：
```
adb shell dumpsys meminfo com.tencent.mm -d //腾讯视频内存信息
** MEMINFO in pid 30792 [com.tencent.mm] **
                   Pss  Private  Private  SwapPss     Heap     Heap     Heap
                 Total    Dirty    Clean    Dirty     Size    Alloc     Free
                ------   ------   ------   ------   ------   ------   ------
  Native Heap     5118     5036        0       35   280576   278773     1802
  Dalvik Heap     2698     2636        0       30     4338     3254     1084
 Dalvik Other     1600     1600        0        0                           
        Stack       36       36        0        0                           
       Ashmem        0        0        0        0                           
    Other dev       36        0       36        0                           
     .so mmap     4689      724     3524       24                           
    .apk mmap     3277        0     2628        0                           
    .ttf mmap        2        0        0        0                           
    .dex mmap    18618       16    18392        0                           
    .oat mmap      397        0        4        0                           
    .art mmap     1605     1020       16        7                           
   Other mmap       18        4        0        0                           
      Unknown     1509     1504        0        5                           
        TOTAL    39704    12576    24600      101   284914   282027     2886
 
 App Summary
                       Pss(KB)
                        ------
           Java Heap:     3672
         Native Heap:     5036
                Code:    25288
               Stack:       36
            Graphics:        0
       Private Other:     3144
              System:     2528
 
               TOTAL:    39704       TOTAL SWAP PSS:      101
 
 Objects
               Views:      239         ViewRootImpl:        0
         AppContexts:        3           Activities:        0
              Assets:        5        AssetManagers:        0
       Local Binders:       25        Proxy Binders:       27
       Parcel memory:       15         Parcel count:       59
    Death Recipients:        0      OpenSSL Sockets:        0
            WebViews:        0
 
 SQL
         MEMORY_USED:        0
  PAGECACHE_OVERFLOW:        0          MALLOC_SIZE:        0

```
查看更多命令：
```
adb shell dumpsys meminfo -h
meminfo dump options: [-a] [-d] [-c] [-s] [--oom] [process]
  -a: include all available information for each process. //包括每个进程的所有可用信息
  -d: include dalvik details. //包括虚拟机细节
  -c: dump in a compact machine-parseable representation. //
  -s: dump only summary of application memory usage.
  -S: dump also SwapPss.
  --oom: only show processes organized by oom adj.
  --local: only collect details locally, don't call process.
  --package: interpret process arg as package, dumping all
             processes that have loaded that package.
  --checkin: dump data for a checkin
  --proto: dump data to proto

```
查看内存相关的更多操作：[meminfo](https://developer.android.com/studio/command-line/dumpsys)。

### 窗口信息

查看窗口列表：

```
adb shell dumpsys window windows
```

查看更多命令：
```
adb shell dumpsys window -h
Window manager dump options:
  [-a] [-h] [cmd] ...
  cmd may be one of:
    l[astanr]: last ANR information
    p[policy]: policy state
    a[animator]: animator state
    s[essions]: active sessions
    surfaces: active surfaces (debugging enabled only)
    d[isplays]: active display contents
    t[okens]: token list
    w[indows]: window list
  cmd may also be a NAME to dump windows.  NAME may
    be a partial substring in a window name, a
    Window hex object identifier, or
    "all" for all windows, or
    "visible" for the visible windows.
    "visible-apps" for the visible app windows.
  -a: include all available server state.
```

### 查看deep link

```
 adb shell dumpsys package domain-preferred-apps
 App verification status:

  Package: com.qiyi.video
  Domains: tw.iqiyi.com m.tw.iqiyi.com m.iqiyi.com
  Status:  undefined

  Package: com.jifen.qukan
  Domains: m.qutoutiao.net
  Status:  undefined

  Package: com.ss.android.article.news
  Domains: m.toutiao.com toutiao.com www.toutiao.com open.toutiao.com d.toutiao.com
  Status:  undefined

  Package: com.taobao.taobao
  Domains: login.wapa.taobao.com travelitem.taobao.com trade.taobao.com tm.m.taobao.com cart.taobao.com wapp.wapa.taobao.com marketingop.taobao.com account.taobao.com m.intl.taobao.com h5.waptest.taobao.com alisec.taobao.com hiv.taobao.com login.taobao.com my.m.taobao.com i.taobao.com im.m.taobao.com fav.m.taobao.com newcart.taobao.com favorite.taobao.com market.m.taobao.com cart.m.tmall.com wapp.m.taobao.com store.taobao.com taobao.com *.taobao.com huodong.m.taobao.com ju.taobao.com m.taobao.com pages.tmall.com fishpage.taobao.com login.waptest.taobao.com qrreg.taobao.com h5.m.taobao.com market.wapa.taobao.com s.m.tmall.com refund.taobao.com tb.cn www.taobao.com login.tmall.com native.taobao.com items.alitrip.com magicmirror.m.taobao.com list.tmall.com goToNative h5.m.tmall.hk item.taobao.com s.m.taobao.com local.m.taobao.com txd.wdk.alibaba.com jusp.tmall.com newcart.m.tmall.com a.m.tmall.com login.m.taobao.com h5.wapa.taobao.com skip.ju.taobao.com a.m.taobao.com reader.taobao.com tqg.taobao.com *.we.app.jae.m.taobao.com pre-wormhole.tmall.com reg.taobao.com login.daily.taobao.net shop.m.taobao.com 30.5.36.255
  Status:  undefined

App linkages for user 0:

  Package: com.qiyi.video
  Domains: tw.iqiyi.com m.tw.iqiyi.com * m.iqiyi.com
  Status:  always : 200000002

  Package: com.jifen.qukan
  Domains: m.qutoutiao.net
  Status:  always : 200000004

  Package: com.ss.android.article.news
  Domains: m.toutiao.com toutiao.com www.toutiao.com open.toutiao.com d.toutiao.com
  Status:  ask

  Package: com.taobao.taobao
  Domains: login.wapa.taobao.com travelitem.taobao.com detail.m.tmall.hk helpandabout trade.taobao.com tm.m.taobao.com cart.taobao.com api.s.m.taobao.com wapp.wapa.taobao.com marketingop.taobao.com account.taobao.com m.intl.taobao.com h5.waptest.taobao.com alisec.taobao.com hiv.taobao.com login.taobao.com my.m.taobao.com shopsearch.taobao.com i.taobao.com im.m.taobao.com fav.m.taobao.com newcart.taobao.com favorite.taobao.com market.m.taobao.com cart.m.tmall.com page.tm wapp.m.taobao.com store.taobao.com taobao.com *.taobao.com huodong.m.taobao.com ju.taobao.com m.taobao.com pages.tmall.com fishpage.taobao.com login.waptest.taobao.com qrreg.taobao.com d.m.taobao.com h5.m.taobao.com go sm market.wapa.taobao.com detail.m.tmall.com s.m.tmall.com refund.taobao.com tb.cn www.taobao.com login.tmall.com s.taobao.com native.taobao.com my.m.ltao.com cun.m.taobao.com items.alitrip.com jhs.m.taobao.com chaoshi.detail.tmall.com magicmirror.m.taobao.com d.waptest.taobao.com list.tmall.com detail.ju.taobao.com goToNative h5.m.tmall.hk item.taobao.com s.m.taobao.com local.m.taobao.com txd.wdk.alibaba.com message jusp.tmall.com newcart.m.tmall.com item.tmall.com d.wapa.taobao.com a.m.tmall.com cal.m.taobao.com login.m.taobao.com h5.wapa.taobao.com skip.ju.taobao.com a.m.taobao.com reader.taobao.com tqg.taobao.com *.we.app.jae.m.taobao.com pre-wormhole.tmall.com api.s.taobao.com dai.taobao.com detail.tmall.com reg.taobao.com login.daily.taobao.net shop.m.taobao.com copyhomecache.m.taobao.com taoke.mdaren.taobao.com taobao.trade.order com.taobao.taobao 30.5.36.255
  Status:  ask

```

