---
author: michael-collins
date: 2021-01-02T13:54:13-07:00
description: In this post, I cover the new SwiftUI application model that was introduced with iOS 14 and how to manage multiple scenes on iPad devices.
draft: true
title: Creating Multiple Scenes in a SwiftUI App
---
The latest iPad and iPad Pro devices are ripe and ready for more powerful business solutions. In this post, I will describe the ability to create multiple scenes in iPad applications.

<!--more-->

## Introduction

One of the reasons for the success of modern mobile platforms such as Apple's iOS and iPhone has been the simplicity of the application model. Since the beginning, iOS always provided a simple application model. The application would start by launching. At a certain point, it may move to the background. If the user switched back to the application, the application comes back into the foreground and starts running again. At some future time, if iOS needs to remove the application from active memory, it will safely terminate the application if it's in the background. This model works so well because complexity is limited. There's only ever one instance of the application running. There's no need to synchronize shared resources between multiple copies of the application. Applications are sandboxed and containerized and don't share resources with other applications, so conflicts between applications are pretty much non-existent.

From the introduction of the original iPad, we could all see where Apple was headed. The iPad is a fantastic consumer device and fills a need for users between having a lightweight phone for travel and a desktop or laptop computer. The iPad is the everything inbetween portable solution that enables us to read our email, read books, or browse social media while sitting in bed, or on the couch, or while eating at a restaurant or at the dinner table. The iPad is also portable enough to take to the park and work outside. But the original iPad led to the iPad Pro with [Apple's Magic Keyboard, Smart Keyboard Folio, and Smart Keyboard](https://www.apple.com/ipad-keyboards/). Apple's goal wasn't just to make a better portable device, but to provide a better platform for doing real work.

Applications for iPad also got stronger. Apple introduced split screen functionality letting two applications share the screen. Apple introduced drag-and-drop allowing sharing data from one application to another application. Imagine dragging an image from your photo library and dropping it into an email or a document. Microsoft produced full versions of Microsoft Office 365 that take advantage of the keyboard for typing in Microsoft Word, managing spreadsheets in Excel, or building presentations with PowerPoint. A number of editing tools such as Ulysses, IA Writer, Bear, Evernote, and others popped up to turn iPads with keyboards into full productivity machines.

With iOS 13, not only did Apple introduce a new way to build applications with SwiftUI, but Apple also introduced their first major revision to the application model directly for use in iPad applications. We still have the single application instance model that has been present in iOS since the start, but with iOS 13, Apple introduced the concept of **scenes**. Using scenes, iPad applications can present multiple tasks or windows concurrently. iPad applications are no longer limited to a single window and navigation tools such as split views or tab views to switch the user between tasks. Now iPad applications can display different scenes to allow users to perform different tasks concurrently, and the scenes can run side-by-side using the iPad split window interface.

In iOS 14, Apple expanded SwiftUI to support scenes as well. Up until iOS 14, iOS developers needed to implement their own application delegates and scene delegates in iOS 13. With iOS 14, SwiftUI gained its own application model, and its own way to manage scenes. While the SwiftUI support for multiple scenes works for about 80% of scenarios, there's sometimes a need to handle something in the remaining 20%, and fortunately Apple's developers left us with a hole to extend the SwiftUI application model.

In this post, I will show you how to use iOS 14 and SwiftUI to create iPad applications that support multiple scenes. I will show you how to launch new scenes, close existing scenes, and I will take you down the rabbit hole of routing external stimulus events such as push notifications and universal links to the correct scene to be handled.

## The Application Model

{{< admonition type=warning title="External Displays" open=true >}}
This post is going to assume the presence of a single screen, the device's screen. In a future post, I will describe how to support external displays such as a television, computer monitor, or projector attached to an iPhone or iPad device.
{{< /admonition >}}

Prior to iOS 13, the application model was pretty simple. iOS starts a single instance of the application and provides the application with a window that is connected to the device's screen.

{{< image src=simple_application_model.png alt="Simple iOS application model with a UIApplication, UIScrene, UIWindow, and the root UIViewController. This was the model used for iOS 12 and earlier." caption="Pre-iOS 13 Application Model" title="Pre-iOS 13 Application Model" >}}

At WWDC 2019, Apple introduced a new concept into the application model: scenes. Apple also introduced a split of the iOS operating system. While they're still nearly identical, iOS was being targeted for use on the iPhone and the new iPadOS was introduced to provide iPad-specific features. While the APIs and the core kernels of the operating systems are identical, there are some features present in each that aren't available on the other device form. While scenes are present in modern iOS applications, they're really only utilized on iPadOS for iPad applications.

Scenes refactor and remove responsibilities from the `UIApplication` class and `UIApplicationDelegate` protocol and move those into the new `UIScene` class and `UISceneDelegate` object. The application delegate is still present and manages the application lifecycle events such as launching and termination, and manages global application state. Scenes and scene delegates take on more of the UI responsibilities and handle the application's user experience being activated or moved to the background.

On an iPad, scenes also allow applications to provide multiple user interfaces for different tasks. Take a document editor similar to Microsoft Word. Instead of needing to manage a single user experience and provide a navigation mechanism to allow a user to switch between different documents, the application can now launch multiple scenes, hosting one document per scene. Each scene has its own window and can present its own user experience. Apple also added a new way to see all of the scenes that the application has created for the user.

{{< image src=show_all_windows_small.png alt="Application menu showing the See All Windows option on the iPadOS Home Screen." caption="Application menu showing the Show All Windows option" title="Show All Windows Menu Option" src_l=show_all_windows.jpeg >}}

The new application model is shown in the diagram below. The application still has the single `UIApplication` instance that manages the application execution. Each application can have one or more scenes. Each scene has its own window with its own root view controller. Each scene can present and manage its own user experience.

{{< image src=multiple_scene_model_small.png alt="Diagram showing the new model for hosting multiple scenes with their own windows and root view controllers." caption="Multiple Scene Application Model" title="Multiple Scene Application Model" src_l=multiple_scene_model.png >}}

Each scene does not need to present a unique user experience. For example, Safari on iOS presents the same tabbed interface for each window that it creates. An example of where alternative user experiences can be presented might be an email application. An email application may host the main email interface in one scene, but might allow the creation of new windows and scenes for viewing or creating an email.

## Can We See Some Code Already?

{{< admonition type=warning title="Software Requirements" open=true >}}
The rest of this post uses features in SwiftUI 2. The sample code in this post will only work on iOS 14 and newer. All of the code should work in the simulator as well as on a device.
{{< /admonition >}}

Yes. Enough of the theory and explanation of multiple scenes. How do we implement multiple scenes using SwiftUI? I'm going to start by creating a new SwiftUI application in Xcode using the new SwiftUI lifecycle.

{{< image src=new_application.png alt="New application wizard in Xcode showing the settings for the new application. The new application will use the new SwiftUI lifecycle." caption="The new application will use the new SwiftUI lifecycle" title="Creating the Sample Application" >}}

By default, Xcode generates an application with a single scene:

{{< gist mfcollins3 4b5b95b247e3334fc1aaca18e7c850e6 "MultipleSceneApp_1.swift" >}}

The `App` protocol in SwiftUI replaces the presence of a `UIApplicationDelegate` object in UIKit and SwiftUI applications. SwiftUI will provide a default `UIApplicationDelegate` implementation that will work for most basic application cases. In this example, `MultipleSceneApp` implements the `App` protocol and implements the `body` property that returns a `Scene` value. The current implementation has a single scene, represented by `WindowGroup` that presents its user experience using the `ContentView` SwiftUI view.

The trick to understand in this implementation is that the `App.body` property is decorated using a special function builder named `SceneBuilder`. This function builder will allow multiple scenes in the `body` implementation and will wrap them all up in a composite scene that will be given to the application. The `SceneBuilder` function builder is the key that we need to implement support for multiple scenes for an iPad application.

Let's start this exploration of multiple scenes by adding code to create a new scene. I'm going to remove the default implementation of `ContentView` and will replace it with a `Button` that when tapped will launch a new scene:

{{< gist mfcollins3 4b5b95b247e3334fc1aaca18e7c850e6 "ContentView_1.swift" >}}

When I run the app initially in the iPad simulator, SwiftUI creates my main scene and hosts my ContentView in the scene:

{{< image src=single_scene_small.png alt="Sample application showing a single scene running in the full screen of the iPad device and displaying a button that can be tapped to create a second scene." caption="Initial Scene" title="Initial Scene" src_l=single_scene.png >}}

Tapping on the **Create Scene** button will cause the new scene to be created. The new scene is created by splitting the device screen and showing the new scene in side-by-side mode:

{{< image src=multiple_scenes_small.png alt="The sample application running multiple scenes side-by-side after tapping the Create Scene button in the initial scene." caption="Multiple scenes running side-by-side" title="Multiple Scenes" src_l=multiple_scenes.png >}}

If we go back to the Home Screen, perform a long touch on the application icon, and choose the **Show All Windows** menu option, we'll see both scenes appear and allow us to tap to resume the scene.

{{< image src=show_multiple_scenes_small.png alt="Showing both scenes of the sample application running in separate windows." caption="Sample application in Show All Windows" title="Multiple Application Scenes" src_l=show_multiple_scenes.png >}}

iPadOS and SwiftUI handle the creation of the initial scene automatically. Creating additional scenes is handled in the `UIApplication` object using the [`requestSceneSessionActivation`](https://developer.apple.com/documentation/uikit/uiapplication/3197901-requestscenesessionactivation#) function. When the application calls `requestSceneSessionActivation` and passes a `nil` reference for the first parameter, we're telling `UIApplication` that we want to create a new scene instead of reusing an existing scene. The end result of this call is that `UIApplication` creates a new instance of the single `WindowGroup` that I defined in the `MultipleSceneApp` struct.

What if I wanted to create a scene with a different user experience? How can I achieve that? The answer leads us into an exploration of the `NSUserActivity` class and **target content identifiers**.

If you've been programming for iOS for a while and have been paying attention, then you should have noticed how `NSUserActivity` has been becoming more and more prominent on iOS. `NSUserActivity` was originally introduced in iOS 8 and powered such activities like Handoff, Siri Shortcuts, and Spotlight search. With scenes, `NSUserActivity` also became the basis of implementing state restoration by allowing scenes to store their state in an `NSUserActivity` object and restore it later (stay tuned for a blog post on that down the road). `NSUserActivity` is fundamental to the ability to create new scenes.

Typically an application is not going to create a new scene just because it can. Scenes will get created to perform a specific task such as editing a document, browsing a web page, editing a video, creating a new email, or performing some other kind of activity. In order to communicate the intent of what the new scene should be doing, the requesting scene can use an `NSUserActivity` object to transfer information to the new scene to use to start its execution.

In iOS 13, Apple enhanced the `targetContentIdentifier` property to `NSUserActivity`. `NSUserActivity` already supports an activity type which is intended to identify the data contained by an `NSUserActivity` object. The `targetContentIdentifier` property was added specifically to support multiple scenes by helping UIKit to route external events to the correct source. We'll come back to this later in the post.

For the purposes of creating a new scene, we can also use `targetContentIdentifier` to tell `UIApplication` and SwiftUI which scene that we would like to create. To start off with, I need the content view for my second scene, which will be pretty simple:

{{< gist mfcollins3 4b5b95b247e3334fc1aaca18e7c850e6 "SecondSceneView_1.swift" >}}

Next, I will update `MultipleSceneApp` to create a new `WindowGroup` to host `SecondSceneView`:

{{< gist mfcollins3 4b5b95b247e3334fc1aaca18e7c850e6 "MultipleSceneApp_2.swift" >}}

Notice the addition of the `handlesExternalEvents` modifier on `WindowGroup`. The `handleExternalEvents` modifier allows me to specify the set of target content identifiers that the `WindowGroup` knows about. By default, a `WindowGroup` can handle all target content identifiers. In this case, I'm limiting the new `WindowGroup` to the target content identifier matching `me.michaelfcollins3.MultipleSceneApp.SceneTwo` target content identifier.

When I run the application, you'll notice that the original scene is created first. When multiple scenes are provided in the `App` implementation, SwiftUI will always use the first scene specified as the default initial scene to use. When I request a new scene to be created and provide a target content identifier, then SwiftUI will look at all of my scenes and will try to find a match.

In order to create the second scene, I have to update `ContentView` to pass an `NSUserActivity` object to `UIApplication.requestSceneSessionActivation`. When I do this, I make sure to set the `targetContentIdentifier` property to `me.michaelfcollins3.MultipleSceneApp.SceneTwo`. When I create the new scene, SwiftUI creates a new scene using my second `WindowGroup` definition:

{{< gist mfcollins3 4b5b95b247e3334fc1aaca18e7c850e6 "ContentView_2.swift" >}}

{{< image src=second_scene_small.png alt="Showing the new second scene being displayed side-by-side with the main scene." caption="The new second scene is shown in side-by-side-mode" title="The Second Scene" src_l=second_scene.png >}}
