---
layout: post
title: "Multitasking in iOS 7"
category: "5"
date: "2013-10-07 07:00:00"
tags: article
author: "<a href=\"https://twitter.com/dcaunt\">David Caunt</a>"

[参考链接](http://blog.jobbole.com/51660/)

---

在iOS7之前，当程序置于后台之后开发者们对他们程序所能做的事情非常有限。除了VOIP和基于地理位置特性以外，唯一能做的地方就是使用后台任务（background tasks）让代码可以执行几分钟。如果你想下载比较大的视频文件以便离线浏览，亦或者备份用户的照片到你的服务器上，你都仅能完成一部分工作。

iOS 7添加了两个新的API以便你的程序可以在后台更新界面以及内容。首先是后台获取（Background Fetch），它允许你定期地从网络获取新的内容。第二个API就是远程通知（Remote Notifications），这是一个当事件发生时可以让推送通知主动提醒应用的新特性，这两者都为你的应用界面保持最新提供了极大的帮助。在新的后台传输服务（Background Transfer Service）中执行定期的任务，也允许你在进程之外可以执行网络传输（下载和上传）工作。

后台获取（Background Fetch）和远程通知（Remote Notification）基于简单的应用程序委托钩子，在应用程序挂起之前的30秒时钟时间开始执行工作。它们不是用于CPU频繁工作或者长时间运行任务，而是用来处理长时间运行的网络请求队列，例如下载一部很大的电影，或者执行快速的内容更新。

对用户来说，多任务处理有一点显而易见的改变就是新的应用切换程序(the new app switcher)，它用来呈现应用到后台时的界面快照。这些快照的存在是有一定理由的--现在你可以在后台完成工作后更新程序快照，以用来呈现新的内容。社交网络、新闻或者天气等应用现在都可以直接呈现最新的内容而不需要用户重新打开应用。我们稍后会介绍如何更新屏幕快照。

## Background Fetch

后台获取（Background Fetch）是一种智能的轮询机制，它很适合需要经常更新内容的程序，像社交网络，新闻或天气的程序。为了在用户启动程序前提前触发后台获取，系统会根据用户行为唤醒应用程序。举个例子，如果用户经常在下午1点使用某个应用程序，系统会学习，适应并在使用周期前执行后台获取。为了减少电池使用，后台获取（Background Fetch）会跨应用程序被设备的无线电合并，如果你向系统报告新数据无法获取，iOS会适应并使用此信息避免会继续获取。

The first step in enabling Background Fetch is to specify that you’ll use the feature in the [`UIBackgroundModes`](https://developer.apple.com/library/ios/documentation/general/Reference/InfoPlistKeyReference/Articles/iPhoneOSKeys.html#//apple_ref/doc/uid/TP40009252-SW22) key in your info plist. The easiest way to do this is to use the new Capabilities tab in Xcode 5’s project editor, which includes a Background Modes section for easy configuration of multitasking options. 

开启后台获取的第一步是在info plist文件中的[UIBackgroundModes](https://developer.apple.com/library/ios/documentation/general/Reference/InfoPlistKeyReference/Articles/iPhoneOSKeys.html#//apple_ref/doc/uid/TP40009252-SW22)健值指定使用的特性。最简单的途径是在Xcode5的project editor中新的性能标签页中（Capabilities tab）设置，这个标签页包含了后台模式部分,可以方便配置多任务选项。

![](http://jbcdn2.b0.upaiyun.com/2013/11/capabilities-on-bgfetch-1-1024x866.jpg)

Alternatively, you can edit the key manually:

或者，你可以手动编辑这个值

    <key>UIBackgroundModes</key>
    <array>
        <string>fetch</string>
    </array>

Next, tell iOS how often you'd like to fetch:  

接下来，告诉iOS多久进行一次数据获取

    - (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions
    {
        [application setMinimumBackgroundFetchInterval:UIApplicationBackgroundFetchIntervalMinimum];
    
        return YES;
    }

The default fetch interval is never, so you'll need to set a time interval or the app won't ever be called in the background. The value of `UIApplicationBackgroundFetchIntervalMinimum` asks the system to manage when your app is woken, as often as possible, but you should specify your own time interval if this is unnecessary. For example, a weather app might only update conditions hourly. iOS will wait at least the specified time interval between background fetches.

iOS默认不进行后台获取，所以你需要设置一个时间间隔，否则，你的应用程序永远不行在后台进行获取数据。UIApplicationBackgroundFetchIntervalMinimum这个值要求系统尽可能经常去管理应用程序什么时候会被唤醒，但如果不需要这个值，你应该指定你的时间间隔。例如，一个天气的应用程序，可能只需要几个小时才更新一次，iOS将会在后台获取之间至少等待你指定的时间间隔。

If your application allows a user to logout, and you know that there won’t be any new data, you may want to set the `minimumBackgroundFetchInterval` back to `UIApplicationBackgroundFetchIntervalNever` to be a good citizen and to conserve resources.

如果你的应用允许用户退出登录，那么就没有获取新数据的需要了，你应该把`minimumBackgroundFetchInterval`设置为`UIApplicationBackgroundFetchIntervalNever`，这样可以节省资源。

The final step is to implement the following method in your application delegate:

最后一步是在应用程序委托中实现下列方法：

    - (void)                application:(UIApplication *)application 
      performFetchWithCompletionHandler:(void (^)(UIBackgroundFetchResult))completionHandler
    {
        NSURLSessionConfiguration *sessionConfiguration = [NSURLSessionConfiguration defaultSessionConfiguration];
        NSURLSession *session = [NSURLSession sessionWithConfiguration:sessionConfiguration];
    
        NSURL *url = [[NSURL alloc] initWithString:@"http://yourserver.com/data.json"];
        NSURLSessionDataTask *task = [session dataTaskWithURL:url 
                                            completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {
        
            if (error) {
                completionHandler(UIBackgroundFetchResultFailed);
                return;
            }
        
            // Parse response/data and determine whether new content was available
            BOOL hasNewData = ...
            if (hasNewData) {
                completionHandler(UIBackgroundFetchResultNewData);
            } else {
                completionHandler(UIBackgroundFetchResultNoData);
            }
        }];
    
        // Start the task
        [task resume];
    }

This is where you can perform work when you are woken by the system. Remember, you only have 30 seconds to determine whether new content is available, to process the new content, and to update your UI. This should be enough time to fetch data from the network and to fetch a few thumbnails for your UI, but not much more. When your network requests are complete and your UI has been updated, you should call the completion handler. 

系统唤醒应用程序后将会执行这个委托方法。需要注意的是，你只有30秒的时间来确定获取的新内容是否可用，然后处理新内容并更新界面。30秒时间应该足够去从网络获取数据和获取界面的缩略图，最多只有30秒。当完成了网络请求和更新界面后，你应该调用完成的处理代码。

The completion handler serves two purposes. First, the system measures the power used by your process and records whether new data was available based on the `UIBackgroundFetchResult` argument you passed. Second, when you call the completion handler, a snapshot of your UI is taken and the app switcher is updated. The user will see the new content when he or she is switching apps. This completion handler snapshotting behavior is common to all of the completion handlers in the new multitasking APIs.

完成的处理代码有两个目的。首先，系统会估量你的进程消耗的电量，并根据你传递的`UIBackgroundFetchResult`参数记录新数据是否可用。其次，当你调用完成的处理代码时，应用的界面缩略图会被采用，并更新应用程序切换器。当用户在应用间切换时，用户将会看到新内容。这种快照行为的完成代码，在新的多任务处理APIs中，很很常见的。

In a real-world application, you should pass the `completionHandler` to sub-components of your application and call it when you've processed data and updated your UI.

在实际应用中，你应该将`completionHandler`传递到应用程序的子组件，然后在处理完数据和更新界面后调用。

At this point, you might be wondering how iOS can snapshot your app's UI when it is running in the background, and how the application lifecycle works with Background Fetch. If your app is currently suspended, the system will wake it before calling `application: performFetchWithCompletionHandler:`. If your app is not running, the system will launch it, calling the usual delegate methods, including `application: didFinishLaunchingWithOptions:`. You can think of it as the app running exactly the same way as if the user had launched it from Springboard, except the UI is invisible, rendered offscreen.

在这里，你可能想知道iOS是如何在应用程序后台运行时获得界面快照的，并且想知道应用程序的生命周期与后台获取之间有什么关系。如果应用程序处于挂起状态，系统会先唤醒应用，然后再调用`application: performFetchWithCompletionHandler:`。如果应用程序还没有启动，系统将会启动它，然后调用常见的委托方法，包括`application: didFinishLaunchingWithOptions:`。你可以把这种应用程序运行的方式想像为用户从Springboard启动这个程序，区别仅仅在于界面是看不见的，在屏幕外渲染的。

In most cases, you'll perform the same work when the application launches in the background as you would in the foreground, but you can detect background launches by looking at the [`applicationState`](https://developer.apple.com/library/ios/documentation/uikit/reference/UIApplication_Class/Reference/Reference.html#//apple_ref/doc/uid/TP40006728-CH3-SW77) property of UIApplication:

大多数情况下，无论应用在后台启动或者在前台，你会执行相同的工作，但你可以通过查看UIApplication的[applicationState](https://developer.apple.com/library/ios/documentation/uikit/reference/UIApplication_Class/Reference/Reference.html#//apple_ref/doc/uid/TP40006728-CH3-SW77)属性来判断应用是不是从后台启动。

    - (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions
    {
        NSLog(@"Launched in background %d", UIApplicationStateBackground == application.applicationState);
    
        return YES;
    }

### Testing Background Fetch

### 测试后台数据获取

There are two ways you can simulate a background fetch. The easiest method is to run your application from Xcode and click _Simulate Background Fetch_ under Xcode's Debug menu while your app is running.

有两种可以模拟后台获取的途径。最简单是从Xcode运行你的应用，当应用运行时，在Xcode的Debug菜单选择 _Simulate Background Fetch_.

Alternatively, you can use a scheme to change how Xcode runs your app. Under the Xcode menu item Product, choose Scheme and then Manage Schemes. From here, edit or add a new scheme and check the _Launch due to a background fetch event_ checkbox as shown below.

第二种方法，使用scheme更改Xcode运行程序的方式。在Xcode菜单的Product选项，选择Scheme然后选择Manage Schemes.在这里，你可以编辑或者添加一个新的scheme，然后选中 _Launch due to a background fetch event_ 。如下图：

![](http://jbcdn2.b0.upaiyun.com/2013/11/edit-scheme-simulate-background-fetch.png)

## Remote Notifications
## 远程通知

Remote notifications allow you to notify your app when important events occur. You might have new instant messages to deliver, breaking news alerts to send, or the latest episode of your user's favorite TV show ready for him or her to download for offline viewing. Remote notifications are great for sporadic but immediately important content, where the delay between background fetches might not be acceptable. Remote Notifications can also be much more efficient than Background Fetch, as your application only launches when necessary.

远程通知允许你在重要事件发生时，告知你的应用。你可能需要发送新的即时信息，突发新闻的提醒，或者用户喜爱电视的最新剧集已经可以下载以便离线观看的消息。远程通知很适合偶尔出现，但当前很重要的内容，这在后台获取之间出现的延迟是不允许的。远程通知会比后台获取更有效率，因为应用程序只有在需要的时候才会启动。

A Remote Notification is really just a normal Push Notification with the `content-available` flag set. You might send a push with an alert message informing the user that something has happened, while you update the UI in the background. But Remote Notifications can also be silent, containing no alert message or sound, used only to update your app’s interface or trigger background work. You might then post a local notification when you've finished downloading or processing the new content.

一条远程通知实际上只是一条普通的带有content-available标志的推送通知。当你在后台更新界面时，你可以发送一条带有提醒信息的推送去告诉用户。但远程通知可以做到在安静地，没有提醒消息或者任何声音的情况下，只去更新应用界面或者触发后台工作。然后你可以在完成下载或者处理完新内容后，发送一条本地通知。

Silent push notifications are rate-limited, so don't be afraid of sending as many as your application needs. iOS and the APNS servers will control how often they are delivered, and you won’t get into trouble for sending too many. If your push notifications are throttled, they might be delayed until the next time the device sends a keep-alive packet or receives another notification.

静默的推送通知有速度限制，所以你可以勇敢地根据应用程序的需要发送通知。iOS和苹果推送服务会控制推送通知多久被递送，发送很多推送通知是没有问题的。如果你的推送通知被禁止，推送通知可能会被延迟，直到设备下次发送保持活动状态的数据包，或者收到另外一个通知。


### Sending Remote Notifications
### 发送远程通知

To send a remote notification, set the content-available flag in a push notification payload. The content-available flag is the same key used to notify Newsstand apps, so most push scripts and libraries already support remote notifications. When you're sending a Remote Notification, you might also want to include some data in the notification payload, so your application can reference the event. This could save you a few networking requests and increase the responsiveness of your app.

要发送一条远程通知，需要在推送通知的有效负载（payload）设置content－available标志。content-available标志和用来通知Newsstand应用的健值是一样的，因此，大多数推送脚本和库都已经支持远程通知。当你发送一条远程通知时，你可能还想要包含一些通知有效负载（payload）中的数据，让你应用程序可以引用时间。这可以为你节省一些网络请求，并提高应用程序的响应度。

I recommend using [Nomad CLI’s Houston](http://nomad-cli.com/#houston) utility to send push messages while developing, but you can use your favorite library or script.

我建议在开发的时候，使用[Nomad CLI’s Houston](http://nomad-cli.com/#houston)工具发送推送消息，你也可以使用你喜欢的库或脚本。

You can install Houston as part of the nomad-cli ruby gem:

你可以通过nomad-cli ruby gem安装Houston

    gem install nomad-cli

And then send a notification with the apn utility included in Nomad

然后通过包含在Nomad的apn实用工具发送一条通知：

    # Send a Push Notification to your Device
    apn push <device token> -c /path/to/key-cert.pem -n -d content-id=42

Here the `-n` flag specifies that the content-available key should be included, and `-d` allows us to add our own data keys to the payload.

在这里，`-n`标志指定应该包含content-available健值，`-d`标志允许添加我们自定义的数据健值到有效负荷（payload）。

The resulting notification payload looks like this:

通知的有效负荷（payload）结果和下面类似：

    {
        "aps" : {
            "content-available" : 1
        },
        "content-id" : 42
    }

iOS 7 adds a new application delegate method, which is called when a push notification with the content-available key is received:

iOS7添加了一个新的应用程序委托方法，当接收到一条带有content－available的推送通知时，这个方法被调用：

    - (void)           application:(UIApplication *)application 
      didReceiveRemoteNotification:(NSDictionary *)userInfo 
            fetchCompletionHandler:(void (^)(UIBackgroundFetchResult))completionHandler
    {
        NSLog(@"Remote Notification userInfo is %@", userInfo);
    
        NSNumber *contentID = userInfo[@"content-id"];
        // Do something with the content ID
        completionHandler(UIBackgroundFetchResultNewData);
    }

Again, the app is launched into the background and given 30 seconds to fetch new content and update its UI, before calling the completion handler. We could perform a quick network request as we did in the Background Fetch example, but let's use the powerful new Background Transfer Service to enqueue a large download task and see how we can update our UI when it completes.

然后，应用程序进入后台启动，有30秒的时间去获取新内容并更新界面，最后调用完成的处理代码。我们可以像后台获取那样，执行快速的网络请求，但我们可以使用新的强大的后台传输服务，处理任务队列，下面看看我们如何在任务完成后更新界面。

## NSURLSession and Background Transfer Service

While `NSURLSession `is a new class in iOS 7, it also refers to the new technology in Foundation networking. Intended to replace `NSURLConnection`, familiar concepts and classes such as `NSURL`, `NSURLRequest`, and `NSURLResponse` are preserved. You’ll work with `NSURLConnection`’s replacement, `NSURLSessionTask`, to make network requests and handle their responses. There are three types of session tasks – data, download, and upload – each of which add syntactic sugar to `NSURLSessionTask`, so you should use the appropriate one for your use case.

NSURLSession是iOS7添加的一个新类，它也是Foundation networking中的新技术。作为NSURLConnection的替代品，一些熟悉的概念和类都保留下来了，例如NSURL，NSURLRequest和NSURLRespond。所以，你可以使用NSURLConnection的替代品——NSURLSessionTask，处理网络请求及响应。一共有3中会话任务：数据，下载和上传。每一种都向NSURLSessionTask添加了语法糖（syntactic sugar），根据你的需要，适当选择一种。

An `NSURLSession` coordinates one or more of these `NSURLSessionTask`s and behaves according to the `NSURLSessionConfiguration` with which it was created. You may create multiple `NSURLSession`s to group related tasks with the same configuration. To interact with the Background Transfer Service, you'll create a session configuration using `[NSURLSessionConfiguration backgroundSessionConfiguration]`. Tasks added to a background session are run in an external process and continue even if your app is suspended, crashes, or is killed.

一个NSURLSession对象协调一个或多个NSURLSessionTask对象，并根据NSURLSessionTask创建的NSURLSessionConfiguration实现不同的功能。使用相同的配置，你也可以创建多组具有相关任务的NSURLSession对象。要利用后台传输服务，你将会使用[NSURLSessionConfiguration backgroundSessionConfiguration]来创建一个会话配置。添加到后台会话的任务在外部进程运行，即使应用程序被挂起，崩溃，或者被杀死，依然会运行。

[`NSURLSessionConfiguration`](https://developer.apple.com/library/ios/documentation/Foundation/Reference/NSURLSessionConfiguration_class/Reference/Reference.html) allows you to set default HTTP headers, configure cache policies, restrict cellular network usage, and more. One option is the `discretionary` flag, which allows the system to schedule tasks for optimal performance. What this means is that your transfers will only go over Wifi when the device has sufficient power. If the battery is low, or only a cellular connection is available, your task won't run. The `discretionary` flag only has an effect if the session configuration object has been constructed by calling the [`backgroundSessionConfiguration:`](https://developer.apple.com/library/ios/documentation/Foundation/Reference/NSURLSessionConfiguration_class/Reference/Reference.html#//apple_ref/occ/clm/NSURLSessionConfiguration/backgroundSessionConfiguration:) method and if the background transfer is initiated while your app is in the foreground. If the transfer is initiated from the background the transfer will _always_ run in discretionary mode.

NSURLSessionConfiguration允许你设置默认的HTTP头部，配置缓存策略，限制使用蜂窝数据等等。其中一个选项是discretionary标志，这个标志允许系统为分配任务进行性能优化。这意味着只有当设备有足够电量时，设备才通过Wifi进行数据传输。如果电量低，或者只仅有一个蜂窝连接，传输任务是不会运行的。后台传输总是在discretionary模式下运行。
 
Now we know a little about `NSURLSession`, and how a background session functions, let's return to our Remote Notification example and add some code to enqueue a download on the background transfer service. When the download completes, we'll notify the user that the file is available for use.

目前为止，我们大概了解了NSURLSession，以及一个后台会话如何进行，接下来，让我们回到远程通知的例子，添加一些代码来处理后台传输服务的下载队列。当下载完成后，我们会通知用户该文件已经可以使用了。

### NSURLSessionDownloadTask

First of all, let's handle a Remote Notification and enqueue an `NSURLSessionDownloadTask` on the background transfer service. In `backgroundURLSession`, we create an `NURLSession` with a background session configuration and add our application delegate as the session delegate. The documentation advises against instantiating multiple sessions with the same identifier, so we use `dispatch_once` to avoid potential issues:

首先，我们先处理一条远程通知，并把一个NSURLSessionDownloadTask添加到后台传输服务的队列。在backgroundURLSession方法中，我们根据后台会话配置，创建一个NSURLSession对象，并把应用程序委托对象（application delegate）作为会话的委托对象。文档反对对于相同的标识符（identifier）创建多个会话对象，所以我们使用dispatch_once来避免潜在的问题：

    - (NSURLSession *)backgroundURLSession
    {
        static NSURLSession *session = nil;
        static dispatch_once_t onceToken;
        dispatch_once(&onceToken, ^{
            NSString *identifier = @"io.objc.backgroundTransferExample";
            NSURLSessionConfiguration* sessionConfig = [NSURLSessionConfiguration backgroundSessionConfiguration:identifier];
            session = [NSURLSession sessionWithConfiguration:sessionConfig 
                                                    delegate:self 
                                               delegateQueue:[NSOperationQueue mainQueue]];
        });
    
        return session;
    }

    - (void)           application:(UIApplication *)application 
      didReceiveRemoteNotification:(NSDictionary *)userInfo 
            fetchCompletionHandler:(void (^)(UIBackgroundFetchResult))completionHandler
    {
        NSLog(@"Received remote notification with userInfo %@", userInfo);
    
        NSNumber *contentID = userInfo[@"content-id"];
        NSString *downloadURLString = [NSString stringWithFormat:@"http://yourserver.com/downloads/%d.mp3", [contentID intValue]];
        NSURL* downloadURL = [NSURL URLWithString:downloadURLString];

        NSURLRequest *request = [NSURLRequest requestWithURL:downloadURL];
        NSURLSessionDownloadTask *task = [[self backgroundURLSession] downloadTaskWithRequest:request];
        task.taskDescription = [NSString stringWithFormat:@"Podcast Episode %d", [contentID intValue]];
        [task resume];
    
        completionHandler(UIBackgroundFetchResultNewData);
    }

We create a download task using the `NSURLSession` class method and configure its request, and provide a description for use later. You must remember to call `[task resume]` to actually start the task, as all session tasks begin in the suspended state.

我们使用NSURLSession类方法创建一个下载任务，配置请求，并提供说明供以后使用。因为所有会话任务一开始处于挂起状态，你必须谨记要调用[task resume]保证开始了任务。

Now we need to implement the `NSURLSessionDownloadDelegate` methods to receive callbacks when the download completes. You may also need to implement `NSURLSessionDelegate` or `NSURLSessionTaskDelegate` methods if you need to handle authentication or other events in the session lifecycle. You should consult Apple's document [Life Cycle of a URL Session with Custom Delegates](https://developer.apple.com/library/ios/documentation/cocoa/Conceptual/URLLoadingSystem/NSURLSessionConcepts/NSURLSessionConcepts.html#//apple_ref/doc/uid/10000165i-CH2-SW42), which explains the full life cycle across all types of session tasks.

现在，我们需要实现NSURLSessionDownloadDelegate的委托方法，当下载完成时，调用回调函数。如果你需要处理认证或会话生命周期的其他事件，你可能还需要实现NSURLSessionDelegate或NSURLSessionTaskDelegate的方法。你应该阅读Apple的[Life Cycle of a URL Session with Custom Delegates](https://developer.apple.com/library/ios/documentation/cocoa/Conceptual/URLLoadingSystem/NSURLSessionConcepts/NSURLSessionConcepts.html#//apple_ref/doc/uid/10000165i-CH2-SW42)文档，它讲解了所有类型的会话任务的完整生命周期。

None of the `NSURLSessionDownloadDelegate` delegate methods are optional, though the only one where we need to take action in this example is `[NSURLSession downloadTask:didFinishDownloadingToURL:]`. When the task finishes downloading, you're provided with a temporary URL to the file on disk. You must move or copy the file to your app's storage, as it will be removed from temporary storage when you return from this delegate method.

NSURLSessionDownloadDelegate中的委托方法全部是必须实现的，尽管在这个例子中我们只需要用到[NSURLSession downloadTask:didFinishDownloadingToURL:]。任务完成下载时，你会得到一个磁盘上该文件的临时URL。你必须把这个文件移动或复制你的应用程序空间，因为当你从这个委托方法返回时，该文件将从临时存储中删除。

    #Pragma Mark - NSURLSessionDownloadDelegate

    - (void)         URLSession:(NSURLSession *)session 
                   downloadTask:(NSURLSessionDownloadTask *)downloadTask
      didFinishDownloadingToURL:(NSURL *)location
    {
        NSLog(@"downloadTask:%@ didFinishDownloadingToURL:%@", downloadTask.taskDescription, location);

        // Copy file to your app's storage with NSFileManager
        // ...

        // Notify your UI
    }

    - (void)  URLSession:(NSURLSession *)session 
            downloadTask:(NSURLSessionDownloadTask *)downloadTask 
       didResumeAtOffset:(int64_t)fileOffset 
      expectedTotalBytes:(int64_t)expectedTotalBytes
    {
    }

    - (void)         URLSession:(NSURLSession *)session 
                   downloadTask:(NSURLSessionDownloadTask *)downloadTask 
                   didWriteData:(int64_t)bytesWritten totalBytesWritten:(int64_t)totalBytesWritten 
      totalBytesExpectedToWrite:(int64_t)totalBytesExpectedToWrite
    {
    }

If your app is still running in the foreground when the background session task completes, the above code will be sufficient. In most cases, however, your app won't be running, or it will be suspended in the background. In these cases, you must implement two application delegates methods so the system can wake your application. Unlike previous delegate callbacks, the application delegate is called twice, as your session and task delegates may receive several messages. The app delegate method `application: handleEventsForBackgroundURLSession:` is called before these `NSURLSession` delegate messages are sent, and `URLSessionDidFinishEventsForBackgroundURLSession` is called afterward. In the former method, you store a background `completionHandler`, and in the latter you call it to update your UI:

当后台会话任务完成时，如果你的应用程序仍然在前台运行，上面的代码已经足够了。然而，在大多数情况下，你的应用程序没有运行，或者在后台被挂起。在这些情况下，你必须实现应用程序委托的两个方法，这样系统就可以唤醒你的应用程序。不同于以往的委托回调，该应用程序委托会被调用两次，因为您的会话和任务委托可能会收到一系列消息。应用程序委托的：handleEventsForBackgroundURLSession：方法，在这些NSURLSession委托的消息发送前被调用，然后，URLSessionDidFinishEventsForBackgroundURLSession被调用。在前面的方法中，储存了一个后台完成处理代码（completionHandler），并在后面的方法中调用该代码更新界面。

    - (void)                  application:(UIApplication *)application 
      handleEventsForBackgroundURLSession:(NSString *)identifier completionHandler:(void (^)())completionHandler
    {
        // You must re-establish a reference to the background session, 
        // or NSURLSessionDownloadDelegate and NSURLSessionDelegate methods will not be called
        // as no delegate is attached to the session. See backgroundURLSession above.
        NSURLSession *backgroundSession = [self backgroundURLSession];
    
        NSLog(@"Rejoining session with identifier %@ %@", identifier, backgroundSession);
    
        // Store the completion handler to update your UI after processing session events
        [self addCompletionHandler:completionHandler forSession:identifier];
    }

    - (void)URLSessionDidFinishEventsForBackgroundURLSession:(NSURLSession *)session
    {
        NSLog(@"Background URL session %@ finished events.\n", session);
    
        if (session.configuration.identifier) {
            // Call the handler we stored in -application:handleEventsForBackgroundURLSession:
            [self callCompletionHandlerForSession:session.configuration.identifier];
        }
    }

    - (void)addCompletionHandler:(CompletionHandlerType)handler forSession:(NSString *)identifier
    {
        if ([self.completionHandlerDictionary objectForKey:identifier]) {
            NSLog(@"Error: Got multiple handlers for a single session identifier.  This should not happen.\n");
        }
    
        [self.completionHandlerDictionary setObject:handler forKey:identifier];
    }

    - (void)callCompletionHandlerForSession: (NSString *)identifier
    {
        CompletionHandlerType handler = [self.completionHandlerDictionary objectForKey: identifier];
    
        if (handler) {
            [self.completionHandlerDictionary removeObjectForKey: identifier];
            NSLog(@"Calling completion handler for session %@", identifier);
        
            handler();
        }
    }


This two-stage process is necessary to update your app UI if you aren't already in the foreground when the background transfer completes. Additionally, if the app is not running at all when the background transfer finishes, iOS will launch it into the background, and the preceding application and session delegate methods are called after `application:didFinishLaunchingWithOptions:`.

如果当后台传输完成时，应用程序不再在前台，那么，对于更新程序界面来说，这两步是必要的。此外，如果当后台传输完成时，应用程序根本没有在运行，iOS将会在后台启动该应用程序，然后前面的应用程序和会话的委托方法会在application:didFinishLaunchingWithOptions:.方法被调用之后被调用。

### Configuration and Limitation
### 配置和限制

We've briefly touched on the power of background transfers, but you should explore the documentation and look at the `NSURLSessionConfiguration` options that best support your use case. For example, `NSURLSessionTasks` support resource timeouts through the `NSURLSessionConfiguration`'s `timeoutIntervalForResource` property. You can use this to specify how long you want to allow for a transfer to complete before giving up entirely. You might use this if your content is only available for a limited time, or if failure to download or upload the resource within the given timeInterval indicates that the user doesn't have sufficient Wifi bandwidth. 

我们简单地体验了后台传输的强大之处，但你应该深入文档，阅读NSURLSessionConfiguration部分，以便最好地满足你的情况。例如，NSURLSessionTasks通过NSURLSessionConfiguration的timeoutIntervalForResource属性，支持资源超时特性。你可以使用这个特性指定你允许完成一个传输所需的最长时间。内容只在有限的时间可用，或者在用户只有有限Wifi带宽的时间内无法下载或上传资源的情况下，你也可以使用这个特性。

In addition to download tasks, `NSURLSession` fully supports upload tasks, so you might upload a video to your server in the background and assure your user that he or she no longer needs to leave the app running, as might have been done in iOS 6. A nice touch would be to set the `sessionSendsLaunchEvents` property of your `NSURLSessionConfiguration` to `NO`, if your app doesn't need launching in the background when the transfer completes. Efficient use of system resources keeps both iOS and the user happy.

除了下载任务，NSURLSession也全面支持上传任务，因此，你可能会在后台将视频上传到服务器，这保证用户不需要再像iOS6那样离开正在运行的应用程序。如果当传输完成时你的应用程序不需要在后台运行，一个比较好的做法是，把NSURLSessionConfiguration的sessionSendsLaunchEvents属性设置为NO。高效利用系统资源，是一件让iOS和用户都高兴的事。

Finally, there are a couple of limitations in using background sessions. As a delegate is required, you can't use the simple block-based callback methods on `NSURLSession`. Launching your app into the background is relatively expensive, so HTTP redirects are always taken. The background transfer service only supports HTTP and HTTPS and you cannot use custom protocols. The system optimizes transfers based on available resources and you cannot force your transfer to progress in the background at all times.

最后，我们来说一说使用后台会话的几个限制。作为一个必须实现的委托，您不能对NSURLSession使用简单的基于块的回调方法。后台启动应用程序，是相对耗费较多资源的，所以总是采用HTTP重定向。后台传输服务只支持HTTP和HTTPS，你不能使用自定义的协议。系统会根据可用的资源进行优化，在任何时候你都不能强制传输任务在后台进行。

Also note that `NSURLSessionDataTasks` are not supported in background sessions at all, and you should only use these tasks for short-lived, small requests, not for downloads or uploads.

另外，要注意，在后台会话中，NSURLSessionDataTasks 是完全不支持的，你应该只出于短期的，小请求为目的使用这些任务，而不是用来下载或上传。

## Summary
## 总结

The powerful new multitasking and networking APIs in iOS 7 open up a whole range of possibilities for both new and existing apps. Consider the use cases in your app which can benefit from out-of-process network transfers and fresh data, and make the most of these fantastic new APIs. In general, implement background transfers as if your application is running in the foreground, making appropriate UI updates, and most of the work is already done for you.

iOS7中新添加的多任务处理和网络的APIs十分强大，它们为现有和新的应用程序开辟了一系列可能。如果你的应用程序可以从进程外的网络传输和数据中获益，那么尽情地使用这些美妙的APIs！一般情况下，实现后台传输，可以假装你的应用程序正在前台运行，并进行适当的界面更新，而这大部分的工作已经为你完成了。

- Use the appropriate new API for your app’s content.
- Be efficient, and call completion handlers as early as possible. 
- Completion handlers update your app’s UI snapshot.

- 使用适当的新API，为你的应用程序提供内容服务。
- 尽可能早地有效率调用完成处理代码。
- 让完成的处理代码为应用程序更新界面快照。


## Further Reading
## 扩展阅读

- [WWDC 2013 session “What’s New with Multitasking”](https://developer.apple.com/wwdc/videos/?id=204)
- [WWDC 2013 session “What’s New in Foundation Networking”](https://developer.apple.com/wwdc/videos/?id=705)
- [URL Loading System Programming Guide](https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/URLLoadingSystem/URLLoadingSystem.html#//apple_ref/doc/uid/10000165i)




