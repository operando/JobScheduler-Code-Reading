# JobScheduler Code Reading

* **Android M 6.0.0_r1.0のソースコードをベースに調べたもの**


# Project Volta

* 簡単に言ってバッテリー消費を削減するプロジェクト
* Android Lで行われたもの
* [Battery Historian](https://github.com/google/battery-historian)
* [JobScheduler](http://developer.android.com/about/versions/android-5.0.html#Power)
* [AlarmManagerが省電力化(4.4 KitKat)](http://developer.android.com/about/versions/android-4.4.html#BehaviorAlarms)


# About JobScheduler

* Android Lから導入されたAPI
* 様々な条件のJobをスケジュールしてくれるAPI
* 使うことで消費電力を意識した実装ができる
* 開発者が頑張らなくていいAPI
* JobSchedulerはAndroid Framework Services(System Service系)
* マルチユーザ用にも設計されている(当たり前か)
* Schedulerであって、Alarmではない
* 特定の時間に実行！みたいな感じではない


# JobSchedulerの使い方

* [Android API21から追加されたJobSchedulerに 慣れていこう](http://blog.techfirm.co.jp/2015/10/19/android-api21%E3%81%8B%E3%82%89%E8%BF%BD%E5%8A%A0%E3%81%95%E3%82%8C%E3%81%9Fjobscheduler%E3%81%AB%E6%85%A3%E3%82%8C%E3%81%A6%E3%81%84%E3%81%93%E3%81%86/)
* [Using the JobScheduler API on Android Lollipop](http://code.tutsplus.com/tutorials/using-the-jobscheduler-api-on-android-lollipop--cms-23562)


# JobScheduler Sample

* [googlesamples/android-JobScheduler](https://github.com/googlesamples/android-JobScheduler)
* [operando/JobScheduler-Sample](https://github.com/operando/JobScheduler-Sample)
 * 実験用


## Android MのAuto BackupのJobSchedulerを使ってる

* Auto Backupがされる条件
* バックアップは24時間ごとに行われる
* バックアップは充電中、WiFi接続、アイドル状態の3つの条件が満たされた時に行われる
* この条件(24時間,充電中,WiFi接続,アイドル状態)を制御してるのがJobScheduler
* http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/services/backup/java/com/android/server/backup/FullBackupJob.java
* ↓参考程度
* http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/services/backup/java/com/android/server/backup/KeyValueBackupJob.java#60
* http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/services/backup/java/com/android/server/backup/


##### http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/services/backup/java/com/android/server/backup/FullBackupJob.java

```java
public static void schedule(Context ctx, long minDelay) {
    JobScheduler js = (JobScheduler) ctx.getSystemService(Context.JOB_SCHEDULER_SERVICE);
    JobInfo.Builder builder = new JobInfo.Builder(JOB_ID, sIdleService)
             .setRequiresDeviceIdle(true)
             .setRequiredNetworkType(JobInfo.NETWORK_TYPE_UNMETERED)
             .setRequiresCharging(true);
    if (minDelay > 0) {
        builder.setMinimumLatency(minDelay);
    }
    js.schedule(builder.build());
}
```


## JobScheduler scheduler = (JobScheduler) getSystemService(Context.JOB_SCHEDULER_SERVICE);で取得できるJobSchedulerの実態

* [JobSchedulerImpl](http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/core/java/android/app/JobSchedulerImpl.java)


## RAMサイズによって同時実行できるJobの数が変わる

* System propertyのro.config.low_ram=trueの場合、**同時実行できるJobの数は1つ**
* System propertyのro.config.low_ram=faseの場合、**同時実行できるJobの数は3つ**
* ココらへん見るとJobの数についてわかる
 * http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/JobSchedulerService.java#77
* ro.config.low_ramはAndroid4.4で導入された、メモリ搭載量が少ないターゲット向けの設定
  * https://source.android.com/devices/tech/config/low-ram.html
* Android Wearとかはro.config.low_ram=trueかな？
* JobServiceContextの数が1 or 3ってこと。JobServiceContextで実際にJobを走らせる
 * 自分でbuildする時に、JobServiceContextを生成する数を増やせば同時実行できるJob数は増える
 * 機種依存とかでいじってるのなくはないかも...

###### http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/JobSchedulerService.java#77

```java
/** The number of concurrent jobs we run at one time. */
private static final int MAX_JOB_CONTEXTS_COUNT
        = ActivityManager.isLowRamDeviceStatic() ? 1 : 3;
```

##### http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/core/java/android/app/ActivityManager.java#isLowRamDeviceStatic

```java
// ActivityManager.java
public static boolean isLowRamDeviceStatic() {
    return "true".equals(SystemProperties.get("ro.config.low_ram", "false"));
}
```

##### http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/JobSchedulerService.java#352

```java
// Create the "runners".
for (int i = 0; i < MAX_JOB_CONTEXTS_COUNT; i++) {
    mActiveServices.add(
            new JobServiceContext(this, mBatteryStats,
                    getContext().getMainLooper()));
}
```


## デバイス再起動後も動くJobが作れる

* JobInfo.Builder#setPersisted(true)でJobが再起動後も実行される
* 開発者が再起動後に自分でまたJobを登録する必要がない
* JobInfo.Builder#setExtras(PersistableBundle)で再起動後も値を引き継げる

#### サンプル(こんな感じ)

```java
PersistableBundle persistableBundle = new PersistableBundle();
persistableBundle.putInt("id", i);
JobInfo jobInfo = new JobInfo.Builder(i, serviceName)
        .setRequiredNetworkType(JobInfo.NETWORK_TYPE_ANY)
        .setPersisted(true)
        .setExtras(persistableBundle)
        .build();
scheduler.schedule(jobInfo);
```


## 再起動後も引き継げるってことはJobの情報をどこかに保存してるぞ...

* ここ → **/data/system/job/jobs.xml**
 * 見るためには要Root
* [JobStore](http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/JobStore.java)でそこら辺管理してる
 * http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/JobStore.java
* JobSchedulerに新しいJob(Persisted = trueのJob)が追加された または Jobが削除された際に、ファイルの内容がSyncされる
 * JobStore#add / JobStore#remove -> JobStore#WriteJobsMapToDiskRunnable -> WriteJobsMapToDiskRunnable
 * 書き込み処理は主にこれ
 * http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/JobStore.java#WriteJobsMapToDiskRunnable

## [/data/system/job/jobs.xmlの中身(サンプル)](./job.xml)

```xml
<?xml version='1.0' encoding='utf-8' standalone='yes' ?>
<job-info version="0">
    <job jobid="800" package="android" class="com.android.server.pm.BackgroundDexOptService" uid="1000">
        <constraints idle="true" charging="true" />
        <one-off delay="1446847721372" />
        <extras />
    </job>
    <job jobid="808" package="android" class="com.android.server.MountServiceIdler" uid="1000">
        <constraints idle="true" charging="true" />
        <one-off delay="1446883201683" />
        <extras />
    </job>
    <job jobid="1" package="com.android.providers.downloads" class="com.android.providers.downloads.DownloadIdleService" uid="10006">
        <constraints idle="true" charging="true" />
        <periodic period="86400000" deadline="1446913897467" delay="1446847721311" />
        <extras />
    </job>
    <job jobid="0" package="com.os.operando.jobschedulersample" class="com.os.operando.jobschedulersample.services.MyJobService" uid="10062">
        <constraints connectivity="true" idle="true" />
        <one-off />
        <extras>
            <int name="id" value="0" />
            <string name="test">value0</string>
        </extras>
    </job>
    <job jobid="20536" package="android" class="com.android.server.backup.FullBackupJob" uid="1000">
        <constraints unmetered="true" idle="true" charging="true" />
        <one-off />
        <extras />
    </job>
</job-info>
```

##### http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/JobStore.java#246

```java
/**
 * Every time the state changes we write all the jobs in one swath, instead of trying to
 * track incremental changes.
 * @return Whether the operation was successful. This will only fail for e.g. if the system is
 * low on storage. If this happens, we continue as normal
 */
private void maybeWriteStatusToDiskAsync() {
    mDirtyOperations++;
    if (mDirtyOperations >= MAX_OPS_BEFORE_WRITE) {
        if (DEBUG) {
            Slog.v(TAG, "Writing jobs to disk.");
        }
        mIoHandler.post(new WriteJobsMapToDiskRunnable());
    }
}
```

## JobInfo.Builder#setExtrasでセットしたPersistableBundle

* PersistableBundleの内容は/data/system/job/jobs.xmlに書き込まれている

```xml
<job jobid="0" package="com.os.operando.jobschedulersample" class="com.os.operando.jobschedulersample.services.MyJobService" uid="10059">
    <constraints connectivity="true" idle="true" charging="true" />
    <periodic period="1000" deadline="1446648619790" delay="1446648618790" />
    <extras>
        <int name="id" value="0" />
        <string name="test">value0</string>
    </extras>
</job>
```

* xmlへの書き込み処理はここでやってる
 * http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/core/java/android/os/PersistableBundle.java#restoreFromXml
* 書き込み処理はJobStoreないから呼び出される
 * http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/JobStore.java#608
* PersistableBundleの情報を保存する処理は、保存するXMLだけ用意してあげれば、どんなものでも使える汎用的なものっぽい。
* restoreFromXmlがhideなので、サードパーティからは使えないけど・・・。


## 端末起動時のJobScheduler実行

* onBootPhaseのphaseがPHASE_THIRD_PARTY_APPS_CAN_STARTだったらJobを実行
* 名前通り端末起動段階で、サードパーティアプリのJobが実行可能になったってこと
* ここでmReadyToRockがtrueになる
 * mReadyToRockがfalseだとJobの追加とかができない
* 同時実行できるJobの数分だけmActiveServices(List<JobServiceContext>)にJobServiceContextをadd
* JobStore(mJobs)からJob(JobStatus)を取り出して、各StateControllerに通知する
 * これは再起動前に/data/system/jobs/job.xmlに書き込まれた情報から取り出す
 * これが再起動後も実行可能はJob。JobInfo.Builder#setPersisted(true)したJob
* 後はmHandler.obtainMessage(MSG_CHECK_JOB).sendToTarget();してJobが実行できるかチェック
 * GO GO GO!


##### http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/JobSchedulerService.java#346

```java
@Override
public void onBootPhase(int phase) {
    if (PHASE_SYSTEM_SERVICES_READY == phase) {
        // Register br for package removals and user removals.
        final IntentFilter filter = new IntentFilter(Intent.ACTION_PACKAGE_REMOVED);
        filter.addDataScheme("package");
        getContext().registerReceiverAsUser(
                mBroadcastReceiver, UserHandle.ALL, filter, null, null);
        final IntentFilter userFilter = new IntentFilter(Intent.ACTION_USER_REMOVED);
        userFilter.addAction(PowerManager.ACTION_DEVICE_IDLE_MODE_CHANGED);
        getContext().registerReceiverAsUser(
                mBroadcastReceiver, UserHandle.ALL, userFilter, null, null);
        mPowerManager = (PowerManager)getContext().getSystemService(Context.POWER_SERVICE);
    } else if (phase == PHASE_THIRD_PARTY_APPS_CAN_START) {
        synchronized (mJobs) {
            // Let's go!
            mReadyToRock = true;
            mBatteryStats = IBatteryStats.Stub.asInterface(ServiceManager.getService(
                    BatteryStats.SERVICE_NAME));
            // Create the "runners".
            for (int i = 0; i < MAX_JOB_CONTEXTS_COUNT; i++) {
                mActiveServices.add(
                        new JobServiceContext(this, mBatteryStats,
                                getContext().getMainLooper()));
            }
            // Attach jobs to their controllers.
            ArraySet<JobStatus> jobs = mJobs.getJobs();
            for (int i=0; i<jobs.size(); i++) {
                JobStatus job = jobs.valueAt(i);
                for (int controller=0; controller<mControllers.size(); controller++) {
                    mControllers.get(controller).deviceIdleModeChanged(mDeviceIdleMode);
                    mControllers.get(controller).maybeStartTrackingJob(job);
                }
            }
            // GO GO GO!
            mHandler.obtainMessage(MSG_CHECK_JOB).sendToTarget();
        }
    }
}
```


## [setBackoffCriteria(long initialBackoffMillis, int backoffPolicy)](http://developer.android.com/intl/ja/reference/android/app/job/JobInfo.Builder.html#setBackoffCriteria(long, int))メソッドの動作

* MAX_BACKOFF_DELAY_MILLIS = 5 * 60 * 60 * 1000;  // 5 hours.

* JobInfo.BACKOFF_POLICY_LINEAR
 * initialBackoffMillis * backoffAttempts

* JobInfo.BACKOFF_POLICY_EXPONENTIAL
 * initialBackoffMillis *  2^backoffAttempts
 * BACKOFF_POLICY_LINEAR or BACKOFF_POLICY_EXPONENTIAL以外指定した場合はこっちで計算される

 * delayMillis = Math.min(delayMillis, JobInfo.MAX_BACKOFF_DELAY_MILLIS)
  * 5時間以上の遅らせるJobは登録できないってことか

##### http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/JobSchedulerService.java#getRescheduleJobForFailure

```java
private JobStatus getRescheduleJobForFailure(JobStatus failureToReschedule) {
    final long elapsedNowMillis = SystemClock.elapsedRealtime();
    final JobInfo job = failureToReschedule.getJob();

    final long initialBackoffMillis = job.getInitialBackoffMillis();
    final int backoffAttempts = failureToReschedule.getNumFailures() + 1;
    long delayMillis;

    switch (job.getBackoffPolicy()) {
        case JobInfo.BACKOFF_POLICY_LINEAR:
            delayMillis = initialBackoffMillis * backoffAttempts;
            break;
        default:
            if (DEBUG) {
                Slog.v(TAG, "Unrecognised back-off policy, defaulting to exponential.");
            }
        case JobInfo.BACKOFF_POLICY_EXPONENTIAL:
            delayMillis =
                    (long) Math.scalb(initialBackoffMillis, backoffAttempts - 1);
            break;
    }
    delayMillis =
            Math.min(delayMillis, JobInfo.MAX_BACKOFF_DELAY_MILLIS);
    return new JobStatus(failureToReschedule, elapsedNowMillis + delayMillis,
            JobStatus.NO_LATEST_RUNTIME, backoffAttempts);
}
```


# StateControllerの種類

* JobManager
* 各条件の監視をして、Jobの状態をコントロールする
* [StateController](http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/controllers/StateController.java) (抽象クラス)
 * [AppIdleController](http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/controllers/AppIdleController.java)
 * [BatteryController](http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/controllers/BatteryController.java)
 * [ConnectivityController](http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/controllers/ConnectivityController.java)
 * [IdleController](http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/controllers/IdleController.java)
 * [JobStatus](http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/controllers/JobStatus.java)
 * [TimeController](http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/controllers/TimeController.java)

## frameworks/base/services/core/java/com/android/server/job/controllers/

* http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/controllers/StateController.java
 * 各controllerの抽象クラス
* public abstract void maybeStartTrackingJob(JobStatus jobStatus);
 * jobをaddする
* public abstract void maybeStopTrackingJob(JobStatus jobStatus);
 * jobをremoveする
* public abstract void dumpControllerState(PrintWriter pw);
 * dumpsysした時にjobの状態をdumpする


##### http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/controllers/IdleController.java

```java
/**
 * StateController interface
 */
@Override
public void maybeStartTrackingJob(JobStatus taskStatus) {
    if (taskStatus.hasIdleConstraint()) {
        synchronized (mTrackedTasks) {
            mTrackedTasks.add(taskStatus);
            taskStatus.idleConstraintSatisfied.set(mIdleTracker.isIdle());
        }
    }
}

@Override
public void maybeStopTrackingJob(JobStatus taskStatus) {
    synchronized (mTrackedTasks) {
        mTrackedTasks.remove(taskStatus);
    }
}

@Override
public void dumpControllerState(PrintWriter pw) {
    synchronized (mTrackedTasks) {
        pw.print("Idle: ");
        pw.println(mIdleTracker.isIdle() ? "true" : "false");
        pw.println(mTrackedTasks.size());
        for (int i = 0; i < mTrackedTasks.size(); i++) {
            final JobStatus js = mTrackedTasks.get(i);
            pw.print("  ");
            pw.print(String.valueOf(js.hashCode()).substring(0, 3));
            pw.println("..");
        }
    }
}
```


## [各Controllerの生成箇所](http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/JobSchedulerService.java#313)

##### http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/JobSchedulerService.java#313

```java
public JobSchedulerService(Context context) {
    super(context);
    // Create the controllers.
    mControllers = new ArrayList<StateController>();
    mControllers.add(ConnectivityController.get(this));
    mControllers.add(TimeController.get(this));
    mControllers.add(IdleController.get(this));
    mControllers.add(BatteryController.get(this));
    mControllers.add(AppIdleController.get(this));

    mHandler = new JobHandler(context.getMainLooper());
    mJobSchedulerStub = new JobSchedulerStub();
    mJobs = JobStore.initAndGet(this);
}
```


## [IdleController](http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/controllers/IdleController.java)

* JobInfo#setRequiresDeviceIdleがtrueのjobだけ登録されている


## [AppIdleController](http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/controllers/AppIdleController.java)

```
Controls when apps are considered idle and if jobs pertaining to those apps should be executed.
Apps that haven't been actively launched or accessed from a foreground app for a certain amount of time (maybe hours or days) are considered idle.
When the app comes out of idle state, it will be allowed to run scheduled jobs.
```

* UsageStatsと関連性のあるjob controllerっぽい
* アプリの使用具合をみてjobを制御してる？？
* idle状態でも実行できるようにしているjobはここでも管理されるのか？
* UsageStatsの状態変化によって、アプリがIdelかどうかを管理してるのかなー
* UsageStatsもちゃんと見ないとだわ
* http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/core/java/android/app/usage/UsageStatsManagerInternal.java#UsageStatsManagerInternal


## Jobの実行可能状態(ready)について

* Jobがready(実行可能)になる条件は、Jobに登録した条件によって異なる
 * [JobStatus#isReady](http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/controllers/JobStatus.java#198)
* readyは実行可能状態なので、readyになったからってすぐにJobが実行されるわけじゃない
* System側が定義してる一定量のJobが溜まったら実行されるとか
 * TODO : ココらへんの実行される条件とかタイミングとか詳しく調べる
* 条件が細かい...
 * TODO : もう少し詳しく調べる

##### http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/controllers/JobStatus.java#198

```java
/**
 * @return Whether or not this job is ready to run, based on its requirements. This is true if
 * the constraints are satisfied <strong>or</strong> the deadline on the job has expired.
 */
public synchronized boolean isReady() {
    // Deadline constraint trumps other constraints
    // AppNotIdle implicit constraint trumps all!
    return (isConstraintsSatisfied()
                || (hasDeadlineConstraint() && deadlineConstraintSatisfied.get()))
            && appNotIdleConstraintSatisfied.get();
}
```


## adb shell dumpsys jobscheduler

* サンプル
* TODO : Dumpされる情報を細かく調べる。意味とか

```java
Started users: u0 u10
Registered jobs:
  326..:[ComponentInfo{com.android.providers.downloads/com.android.providers.downloads.DownloadIdleService},jId=1,u10,R=(-02:40,23:57:19),N=0,C=true,I=true,F=0,P=false,ANI=true]
  425..:[ComponentInfo{android/com.android.server.pm.BackgroundDexOptService},jId=800,u0,R=(-2:20:18,none),N=0,C=true,I=true,F=0,P=false,ANI=true]
  981..:[ComponentInfo{android/com.android.server.backup.KeyValueBackupJob},jId=20537,u0,R=(1:43:07,21:39:44),N=1,C=true,I=false,F=0,P=false,ANI=true]
  105..:[ComponentInfo{com.os.operando.jobschedulersample/com.os.operando.jobschedulersample.services.MyJobService},jId=0,u0,R=(none,none),N=0,C=true,I=false,F=0,P=false,ANI=true]
  174..:[ComponentInfo{android/com.android.server.MountServiceIdler},jId=808,u0,R=(16:47:50,none),N=0,C=true,I=true,F=0,P=false,ANI=true]
  202..:[ComponentInfo{com.os.operando.jobschedulersample/com.os.operando.jobschedulersample.services.MyJobService},jId=0,u10,R=(none,none),N=0,C=true,I=false,F=0,P=false,ANI=true]
  203..:[ComponentInfo{com.android.providers.downloads/com.android.providers.downloads.DownloadIdleService},jId=1,u0,R=(-20:17,23:39:42),N=0,C=true,I=true,F=0,P=true,ANI=true]
  212..:[ComponentInfo{android/com.android.server.backup.FullBackupJob},jId=20536,u0,R=(none,none),N=2,C=true,I=true,F=0,P=false,ANI=true]

Conn.
connected: true unmetered: true
981..: C=true, UM=false
212..: C=false, UM=true

Alarms (8429004)
Next delay alarm in 6187s
Next deadline alarm in 77984s
Tracking:
981..: (14616797, 86413281)
203..: (7211953, 93611953)
326..: (8268760, 94668760)
174..: (68900000, N/A)

Idle: false
5
  425..
  174..
  203..
  326..
  212..

Batt.
Stable power: false
42516548,174129466,98154298,203795623,32629981,212117473,105396486,202238442

AppIdle
Parole On: false
android:idle=false, android:idle=false, android:idle=false, com.android.providers.downloads:idle=false, com.android.providers.downloads:idle=false, android:idle=false, com.os.operando.jobschedulersample:idle=false, com.os.operando.jobschedulersample:idle=false,

Pending:

Active jobs:

mReadyToRock=true
mDeviceIdleMode=false
```

### マルチユーザ

* マルチユーザのJobも管理してる

```
Started users: u0 u10
```

### JobInfo.Builder#setRequiredNetworkTypeでセットした状態

 * N=0 : JobInfo.NETWORK_TYPE_NONE
 * N=1 : JobInfo.NETWORK_TYPE_ANY
 * N=2 : JobInfo.NETWORK_TYPE_UNMETERED(Wi-Fi)

```
212..:[ComponentInfo{android/com.android.server.backup.KeyValueBackupJob},jId=20537,u0,R=(3:16:08,23:09:42),N=1,C=true,I=false,F=0,P=false,ANI=true]
231..:[ComponentInfo{android/com.android.server.backup.FullBackupJob},jId=20536,u0,R=(none,none),N=2,C=true,I=true,F=0,P=false,ANI=true]
```

### (READY)
 * Jobが実行可能状態になったことを意味する

```
144..:[ComponentInfo{com.os.operando.jobschedulersample/com.os.operando.jobschedulersample.services.MyJobService},jId=0,u0,R=(-00:18,00:38),N=1,C=false,I=false,F=0,P=false,ANI=true(READY)
```

### Pending

* Pending中のJob
 * Active Jobsの枠が開けば、実行される

```
Pending:
43620931
961017
228486462
229742495
```


### Active Jobs

* 実行中のJob
 * timeoutとかも設定されてる

```
Active jobs:
Running for: 2s timeout=18770473 fromnow=597716
  235..:[ComponentInfo{com.os.operando.jobschedulersample/com.os.operando.jobschedulersample.services.MyJobService},jId=0,u0,R=(-00:59,-00:02),N=1,C=false,I=false,F=0,P=false,ANI=true(READY)]
```


## Debugging JobScheduler

* adb shell dumpsys jobscheduler
 * JobSchedulerに登録されているJobをDumpする
 * めっちゃ使う
* adb logcat -s JobSchedulerService
 * JobSchedulerServiceのログ。あんまり出ないけど...
* idle状態にするコマンド
 * adb shell dumpsys battery unplug
 * adb shell dumpsys deviceidle enable
 * adb shell dumpsys deviceidle step
 * adb shell dumpsys deviceidle force-idle


## アプリアンインストール時・ユーザ削除時のJobSchedulerServiceのHandling

* IntentFilterを使って、Handlingしてる
* アンインストールされたアプリのJobの削除
* 削除されたユーザのJobの削除
* onBootPhaseはSystem Service系の起動段階を通知するものっぽい

##### http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/JobSchedulerService.java#141

```java
/**
 * Cleans up outstanding jobs when a package is removed. Even if it's being replaced later we
 * still clean up. On reinstall the package will have a new uid.
 */
private final BroadcastReceiver mBroadcastReceiver = new BroadcastReceiver() {
    @Override
    public void onReceive(Context context, Intent intent) {
        Slog.d(TAG, "Receieved: " + intent.getAction());
        if (Intent.ACTION_PACKAGE_REMOVED.equals(intent.getAction())) {
            // If this is an outright uninstall rather than the first half of an
            // app update sequence, cancel the jobs associated with the app.
            if (!intent.getBooleanExtra(Intent.EXTRA_REPLACING, false)) {
                int uidRemoved = intent.getIntExtra(Intent.EXTRA_UID, -1);
                if (DEBUG) {
                    Slog.d(TAG, "Removing jobs for uid: " + uidRemoved);
                }
                cancelJobsForUid(uidRemoved);
            }
        } else if (Intent.ACTION_USER_REMOVED.equals(intent.getAction())) {
            final int userId = intent.getIntExtra(Intent.EXTRA_USER_HANDLE, 0);
            if (DEBUG) {
                Slog.d(TAG, "Removing jobs for user: " + userId);
            }
            cancelJobsForUser(userId);
        } else if (PowerManager.ACTION_DEVICE_IDLE_MODE_CHANGED.equals(intent.getAction())) {
            updateIdleMode(mPowerManager != null ? mPowerManager.isDeviceIdleMode() : false);
        }
    }
};
```


##### http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/JobSchedulerService.java#335

```java
@Override
public void onBootPhase(int phase) {
    if (PHASE_SYSTEM_SERVICES_READY == phase) {
        // Register br for package removals and user removals.
        final IntentFilter filter = new IntentFilter(Intent.ACTION_PACKAGE_REMOVED);
        filter.addDataScheme("package");
        getContext().registerReceiverAsUser(
                mBroadcastReceiver, UserHandle.ALL, filter, null, null);
        final IntentFilter userFilter = new IntentFilter(Intent.ACTION_USER_REMOVED);
        userFilter.addAction(PowerManager.ACTION_DEVICE_IDLE_MODE_CHANGED);
        getContext().registerReceiverAsUser(
                mBroadcastReceiver, UserHandle.ALL, userFilter, null, null);
        mPowerManager = (PowerManager)getContext().getSystemService(Context.POWER_SERVICE);
    } else if (phase == PHASE_THIRD_PARTY_APPS_CAN_START) {
        synchronized (mJobs) {
            // Let's go!
            mReadyToRock = true;
            mBatteryStats = IBatteryStats.Stub.asInterface(ServiceManager.getService(
                    BatteryStats.SERVICE_NAME));
            // Create the "runners".
            for (int i = 0; i < MAX_JOB_CONTEXTS_COUNT; i++) {
                mActiveServices.add(
                        new JobServiceContext(this, mBatteryStats,
                                getContext().getMainLooper()));
            }
            // Attach jobs to their controllers.
            ArraySet<JobStatus> jobs = mJobs.getJobs();
            for (int i=0; i<jobs.size(); i++) {
                JobStatus job = jobs.valueAt(i);
                for (int controller=0; controller<mControllers.size(); controller++) {
                    mControllers.get(controller).deviceIdleModeChanged(mDeviceIdleMode);
                    mControllers.get(controller).maybeStartTrackingJob(job);
                }
            }
            // GO GO GO!
            mHandler.obtainMessage(MSG_CHECK_JOB).sendToTarget();
        }
    }
}
```


## JobService#onStartJob

* MainThreadで実行される
* なので重い処理はAnother Threadを作って処理する
* jobFinishedしないとjobがはけないから気をつけて。新しいjobが実行できなくなる


## JobService#onStopJob

* 戻り値でtrueを返せばjob作成時に、指定された再試行の基準に基づいてjobを再スケジューリング
* 戻り値でfalseを返せばjobは止まる
* onStopJobが呼ばれるのは、保留中・実行中のjobが何かにより(登録したjobの更新)cancelされた際に呼ばれるっぽい
* job実行中にcancelAllとかすれば呼ばれる


# scheduleの流れ

## JobScheduler.schedule(JobInfo)

* プロセス間通信でJobSchedulerStub#scheduleを呼び出す
 * http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/core/java/android/app/JobSchedulerImpl.java#schedule
* Jobの登録ができればJobScheduler.RESULT_SUCCESSが返ってくる。登録に失敗した場合JobScheduler.RESULT_FAILUREが返ってくる。
* なので、Jobの登録ができたかどうかしっかり見てあげよう


## [JobSchedulerStub#schedule](http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/JobSchedulerService.java#813)

* enforceValidJobRequestやcanPersistJobsでJob登録ができるかどうかとチェック
* JobSchedulerService#scheduleを呼び出す
* http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/JobSchedulerService.java#813


## [JobSchedulerService#schedule](http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/JobSchedulerService.java#183)

* JobInfoとアプリのUIDでJobStatusを生成
 * Framework側ではJobInfoはJobStatusとして管理される
* cancelJob + cancelJobImplで同じJobが登録されていたらcancelする
 * 同じJob = Job ID + UIDが同じJobStatus
* pendingとしてqueueに溜まってるJobもcancel
* ActiveServices(実際に動いてるJob)を全部チェックして、同じJobがあればcancel処理を行う


## [JobSchedulerService#stopTrackingJobs](http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/JobSchedulerService.java#400)

* JobStoreにある同じJobをremove
* 各StateControllerから同じJobをremove(removeするにもJobStatusが各StateControllerの条件にあっているかどうかをチェックしてる)
* 全JobStatusを管理するクラスとしてJobStoreが使われている


## [JobSchedulerService#startTrackingJob](http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/JobSchedulerService.java#378)

* 基本的にJobSchedulerService#stopTrackingJobと逆のことする(JobStatusのadd)
* 各StateControllerにJobStatusをaddする(addするにもJobStatusが各StateControllerの条件にあっているかどうかをチェックしてる)
* mReadyToRock(JobSchedulerの準備??)がtrueになっていないと、基本的にはaddもremoveもできない
 * http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/JobSchedulerService.java#373


## [JobSchedulerService#maybeQueueReadyJobsForExecutionLockedH](http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/JobSchedulerService.java#622)

* ready(実行可能)になっているJobがいくつ存在するかでJobをPendingにするか決めてる?
 * Right now the policy is such:
 * If >1 of the ready jobs is idle mode we send all of them off
 * if more than 2 network connectivity jobs are ready we send them all off.
 * If more than 4 jobs total are ready we send them all off.
* 検証
 * JobInfo.Builder#setRequiredNetworkType(JobInfo.NETWORK_TYPE_ANY)のjobを１つずつ登録
 * 1個目のJobを登録したら、JobはREADYのままで止まっている(Pendingにも入ってない)
 * 2個目のJobを登録したら、JobがActiveになって実行された
 * 上の条件(2 network connectivity jobs are ready)に合致したので、Jobをpending queueに移動させて、実行したのだと推測
* Jobがready(実行可能)になっても、登録されているJob全体として条件を満たしてなければJobがPendingまで移行しないってことっぽい
 * TODO : 他にもPendingになるルートがあるから、そこ調べる


##### http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/JobSchedulerService.java#622

```java
/**
 * The state of at least one job has changed. Here is where we could enforce various
 * policies on when we want to execute jobs.
 * Right now the policy is such:
 * If >1 of the ready jobs is idle mode we send all of them off
 * if more than 2 network connectivity jobs are ready we send them all off.
 * If more than 4 jobs total are ready we send them all off.
 * TODO: It would be nice to consolidate these sort of high-level policies somewhere.
 */
private void maybeQueueReadyJobsForExecutionLockedH() {
    int chargingCount = 0;
    int idleCount =  0;
    int backoffCount = 0;
    int connectivityCount = 0;
    List<JobStatus> runnableJobs = new ArrayList<JobStatus>();
    ArraySet<JobStatus> jobs = mJobs.getJobs();
    for (int i=0; i<jobs.size(); i++) {
        JobStatus job = jobs.valueAt(i);
        if (isReadyToBeExecutedLocked(job)) {
            if (job.getNumFailures() > 0) {
                backoffCount++;
            }
            if (job.hasIdleConstraint()) {
                idleCount++;
            }
            if (job.hasConnectivityConstraint() || job.hasUnmeteredConstraint()) {
                connectivityCount++;
            }
            if (job.hasChargingConstraint()) {
                chargingCount++;
            }
            runnableJobs.add(job);
        } else if (isReadyToBeCancelledLocked(job)) {
            stopJobOnServiceContextLocked(job);
        }
    }
    if (backoffCount > 0 ||
            idleCount >= MIN_IDLE_COUNT ||
            connectivityCount >= MIN_CONNECTIVITY_COUNT ||
            chargingCount >= MIN_CHARGING_COUNT ||
            runnableJobs.size() >= MIN_READY_JOBS_COUNT) {
        if (DEBUG) {
            Slog.d(TAG, "maybeQueueReadyJobsForExecutionLockedH: Running jobs.");
        }
        for (int i=0; i<runnableJobs.size(); i++) {
            mPendingJobs.add(runnableJobs.get(i));
        }
    } else {
        if (DEBUG) {
            Slog.d(TAG, "maybeQueueReadyJobsForExecutionLockedH: Not running anything.");
        }
    }
    if (DEBUG) {
        Slog.d(TAG, "idle=" + idleCount + " connectivity=" +
        connectivityCount + " charging=" + chargingCount + " tot=" +
                runnableJobs.size());
    }
}
```

## [JobService#jobFinished](http://developer.android.com/reference/android/app/job/JobService.html#jobFinished(android.app.job.JobParameters, boolean))の流れ

* [JobServiceContext#jobFinished](http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/JobServiceContext.java#jobFinished)を呼び出す
* [JobServiceContext$JobServiceHandler#handleMessage](http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/JobServiceContext.java#308)
* [JobServiceContext$JobServiceHandler#handleFinishedH](http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/JobServiceContext.java#handleFinishedH)
* [JobServiceContext$JobServiceHandler#closeAndCleanupJobH](http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/JobServiceContext.java#closeAndCleanupJobH)
* [JobSchedulerService#onJobCompleted](http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/JobSchedulerService.java#509)
 * needsRescheduleがtrueなら失敗jobと扱い再スケジューリング(getRescheduleJobForFailure)
 * setPeriodicをセットしてればjobを再スケジューリング(getRescheduleJobForPeriodic)


## [JobSchedulerService#isReadyToBeExecutedLocked](http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/JobSchedulerService.java#681)

* Jobをpending queueに移動するための条件をチェックする
* It's ready. = JobStatus#isReady
* It's not pending. = PendingJobs#contains
* It's not already running on a JSC(JobServiceContext). = JobSchedulerService#isCurrentlyActiveLocked(jobが実行中じゃないかどうか)
* The user that requested the job is running. = mStartedUsers#contains = 動いているUserのJobかどうか(動いている ≠ 今選択してるユーザ)
 * 選択中のユーザ以外のJobも条件を満たせば動くってこと？オーナーのJobなら動いてもおかしくないか？
* http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/JobSchedulerService.java##isReadyToBeExecutedLocked

##### http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/JobSchedulerService.java##isReadyToBeExecutedLocked

```java
/**
 * Criteria for moving a job into the pending queue:
 *      - It's ready.
 *      - It's not pending.
 *      - It's not already running on a JSC.
 *      - The user that requested the job is running.
 */
private boolean isReadyToBeExecutedLocked(JobStatus job) {
    final boolean jobReady = job.isReady();
    final boolean jobPending = mPendingJobs.contains(job);
    final boolean jobActive = isCurrentlyActiveLocked(job);
    final boolean userRunning = mStartedUsers.contains(job.getUserId());
    if (DEBUG) {
        Slog.v(TAG, "isReadyToBeExecutedLocked: " + job.toShortString()
                + " ready=" + jobReady + " pending=" + jobPending
                + " active=" + jobActive + " userRunning=" + userRunning);
    }
    return userRunning && jobReady && !jobPending && !jobActive;
}
```


## [JobSchedulerService#isReadyToBeCancelledLocked](http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/JobSchedulerService.java#701)

 * It's not ready
 * It's running on a JSC.
 * http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/JobSchedulerService.java#isReadyToBeCancelledLocked

##### http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/JobSchedulerService.java#701

```java
/**
 * Criteria for cancelling an active job:
 *      - It's not ready
 *      - It's running on a JSC.
 */
private boolean isReadyToBeCancelledLocked(JobStatus job) {
    return !job.isReady() && isCurrentlyActiveLocked(job);
}
```


## JobInfo.Builder#setRequiresDeviceIdle

* DeviceがIdelの時の実行するようにする
* 他の条件と合わせると(setMinimumLatencyとか)Idel状態じゃなくても実行される??
 * 他の条件が整ったら実行されるって感じかな？
 * そんなことないか・・・
 * TODO : 要検証
* ドキュメントに「As such, it is a good time to perform resource heavy jobs.」って書いてある
 * デバイスがIdelということは、あまり他のアプリも動いてないからJobを動かすには最適ってことかな？
 * 省電力には協力的なのかどうか・・・


## Command Memo

* adb shell am stop-user 10
* adb logcat -s JobSchedulerService
* adb shell dumpsys jobscheduler
* App Standby
 * adb shell dumpsys battery unplug
 * adb shell am set-inactive <packageName> true
 * adb shell am set-inactive <packageName> false

## Android wear

* JobSchedulerあるね
* Jobの同時実行数が1つかもしれない
 * **TODO : 要調査**

```
Started users: u0
Registered jobs:
  162..:[ComponentInfo{android/com.android.server.MountServiceIdler},jId=808,u0,R=(17:57:02,none),N=0,C=true,I=true,F=0,P=false]
  312..:[ComponentInfo{android/com.android.server.pm.BackgroundDexOptService},jId=800,u0,R=(none,none),N=0,C=true,I=true,F=0,P=false]
  526..:[ComponentInfo{com.android.providers.downloads/com.android.providers.downloads.DownloadIdleService},jId=1,u0,R=(-00:51,23:59:08),N=0,C=true,I=true,F=0,P=false]
  702..:[ComponentInfo{com.google.android.wearable.app/com.google.android.clockwork.home.watchfaces.WatchFacePackageJobService},jId=710751,u0,R=(-00:45,23:59:14),N=0,C=true,I=true,F=0,P=false]

Conn.
connected: false unmetered: false

Alarms (145279)
Next delay alarm in 64622s
Next deadline alarm in 86348s
Tracking:
526..: (93711, 86493711)
702..: (99646, 86499646)
162..: (64767413, N/A)

Idle: false
4
  162..
  312..
  526..
  702..

Batt.
Stable power: false
162893216,312256788,526538261,702085409

Pending:

Active jobs:

mReadyToRock=true
```


## DeviceIdleController

* Dozeに関係あり
* http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/services/core/java/com/android/server/DeviceIdleController.java
* adb shell dumpsys deviceidle


## 実験(済)

* 10000件のJobを登録してみた
 * 登録できた。途中で落ちるかなーと思ったけど。OOMとか。

## 実験(未検証/検証するもの)

* すぐには実行されなそうなJobを登録しまくってOOMでJobSchedulerServiceをCrashさせて、多分System Crashして再起動するんではないのかなーと期待
 * JobをSetで管理してる(JobStoreで)から、数十万件Jobとか登録すればOOMで死ぬかもしれない

* JobScheduler#setPersisted(true)のJobをセットしまくって、XMLを肥大化させるアプローチもありかも
 * 肥大化すれば再起動時に読み込むXMLがでかすぎるて落ちるとか、そもそもJob登録中にXMLが肥大化して処理が終わらないとかなりそう


## 疑問

* setRequiresDeviceIdle(true)のjobのJobStatus-appNotIdleConstraintSatisfiedフィールドの値はどうなるのか？trueになる？
 * JobStatus#isReadyでは、アプリがIdelじゃなかったらって条件がある
 * じゃidleだったらsetRequiresDeviceIdle(true)したjobも動かないのか？？


## bug??

* TODO: 要検証

* ManifestにRECEIVE_BOOT_COMPLETEDを定義してても、この例外が出る。
* 例外投げてるところ。この辺。canPersistJobsが重要
 * http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/JobSchedulerService.java##824
 * http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/JobSchedulerService.java#canPersistJobs

```java
Shutting down VM
FATAL EXCEPTION: main
Process: com.os.operando.jobschedulersample, PID: 21464
java.lang.IllegalArgumentException: Error: requested job be persisted without holding RECEIVE_BOOT_COMPLETED permission.
	at android.os.Parcel.readException(Parcel.java:1603)
	at android.os.Parcel.readException(Parcel.java:1552)
	at android.app.job.IJobScheduler$Stub$Proxy.schedule(IJobScheduler.java:121)
	at android.app.JobSchedulerImpl.schedule(JobSchedulerImpl.java:42)
	at com.os.operando.jobschedulersample.MainActivity$1.onClick(MainActivity.java:42)
	at android.view.View.performClick(View.java:5198)
	at android.view.View$PerformClick.run(View.java:21147)
	at android.os.Handler.handleCallback(Handler.java:739)
	at android.os.Handler.dispatchMessage(Handler.java:95)
	at android.os.Looper.loop(Looper.java:148)
	at android.app.ActivityThread.main(ActivityThread.java:5417)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:726)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:616)
```

* mPersistCacheにアプリがRECEIVE_BOOT_COMPLETEDのPermissionを許可されているかどうかが保持されている
 * mPersistCacheは、JobSchedulerStubのフィールド(SparseArray<Boolean>)
* uidをKeyにしてる。valueがなければ、Context#checkPermissionしてPermission許可されているかどうかチェック
* その結果とmPersistCacheにputする
* んで、このmPersistCacheにキャッシュされると、アプリを更新してもRECEIVE_BOOT_COMPLETEDのPermissionを許可されているかどうかの値が変わらない？
 * 変わるタイミングがない。アプリのインストール/アップデート時にremoveするとかないから、uidが同じだったら、アプリのManifestを更新してもダメじゃね？
 * アンインストールすれば直ったけど、アプリの更新じゃダメそう


##### http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/JobSchedulerService.java#canPersistJobs

```java
/** Cache determination of whether a given app can persist jobs
 * key is uid of the calling app; value is undetermined/true/false
 */
private final SparseArray<Boolean> mPersistCache = new SparseArray<Boolean>();

private boolean canPersistJobs(int pid, int uid) {
    // If we get this far we're good to go; all we need to do now is check
    // whether the app is allowed to persist its scheduled work.
    final boolean canPersist;
    synchronized (mPersistCache) {
        Boolean cached = mPersistCache.get(uid);
        if (cached != null) {
            canPersist = cached.booleanValue();
        } else {
            // Persisting jobs is tantamount to running at boot, so we permit
            // it when the app has declared that it uses the RECEIVE_BOOT_COMPLETED
            // permission
            int result = getContext().checkPermission(
                    android.Manifest.permission.RECEIVE_BOOT_COMPLETED, pid, uid);
            canPersist = (result == PackageManager.PERMISSION_GRANTED);
            mPersistCache.put(uid, canPersist);
        }
    }
    return canPersist;
}
```


# 関連クラス

* [JobSchedulerImpl](http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/core/java/android/app/JobSchedulerImpl.java)

* [/frameworks/base/core/java/android/app/job/](http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/core/java/android/app/job/)
 * [IJobCallback.aidl](http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/core/java/android/app/job/IJobCallback.aidl)
 * [IJobScheduler.aidl](http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/core/java/android/app/job/IJobScheduler.aidl)
 * [IJobService.aidl](http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/core/java/android/app/job/IJobService.aidl)
 * [JobInfo.aidl](http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/core/java/android/app/job/JobInfo.aidl)
 * [JobInfo](http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/core/java/android/app/job/JobInfo.java)
 * [JobParameters.aidl](http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/core/java/android/app/job/JobParameters.aidl)
 * [JobParameters](http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/core/java/android/app/job/JobParameters.java)
 * [JobScheduler](http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/core/java/android/app/job/JobScheduler.java)
 * [JobService](http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/core/java/android/app/job/JobService.java)

* [/frameworks/base/services/core/java/com/android/server/job/](http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/)
 * [JobCompletedListener](http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/JobCompletedListener.java)
 * [JobSchedulerService](http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/JobSchedulerService.java)
 * [JobServiceContext](http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/JobServiceContext.java)
 * [JobStore](http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/JobStore.java)
 * [StateChangedListener](http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/StateChangedListener.java)

* [/frameworks/base/services/core/java/com/android/server/job/controllers](http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/controllers)
 * [StateController](http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/controllers/StateController.java)
 * [AppIdleController](http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/controllers/AppIdleController.java)
 * [BatteryController](http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/controllers/BatteryController.java)
 * [ConnectivityController](http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/controllers/ConnectivityController.java)
 * [IdleController](http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/controllers/IdleController.java)
 * [JobStatus](http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/controllers/JobStatus.java)
 * [TimeController](http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/controllers/TimeController.java)

* Auto BackのJobSchedular
 * [FullBackupJob.java](http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/services/backup/java/com/android/server/backup/FullBackupJob.java)


# 面白コメント

## // Let's go!

* http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/JobSchedulerService.java#348


## // GO GO GO!

* http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/JobSchedulerService.java#367

## [Doze and App Standby](http://developer.android.com/about/versions/marshmallow/android-6.0-changes.html#behavior-power)

* JobSchedularココらへんとも絡みそう
* TODO : 関連性を調べる
* http://developer.android.com/about/versions/marshmallow/android-6.0-changes.html#behavior-power


## 雑memo

* http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/core/java/android/app/SystemServiceRegistry.java#652
 * JobSchedulerをSystem Serviceに設定してる場所

* http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/services/java/com/android/server/SystemServer.java#864
 * JobSchedulerServiceの起動

* http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/core/java/android/app/JobSchedulerImpl.java
 * JobSchedulerの実装

* http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/JobSchedulerService.java#813
 * scheduleの実装

* http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/JobSchedulerService.java#canPersistJobs
 * 実装が面白い
 * http://developer.android.com/intl/ja/reference/android/content/Context.html#checkPermission(java.lang.String, int, int)

* android.permission.DUMP
 * http://developer.android.com/intl/ja/reference/android/Manifest.permission.html#DUMP

* http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/services/core/java/com/android/server/SystemServiceManager.java#startBootPhase
 * 各System Serviceに起動段階(端末の起動)の変化を通知してる
 * http://tools.oesf.biz/android-6.0.0_r1.0/search?q=onBootPhase&defs=&refs=&path=&hist=
 * http://tools.oesf.biz/android-6.0.0_r1.0/xref/frameworks/base/services/java/com/android/server/SystemServer.java#1146
