Android(O) 8.0 怎样在后台启动service
2018年03月23日 19:15:00 c1392851600 阅读数：4010

在Android8.0之后, google对后台启动service进行了更加严格的限制, 具体可以参考官网文档, 上不了外网的朋友看这个网址:android中文官网, 这里摘出和后台service相关的内容:
后台服务限制

在后台中运行的服务会消耗设备资源，这可能降低用户体验。 为了缓解这一问题，系统对这些服务施加了一些限制。系统可以区分 前台 和 后台 应用。（用于服务限制目的的后台定义与内存管理使用的定义不同；一个应用按照内存管理的定义可能处于后台，但按照能够启动服务的定义又处于前台。）如果满足以下任意条件，应用将被视为处于前台：
• 具有可见 Activity（不管该 Activity 已启动还是已暂停）。
• 具有前台服务。
• 另一个前台应用已关联到该应用（不管是通过绑定到其中一个服务，还是通过使用其中一个内容提供程序）。 例如，如果另一个应用绑定到该应用的服务，那么该应用处于前台：
1. IME
2. 壁纸服务
3. 通知侦听器
4. 语音或文本服务
如果以上条件均不满足，应用将被视为处于后台。
处于前台时，应用可以自由创建和运行前台服务与后台服务。 进入后台时，在一个持续数分钟的时间窗内，应用仍可以创建和使用服务。
在该时间窗结束后，应用将被视为处于 空闲状态。 此时，系统将停止应用的后台服务，就像应用已经调用服务的“Service.stopSelf()”方法。
在这些情况下，后台应用将被置于一个临时白名单中并持续数分钟。 位于白名单中时，应用可以无限制地启动服务，并且其后台服务也可以运行。
处理对用户可见的任务时，应用将被置于白名单中，例如：
• 处理一条高优先级 Firebase 云消息传递 (FCM) 消息。
• 接收广播，例如短信/彩信消息。
• 从通知执行 PendingIntent。

绑定服务不受影响这些规则不会对绑定服务产生任何影响。 如果您的应用定义了绑定服务，则不管应用是否处于前台，其他组件都可以绑定到该服务。

官方也给出了响应的解决办法:

    在很多情况下，您的应用都可以使用 JobScheduler 作业替换后台服务。 例如，CoolPhotoApp
    需要检查用户是否已经从朋友那里收到共享的照片，即使该应用未在前台运行。
    Android 8.0 引入了一种全新的方法，即Context.startForegroundService()，以在前台启动新服务。在系统创建服务后，应用有五秒的时间来调用该服务的startForeground() 方法以显示新服务的用户可见通知。如果应用在此时间限制内未调用
    startForeground()，则系统将停止服务并声明此应用为 ANR。

先来看第一个方法利用startForegroundService启动后台service

Intent service = new Intent(TheApplication.getInstance().getApplicationContext(), MyBackgroundService.class);
service.putExtra("startType", 1);
if (Build.VERSION.SDK_INT >= 26) {
    TheApplication.getInstance().startForegroundService(service);
} else {
    TheApplication.getInstance().startService(service);
}

    1
    2
    3
    4
    5
    6
    7

简单介绍说明一下, 主要就是第四行和第六行的代码, 第四行就是启动前台service, 在启动之前判断一下如果当前手机系统版本高于26就启用前台service, 否则还是正常startService即可
启动完前台service, 一定记得在5s以内要执行如下代码, 否则程序会报ANR问题

  if (Build.VERSION.SDK_INT >= 26) {
      startForeground(1, new Notification());
  }

    1
    2
    3

一样还是先判断系统版本, 如果高于26就调用startForeground方法好了, 使用startForegroundService 方法启动后台service这么使用即可
第二种方法使用JobScheduler代替Service

    使用步骤如下
    1.创建一个类继承自JobService

public class TestJobService extends JobService {
    @Override
    public boolean onStartJob(JobParameters params) {
        return true;
    }

    @Override
    public boolean onStopJob(JobParameters params) {
        return false;
    }
}

    1
    2
    3
    4
    5
    6
    7
    8
    9
    10
    11

这两个方法很好理解, 一个是开始任务, 一个是停止任务, 这两个方法都是系统调用的
onStartJob实在任务开始的时候执行, 返回值如果是false表明调用onStartJob后系统任务任务已经处理完成, 注意:当返回false的时候, 当系统在收到取消请求的时候, 会认为当前已经没有任务在运行, 就不会调用对应的onStopJob方法了; 如果返回的是true, 表示告诉系统我的任务还在执行, 这时候系统就会把任务的结束调用交给用户去做, 也就是我们需要在任务执行完毕的时候手动去调用jobFinished(JobParameters params, boolean needsRescheduled)来通知系统. 这里的第一个参数params就是onStartJob方法的params, 第二个参数是表示是否在条件满足时重复执行
注意onStartJob本身是在主线程中的, 所以如果要做耗时的操作记得另开子线程处理
例如:

    @Override
    public boolean onStartJob(JobParameters params) {
        Log.d(TAG, "===onStartJob===");
        task = new Task();
        task.setJobParameters(params);
        // 启动对应的任务
        task.start();
        return true;
    }

    1
    2
    3
    4
    5
    6
    7
    8
    9

在里面开一个子线程处理我们的任务, 然后返回true, 这样系统就知道我们会主动去掉用jobFinished方法来通知系统任务完成
然后我们可以在task里当我们的耗时任务执行完毕的时候去调用jobFinished()方法， 以下就是整个JobScheduler的示例代码:

public class TestJobScheduler extends JobService {

    private String TAG = "TestJobScheduler";

    private Task task;

    @Override
    public void onCreate() {
        super.onCreate();
        Log.d(TAG, "===onCreate===");
    }

    @Override
    public void onDestroy() {
        super.onDestroy();
        Log.d(TAG, "===onDestroy===");
    }

    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {

        Log.d(TAG, "===onStartCommand===");

        return super.onStartCommand(intent, flags, startId);
    }

    @Override
    public boolean onStartJob(JobParameters params) {
        Log.d(TAG, "===onStartJob===");
        task = new Task();
        task.setJobParameters(params);
        // 启动对应的任务
        task.start();
        return true;
    }

    @Override
    public boolean onStopJob(JobParameters params) {
        Log.d(TAG, "===onStopJob===");
        return false;
    }

    class Task extends Thread {

        private JobParameters params;
        public void setJobParameters(JobParameters params) {
            this.params = params;
        }

        @Override
        public void run() {
            Log.d(TAG, " start record ");

            try {
                Thread.sleep(2000);
            } catch (Exception e) {
                e.printStackTrace();
            }
            dealResult(params);
        }
    }

    private void dealResult(JobParameters params) {
        try {
            // do something
        } catch (Exception e) {

        } finally {
            Log.d(TAG, "===finally===");
            jobFinished(params, false);
        }
    }
}

    1
    2
    3
    4
    5
    6
    7
    8
    9
    10
    11
    12
    13
    14
    15
    16
    17
    18
    19
    20
    21
    22
    23
    24
    25
    26
    27
    28
    29
    30
    31
    32
    33
    34
    35
    36
    37
    38
    39
    40
    41
    42
    43
    44
    45
    46
    47
    48
    49
    50
    51
    52
    53
    54
    55
    56
    57
    58
    59
    60
    61
    62
    63
    64
    65
    66
    67
    68
    69
    70
    71
    72
    73

最后就是怎么开启一个任务了, 先看代码

        JobScheduler mJobScheduler = (JobScheduler) context.getSystemService(Context.JOB_SCHEDULER_SERVICE);
        JobInfo.Builder builder = new JobInfo.Builder( 1,
                new ComponentName("com.wb.demo.jobschedule",
                        TestJobScheduler.class.getName() ) );
        builder.setMinimumLatency(10);
        builder.setPersisted(false);
        mJobScheduler.schedule(builder.build());

    1
    2
    3
    4
    5
    6
    7

第一个行就是获取一个JobScheduler对象, 第2到4行就是创建一个JobInfo.Builder对象,并设置响应的属性, 第一个参数1 是一个类似任务id的参数, 第二个ComponentName就是要启动的JobSchedulerService的包名和类名, 在调用schedule()这个方法的后, 如果预设置的条件满足, 系统就会调用你的JobSchedulerService的onStartJob()方法, 任务就会开始执行.
关于其他的几个设置介绍如下:
- setMinimumLatency(long minLatencyMillis): 表示多少毫秒后开始执行任务,这个函数与setPeriodic(long time)方法互斥，这两个方法同一时候调用了就会引起异常, 与setPeriodic(long time)互斥；
- setPeriodic(long intervalMillis): 表示多长时间重复执行一次
- setOverrideDeadline(long maxExecutionDelayMillis): 设置任务最大的延迟时间,也就是到设置的时间后, 不管其他条件是否满足都会开始任务. 与setMinimumLatency(long time)一样，该方法也会与setPeriodic(long time)互斥。
- setPersisted(boolean isPersisted): 表明当设备重新启动后你的任务是否还要继续运行。
- setRequiresCharging(boolean requiresCharging): 是否需要当设备在充电时任务才会被运行。
- setRequiresDeviceIdle(boolean requiresDeviceIdle): 表示任务只有当用户没有在使用该设备且有一段时间没有使用时才会启动。
- setRequiredNetworkType(int networkType): 指明在满足指定的网络条件时才会被运行。默认条件是JobInfo.NETWORK_TYPE_NONE, 即无论是否有网络这个任务都会被运行。另外两个可选类型。一种是JobInfo.NETWORK_TYPE_ANY，它表明须要随意一种网络才使得任务能够运行。还有一种是JobInfo.NETWORK_TYPE_UNMETERED，它表示设备不是蜂窝网络( 比方在WIFI连接时 )时任务才会被运行。剩下的还有两种分别是NETWORK_TYPE_NOT_ROAMING 不是漫游, NETWORK_TYPE_METERED. wifi
