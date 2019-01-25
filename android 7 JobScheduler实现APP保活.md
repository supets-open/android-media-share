
  [https://blog.csdn.net/findsafety/article/details/80388430]()

[https://blog.csdn.net/qq_33689414/article/details/54668889]()

JobScheduler：当一系列预置的条件被满足时，JobScheduler API为你的应用执行一个操作，例如当设备接通电源适配器或者连接到WIFI，在API 21 ( Android 5.0(Lollipop) )中，google提供了一个新叫做JobScheduler API的组件来处理这样的场景。 
在API 24 ( Android 7.0(N) )的新特性中，Google对于新设备功耗要求越来越严格，对于APP的限制也越来约多，想继续像Android5.0、6.0一样简单处理定时执行执行周期性作业，需要做出一些代码调整，针对该需求，总结了2种解决方案供参考。

    setPeriodic 
    setPeriodic：按时间间隔执行周期性作业，在Android 5、6平台版本下可以间隔任何时间运行，在Android7.0平台版本上需设置定期作业的间隔时间>=15分钟时才能运行。

    setMinimumLatency 
    设置作业延迟执行的时间，与setPeriodic不可同时执行，可配置setOverrideDeadline设置作业最大延迟执行时间。

    setRequiredNetworkType 
    设置作业只有在满足指定的网络条件时才会被执行

    /** 默认条件，不管是否有网络这个作业都会被执行 */
    public static final int NETWORK_TYPE_NONE = 0;
    /** 任意一种网络这个作业都会被执行 */
    public static final int NETWORK_TYPE_ANY = 1;
    /** 不是蜂窝网络( 比如在WIFI连接时 )时作业才会被执行 */
    public static final int NETWORK_TYPE_UNMETERED = 2;
    /** 不在漫游时作业才会被执行 */
    public static final int NETWORK_TYPE_NOT_ROAMING = 3;

    1
    2
    3
    4
    5
    6
    7
    8

解决方案1

使用Context.getSystemService(Context.JOB_SCHEDULER_SERVICE)创建JobScheduler对象，配置调度工作setMinimumLatency等参数，延迟运行Job Service，在JobSchedulerService服务的onStartJob方法中启动保活服务，再创建一个新的JobScheduler任务，并结束当前JobScheduler任务。

    如果正在运行需要很长时间的任务，则下一个任务将安排在当前任务完成时间+ PROVIDED_TIME_INTERVAL

    创建JobScheduler 
    设置setExtras参数，将需要保活的服务名称传递至Job Service

     //API大于24
     if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.N) {
                        //7.0+
                        mJobScheduler = (JobScheduler) mContext.getSystemService(Context.JOB_SCHEDULER_SERVICE);
                        JobInfo.Builder builder = new JobInfo.Builder(1,
                                new ComponentName(mContext.getPackageName(), JobSchedulerService.class.getName()));
                        builder.setMinimumLatency(2 * 60 * 1000);
                        PersistableBundle persiBundle = new PersistableBundle();
                        persiBundle.putString("servicename", service.getName());
                        builder.setExtras(persiBundle);
                        if (mJobScheduler.schedule(builder.build()) <= 0) {
                            //If something goes wrong
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

    执行JobSchedulerService 工作任务 
    在onStartJob方法中，执行startService方法启动保活服务，创建一个新的JobScheduler对象，手动调用jobFinished(params, false)结束作业，并将返回值设为true

    onStartJob的返回值区别：
    (1) false：框架认为你作业已经执行完毕了，那么下一个作业就立刻展开了
    (2) true：框架将作业结束状态交给你去处理。因为我们可能会异步的通过线程等方式去执行工作，这个时间肯定不能放在主线程里面去控制，这时候需要手动调用jobFinished(JobParameters params, boolean needsReschedule)方法去告诉框架作业结束了，其中needsReschedule表示是否重复执行

    1
    2
    3

    public boolean onStartJob(JobParameters params) {
            try {
                Log.d("JobSchedulerService","start~~"+ System.currentTimeMillis());
                if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.N) {
                    Log.d("JobSchedulerService", "7.0 handleMessage task running");
                    String servicename = params.getExtras().getString("servicename");
                    Class service = getClassLoader().loadClass(servicename);
                    if (service != null) {
                        Log.d("JobSchedulerService", "7.0 handleMessage task running ~~2~~"+service.hashCode());
                        //判断保活的service是否被杀死
                        if (!isMyServiceRunning(service)) {
                            //重启service
                            startService(new Intent(getApplicationContext(), service));
                        }
                    }
                    //创建一个新的JobScheduler任务
                    scheduleRefresh(servicename);
                    jobFinished(params, false);
                    Log.d("JobSchedulerService","7.0 handleMessage task end~~"+ System.currentTimeMillis());
                    return true;
                }
            } catch (Exception e) {
                e.printStackTrace();
            }
            return false;
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

    scheduleRefresh

    private void scheduleRefresh(String serviceName) {
            JobScheduler mJobScheduler = (JobScheduler)getApplicationContext()
                    .getSystemService(JOB_SCHEDULER_SERVICE);
            //jobId可根据实际情况设定        
            JobInfo.Builder mJobBuilder =
                    new JobInfo.Builder(0,
                            new ComponentName(getPackageName(),
                                    JobSchedulerService.class.getName()));
     
            if (android.os.Build.VERSION.SDK_INT >= Build.VERSION_CODES.N) {
                mJobBuilder.setMinimumLatency(2* 60 * 1000).setRequiredNetworkType(JobInfo.NETWORK_TYPE_ANY);
                PersistableBundle persiBundle = new PersistableBundle();
                persiBundle.putString("servicename", serviceName);
                mJobBuilder.setExtras(persiBundle);
            }
     
            if (mJobScheduler != null && mJobScheduler.schedule(mJobBuilder.build())
                    <= JobScheduler.RESULT_FAILURE) {
                //Scheduled Failed/LOG or run fail safe measures
                Log.d("JobSchedulerService", "7.0 Unable to schedule the service FAILURE!");
            }else{
                Log.d("JobSchedulerService", "7.0 schedule the service SUCCESS!");
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

    判断保活Service是否在运行

        public boolean isMyServiceRunning(Class<?> serviceClass) {
            ActivityManager manager = (ActivityManager) getSystemService(Context.ACTIVITY_SERVICE);
            for (ActivityManager.RunningServiceInfo service : manager.getRunningServices(Integer.MAX_VALUE)) {
                if (serviceClass.getName().equals(service.service.getClassName())) {
                    return true;
                }
            }
            return false;
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

解决方案（作业间隔时间>=15分钟）

设定作业的间隔时间大于15分钟，采用setPeriodic方法实现定时执行周期性作业。

    private static final long REFRESH_INTERVAL = 15 * 60 * 1000;

    Job Scheduler代码

    public static void scheduleJob(Context context) {
        ComponentName serviceComponent = new ComponentName(context, PAJobService.class);
        JobInfo.Builder builder = new JobInfo.Builder(JOB_ID, serviceComponent);
        builder.setPeriodic(15 * 60 * 1000, 5 * 60 *1000);
     
        JobScheduler jobScheduler = context.getSystemService(JobScheduler.class);
        int ret = jobScheduler.schedule(builder.build());
        if (ret == JobScheduler.RESULT_SUCCESS) {
            Log.d(TAG, "Job scheduled successfully");
        } else {
            Log.d(TAG, "Job scheduling failed");
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

    JobService服务

    public class PAJobService extends JobService {
        private static final String TAG = PRE_TAG + PAJobService.class.getSimpleName();
        private LocationManager mLocationManager;
     
        public boolean onStartJob(JobParameters params) {
            Log.d(TAG, "onStartJob");
            Toast.makeText(getApplicationContext(), "Job Started", Toast.LENGTH_SHORT).show();
            return false;
        }
     
        public boolean onStopJob(JobParameters params) {
            Log.d(TAG, "onStopJob");
            return false;
        }
    }