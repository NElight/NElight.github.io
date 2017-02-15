# iOS10新的推送框架UserNotifications
### 1. 申请权限
iOS8 - iOS9
```
[[UIApplication sharedApplication] registerUserNotificationSettings:[UIUserNotificationSettings settingsForTypes:UIUserNotificationTypeAlert | UIUserNotificationTypeBadge | UIUserNotificationTypeSound categories:nil]];
```
iOS10

```
[[UNUserNotificationCenter currentNotificationCenter] requestAuthorizationWithOptions:UNAuthorizationOptionAlert | UNAuthorizationOptionBadge | UNAuthorizationOptionSound completionHandler:^(BOOL granted, NSError * _Nullable error) {
    if (granted) {
        NSLog(@"授权成功");
    }else {
        NSLog(@"授权失败");
    }
}];
```
如果是本地通知的话已经到此已经可以发出通知了。如果是远程的推送则需要获取devicetoken来注册远程推送。获取devicetoken的方法与之前相同。调用[[UIApplication shareApplicate registerForRemoteNotifications]方法。在下面的回调中获取devicetoke。

```
- (void)application:(UIApplication *)application didRegisterForRemoteNotificationsWithDeviceToken:(NSData *)deviceToken {
    NSLog(@"device token ----------------- %@", [NSString stringWithFormat:@"%@", deviceToken]);
}

- (void)application:(UIApplication *)application didFailToRegisterForRemoteNotificationsWithError:(NSError *)error {
    NSLog(@"device token ------------------ %@", error);
}
```
另外用户可以关闭全部或者一部分通知的形式。这时我们应该使用``
[[UNUserNotificationCenter currentNotificationCenter] getNotificationSettingsWithCompletionHandler:^(UNNotificationSettings * _Nonnull settings){}]
``方法来检测用户的设置，不应该更改用户的行为。
### 发送通知
iOS10之前
```

    UILocalNotification *localNote = [[UILocalNotification alloc] init];
// 2.设置本地通知的内容
    // 2.1.设置通知发出的时间
    localNote.fireDate = [NSDate dateWithTimeIntervalSinceNow:3.0];
    // 2.2.设置通知的内容
    localNote.alertBody = @"在干吗?";
    // 2.3.设置滑块的文字（锁屏状态下：滑动来“解锁”）
    localNote.alertAction = @"解锁";
    // 2.4.决定alertAction是否生效
    localNote.hasAction = NO;
    // 2.5.设置点击通知的启动图片
    localNote.alertLaunchImage = @"123Abc";
    // 2.6.设置alertTitle
    localNote.alertTitle = @"你有一条新通知";
    // 2.7.设置有通知时的音效
    localNote.soundName = @"buyao.wav";
    // 2.8.设置应用程序图标右上角的数字
    localNote.applicationIconBadgeNumber = 999;

    // 2.9.设置额外信息
    localNote.userInfo = @{@"type" : @1};

    // 3.调用通知
    [[UIApplication sharedApplication] scheduleLocalNotification:localNote];
```
iOS10之后

```
UNMutableNotificationContent *content = [[UNMutableNotificationContent alloc]init];
    content.title = @"time interval notification";
    content.body = @"my first notification";
    content.categoryIdentifier = @"myNotificationCategory";
    UNTimeIntervalNotificationTrigger *trigger = [UNTimeIntervalNotificationTrigger triggerWithTimeInterval:5 repeats:NO];
    NSString *requestIdentifier = @"com.yioks.teacher";
    UNNotificationRequest *req = [UNNotificationRequest requestWithIdentifier:requestIdentifier content:content trigger:trigger];
    [[UNUserNotificationCenter currentNotificationCenter] addNotificationRequest:req withCompletionHandler:^(NSError * _Nullable error) {
        if (error) {
            NSLog(@"local notification error ------------ %@", error);
        }
    }];
```
iOS10之后我们可以取消还未展示的通知，更新还未展示的通知，移除已经展示的通知，更新已经展示过的通知。使用
``
[[UNUserNotificationCenter currentNotificationCenter] removeDeliveredNotificationsWithIdentifiers:@[requestIdentifier]]
``根据通知的标识来移除已经展示的通知。类似的使用``
removePendingNotificationRequests``来取消还未展示的通知。
如果需要更新通知内容。则只要重新提交相同标识的通知即可。
### 处理通知
UNUserNotificationCenterDelegate带里提供了两个方法，用来在应用内展示通知，和收到通知响应时如何处理。

```
//应用内接受通知
- (void)userNotificationCenter:(UNUserNotificationCenter *)center willPresentNotification:(UNNotification *)notification withCompletionHandler:(void (^)(UNNotificationPresentationOptions options))completionHandler;

- (void)userNotificationCenter:(UNUserNotificationCenter *)center didReceiveNotificationResponse:(UNNotificationResponse *)response withCompletionHandler:(void(^)())completionHandler;
```
收到通知后在第二个方法中可以用``
response.notification.request.content.userInfo
``来获取本地通知和远程推送中的用户自定义信息。
### 绑定action
使用如下代码进行绑定Action操作。
```
UNTextInputNotificationAction *inputAction = [UNTextInputNotificationAction actionWithIdentifier:@"action.input" title:@"Input" options:UNNotificationActionOptionForeground textInputButtonTitle:@"Send" textInputPlaceholder:@"what do you want to say..."];
    
    UNNotificationAction *goodbyeAction = [UNNotificationAction actionWithIdentifier:@"action.goodbye" title:@"Goodbye" options:UNNotificationActionOptionForeground];
    
    UNNotificationAction *cancelAction = [UNNotificationAction actionWithIdentifier:@"action.cancel" title:@"Cancel" options:UNNotificationActionOptionDestructive];
    
    UNNotificationCategory *category = [UNNotificationCategory categoryWithIdentifier:@"com.yioks.notiAction" actions:@[inputAction, goodbyeAction, cancelAction] intentIdentifiers:@[] options:UNNotificationCategoryOptionCustomDismissAction];
    
    [[UNUserNotificationCenter currentNotificationCenter] setNotificationCategories:[NSSet setWithObjects:category, nil]];
```
绑定action，展开通知后就可以看到对应的action了。
在``didReceiveResponse``方法中通过``response.notification.request.content.categoryIdentifier``来辨认category，通过``response.actionIdentifier``来判断对应的action，来完成不同的操作。
### Extension
ios10为通知添加了两个extension，分别为``Service Extension``和``Content Extension``
##### 1. Service Extension

```
- (void)didReceiveNotificationRequest:(UNNotificationRequest *)request withContentHandler:(void (^)(UNNotificationContent * _Nonnull))contentHandler;
- (void)serviceExtensionTimeWillExpire;
```
第一个方法用来修改接受到的通知，在方法的最后把修改过后的通知返回给系统来展现。

第二个方法是在时间过长时，会强行旧的通知展现出来。

##### 2. Content Extension
它是用来自定义通知视图的。

不管是哪种extension，最重要的是在info.plist中配置UNNotificationExtensionCategory的值。以便用来使用和区分extension。



