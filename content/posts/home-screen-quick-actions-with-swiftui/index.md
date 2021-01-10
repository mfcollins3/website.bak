---
author: michael-collins
categories:
- iOS Development
date: 2021-01-10T12:14:00-07:00
description: Home Screen quick actions are a great way to expose common actions that your customers perform using your application. When SwiftUI 2 was released, there was surprisingly little said about how to implement quick actions in new applications. I will show you how to use custom scene delegates to support Home Screen quick actions in your SwiftUI apps.
draft: false
resources:
- name: "featured-image"
  src: "home_screen_quick_action.png"
- name: "featured-image-preview"
  src: "preview_image.png"
tags:
- iOS
- SwiftUI
- Home Screen
- Quick Action
title: Home Screen Quick Actions With SwiftUI
---
Home Screen quick actions are a great way to expose common actions that your customers perform using your application. When SwiftUI 2 was released, there was surprisingly little said about how to implement quick actions in new applications. I will show you how to use custom scene delegates to support Home Screen quick actions in your SwiftUI apps.

<!--more-->

One of the most common features that I see in app reviews from customers of my client's apps is the ability to add shortcuts to the home screen for frequent actions. For a gym app, the request could be a shortcut to bring up the barcode that the gym scans to check in the customer at the front desk. A business customer might want a shortcut to book a room at a hotel that the customer frequently stays at. These shortcuts are formally known as [Home Screen quick actions](https://developer.apple.com/design/human-interface-guidelines/ios/system-capabilities/home-screen-actions/) and can be a great way to win customer favor by exposing common actions.

Home Screen quick actions were introduced as a cool feature for iOS 3D Touch technology. With the deprecation of 3D Touch in newer iPhone models, the feature has been adopted now into long-pressing the app icon on the Home Screen. When SwiftUI 2 was introduced with its new application model, it has not been made clear by Apple on how to implement Home Screen quick actions with the new application model. Since this is a common feature request, it's one of the first areas that I started looking to when I made the jump to SwiftUI 2 during the iOS 14 beta period. In this post, I will show you how to add support for Home Screen quick actions in your SwiftUI applications.

{{< admonition type=warning title="Requires iOS 14+ and SwiftUI 2" open=true >}}
The article is only relevant for iOS 14+ and SwiftUI 2+.
{{< /admonition >}}

## The Anatomy of Home Screen Quick Actions

Home Screen quick actions come in two varieties:

* Static quick actions
* Dynamic quick actions

Static quick actions are defined in the application's `Info.plist` file using the `UIApplicationShortcutItems` key. `UIApplicationShortcutItems` is an array of dictionaries, with each dictionary containing the properties for a single Home Screen quick action. Each quick action item can have the following attributes:

* `UIApplicationShortcutItemType`: An application-specific string value that uniquely identifies the quick action. The application will use this value to determine how to interpret and respond to the quick action when the application is launched or activated by the quick action.
* `UIApplicationShortcutItemTitle`: The title of the quick action that is displayed to the user in the app's menu.
* `UIApplicationShortcutItemSubtitle`: A string that is displayed below the title on the application menu. (**OPTIONAL**)
* `UIApplicationShortcutItemIconType`: The name of the icon to use from the system-provided icon library. (**OPTIONAL**)
* `UIApplicationShortcutItemIconFile`: The path to an icon image to use from the app's bundle, or the name of an icon image in the application's asset catalog. (**OPTIONAL**)
* `UIApplicationShortcutItemUserInfo`: A dictionary containing additional information that is passed to the application in the shortcut item. (**OPTIONAL**)

An example quick action is shown below:

{{< highlight xml "linenos=true" >}}
<key>UIApplicationShortcutItems</key>
<array>
  <dict>
    <key>UIApplicationShortcutItemType</key>
    <string>me.michaelfcollins3.QuickActionDemo.CreateNoteAction</string>
    <key>UIApplicationShortcutItemIconType</key>
    <string>UIApplicationShortcutIconTypeCompose</string>
    <key>UIApplicationShortcutItemTitle</key>
    <string>Create a Note</string>
  </dict>
</array>
{{< /highlight >}}

Dynamic quick actions can be added at runtime by the application. Dynamic quick actions are created by creating a `UIApplicationShortcutItem` object and adding it to the `UIApplication.shortcutItems` property. Dynamic shortcut items are effective for scenarios where you want to provide more information. For example, a quick action for reading messages in an inbox may want to include the number of unread messages. A bank app or financial app may want to show a current balance for an account-specific activity.

The Home Screen limits the number of Home Screen quick actions that can be displayed in the app's menu. This is typically a function of the screen size and icon position on the Home Screen. Static quick actions will always appear before dynamic quick actions. Applications should limit the number of quick actions that they define to an amount that makes sense. 

## Supporting Home Screen Quick Actions

As I mentioned, when SwiftUI 2 came out in June 2020, I dove into the first beta to take a look at the new application model. Since Home Screen quick actions are one of my frequent feature requests, it actually happened to be the first thing that I looked at, and immediately ran into a roadblock with.

Prior to iOS 13, quick actions were routed to the application delegate object using the `UIApplicationDelegate.applciation(_:performActionFor:completionHandler:)` function. In iOS 13, Apple introduced scenes into the applications and the quick action handler was moved from the `UIApplicationDelegate` protocol to the `UIWindowSceneDelegate` protocol using the `windowScene(_:performActionFor:completionHandler:)` function.

If you haven't read [my previous post]({{< ref "creating-multiple-scenes-in-a-swiftui-app" >}}), you may not recongize the immediate issue confronting us with the new Swift UI 2 application model. First, using the new application model, we have neither an application delegate nor a scene delegate. The application delegate was replaced with the SwiftUI `App` type and scenes are now implemented using `WindowGroup`. But by default, there are no delegates for the application or scene. SwiftUI does allow us to use the `UIApplicationDelegateAdaptor` property wrapper in our `App` implementation type to specify a custom application delegate type to be used for initializing global state and services such as an analytics service. But SwiftUI 2 does not provide us any way to specify a custom scene delegate to be used.

As it turns out, life as a SwiftUI developer is not as bleak as it may seem. Fortunately, it appears that the SwiftUI team thought about this a little (not enough apparently to document this; but they obviously thought about it). As it turns out, if you specify a custom `UIApplication` delegate using the `UIApplicationDelegateAdaptor` property wrapper, and in your custom application delegate class implement the `application(_:configurationForConnecting:options:)` function, you can provide a custom scene delegate and SwiftUI will use that for screen events.

I am going to start with a new iOS application using the new SwiftUI life cycle. Xcode will automatically generate me an `App` type that looks like this:

{{< gist mfcollins3 76dd8cd29b1444f959699e422790eda1 "QuickActionDemoApp_1.swift" >}}

I will update this definition to associate a custom `AppDelegate` class that has not been created yet with the SwiftUI application:

{{< gist mfcollins3 76dd8cd29b1444f959699e422790eda1 "QuickActionDemoApp_2.swift" >}}

My next step is to create the custom `AppDelegate` class and to customize the `application(_:configurationForConnecting:options:)` function to associate a custom scene delegate with my application's scene:

{{< gist mfcollins3 76dd8cd29b1444f959699e422790eda1 "AppDelegate_1.swift" >}}

I will create the `MainSceneDelegate` class and implement the `scene(_:willConnectTo:options:)` function and the `windowScene(_:performActionFor:completionHandler:)` function to handle the quick action:

{{< gist mfcollins3 76dd8cd29b1444f959699e422790eda1 "MainSceneDelegate_1.swift" >}}

I am using the static quick action definition that I showed earlier in my `Info.plist` file. I have to implement both `scene(_:willConnectTo:options)` and `windowScene(_:performActionFor:completionHandler:)` in the scene delegate. The `scene(_:willConnectTo:options:)` function will be called if the application is not running and I will need to handle the shortcut when the scene is created and connected to the scene session. If the application is running and the scene is connected to the session, then iOS will invoke the `windowScene(_:performActionFor:completionHandler:)` function to handle the quick action.

In the delegate functions, I am checking the type value for the shortcut item. If the type matches what I am expecting, then I am initiating a navigation to a handler that can handle the quick action. Notice that my custom scene delegate can reference the `openURL` environment property. `openURL` acts as a bridge allowing me to execute actions in SwiftUI from my UIKit code. I have not seen another way to get access to the SwiftUI views when using the `WindowGroup` to create the scene, so `openURL` seems like the only way unless I want to create the UI myself in the scene delegate. It appears that if I do not create the UI myself, then the `WindowGroup` will create its UI, but I can still safely customize the session connection event in the scene delegate.

I will update my `ContentView` view type to handle the URL that I passed to `openURL`. For the time being, I will only display an alert to validate that the application has successfully received and routed the quick action.

{{< gist mfcollins3 76dd8cd29b1444f959699e422790eda1 "ContentView_1.swift" >}}

Here is what the application looks like when using the quick action:

{{< image src=quick_action_demo.gif alt="Demonstration showing that selecting the quick action on the Home Screen menu will make the alert appear." caption="Selecting the quick action displays an alert" title="Home Screen Quick Action Demo" >}}

## The Problem with Multiple Scenes

If you read [post on multiple scenes with SwiftUI 2]({{< ref "creating-multiple-scenes-in-a-swiftui-app" >}}), there is one little trap that you will probably run into which I have not been able to solve yet. In my post, I talk about how target content identifiers can be used to route events to the correct scene. If you look at the `UIApplicationShortcutItem` class, you'll see that it also has a target content identifier field.

The problem that I have run into with shortcut items is that I have not figured out how to set the target content identifier property for a quick action. For dynamic quick actions, you cannot set the property directly since it only publicly exposes a *getter*. For static quick actions, I have tried using the `UIApplicationShortcutItemTargetContentIdentifier` key in the dictionary and that does not appear to work either. So at this point, I have no idea of how to route the quick action to the correct scene.

The behavior that I have observed is that the quick action will route to whichever scene was most recently active. Therefore, if you are building an application that has multiple scenes and also implements quick actions, you will need to implement support for the quick actions in all of your scene delegates. You might want to consider using a common base class.

I have an open feedback request with Apple about this issue and will publish an update if I ever get a response.

## What We Learned

In this post, I demonstrated how you can implement support for home screen quick actions in a SwiftUI 2 application. Home Screen quick actions provide a great way to expose shortcuts to common actions that your customers perform using your application. While SwiftUI 2 does not natively implement support for handling quick actions like other external events, adding support for them is fairly easy through the creation of custom scene delegates.