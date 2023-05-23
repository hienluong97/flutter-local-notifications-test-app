# flutter_local_notifications

### 利用パッケージ

flutter_local_notifications : 12.0.0

### flutter_local_notifications パッケージのインストール

#### pubspec.yaml 　

```
dependencies:
 flutter:
   sdk: flutter
 flutter_local_notifications: ^12.0.0 # 追加

```

インストール コマンド

```
flutter pub get

```

## OS 設定

### Android

#### android/app/src/main/AndroidManifest.xml

```

<manifest xmlns:android="http://schemas.android.com/apk/res/android"
   package="com.example.local_notifcations">

 <!-- ここから追加 -->
 <uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED"/>
 <uses-permission android:name="android.permission.VIBRATE" />
 <!-- ここまで追加 -->

  <application
       android:label="local_notifcations"
       android:icon="@mipmap/ic_launcher">

       <!-- ここから追加 -->
       <receiver android:name="com.dexterous.flutterlocalnotifications.ScheduledNotificationBootReceiver">
       <intent-filter>
         <action android:name="android.intent.action.BOOT_COMPLETED"/>
         <!-- 再起動時のローカル通知継続用設定 -->
         <action android:name="android.intent.action.QUICKBOOT_POWERON" />
         <action android:name="com.htc.intent.action.QUICKBOOT_POWERON"/>


         <action android:name="android.intent.action.MY_PACKAGE_REPLACED"/>
       </intent-filter>
       </receiver>
       <receiver android:name="com.dexterous.flutterlocalnotifications.ScheduledNotificationReceiver" />
       <!-- ここまで追加 -->

       <activity
           android:name=".MainActivity"

```

#### 通知アイコンを配置

置く場所　 android/app/src/main/res/drawable/app_icon.png

（例）android/app/src/main/res/drawable/mipmap-hdpi/ic_launcher.png(72×72)をコピー、リネームして作成。

### iOS

#### iOS/Runner/AppDelegate.swift

```

import UIKit
import Flutter

@UIApplicationMain
@objc class AppDelegate: FlutterAppDelegate {
 override func application(
   _ application: UIApplication,
   didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?
 ) -> Bool {

   /*　ここから追加　*/
   if #available(iOS 10.0, *) {
    UNUserNotificationCenter.current().delegate = self as? UNUserNotificationCenterDelegate
   }
   /*　ここまで追加　*/

   GeneratedPluginRegistrant.register(with: self)
   return super.application(application, didFinishLaunchingWithOptions: launchOptions)
 }
}

```

### ローカル通知の組み込み

#### lib/main.dart 編集

##### パーミッション取得ダイアログの表示　※iOS のみ

\_requestIOSPerssion()を呼ばなくても後述の\_initializePlatformSpecifics()を呼んだタイミングで自動で出るけど、通知サウンドのパーミッションを追加したければ自前で呼ぶ必要があるみたい。

パッケージ追加

```
 import 'package:flutter_local_notifications/flutter_local_notifications.dart';

```

```

class _MyHomePageState extends State<MyHomePage> {
 final FlutterLocalNotificationsPlugin flutterLocalNotificationsPlugin =
     FlutterLocalNotificationsPlugin(); // 追加

 @override
 void initState() {
   super.initState();

   // ここから追加
   if (Platform.isIOS) {
     _requestIOSPermission();
   }
   // ここまで追加
 }

 // ここから追加
 void _requestIOSPermission() {
   flutterLocalNotificationsPlugin
       .resolvePlatformSpecificImplementation<
           IOSFlutterLocalNotificationsPlugin>()
       .requestPermissions(
         alert: false,
         badge: true,
         sound: false,
       );
 }
 // ここまで追加

```

## 初期化 --- ローカル通知を単純に表示

#### lib/main.dart 編集

```

class _MyHomePageState extends State<MyHomePage> {
 final FlutterLocalNotificationsPlugin flutterLocalNotificationsPlugin =
     FlutterLocalNotificationsPlugin();

 @override
 void initState() {
   super.initState();

   _requestIOSPermission();

   //　ここから追加
   _initializePlatformSpecifics();
   _showNotification(); // ローカル通知を表示
　　　　　// ここまで追加
 }

 〜〜〜

 // ここから追加
 void _initializePlatformSpecifics() {
   var initializationSettingsAndroid =
       AndroidInitializationSettings('app_icon');

   var initializationSettingsIOS = DarwinInitializationSettings(
     requestAlertPermission: true,
     requestBadgePermission: true,
     requestSoundPermission: false,
     onDidReceiveLocalNotification: (id, title, body, payload) async {
       // your call back to the UI
     },
   );

   var initializationSettings = InitializationSettings(
       android: initializationSettingsAndroid, iOS: initializationSettingsIOS);

   flutterLocalNotificationsPlugin.initialize(initializationSettings,
       onDidReceiveNotificationResponse: (NotificationResponse res) {
     debugPrint('payload:${res.payload}');
   });
 }

 Future<void> _showNotification() async {
   var androidChannelSpecifics = AndroidNotificationDetails(
     'CHANNEL_ID',
     'CHANNEL_NAME',
     channelDescription: "CHANNEL_DESCRIPTION",
     importance: Importance.max,
     priority: Priority.high,
     playSound: false,
     timeoutAfter: 5000,
     styleInformation: DefaultStyleInformation(true, true),
   );

   var iosChannelSpecifics = DarwinNotificationDetails();

   var platformChannelSpecifics = NotificationDetails(
       android: androidChannelSpecifics, iOS: iosChannelSpecifics);

   await flutterLocalNotificationsPlugin.show(
     0, // Notification ID
     'Test Title', // Notification Title
     'Test Body', // Notification Body, set as null to remove the body
     platformChannelSpecifics,
     payload: 'New Payload', // Notification Payload
   );
 }
 // ここまで追加

```

### ローカル通知を指定日時に表示

#### lib/main.dart 編集

あとで通知チャンネル ID でキャンセルすることができる。
「zonedSchedule(0, …」の部分。

```
import 'package:timezone/data/latest_all.dart' as tz;
import 'package:timezone/timezone.dart' as tz;

```

コマンド

```
flutter pub get

```

#### lib/main.dart 編集

```
@override
 void initState() {
   super.initState();

   _requestIOSPermission();
   _initializePlatformSpecifics();
   //_showNotification(); // コメントアウト
   _scheduleNotification(); // 追加

   // ここから追加
   // タイムゾーンデータベースの初期化
   tz.initializeTimeZones();
   // ローカルロケーションのタイムゾーンを東京に設定
   tz.setLocalLocation(tz.getLocation("Asia/Tokyo"));
　　　　　　// ここまで追加
 }

 // ここから追加
 Future<void> _scheduleNotification() async {
   // 5秒後
   var scheduleNotificationDateTime = DateTime.now().add(Duration(seconds: 5));

   var androidChannelSpecifics = AndroidNotificationDetails(
     'CHANNEL_ID 1',
     'CHANNEL_NAME 1',
     channelDescription: "CHANNEL_DESCRIPTION 1",
     icon: 'app_icon',
     //sound: RawResourceAndroidNotificationSound('my_sound'),
     largeIcon: DrawableResourceAndroidBitmap('app_icon'),
     enableLights: true,
     color: const Color.fromARGB(255, 255, 0, 0),
     ledColor: const Color.fromARGB(255, 255, 0, 0),
     ledOnMs: 1000,
     ledOffMs: 500,
     importance: Importance.max,
     priority: Priority.high,
     playSound: false,
     timeoutAfter: 5000,
     styleInformation: DefaultStyleInformation(true, true),
   );

   var iosChannelSpecifics = DarwinNotificationDetails(
     //sound: 'my_sound.aiff',
   );

   var platformChannelSpecifics = NotificationDetails(
     android: androidChannelSpecifics,
     iOS: iosChannelSpecifics,
   );

   await flutterLocalNotificationsPlugin.zonedSchedule(
     0,
     'Test Title',
     'Test Body',
     tz.TZDateTime.from(scheduleNotificationDateTime, tz.local),　// 5秒後に表示
     platformChannelSpecifics,
     payload: 'Test Payload',
     androidAllowWhileIdle: true,
     uiLocalNotificationDateInterpretation: UILocalNotificationDateInterpretation.absoluteTime,
   );
 }
 // ここまで追加


```

### 予約済みのローカル通知の数を取得する

#### lib/main.dart 編集

```
 @override
 void initState() {
   super.initState();

   _requestIOSPermission();
   _initializePlatformSpecifics();
   //_showNotification(); //
   _scheduleNotification();

  // 追加
   _getPendingNotificationCount().then((value) =>
       debugPrint('getPendingNotificationCount:' + value.toString()));
 }

 // ここから追加
 Future<int> _getPendingNotificationCount() async {
   List<PendingNotificationRequest> p =
       await flutterLocalNotificationsPlugin.pendingNotificationRequests();
   return p.length;
 }
 // ここまで追加
```

### 予約済みのローカル通知をキャンセルする

#### lib/main.dart 編集

通知チャンネル ID 指定キャンセル、全キャンセルの両方が可能。

```
@override
 void initState() {
   super.initState();

   _requestIOSPermission();
   _initializePlatformSpecifics();
   //_showNotification(); //
   _scheduleNotification();

   _getPendingNotificationCount().then((value) =>
       debugPrint('getPendingNotificationCount:' + value.toString()));

   // 追加
   _cancelNotification().then((value) => debugPrint('cancelNotification'));
}

// ここから追加
Future<void> cancelNotification() async {
   await flutterLocalNotificationsPlugin.cancel(0);
}
Future<void> cancelAllNotification() async {
   await flutterLocalNotificationsPlugin.cancelAll();
}
// ここまで追加

```

### ローカル通知ダイアログをタップしたとき

#### lib/main.dart 編集

初期化時に指定した onDidReceiveNotificationResponse が発火。
その際、通知をセットしたときの payload 値が渡る。

```
await flutterLocalNotificationsPlugin.zonedSchedule(
     ..,
     payload: 'Test Payload',
     ..,
   );
 }

flutterLocalNotificationsPlugin.initialize(initializationSettings,
   onDidReceiveNotificationResponse: (NotificationResponse res) {
   debugPrint('payload:${res.payload}');
});

```

## 最後に全文

#### lib/main.dart

```
import 'dart:io';
import 'package:flutter/material.dart';
import 'package:flutter_local_notifications/flutter_local_notifications.dart';
import 'package:timezone/data/latest_all.dart' as tz;
import 'package:timezone/timezone.dart' as tz;

void main() {
  runApp(const MyApp());
}

class MyApp extends StatelessWidget {
  const MyApp({Key? key}) : super(key: key);

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Flutter Demo',
      theme: ThemeData(
        primarySwatch: Colors.blue,
      ),
      home: const MyHomePage(),
    );
  }
}

class MyHomePage extends StatefulWidget {
  const MyHomePage({Key? key}) : super(key: key);

  @override
  MyHomePageState createState() => MyHomePageState();
}

class MyHomePageState extends State<MyHomePage> {
  final FlutterLocalNotificationsPlugin flutterLocalNotificationsPlugin =
      FlutterLocalNotificationsPlugin();
  final scheduleId = 0;
  final scheduleAddSec = 3;

  @override
  void initState() {
    super.initState();

    // 通知パーミッション許可ダイアログの表示
    if (Platform.isIOS) {
      _requestIOSPermission();
    }

    // タイムゾーンデータベースの初期化
    tz.initializeTimeZones();
    // ローカルロケーションのタイムゾーンを東京に設定
    tz.setLocalLocation(tz.getLocation("Asia/Tokyo"));

    _initializePlatformSpecifics();

    // 設定済みの通知数を取得
    _getPendingNotificationCount().then((value) {
      debugPrint('getPendingNotificationCount:$value');
    });

    // 設定済みの通知をすべてキャンセル
    _cancelAllNotification().then((value) => debugPrint('cancelNotification'));
  }

  void _requestIOSPermission() {
    flutterLocalNotificationsPlugin
        .resolvePlatformSpecificImplementation<
            IOSFlutterLocalNotificationsPlugin>()!
        .requestPermissions(
          alert: false,
          badge: true,
          sound: false,
        );
  }

  void _initializePlatformSpecifics() {
    const initializationSettingsAndroid =
        AndroidInitializationSettings('app_icon');

    var initializationSettingsIOS = DarwinInitializationSettings(
      requestAlertPermission: true,
      requestBadgePermission: true,
      requestSoundPermission: false,
      onDidReceiveLocalNotification: (id, title, body, payload) async {},
    );

    var initializationSettings = InitializationSettings(
        android: initializationSettingsAndroid, iOS: initializationSettingsIOS);

    flutterLocalNotificationsPlugin.initialize(initializationSettings,
        onDidReceiveNotificationResponse: (NotificationResponse res) {
      debugPrint('payload:${res.payload}');
    });
  }

  Future<void> _showNotification() async {
    var androidChannelSpecifics = const AndroidNotificationDetails(
      'CHANNEL_ID',
      'CHANNEL_NAME',
      channelDescription: "CHANNEL_DESCRIPTION",
      importance: Importance.max,
      priority: Priority.high,
      playSound: false,
      timeoutAfter: 5000,
      styleInformation: DefaultStyleInformation(
        true,
        true,
      ),
    );

    var iosChannelSpecifics = const DarwinNotificationDetails();

    var platformChannelSpecifics = NotificationDetails(
      android: androidChannelSpecifics,
      iOS: iosChannelSpecifics,
    );

    await flutterLocalNotificationsPlugin.show(
      0, // Notification ID
      'Test Title', // Notification Title
      'Test Body', // Notification Body, set as null to remove the body
      platformChannelSpecifics,
      payload: 'New Payload', // Notification Payload
    );
  }

  Future<void> _scheduleNotification() async {
    var scheduleNotificationDateTime =
        DateTime.now().add(Duration(seconds: scheduleAddSec));

    var androidChannelSpecifics = const AndroidNotificationDetails(
      'CHANNEL_ID 1',
      'CHANNEL_NAME 1',
      channelDescription: "CHANNEL_DESCRIPTION 1",
      icon: 'app_icon',
      //sound: RawResourceAndroidNotificationSound('my_sound'),
      //largeIcon: DrawableResourceAndroidBitmap('app_icon'),
      enableLights: true,
      color: Color.fromARGB(255, 255, 0, 0),
      ledColor: Color.fromARGB(255, 255, 0, 0),
      ledOnMs: 1000,
      ledOffMs: 500,
      importance: Importance.max,
      priority: Priority.high,
      playSound: false,
      timeoutAfter: 10000,
      styleInformation: DefaultStyleInformation(true, true),
    );

    var iosChannelSpecifics = const DarwinNotificationDetails(
        // sound: 'my_sound.aiff',
        );

    var platformChannelSpecifics = NotificationDetails(
      android: androidChannelSpecifics,
      iOS: iosChannelSpecifics,
    );

    await flutterLocalNotificationsPlugin.zonedSchedule(
      scheduleId,
      'Test Title',
      'Test Body',
      tz.TZDateTime.from(scheduleNotificationDateTime, tz.local),
      platformChannelSpecifics,
      payload: 'Test Payload',
      androidAllowWhileIdle: true,
      uiLocalNotificationDateInterpretation:
          UILocalNotificationDateInterpretation.absoluteTime,
    );
  }

  Future<int> _getPendingNotificationCount() async {
    List<PendingNotificationRequest> p =
        await flutterLocalNotificationsPlugin.pendingNotificationRequests();
    return p.length;
  }

  Future<void> _cancelNotification(int id) async {
    await flutterLocalNotificationsPlugin.cancel(id);
  }

  Future<void> _cancelAllNotification() async {
    await flutterLocalNotificationsPlugin.cancelAll();
  }
  // ここまで追加

  @override
  void dispose() {
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('ローカルプッシュ通知テスト'),
      ),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            ElevatedButton(
              onPressed: () {
                _showNotification(); // 通知をすぐに表示
              },
              child: const Text('すぐに通知を表示'),
            ),
            ElevatedButton(
              onPressed: () {
                _scheduleNotification(); // 指定日時に通知を表示
              },
              child: Text('指定日時に通知を表示（$scheduleAddSec秒後）'),
            ),
            ElevatedButton(
              onPressed: () {
                _cancelNotification(scheduleId);
              },
              child: const Text('設定済みの通知を削除'),
            ),
          ],
        ),
      ),
    );
  }
}

```
