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


## Android Mから同時実行できるJobの数に変更はあったのか

* なんか変わったっぽい
* Android Mの時のまとめは以下
 * https://github.com/operando/JobScheduler-Code-Reading#ramサイズによって同時実行できるjobの数が変わる
* 最大で16こ Jobが同時に実行できるようになってるのかな？

http://tools.oesf.biz/android-7.1.1_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/JobSchedulerService.java#106

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
*
* http://tools.oesf.biz/android-7.1.1_r1.0/xref/frameworks/base/core/java/android/provider/Settings.java#8056
* Settings.Global.getStringの処理
 * http://tools.oesf.biz/android-7.1.1_r1.0/xref/frameworks/base/core/java/android/provider/Settings.java#8650


## Let's go!とGO GO GO!は健在

Android Mのコードにもあった面白コメント

### Let's go!

* http://tools.oesf.biz/android-7.1.1_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/JobSchedulerService.java#773


### GO GO GO!

* http://tools.oesf.biz/android-7.1.1_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/JobSchedulerService.java#795


## 雑メモ

* なんかまた内部向け独自パーサー見つけた
 * http://tools.oesf.biz/android-7.1.1_r1.0/xref/frameworks/base/core/java/android/util/KeyValueListParser.java
 * Settingの値とかをパースするのに使われてるっぽい。以下とか。
 * http://tools.oesf.biz/android-7.1.1_r1.0/xref/frameworks/base/core/java/android/provider/Settings.java#ALARM_MANAGER_CONSTANTS


* http://tools.oesf.biz/android-7.1.1_r1.0/xref/frameworks/base/services/core/java/com/android/server/job/JobSchedulerService.java#
