---
layout: post
title: Android ANR 触发原理
categories: [Android]
description: 在Android应用程序的主线程中，如果某个事件没有在系统规定的时间范围内执行完成，就会触发ANR。
keywords: Android, ANR
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

## ANR  简介

ANR(Application Not Responding)：程序无响应。**在Android应用程序的主线程中，如果某个事件没有在系统规定的时间范围内执行完成，就会触发ANR**。通常，ANR 会对用户的体验会产生致命的影响，因为当发生ANR时，用户不能再与应用程序进行交互。所以，我们也会统计应用程序的线上ANR率，以此来衡量一个应用程序的稳定性。ANR率 是应用程序的一个非常重要的性能指标。解决ANR 也是一个重要紧急的事情。本文主要先介绍ANR 触发原理，下一篇再介绍如何处理 ANR 问题。

## ANR 触发场景

在Android 系统中，触发ANR的事件场景通常有四种：

- Service Timeout：Service在特定的时间内无法处理完成
- BroadcastQueue Timeout：BroadcastReceiver在特定时间内无法处理完成
- ContentProvider Timeout：内容提供者执行超时
- inputDispatching Timeout：按键或触摸事件在特定时间内无响应。

| 场景               | 事件                                                      | 时间限制 |
| ---------------- | ------------------------------------------------------- | ---- |
| Service          | ActiveServices#SERVICE_TIMEOUT                          | 20s  |
| Service          | ActiveServices#SERVICE_BACKGROUND_TIMEOUT               | 200s |
| BroadcastQueue   | ActivityManagerService#BROADCAST_FG_TIMEOUT             | 10s  |
| BroadcastQueue   | ActivityManagerService#BROADCAST_BG_TIMEOUT             | 60s  |
| ContentProvider  | ActivityManagerService#CONTENT_PROVIDER_PUBLISH_TIMEOUT | 10s  |
| inputDispatching | ActivityTaskManagerService#KEY_DISPATCHING_TIMEOUT      | 5s   |

## ANR 触发原理

> 文中源码基于Android API 29 Platform

这里以 Service 触发场景为例子，其他的都类似（原理类似）。要 分析 Service ANR 的原因，需要从  Service 的创建过程开始：

1. 启动 Service：`ContextWrapper#startService`  &rarr; `ContextImpl#startService` &rarr; `ContextImpl#startServiceCommon`  &rarr; `ActivityManagerService#startService` 

2. AMS 处理： `ActivityManagerService#startService`   &rarr;  `ActiveServices#startServiceLocked`

3. 接下来在 `ActiveServices` 处理事件中，会发送一个延迟消息：`startServiceLocked` &rarr; `retrieveServiceLocked` &rarr;`startServiceInnerLocked` &rarr; `bringUpServiceLocked` &rarr; **realStartServiceLocked** &rarr; `bumpServiceExecutingLocked` &rarr;  **scheduleServiceTimeoutLocked** &rarr; 发送触发ANR的延迟消息

4. 另外，在**realStartServiceLocked** 方法中，执行`bumpServiceExecutingLocked`方法后，紧接着就会执行`app.thread.scheduleCreateService(r, r.serviceInfo,
   
                    mAm.compatibilityInfoForPackage(r.serviceInfo.applicationInfo),
                    app.getReportedProcState());`代码，也就是执行 `ApplicationThread` 的`scheduleCreateService`方法，接下来就交给 `ActivityThread` 的`handleCreateService`方法，在这个方法中就开始创建 Service，ContextImpl，Application对象，并让 Service 和 ContextImpl 关联起来，最后调用Service 的 `onCreate` 方法，完成Service的创建。

5. 在 `ActivityThread` 的`handleCreateService`方法 最后会依次执行： `ActivityManagerService#serviceDoneExecuting` &rarr; `ActiveServices#serviceDoneExecutingLocked(ServiceRecord r, int type, int startId, int res)` &rarr;  **ActiveServices#serviceDoneExecutingLocked(r, inDestroying, inDestroying)** ，在 serviceDoneExecutingLocked方法中，执行： `mAm.mHandler.removeMessages(ActivityManagerService.SERVICE_TIMEOUT_MSG, r.app);`也就是移除前面在 **scheduleServiceTimeoutLocked**  方法中发送的触发ANR的延迟消息。

现在先查看 **scheduleServiceTimeoutLocked**  方法的具体实现：

```
    void scheduleServiceTimeoutLocked(ProcessRecord proc) {
        if (proc.executingServices.size() == 0 || proc.thread == null) {
            return;
        }
        Message msg = mAm.mHandler.obtainMessage(
                ActivityManagerService.SERVICE_TIMEOUT_MSG);
        msg.obj = proc;
        mAm.mHandler.sendMessageDelayed(msg,
                proc.execServicesFg ? SERVICE_TIMEOUT : SERVICE_BACKGROUND_TIMEOUT);
    }
```

在这个方法中， 会 通过 `mAm` (也就是`ActivityManagerService` )的 `mHandler` 发送一个延迟消息。这个消息的基本参数值信息是：

- `mHandler` ：通过子线程的 Looper 创建的一个 Handler，所以这个消息是放在子线程的消息队列中，不影响主线程。
- `what` ：`ActivityManagerService.SERVICE_TIMEOUT_MSG`(值12)。
- `delayMillis` ：如果是前台服务就是 `SERVICE_TIMEOUT`(值20*1000豪秒)，否则是`SERVICE_BACKGROUND_TIMEOUT`(10 * SERVICE_TIMEOUT豪秒)。

如果 Service 成功启动，也就是在规定的 SERVICE_TIMEOUT 或 SERVICE_BACKGROUND_TIMEOUT 时间范围内执行完成该事件，就会移除掉这个延迟消息：

```
    private void serviceDoneExecutingLocked(ServiceRecord r, boolean inDestroying,
            boolean finishing) {
    //...省略
                    mAm.mHandler.removeMessages(ActivityManagerService.SERVICE_TIMEOUT_MSG, r.app);
    //...省略 
    }
```

否则就会在`ActivityManagerService`中触发这个延迟消息：

```
final class MainHandler extends Handler {

        public MainHandler(Looper looper) {
            super(looper, null, true);
        }

        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
            //...其它消息
            case SERVICE_TIMEOUT_MSG: {
                mServices.serviceTimeout((ProcessRecord)msg.obj);
            } break;
         }
 }
```

再看 ActiveServices 的 `serviceTimeout` 方法：

```
      void serviceTimeout(ProcessRecord proc) {
        //...收集 serviceTimeout 的一些相关的服务信息，也就是给 anrMessage 赋值
        if (anrMessage != null) {
            proc.appNotResponding(null, null, null, null, false, anrMessage);
        }
      }
```

再看 proc(也就是 ProcessRecord) 的 `appNotResponding` 方法：

```
    void appNotResponding(String activityShortComponentName, ApplicationInfo aInfo,
            String parentShortComponentName, WindowProcessController parentProcess,
            boolean aboveSystem, String annotation) {
        ArrayList<Integer> firstPids = new ArrayList<>(5);
        SparseArray<Boolean> lastPids = new SparseArray<>(20);

        mWindowProcessController.appEarlyNotResponding(annotation, () -> kill("anr", true));

        long anrTime = SystemClock.uptimeMillis();
        if (isMonitorCpuUsage()) {
            mService.updateCpuStatsNow();
        }

        synchronized (mService) {
            // PowerManager.reboot() can block for a long time, so ignore ANRs while shutting down.
            if (mService.mAtmInternal.isShuttingDown()) {
                Slog.i(TAG, "During shutdown skipping ANR: " + this + " " + annotation);
                return;
            } else if (isNotResponding()) {
                Slog.i(TAG, "Skipping duplicate ANR: " + this + " " + annotation);
                return;
            } else if (isCrashing()) {
                Slog.i(TAG, "Crashing app skipping ANR: " + this + " " + annotation);
                return;
            } else if (killedByAm) {
                Slog.i(TAG, "App already killed by AM skipping ANR: " + this + " " + annotation);
                return;
            } else if (killed) {
                Slog.i(TAG, "Skipping died app ANR: " + this + " " + annotation);
                return;
            }

            // In case we come through here for the same app before completing
            // this one, mark as anring now so we will bail out.
            setNotResponding(true);

            // Log the ANR to the event log.
            EventLog.writeEvent(EventLogTags.AM_ANR, userId, pid, processName, info.flags,
                    annotation);

            // Dump thread traces as quickly as we can, starting with "interesting" processes.
            firstPids.add(pid);

            // Don't dump other PIDs if it's a background ANR
            if (!isSilentAnr()) {
                int parentPid = pid;
                if (parentProcess != null && parentProcess.getPid() > 0) {
                    parentPid = parentProcess.getPid();
                }
                if (parentPid != pid) firstPids.add(parentPid);

                if (MY_PID != pid && MY_PID != parentPid) firstPids.add(MY_PID);
                //1. 收集应用程序最近使用的的进程pid
                for (int i = getLruProcessList().size() - 1; i >= 0; i--) {
                    ProcessRecord r = getLruProcessList().get(i);
                    if (r != null && r.thread != null) {
                        int myPid = r.pid;
                        if (myPid > 0 && myPid != pid && myPid != parentPid && myPid != MY_PID) {
                            if (r.isPersistent()) {
                                firstPids.add(myPid);
                                if (DEBUG_ANR) Slog.i(TAG, "Adding persistent proc: " + r);
                            } else if (r.treatLikeActivity) {
                                firstPids.add(myPid);
                                if (DEBUG_ANR) Slog.i(TAG, "Adding likely IME: " + r);
                            } else {
                                lastPids.put(myPid, Boolean.TRUE);
                                if (DEBUG_ANR) Slog.i(TAG, "Adding ANR proc: " + r);
                            }
                        }
                    }
                }
            }
        }

        // Log the ANR to the main log.
        // 2.在控制台打印ANR相关的日志，日志包含：当前进程名字，当前组件信息，当前进程id，前五个进程的相关信息，cpu信息等
        StringBuilder info = new StringBuilder();
        info.setLength(0);
        info.append("ANR in ").append(processName);
        if (activityShortComponentName != null) {
            info.append(" (").append(activityShortComponentName).append(")");
        }
        info.append("\n");
        info.append("PID: ").append(pid).append("\n");
        if (annotation != null) {
            info.append("Reason: ").append(annotation).append("\n");
        }
        if (parentShortComponentName != null
                && parentShortComponentName.equals(activityShortComponentName)) {
            info.append("Parent: ").append(parentShortComponentName).append("\n");
        }

        ProcessCpuTracker processCpuTracker = new ProcessCpuTracker(true);

        // don't dump native PIDs for background ANRs unless it is the process of interest
        String[] nativeProcs = null;
        if (isSilentAnr()) {
            for (int i = 0; i < NATIVE_STACKS_OF_INTEREST.length; i++) {
                if (NATIVE_STACKS_OF_INTEREST[i].equals(processName)) {
                    nativeProcs = new String[] { processName };
                    break;
                }
            }
        } else {
            nativeProcs = NATIVE_STACKS_OF_INTEREST;
        }

        int[] pids = nativeProcs == null ? null : Process.getPidsForCommands(nativeProcs);
        ArrayList<Integer> nativePids = null;

        if (pids != null) {
            nativePids = new ArrayList<>(pids.length);
            for (int i : pids) {
                nativePids.add(i);
            }
        }

        // For background ANRs, don't pass the ProcessCpuTracker to
        // avoid spending 1/2 second collecting stats to rank lastPids.
        File tracesFile = ActivityManagerService.dumpStackTraces(firstPids,
                (isSilentAnr()) ? null : processCpuTracker, (isSilentAnr()) ? null : lastPids,
                nativePids);

        String cpuInfo = null;
        if (isMonitorCpuUsage()) {
            mService.updateCpuStatsNow();
            synchronized (mService.mProcessCpuTracker) {
                cpuInfo = mService.mProcessCpuTracker.printCurrentState(anrTime);
            }
            info.append(processCpuTracker.printCurrentLoad());
            info.append(cpuInfo);
        }

        info.append(processCpuTracker.printCurrentState(anrTime));

        Slog.e(TAG, info.toString());
        if (tracesFile == null) {
            // There is no trace file, so dump (only) the alleged culprit's threads to the log
            // 3.向进程发送放弃信号，也就是退出进程信号，这里会打印日志
            Process.sendSignal(pid, Process.SIGNAL_QUIT);
        }

        StatsLog.write(StatsLog.ANR_OCCURRED, uid, processName,
                activityShortComponentName == null ? "unknown": activityShortComponentName,
                annotation,
                (this.info != null) ? (this.info.isInstantApp()
                        ? StatsLog.ANROCCURRED__IS_INSTANT_APP__TRUE
                        : StatsLog.ANROCCURRED__IS_INSTANT_APP__FALSE)
                        : StatsLog.ANROCCURRED__IS_INSTANT_APP__UNAVAILABLE,
                isInterestingToUserLocked()
                        ? StatsLog.ANROCCURRED__FOREGROUND_STATE__FOREGROUND
                        : StatsLog.ANROCCURRED__FOREGROUND_STATE__BACKGROUND,
                getProcessClassEnum(),
                (this.info != null) ? this.info.packageName : "");
        final ProcessRecord parentPr = parentProcess != null
                ? (ProcessRecord) parentProcess.mOwner : null;
                //4. 将这次"ANR"的进程相关信息，组件信息，cpu信息和traces文件保存到dropbox，即data/system/dropbox目录
        mService.addErrorToDropBox("anr", this, processName, activityShortComponentName,
                parentShortComponentName, parentPr, annotation, cpuInfo, tracesFile, null);

        if (mWindowProcessController.appNotResponding(info.toString(), () -> kill("anr", true),
                () -> {
                    synchronized (mService) {
                        mService.mServices.scheduleServiceTimeoutLocked(this);
                    }
                })) {
            return;
        }

        synchronized (mService) {
            // mBatteryStatsService can be null if the AMS is constructed with injector only. This
            // will only happen in tests.
            if (mService.mBatteryStatsService != null) {
                mService.mBatteryStatsService.noteProcessAnr(processName, uid);
            }

            if (isSilentAnr() && !isDebugging()) {
                kill("bg anr", true);
                return;
            }

            // Set the app's notResponding state, and look up the errorReportReceiver
            makeAppNotRespondingLocked(activityShortComponentName,
                    annotation != null ? "ANR " + annotation : "ANR", info.toString());

            // mUiHandler can be null if the AMS is constructed with injector only. This will only
            // happen in tests.
            //5. 交给 ActivityManagerService 处理，再交给AppErrors 处理，然后弹出程序无响应对话框
            if (mService.mUiHandler != null) {
                // Bring up the infamous App Not Responding dialog
                Message msg = Message.obtain();
                msg.what = ActivityManagerService.SHOW_NOT_RESPONDING_UI_MSG;
                msg.obj = new AppNotRespondingDialog.Data(this, aInfo, aboveSystem);

                mService.mUiHandler.sendMessage(msg);
            }
        }
    }
```

这个方法做的主要事情是：

1. 收集应用程序最近使用的的进程 pid，只收集前五个进程 pid；
2. 在控制台打印ANR相关的日志，日志包含：当前进程名字，当前组件信息，当前进程id，前五个进程的相关信息，cpu信息等；
3. 向进程发送放弃信号，也就是退出进程信号，这里会打印日志；
4. 通过`AtivityManagerService#addErrorToDropBox`，将这次"ANR"的进程相关信息，组件信息，cpu信息和traces文件保存到dropbox，即data/system/dropbox目录中；
5. 交给 ActivityManagerService 处理，再交给AppErrors 处理，然后**弹出程序无响应对话框**。

## 总结

综上，创建 Service 触发 ANR的原理就是：在创建 Service 的时候，也就是调用 Service 的 `onCreate`  方法前会发送一个延时消息（如果是前台服务，延时时间就是 `SERVICE_TIMEOUT`(值20*1000豪秒)，否则延时时间是`SERVICE_BACKGROUND_TIMEOUT`(10 * SERVICE_TIMEOUT豪秒)）。），如果`onCreate` 方法在20 /200秒内执行完成，就会移除这个延时消息，不会触发 ANR；否则就会执行这个延时消息，并触发 ANR。 

---

参考文档：

- [Input系统—ANR原理分析](http://gityuan.com/2017/01/01/input-anr/%29)
