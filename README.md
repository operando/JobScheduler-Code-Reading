# JobScheduler Code Reading

**Android M 7.1.1_r1.0のソースコードをベースに調べたもの**

http://tools.oesf.biz/android-7.1.1_r1.0/

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


## Android Mの実装と比較してみて

* 色々変わってそう
* Mの時より複雑な実装になってる印象...


## Android Mの時と比べて同時実行できるJobの数に変更はあったのか

* なんか変わったっぽい
* Android Mの時のまとめは以下
 * https://github.com/operando/JobScheduler-Code-Reading#ramサイズによって同時実行できるjobの数が変わる
* 最大で16こ Jobが同時に実行できるようになってるのかな？

http://tools.oesf.biz/android-7.1.1_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/JobSchedulerService.java#109

```java
/** The maximum number of concurrent jobs we run at one time. */
private static final int MAX_JOB_CONTEXTS_COUNT = 16;
```


## なんかSettingの値を取り出してJobSchedulerの設定値みたいなの作ってるところ

* これややこしいなー
* なんかSettingを値を読み込んで処理するところがあるんだけどー
 * 読み込んでる値が`ALARM_MANAGER_CONSTANTS = "alarm_manager_constants"` ってやつなんだよね
 * http://tools.oesf.biz/android-7.1.1_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/JobSchedulerService.java#302
 * `ALARM_MANAGER_CONSTANTS`
 * http://tools.oesf.biz/android-7.1.1_r1.0/xref/frameworks/base/core/java/android/provider/Settings.java#ALARM_MANAGER_CONSTANTS
* JobSchedulerService#startメソッドでContentResolverを作ってる
 * http://tools.oesf.biz/android-7.1.1_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/JobSchedulerService.java#287
* http://tools.oesf.biz/android-7.1.1_r1.0/xref/frameworks/base/core/java/android/provider/Settings.java#8056
* Settings.Global.getStringの処理
 * http://tools.oesf.biz/android-7.1.1_r1.0/xref/frameworks/base/core/java/android/provider/Settings.java#8650

## JobServiceContextに残された謎なコード

* Android Mのことのコードと同じだ
* でもこの定数どこからもアクセスされてない
* http://tools.oesf.biz/android-7.1.1_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/JobServiceContext.java#68

```java
68     /** Define the maximum # of jobs allowed to run on a service at once. */
69     private static final int defaultMaxActiveJobsPerService =
70             ActivityManager.isLowRamDeviceStatic() ? 1 : 3;
```

## BatteryController

http://tools.oesf.biz/android-7.1.1_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/controllers/BatteryController.java

### ChargingTracker

* http://tools.oesf.biz/android-7.1.1_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/controllers/BatteryController.java122
* BroadcastReceiverを継承してる
* chargingのステータスをトラッキングしてるやーつー

* Battery系に関するIntentFilterを設定してる
* LocalServices.getService(BatteryManagerInternal.class)取得できるのはこれ？
 * http://tools.oesf.biz/android-7.1.1_r1.0/xref/frameworks/base/services/core/java/com/android/server/BatteryService.java#883

http://tools.oesf.biz/android-7.1.1_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/controllers/BatteryController.java#134

```java
134         public void startTracking() {
135             IntentFilter filter = new IntentFilter();
136
137             // Battery health.
138             filter.addAction(Intent.ACTION_BATTERY_LOW);
139             filter.addAction(Intent.ACTION_BATTERY_OKAY);
140             // Charging/not charging.
141             filter.addAction(BatteryManager.ACTION_CHARGING);
142             filter.addAction(BatteryManager.ACTION_DISCHARGING);
143             mContext.registerReceiver(this, filter);
144
145             // Initialise tracker state.
146             BatteryManagerInternal batteryManagerInternal =
147                     LocalServices.getService(BatteryManagerInternal.class);
148             mBatteryHealthy = !batteryManagerInternal.getBatteryLevelLow();
149             mCharging = batteryManagerInternal.isPowered(BatteryManager.BATTERY_PLUGGED_ANY);
150         }
```

http://tools.oesf.biz/android-7.1.1_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/controllers/BatteryController.java#156


```java
// onReceiveの処理
156         @Override
157         public void onReceive(Context context, Intent intent) {
158             onReceiveInternal(intent);
159         }
160
161         @VisibleForTesting
162         public void onReceiveInternal(Intent intent) {
163             final String action = intent.getAction();
164             if (Intent.ACTION_BATTERY_LOW.equals(action)) {
165                 if (DEBUG) {
166                     Slog.d(TAG, "Battery life too low to do work. @ "
167                             + SystemClock.elapsedRealtime());
168                 }
169                 // If we get this action, the battery is discharging => it isn't plugged in so
170                 // there's no work to cancel. We track this variable for the case where it is
171                 // charging, but hasn't been for long enough to be healthy.
172                 mBatteryHealthy = false;
173             } else if (Intent.ACTION_BATTERY_OKAY.equals(action)) {
174                 if (DEBUG) {
175                     Slog.d(TAG, "Battery life healthy enough to do work. @ "
176                             + SystemClock.elapsedRealtime());
177                 }
178                 mBatteryHealthy = true;
179                 maybeReportNewChargingState();
180             } else if (BatteryManager.ACTION_CHARGING.equals(action)) {
181                 if (DEBUG) {
182                     Slog.d(TAG, "Received charging intent, fired @ "
183                             + SystemClock.elapsedRealtime());
184                 }
185                 mCharging = true;
186                 maybeReportNewChargingState();
187             } else if (BatteryManager.ACTION_DISCHARGING.equals(action)) {
188                 if (DEBUG) {
189                     Slog.d(TAG, "Disconnected from power.");
190                 }
191                 mCharging = false;
192                 maybeReportNewChargingState();
193             }
194         }
195     }
```


* jobの実装とかしてる

http://tools.oesf.biz/android-7.1.1_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/controllers/BatteryController.java#97

```java
97     private void maybeReportNewChargingState() {
98         final boolean stablePower = mChargeTracker.isOnStablePower();
99         if (DEBUG) {
100             Slog.d(TAG, "maybeReportNewChargingState: " + stablePower);
101         }
102         boolean reportChange = false;
103         synchronized (mLock) {
104             for (JobStatus ts : mTrackedTasks) {
105                 boolean previous = ts.setChargingConstraintSatisfied(stablePower);
106                 if (previous != stablePower) {
107                     reportChange = true;
108                 }
109             }
110         }
111         // Let the scheduler know that state has changed. This may or may not result in an
112         // execution.
113         if (reportChange) {
114             mStateChangedListener.onControllerStateChanged();
115         }
116         // Also tell the scheduler that any ready jobs should be flushed.
117         if (stablePower) {
118             mStateChangedListener.onRunJobNow(null);
119         }
120     }
```

## mMaxActiveJobsの謎をとく

* 動的に変わる
* adb shell dumpsys jobschedulerすると今どのくらいなのかわかる

```java
161     /**
162      * Current limit on the number of concurrent JobServiceContext entries we want to
163      * keep actively running a job.
164      */
165     int mMaxActiveJobs = 1;
```


## adb shell dumpsys jobscheduler用のコマンドがAndroid Nから増えてる

* http://tools.oesf.biz/android-7.1.1_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/JobSchedulerShellCommand.java
* adb shell dumpsys jobscheduler -h

## Debugging JobScheduler

* adb shell dumpsys jobscheduler
 * JobSchedulerに登録されているJobをDumpする
 * めっちゃ使う
 * 実際に処理してるクラスは以下
 * http://tools.oesf.biz/android-7.1.1_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/JobSchedulerShellCommand.java
* adb logcat -s JobSchedulerService
 * JobSchedulerServiceのログ。あんまり出ないけど...
* idle状態にするコマンド
 * adb shell dumpsys battery unplug
 * adb shell dumpsys deviceidle enable
 * adb shell dumpsys deviceidle step
 * adb shell dumpsys deviceidle force-idle


# StateControllerの種類

* JobManager
* 各条件の監視をして、Jobの状態をコントロールする
 * [StateController](http://tools.oesf.biz/android-7.1.1_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/controllers/StateController.java) (抽象クラス)
 * [AppIdleController](http://tools.oesf.biz/android-7.1.1_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/controllers/AppIdleController.java)
 * [BatteryController](http://tools.oesf.biz/android-7.1.1_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/controllers/BatteryController.java)
 * [ConnectivityController](http://tools.oesf.biz/android-7.1.1_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/controllers/ConnectivityController.java)
 * [IdleController](http://tools.oesf.biz/android-7.1.1_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/controllers/IdleController.java)
 * [JobStatus](http://tools.oesf.biz/android-7.1.1_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/controllers/JobStatus.java)
 * [TimeController](http://tools.oesf.biz/android-7.1.1_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/controllers/TimeController.java)


## Let's go!とGO GO GO!は健在

Android Mのコードにもあった面白コメント

### Let's go!

* http://tools.oesf.biz/android-7.1.1_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/JobSchedulerService.java#834


### GO GO GO!

* http://tools.oesf.biz/android-7.1.1_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/JobSchedulerService.java#856


## adb shell dumpsys jobschedulerのメモ

* Job historyが見れるようになってる
 * 過去に実行したものかな？
* dump処理はここでやってる
 * http://tools.oesf.biz/android-7.1.1_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/JobSchedulerService.java#1807
 * http://tools.oesf.biz/android-7.1.1_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/JobPackageTracker.java
  * 色んなところで

```java
  Job history:
     -1h09m18s553ms START: u0a38 com.android.calendar/com.google/hogehogehogehogehoge@gmail.com:android
     -1h09m18s537ms  STOP: u0a38 com.android.calendar/com.google/hogehogehogehogehoge@gmail.com:android
     -1h09m18s530ms  STOP: u0a38 com.android.calendar/com.google/hogehogehogehogehoge@gmail.com:android
     -1h09m17s480ms  STOP: u0a38 com.android.calendar/com.google/hogehogehogehogehoge@gmail.com:android
     -1h09m17s464ms START: u0a38 com.android.calendar/com.google/hogehogehogehogehoge@gmail.com:android
     -1h09m17s458ms START: u0a38 com.android.calendar/com.google/hogehogehogehogehoge@gmail.com:android
     -1h09m17s437ms  STOP: u0a38 com.android.calendar/com.google/hogehogehogehogehoge@gmail.com:android
     -1h09m16s660ms  STOP: u0a38 com.android.calendar/com.google/hogehogehogehogehoge@gmail.com:android
     -1h09m16s640ms START: u0a38 com.android.calendar/com.google/hogehogehogehogehoge@gmail.com:android
     -1h09m15s896ms  STOP: u0a38 com.android.calendar/com.google/hogehogehogehogehoge@gmail.com:android
     -1h08m36s297ms START: u0a63 com.google.android.apps.photos/com.google.android.libraries.social.mediamonitor.MediaMonitorJobSchedulerService
     -1h08m36s293ms START: u0a63 com.google.android.apps.photos/.camerashortcut.CameraShortcutJobSchedulerService
     -1h08m36s068ms  STOP: u0a63 com.google.android.apps.photos/.camerashortcut.CameraShortcutJobSchedulerService
     -1h08m36s065ms  STOP: u0a63 com.google.android.apps.photos/com.google.android.libraries.social.mediamonitor.MediaMonitorJobSchedulerService
     -1h08m30s410ms START: u0a63 com.google.android.apps.photos/.dbprocessor.impl.DatabaseProcessorJobService
     -1h08m30s245ms  STOP: u0a63 com.google.android.apps.photos/.dbprocessor.impl.DatabaseProcessorJobService
     -1h08m19s747ms START: u0a24 DownloadManager:com.android.providers.downloads
     -1h08m18s751ms  STOP: u0a24 DownloadManager:com.android.providers.downloads
     -1h07m14s726ms START: u0a13 com.google.android.gms.fitness/com.google/hogehogehogehogehoge@gmail.com:android
     -1h07m14s579ms  STOP: u0a13 com.google.android.gms.fitness/com.google/hogehogehogehogehoge@gmail.com:android
     -1h07m14s427ms START: u0a24 DownloadManager:com.android.providers.downloads
     -1h07m14s076ms START: u0a84 com.google.android.apps.plus.content.EsProvider/com.google/hogehogehogehogehoge@gmail.com:android
     -1h07m12s364ms  STOP: u0a24 DownloadManager:com.android.providers.downloads
     -1h07m10s852ms  STOP: u0a84 com.google.android.apps.plus.content.EsProvider/com.google/hogehogehogehogehoge@gmail.com:android
     -1h04m06s448ms START: u0a63 com.google.android.apps.photos/com.google.android.libraries.social.mediamonitor.MediaMonitorJobSchedulerService
     -1h04m06s446ms START: u0a63 com.google.android.apps.photos/.camerashortcut.CameraShortcutJobSchedulerService
     -1h04m06s208ms  STOP: u0a63 com.google.android.apps.photos/.camerashortcut.CameraShortcutJobSchedulerService
     -1h04m06s206ms  STOP: u0a63 com.google.android.apps.photos/com.google.android.libraries.social.mediamonitor.MediaMonitorJobSchedulerService
     -1h04m00s221ms START: u0a63 com.google.android.apps.photos/.dbprocessor.impl.DatabaseProcessorJobService
     -1h03m59s711ms  STOP: u0a63 com.google.android.apps.photos/.dbprocessor.impl.DatabaseProcessorJobService
     -1h03m59s051ms START: u0a24 DownloadManager:com.android.providers.downloads
     -1h03m06s618ms  STOP: u0a24 DownloadManager:com.android.providers.downloads
     -1h01m51s953ms START: u0a24 DownloadManager:com.android.providers.downloads
     -1h01m50s347ms  STOP: u0a24 DownloadManager:com.android.providers.downloads
     -1h00m44s706ms START: u0a69 gmail-ls/com.google/hogehogehogehogehoge@gmail.com:android
     -1h00m36s208ms  STOP: u0a69 gmail-ls/com.google/hogehogehogehogehoge@gmail.com:android
     -1h00m33s197ms START: u0a5 DownloadManager:com.android.providers.downloads
     -1h00m33s031ms  STOP: u0a5 DownloadManager:com.android.providers.downloads
     -1h00m10s056ms START: u0a24 DownloadManager:com.android.providers.downloads
     -1h00m08s859ms  STOP: u0a24 DownloadManager:com.android.providers.downloads
       -59m12s824ms START: u0a24 DownloadManager:com.android.providers.downloads
       -59m11s876ms  STOP: u0a24 DownloadManager:com.android.providers.downloads
       -58m27s773ms START: u0a24 DownloadManager:com.android.providers.downloads
       -58m26s496ms  STOP: u0a24 DownloadManager:com.android.providers.downloads
       -57m08s811ms START: u0a24 DownloadManager:com.android.providers.downloads
       -57m08s044ms  STOP: u0a24 DownloadManager:com.android.providers.downloads
       -56m18s764ms START: u0a24 DownloadManager:com.android.providers.downloads
       -56m14s989ms  STOP: u0a24 DownloadManager:com.android.providers.downloads
       -54m47s067ms START: u0a63 com.google.android.apps.photos/.camerashortcut.CameraShortcutJobSchedulerService
       -54m47s064ms START: u0a63 com.google.android.apps.photos/com.google.android.libraries.social.mediamonitor.MediaMonitorJobSchedulerService
       -54m46s918ms  STOP: u0a63 com.google.android.apps.photos/.camerashortcut.CameraShortcutJobSchedulerService
       -54m46s780ms  STOP: u0a63 com.google.android.apps.photos/com.google.android.libraries.social.mediamonitor.MediaMonitorJobSchedulerService
       -54m40s155ms START: u0a63 com.google.android.apps.photos/.dbprocessor.impl.DatabaseProcessorJobService
       -54m39s279ms  STOP: u0a63 com.google.android.apps.photos/.dbprocessor.impl.DatabaseProcessorJobService
       -54m30s790ms START: u0a24 DownloadManager:com.android.providers.downloads
       -54m28s403ms  STOP: u0a24 DownloadManager:com.android.providers.downloads
       -53m59s214ms START: u0a63 com.google.android.apps.photos/.dbprocessor.impl.DatabaseProcessorJobService
       -53m58s569ms  STOP: u0a63 com.google.android.apps.photos/.dbprocessor.impl.DatabaseProcessorJobService
       -53m02s793ms START: u0a24 DownloadManager:com.android.providers.downloads
       -53m01s628ms  STOP: u0a24 DownloadManager:com.android.providers.downloads
       -52m30s913ms START: u0a63 com.google.android.apps.photos/.dbprocessor.impl.DatabaseProcessorJobService
       -52m26s506ms  STOP: u0a63 com.google.android.apps.photos/.dbprocessor.impl.DatabaseProcessorJobService
       -51m38s043ms START: u0a24 DownloadManager:com.android.providers.downloads
       -51m35s364ms  STOP: u0a24 DownloadManager:com.android.providers.downloads
       -50m12s114ms START: u0a63 com.google.android.apps.photos/.dbprocessor.impl.DatabaseProcessorJobService
       -50m11s764ms  STOP: u0a63 com.google.android.apps.photos/.dbprocessor.impl.DatabaseProcessorJobService
       -47m54s434ms START: u0a63 com.google.android.apps.photos/com.google.android.libraries.social.mediamonitor.MediaMonitorJobSchedulerService
       -47m54s422ms START: u0a63 com.google.android.apps.photos/.camerashortcut.CameraShortcutJobSchedulerService
       -47m54s339ms  STOP: u0a63 com.google.android.apps.photos/.camerashortcut.CameraShortcutJobSchedulerService
       -47m54s337ms  STOP: u0a63 com.google.android.apps.photos/com.google.android.libraries.social.mediamonitor.MediaMonitorJobSchedulerService
       -47m30s004ms START: u0a24 DownloadManager:com.android.providers.downloads
       -47m29s191ms  STOP: u0a24 DownloadManager:com.android.providers.downloads
       -45m52s917ms START: u0a24 DownloadManager:com.android.providers.downloads
       -45m49s560ms  STOP: u0a24 DownloadManager:com.android.providers.downloads
       -45m19s885ms START: u0a63 com.google.android.apps.photos/.dbprocessor.impl.DatabaseProcessorJobService
       -45m19s430ms  STOP: u0a63 com.google.android.apps.photos/.dbprocessor.impl.DatabaseProcessorJobService
       -45m04s907ms START: u0a24 DownloadManager:com.android.providers.downloads
       -45m03s907ms  STOP: u0a24 DownloadManager:com.android.providers.downloads
       -44m05s837ms START: u0a24 DownloadManager:com.android.providers.downloads
       -44m05s072ms  STOP: u0a24 DownloadManager:com.android.providers.downloads
       -43m49s010ms START: u0a13 sync:android
       -43m48s990ms  STOP: u0a13 sync:android
       -35m45s066ms START: u0a63 com.google.android.apps.photos/.dbprocessor.impl.DatabaseProcessorJobService
       -35m44s858ms  STOP: u0a63 com.google.android.apps.photos/.dbprocessor.impl.DatabaseProcessorJobService
       -33m57s905ms START: u0a71 tv.abema/com.evernote.android.job.v21.PlatformJobService
       -33m54s103ms  STOP: u0a71 tv.abema/com.evernote.android.job.v21.PlatformJobService
       -19m06s748ms START: u0a78 com.facebook.orca/com.facebook.analytics2.logger.LollipopUploadService
       -19m06s722ms START: u0a63 com.google.android.apps.photos/.dbprocessor.impl.DatabaseProcessorJobService
       -19m05s954ms  STOP: u0a78 com.facebook.orca/com.facebook.analytics2.logger.LollipopUploadService
       -19m05s932ms  STOP: u0a63 com.google.android.apps.photos/.dbprocessor.impl.DatabaseProcessorJobService
        -7m14s698ms START: u0a13 com.google.android.gms.fitness/com.google/hogehogehogehogehoge@gmail.com:android
        -7m14s516ms  STOP: u0a13 com.google.android.gms.fitness/com.google/hogehogehogehogehoge@gmail.com:android
        -7m13s958ms START: u0a84 com.google.android.apps.plus.content.EsProvider/com.google/hogehogehogehogehoge@gmail.com:android
        -7m12s930ms  STOP: u0a84 com.google.android.apps.plus.content.EsProvider/com.google/hogehogehogehogehoge@gmail.com:android
        -2m45s701ms START: u0a71 tv.abema/com.evernote.android.job.v21.PlatformJobService
        -2m45s680ms START: u0a145 com.google.android.apps.docs.editors.kix/com.google/hogehogehogehogehoge@gmail.com:android
        -2m45s438ms  STOP: u0a145 com.google.android.apps.docs.editors.kix/com.google/hogehogehogehogehoge@gmail.com:android
        -2m45s397ms START: u0a145 com.google.android.apps.docs.editors.kix/com.google/hogehogehogehogehoge@gmail.com:android
        -2m45s342ms  STOP: u0a145 com.google.android.apps.docs.editors.kix/com.google/hogehogehogehogehoge@gmail.com:android
        -2m43s883ms  STOP: u0a71 tv.abema/com.evernote.android.job.v21.PlatformJobService
```

## 雑メモ

* なんかまた内部向け独自パーサー見つけた
 * http://tools.oesf.biz/android-7.1.1_r1.0/xref/frameworks/base/core/java/android/util/KeyValueListParser.java
 * Settingの値とかをパースするのに使われてるっぽい。以下とか。
 * http://tools.oesf.biz/android-7.1.1_r1.0/xref/frameworks/base/core/java/android/provider/Settings.java#ALARM_MANAGER_CONSTANTS


* http://tools.oesf.biz/android-7.1.1_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/JobSchedulerService.java#
