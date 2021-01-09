---
title: "Home Screen Quick Actions With Swiftui"
date: 2021-01-09T08:58:05-07:00
draft: true
---

## Supporting Home Screen Quick Actions

Now I would normally have been fine not knowing any of these details about creating custom scene delegates and multiple scenes, but the use case that led me to learning about this stuff happened to be the first thing that I decided to try with my first SwiftUI application using the new application model was implementing [Home Screen quick actions](https://developer.apple.com/design/human-interface-guidelines/ios/system-capabilities/home-screen-actions/). Surprisingly, this is a feature that I have received frequent requests for, so I try to put them into new apps when I get started. However, I ran into a roadblock. While SwiftUI 2 has added support for universal links, deep links, and [Handoff](https://developer.apple.com/documentation/foundation/task_management/implementing_handoff_in_your_app), SwiftUI does not have any direct support for quick actions. But now that I know how to set up a custom scene delegate, it's fairly simple to add support for quick actions.

I will start by adding a Home Screen quick action definition in my `Info.plist` file:

{{< highlight xml "linenos=true" >}}
<key>UIApplicationShortcutItems</key>
<array>
  <dict>
    <key>UIApplicationShortcutItemType</key>
    <string>me.michaelfcollins3.MultipleSceneApp.CreateNoteAction</string>
    <key>UIApplicationShortcutItemIconType</key>
    <string>UIApplicationShortcutIconTypeCompose</string>
    <key>UIApplicationShortcutItemTitle</key>
    <string>Create a Note</string>
  </dict>
</array>
{{< /highlight >}}

When I long press on the app icon in the iPad Simulator, I see my new quick action:

{{< image src=home_screen_quick_action_small.png alt="The Home Screen quick action that I added to the app menu." caption="The quick action shows when long pressing the icon." title="Home Screen Quick Action" src_l=home_screen_quick_action.png >}}

If I just tap on the quick action, the application opens to the main scene but does not do anything. I need to implement support for that. Earlier, I did not create a custom scene delegate for the main scene, but I'll do that now.

{{< gist mfcollins3 4b5b95b247e3334fc1aaca18e7c850e6 "MainSceneDelegate_1.swift" >}}

I also updated my custom application delegate to specify `MainSceneDelegate` for the main scene's configuration:

{{< gist mfcollins3 4b5b95b247e3334fc1aaca18e7c850e6 "AppDelegate_2.swift" >}}

I added the `multiplesceneapp` URL scheme to my application on the Capabilities tab of my project, which added it to the `Info.plist` file. When the **Compose a Note** quick action executes, it will be routed to `MainSceneDelegate`. `MainSceneDelegate` implements the `windowScene(_:performActionFor:completionHandler:)` function to handle the quick action request. The implementation obtains the `openURL` function from the SwiftUI environment, and then will pass the `multiplesceneapp://app/createnote` URL to the SwiftUI view. Now, I need to modify the main scene's view to handle the URL. I updated `ContentView` to display a temporary **TODO** alert as a placeholder to show that the action was received:

{{< gist mfcollins3 4b5b95b247e3334fc1aaca18e7c850e6 "ContentView_3.swift" >}}
