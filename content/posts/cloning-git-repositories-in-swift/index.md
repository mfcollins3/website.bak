---
categories:
- Naked Blogging
date: 2021-01-31T14:10:00-07:00
description: In my previous post, I showed you how I am building libgit2 for iOS to use it in Naked Blogging. In this post, I will begin using it to clone my website repository to my iPad.
draft: false
resources:
- name: "featured-image"
  src: "feature.jpg"
- name: "featured-image-preview"
  src: "feature_small.jpg"
tags:
- iOS
- libgit2
- Swift
title: "Cloning Git Repositories"
---
In my [last Naked Blogging post]({{< ref build-libgit2-for-ios >}}), I showed you how I built libgit2, libssh2, and OpenSSL for iOS and package them as XCFrameworks so that I can use them in my Naked Blogging application. In this post, I will implement my first Git feature: cloning my website repository on GitHub.

<!--more-->

This blog that you are reading right now is hosted on [GitHub](https://github.com) using a service that GitHub provides called [GitHub Pages](https://pages.github.com). GitHub Pages allows me to create a public repository using my account name that GitHub will treat as a static website and serve from the GitHub HTTP endpoints. The name of my website repository is [mfcollins3.github.com](https://github.com/mfcollins3/mfcollins3.github.com). GitHub Pages also has support for generating a static website using a static website generator named [Jekyll](https://jekyllrb.com). I used Jekyll for an earlier version of my website, but now I use another static site generator named [Hugo](https://gohugo.io). My Hugo source code is stored in [another repository](https://github.com/mfcollins3/website) and I use [GitHub Actions](https://github.com/features/actions) to build my website and then copy the static website files to my other repository. This is how my website works.

My end goal with Naked Blogging is that I want to be able to create and publish new content for my website from my iPad Pro, or even my iPhone if I want to share a random picture or microblog. The first step to being able to do that is that I need to be able to access and change the content in my source Git repository for my website. I need to be able to clone my website repository from GitHub and store the clone on my iPad Pro device so that I can make changes, commit my changes, and publish my changes back to GitHub. In my [last post]({{< ref build-libgit2-for-ios >}}), I built an XCFramework containing [libgit2](https://libgit2.org). Now I will use it to clone my repository.

## Error Handling

libgit2 is a C library. Most libgit2 C functions return an integer value that indicate if an error occurred. If the result is 0, then a function can be assumed to have succeeded. If the result is a negative integer, then that is an indication that an error occurred. I'm going to tackle error reporting first so that I have a strategy for reporting and dealing with errors other than crashing my application as I build out the features.

In Swift, errors are represented using the [`Error`](https://developer.apple.com/documentation/swift/error#) protocol, and implementations of `Error` can be thrown and caught as exceptions like in other programming languages. The first thing that I will build is a bridge to map Git errors into errors that my Swift application can handle and set up the use of the Swift exception handling model for dealing with errors in my source code.

When an error occurs in a libgit2 function, libgit2 will store the error information in thread local storage. The error can then be retrieved using the [`git_error_last`](https://libgit2.org/libgit2/#HEAD/group/error/git_error_last) function. Thread local storage is used to store the error because libgit2 may be called from multiple threads concurrently and the error that is returned should be relevant to the calling thread.

A Git error has two values:

* `klass`: An error class value to indicate the kind of error that occurred or Git subsystem where the error occurred
* `message`: A human-readable description of the error

The first thing that I will do is map the error class into a new `GitErrorClass` enumeration type:

{{< highlight swift "linenos=table" >}}
enum GitErrorClass {
  case callback
  case checkout
  case cherrypick
  case config
  case describe
  case fetchHead
  case fileSystem
  case filter
  case http
  case index
  case indexer
  case invalid
  case `internal`
  case merge
  case net
  case noMemory
  case none
  case object
  case odb
  case os
  case patch
  case rebase
  case reference
  case regex
  case repository
  case revert
  case sha1
  case ssh
  case ssl
  case stash
  case submodule
  case tag
  case thread
  case tree
  case worktree
  case zlib

  init(errorClass: Int32) {
    switch git_error_t(UInt32(errorClass)) {
    case GIT_ERROR_NONE: self = .none
    case GIT_ERROR_NOMEMORY: self = .noMemory
    case GIT_ERROR_OS: self = .os
    case GIT_ERROR_INVALID: self = .invalid
    case GIT_ERROR_REFERENCE: self = .reference
    case GIT_ERROR_ZLIB: self = .zlib
    case GIT_ERROR_REPOSITORY: self = .repository
    case GIT_ERROR_CONFIG: self = .config
    case GIT_ERROR_REGEX: self = .regex
    case GIT_ERROR_ODB: self = .odb
    case GIT_ERROR_INDEX: self = .index
    case GIT_ERROR_OBJECT: self = .object
    case GIT_ERROR_NET: self = .net
    case GIT_ERROR_TAG: self = .tag
    case GIT_ERROR_TREE: self = .tree
    case GIT_ERROR_INDEXER: self = .indexer
    case GIT_ERROR_SSL: self = .ssl
    case GIT_ERROR_SUBMODULE: self = .submodule
    case GIT_ERROR_THREAD: self = .thread
    case GIT_ERROR_STASH: self = .stash
    case GIT_ERROR_CHECKOUT: self = .checkout
    case GIT_ERROR_FETCHHEAD: self = .fetchHead
    case GIT_ERROR_MERGE: self = .merge
    case GIT_ERROR_SSH: self = .ssh
    case GIT_ERROR_FILTER: self = .filter
    case GIT_ERROR_REVERT: self = .revert
    case GIT_ERROR_CALLBACK: self = .callback
    case GIT_ERROR_CHERRYPICK: self = .cherrypick
    case GIT_ERROR_DESCRIBE: self = .describe
    case GIT_ERROR_REBASE: self = .rebase
    case GIT_ERROR_FILESYSTEM: self = .fileSystem
    case GIT_ERROR_PATCH: self = .patch
    case GIT_ERROR_WORKTREE: self = .worktree
    case GIT_ERROR_SHA1: self = .sha1
    case GIT_ERROR_HTTP: self = .http
    case GIT_ERROR_INTERNAL: self = .internal
    default: fatalError("Unknown Git error class: \(errorClass)")
    }
  }
}
{{< /highlight >}}

You may notice the call to `fatalError` if a new error class gets introduced. I'm not sure what I can possibly do in that case, so I'm going to err on the side of failing instead of trying to handle a new error class that I know nothing about.

I can now create a `GitError` structure type that represents an error from the libgit2 library:

{{< highlight swift "linenos=table" >}}
struct GitError: Error {
  let errorClass: GitErrorClass
  let localizedDescription: String

  static func last() -> GitError? {
    guard let error = git_error_last() else {
      return nil
    }

    let errorClass = GitErrorClass(errorClass: error.pointee.klass)
    let message = String(cString: error.pointee.message)
    return GitError(errorClass: errorClass, localizedDescription: message)
  }
}
{{< /highlight >}}

I can now respond to and handle errors from libgit2 within my Naked Blogging application.

## A Simple Clone

libgit2 provides several options for how to clone the repository. I am going to start by doing a very simple clone to demonstrate that my libgit2 XCFramework works correctly in the simulator and on my device and that I can clone a Git repository. This isn't the implementation that I will use in my application, but this is simply a proof of concept.

In order to clone a repository, I need to provide libgit2 with two values:

1. The URL of the original repository that I want to clone.
2. The path of the directory on disk (or in the device's file system in this case) where the cloned repository should be stored.

Before I begin, there's a little wrinkle here. libgit2 accepts a path or a URL for the original repository, but accepts a file path for the directory where the repository will be cloned to. These values are both strings. In Swift, we use the [`FileManager`](https://developer.apple.com/documentation/foundation/filemanager#) class to find directories and manipulate the file system, and `FileManager` uses URLs for file paths. I want my Swift wrapper to feel iOS-like and *Swifty*, so I will implement translation between URLs and path strings in my wrapper. Naked Blogging will use URLs for file locations, but my `GitRepository` class will do the translation so all file locations will be converted to or from path strings.

Here is the first version of my `GitRepository` class that has a simple `clone(_:to:)` function:

{{< highlight swift "linenos=table" >}}
final class GitRepository {
  private let repository: OpaquePointer

  init(repository: OpaquePointer) {
    self.repository = repository
  }

  deinit {
    git_repository_free(repository)
  }

  static func clone(
    _ repositoryURL: URL,
    to directoryURL: URL
  ) throws -> GitRepository {
    var pointer: OpaquePointer?
    try succeed {
      git_clone(&pointer, repositoryURL.absoluteString, directoryURL.path, nil)
    }

    guard let repository = pointer else {
      fatalError("The repository is nil")
    }

    return GitRepository(repository: repository)
  }
}
{{< /highlight >}}

{{< admonition type="warning" title="Watch Your Memory" open="true" >}}
I mentioned this in the previous post, but I will mention it again. Because libgit2 is C, we get back pointers to objects and data structures for objects like repositories. These objects are in heap and aren't subject to Swift's ARC memory management, so I need to ensure that the memory is freed when no longer needed. In Swift, pointers are represented using the [`OpaquePointer`](https://developer.apple.com/documentation/swift/opaquepointer#) type. I store this value in my wrapper class. To ensure that the memory for the repository is freed when no longer needed, I implemented a [deinitializer](https://docs.swift.org/swift-book/LanguageGuide/Deinitialization.html) that will be called when a `GitRepository` object goes out of scope and can be deallocated by ARC. When the deinitializer is called, `GitRepository` will invoke `git_repository_free` to free the unmanaged memory for the repository.
{{< /admonition >}}

I can wire this up in my application's SwiftUI view to try it out. I will replace the standard `ContentView` with a new view that has a button to trigger cloning my website repository:

{{< highlight swift "linenos=table" >}}
struct ContentView: View {
  var body: some View {
    Button("Clone Repository") {
      do {
        let fileManager = FileManager.default
        let documentDirectoryURL = try fileManager.url(
          for: .documentDirectory,
          in: .userDomainMask,
          appropriateFor: nil,
          create: true
        )
        let directoryURL =
          documentDirectoryURL.appendingPathComponent("repo", isDirectory: true)
        if fileManager.fileExists(atPath: directoryURL.path, isDirectory: nil) {
          try fileManager.removeItem(at: directoryURL)
        }

        let repositoryURL =
          URL(string: "https://github.com/mfcollins3/website.git")!
        let repository =
          try GitRepository.clone(repositoryURL, to: directoryURL)
      } catch {
        print("ERROR: \(error)")
      }
    }
  }
}
{{< /highlight >}}

When I run my application, I see the **Clone Repository** button. When I tap it, libgit2 successfully cloned my repository to Naked Blogging's Documents directory in a `repo` subdirectory. Everything works!

{{< admonition type="note" title="Verifying Deinitializion" open="true" >}}
If you are using my code in your own application, you can set a breakpoint on the `GitRepository.deinit` deinitializer. Immediately after the `do` block completes in the `Button` action handler, you should see the breakpoint in the deinitializer get hit showing you that `repository` fell out of scope and ARC released the `GitRepository` object.
{{< /admonition >}}

## Reporting Progress

In the previous code sample, the call to `git_clone` happened synchronously on the application main thread. It also did not happen immediately. While my website repository is not huge, it's big enough that it takes a second or two to download all of the data and build the local copy of the Git repository. When I clone the repository from the command line using the Git CLI, I see the following output in my terminal:

    Cloning into 'website2'...
    remote: Enumerating objects: 229, done.
    remote: Counting objects: 100% (229/229), done.
    remote: Compressing objects: 100% (149/149), done.
    remote: Total 229 (delta 67), reused 194 (delta 46), pack-reused 0
    Receiving objects: 100% (229/229), 29.44 MiB | 17.30 MiB/s, done.
    Resolving deltas: 100% (67/67), done.

It would be great if I could get similar information from libgit2 so that I could display progress information such as a progress bar or summary text so that my user can see some kind of activity if they are dealing with a larger repository. Fortunately `git_clone` does support that through optional callbacks that can be provided in the final parameter that I skipped last time.

In the libgit2 [Clone (Progress)](https://libgit2.org/docs/guides/101-samples/#repositories_clone_progress) sample, it shows that we can receive information while the changes from the remote repository are being fetched, and a second callback that provides information as the working directory is being built by checking out the default branch. I will tap into both of those to see what kind of information gets returned:

{{< highlight swift "linenos=table" >}}
static func clone(
  _ repositoryURL: URL,
  to directoryURL: URL
) throws -> GitRepository {
  var cloneOptions = git_clone_options()
  try succeed {
    git_clone_options_init(&cloneOptions, UInt32(GIT_CLONE_OPTIONS_VERSION))
  }
  cloneOptions.checkout_opts.checkout_strategy = GIT_CHECKOUT_SAFE.rawValue
  cloneOptions.checkout_opts.progress_cb = { (cPath, completedSteps, totalSteps, _) in
    var path = "NOPATH"
    if let pathStr = cPath {
      path = String(cString: pathStr)
    }

    print("CHECKOUT PROGRESS: \(path) \(completedSteps) \(totalSteps)")
  }
  cloneOptions.fetch_opts.callback.transfer_progress = { (progress, _) in
    guard let progress = progress else {
      return 0
    }

    let indexedDeltas = progress.pointee.indexed_deltas
    let indexedObjects = progress.pointee.indexed_objects
    let localObjects = progress.pointee.local_objects
    let receivedBytes = progress.pointee.received_bytes
    let receivedObjects = progress.pointee.received_objects
    let totalDeltas = progress.pointee.total_deltas
    let totalObjects = progress.pointee.total_objects
    print("TRANSFER PROGRESS: \(localObjects) \(receivedBytes) \(receivedObjects) \(indexedDeltas) \(indexedObjects) \(totalDeltas) \(totalObjects)")

    return 0
  }

  var pointer: OpaquePointer?
  try succeed {
    git_clone(
      &pointer,
      repositoryURL.absoluteString,
      directoryURL.path,
      &cloneOptions
    )
  }

  guard let repository = pointer else {
    fatalError("The repository is nil")
  }

  return GitRepository(repository: repository)
}
{{< /highlight >}}

I see the following output in my debug log:

{{< highlight plain "linenos=table" >}}
TRANSFER PROGRESS: 0 15738 0 0 0 0 229
TRANSFER PROGRESS: 0 15738 1 0 1 0 229
TRANSFER PROGRESS: 0 15738 2 0 2 0 229
TRANSFER PROGRESS: 0 15738 3 0 3 0 229
TRANSFER PROGRESS: 0 15738 4 0 4 0 229
TRANSFER PROGRESS: 0 15738 5 0 5 0 229
TRANSFER PROGRESS: 0 15738 6 0 6 0 229
TRANSFER PROGRESS: 0 15738 7 0 7 0 229
TRANSFER PROGRESS: 0 15738 8 0 8 0 229
TRANSFER PROGRESS: 0 15738 9 0 9 0 229
.
.
.
TRANSFER PROGRESS: 0 30892749 228 0 162 0 229
TRANSFER PROGRESS: 0 30892749 229 0 162 0 229
TRANSFER PROGRESS: 0 30892749 229 0 162 0 229
TRANSFER PROGRESS: 0 30892749 229 1 163 67 229
TRANSFER PROGRESS: 0 30892749 229 2 164 67 229
TRANSFER PROGRESS: 0 30892749 229 3 165 67 229
TRANSFER PROGRESS: 0 30892749 229 4 166 67 229
TRANSFER PROGRESS: 0 30892749 229 5 167 67 229
TRANSFER PROGRESS: 0 30892749 229 6 168 67 229
TRANSFER PROGRESS: 0 30892749 229 7 169 67 229
TRANSFER PROGRESS: 0 30892749 229 8 170 67 229
TRANSFER PROGRESS: 0 30892749 229 9 171 67 229
TRANSFER PROGRESS: 0 30892749 229 10 172 67 229
.
.
.
TRANSFER PROGRESS: 0 30892749 229 66 228 67 229
TRANSFER PROGRESS: 0 30892749 229 67 229 67 229
CHECKOUT PROGRESS: NOPATH 0 56
CHECKOUT PROGRESS: .github/workflows/publish_website.yaml 1 56
CHECKOUT PROGRESS: .gitignore 2 56
CHECKOUT PROGRESS: .gitmodules 3 56
CHECKOUT PROGRESS: archetypes/default.md 4 56
CHECKOUT PROGRESS: config/_default/config.yaml 5 56
CHECKOUT PROGRESS: config/development/config.yaml 6 56
CHECKOUT PROGRESS: content/.gitkeep 7 56
CHECKOUT PROGRESS: content/posts/build-libgit2-for-ios/feature.jpg 8 56
CHECKOUT PROGRESS: content/posts/build-libgit2-for-ios/feature_small.jpg 9 56
CHECKOUT PROGRESS: content/posts/build-libgit2-for-ios/index.md 10 56
CHECKOUT PROGRESS: content/posts/creating-multiple-scenes-in-a-swiftui-app/close_second_scene.gif 11 56
CHECKOUT PROGRESS: content/posts/creating-multiple-scenes-in-a-swiftui-app/index.md 12 56
CHECKOUT PROGRESS: content/posts/creating-multiple-scenes-in-a-swiftui-app/multiple_scene_model.png 13 56
CHECKOUT PROGRESS: content/posts/creating-multiple-scenes-in-a-swiftui-app/multiple_scene_model_small.png 14 56
CHECKOUT PROGRESS: content/posts/creating-multiple-scenes-in-a-swiftui-app/multiple_scenes.png 15 56
CHECKOUT PROGRESS: content/posts/creating-multiple-scenes-in-a-swiftui-app/multiple_scenes_small.png 16 56
CHECKOUT PROGRESS: content/posts/creating-multiple-scenes-in-a-swiftui-app/new_application.png 17 56
CHECKOUT PROGRESS: content/posts/creating-multiple-scenes-in-a-swiftui-app/second_scene.png 18 56
CHECKOUT PROGRESS: content/posts/creating-multiple-scenes-in-a-swiftui-app/second_scene_small.png 19 56
CHECKOUT PROGRESS: content/posts/creating-multiple-scenes-in-a-swiftui-app/show_all_windows.jpeg 20 56
CHECKOUT PROGRESS: content/posts/creating-multiple-scenes-in-a-swiftui-app/show_all_windows_small.png 21 56
CHECKOUT PROGRESS: content/posts/creating-multiple-scenes-in-a-swiftui-app/show_multiple_scenes.png 22 56
CHECKOUT PROGRESS: content/posts/creating-multiple-scenes-in-a-swiftui-app/show_multiple_scenes_small.png 23 56
CHECKOUT PROGRESS: content/posts/creating-multiple-scenes-in-a-swiftui-app/simple_application_model.png 24 56
CHECKOUT PROGRESS: content/posts/creating-multiple-scenes-in-a-swiftui-app/single_scene.png 25 56
CHECKOUT PROGRESS: content/posts/creating-multiple-scenes-in-a-swiftui-app/single_scene_small.png 26 56
CHECKOUT PROGRESS: content/posts/deliver-naked-blogging/Icon.png 27 56
CHECKOUT PROGRESS: content/posts/deliver-naked-blogging/app_center_install.png 28 56
CHECKOUT PROGRESS: content/posts/deliver-naked-blogging/code_coverage_report.png 29 56
CHECKOUT PROGRESS: content/posts/deliver-naked-blogging/code_signing_settings.png 30 56
CHECKOUT PROGRESS: content/posts/deliver-naked-blogging/feature.jpg 31 56
CHECKOUT PROGRESS: content/posts/deliver-naked-blogging/feature_small.jpg 32 56
CHECKOUT PROGRESS: content/posts/deliver-naked-blogging/framed.png 33 56
CHECKOUT PROGRESS: content/posts/deliver-naked-blogging/index.md 34 56
CHECKOUT PROGRESS: content/posts/deliver-naked-blogging/signing_and_capabilities.png 35 56
CHECKOUT PROGRESS: content/posts/home-screen-quick-actions-with-swiftui/home_screen_quick_action.png 36 56
CHECKOUT PROGRESS: content/posts/home-screen-quick-actions-with-swiftui/home_screen_quick_action_small.png 37 56
CHECKOUT PROGRESS: content/posts/home-screen-quick-actions-with-swiftui/index.md 38 56
CHECKOUT PROGRESS: content/posts/home-screen-quick-actions-with-swiftui/preview_image.png 39 56
CHECKOUT PROGRESS: content/posts/home-screen-quick-actions-with-swiftui/quick_action_demo.gif 40 56
CHECKOUT PROGRESS: content/posts/hotp-and-totp/feature.jpg 41 56
CHECKOUT PROGRESS: content/posts/hotp-and-totp/feature_small.jpg 42 56
CHECKOUT PROGRESS: content/posts/hotp-and-totp/index.md 43 56
CHECKOUT PROGRESS: content/posts/introducing-naked-blogging/feature.jpg 44 56
CHECKOUT PROGRESS: content/posts/introducing-naked-blogging/feature_small.jpg 45 56
CHECKOUT PROGRESS: content/posts/introducing-naked-blogging/index.md 46 56
CHECKOUT PROGRESS: content/posts/welcome-back-to-the-imaginary-road/cover.jpg 47 56
CHECKOUT PROGRESS: content/posts/welcome-back-to-the-imaginary-road/index.md 48 56
CHECKOUT PROGRESS: data/.gitkeep 49 56
CHECKOUT PROGRESS: data/authors/michael-collins.json 50 56
CHECKOUT PROGRESS: layouts/.gitkeep 51 56
CHECKOUT PROGRESS: layouts/_default/baseof.html 52 56
CHECKOUT PROGRESS: layouts/partials/head/cookie_solution.html 53 56
CHECKOUT PROGRESS: layouts/shortcodes/rawhtml.html 54 56
CHECKOUT PROGRESS: static/CNAME 55 56
CHECKOUT PROGRESS: themes/uBlogger 56 56
{{< /highlight >}}

When comparing this output to that from the command line, I can see that the checkout progress isn't reported. The transfer progress matches the information shown on the last two lines of the output. First, it shows the progress as the objects are downloaded from the remote repository. Next, it shows the deltas being resolved and applied to the new repository. We don't see any of the information from the remote repository though.

Looking at the `git_remote_callbacks` C structure, there are additional callbacks available. I see a `sideband_progress` field that indicates it is called with textual progress from the remote. I will handle that callback to see what it returns:

{{< highlight swift "linenos=table,hl_lines=34-44" >}}
static func clone(
  _ repositoryURL: URL,
  to directoryURL: URL
) throws -> GitRepository {
  var cloneOptions = git_clone_options()
  try succeed {
    git_clone_options_init(&cloneOptions, UInt32(GIT_CLONE_OPTIONS_VERSION))
  }
  cloneOptions.checkout_opts.checkout_strategy = GIT_CHECKOUT_SAFE.rawValue
  cloneOptions.checkout_opts.progress_cb = { (cPath, completedSteps, totalSteps, _) in
    var path = "NOPATH"
    if let pathStr = cPath {
      path = String(cString: pathStr)
    }

    print("CHECKOUT PROGRESS: \(path) \(completedSteps) \(totalSteps)")
  }
  cloneOptions.fetch_opts.callback.transfer_progress = { (progress, _) in
    guard let progress = progress else {
      return 0
    }

    let indexedDeltas = progress.pointee.indexed_deltas
    let indexedObjects = progress.pointee.indexed_objects
    let localObjects = progress.pointee.local_objects
    let receivedBytes = progress.pointee.received_bytes
    let receivedObjects = progress.pointee.received_objects
    let totalDeltas = progress.pointee.total_deltas
    let totalObjects = progress.pointee.total_objects
    print("TRANSFER PROGRESS: \(localObjects) \(receivedBytes) \(receivedObjects) \(indexedDeltas) \(indexedObjects) \(totalDeltas) \(totalObjects)")

    return 0
  }
  cloneOptions.fetch_ops.callbacks.sideband_progress = { (str, len, _) in
    guard let str = str else {
      return 0
    }

    let data = Data(bytes: UnsafeRawPointer(str), count: Int(len))
    let message = String(data: data, encoding.utf8)!
    print("SIDEBAND: \(message)")

    return 0
  }

  var pointer: OpaquePointer?
  try succeed {
    git_clone(
      &pointer,
      repositoryURL.absoluteString,
      directoryURL.path,
      &cloneOptions
    )
  }

  guard let repository = pointer else {
    fatalError("The repository is nil")
  }

  return GitRepository(repository: repository)
}
{{< /highlight >}}

Adding the sideband callback, I get the sideband output before and of the transfer progress events occur:

{{< highlight plain "linenos=table" >}}
SIDEBAND: Enumerating objects: 229, done.

SIDEBAND: Counting objects:   0% (1/229)
SIDEBAND: Counting objects:   1% (3/229)
SIDEBAND: Counting objects:   2% (5/229)
SIDEBAND: Counting objects:   3% (7/229)
SIDEBAND: Counting objects:   4% (10/229)
Counting objects:   5% (12/229)
SIDEBAND: Counting objects:   6% (14/229)
Counting objects:   7% (17/229)
Counting objects:   8% (19/229)
SIDEBAND: Counting objects:   9% (21/229)
Counting objects:  10% (23/229)
SIDEBAND: Counting objects:  11% (26/229)
Counting objects:  12% (28/229)
SIDEBAND: Counting objects:  13% (30/229)
Counting objects:  14% (33/229)
Counting objects:  15% (35/229)
Counting objects:  16% (37/229)
SIDEBAND: Counting objects:  17% (39/229)
SIDEBAND: Counting objects:  18% (42/229)
SIDEBAND: Counting objects:  19% (44/229)
Counting objects:  20% (46/229)
Counting objects:  21% (49/229)
Counting objects:  22% (51/229)
SIDEBAND: Counting objects:  23% (53/229)
Counting objects:  24% (55/229)
Counting objects:  25% (58/229)
Counting objects:  26% (60/229)
SIDEBAND: Counting objects:  27% (62/229)
Counting objects:  28% (65/229)
SIDEBAND: Counting objects:  29% (67/229)
Counting objects:  30% (69/229)
Counting objects:  31% (71/229)
Counting objects:  32% (74/229)
SIDEBAND: Counting objects:  33% (76/229)
Counting objects:  34% (78/229)
Counting objects:  35% (81/229)
Counting objects:  36% (83/229)
SIDEBAND: Counting objects:  37% (85/229)
Counting objects:  38% (88/229)
SIDEBAND: Counting objects:  39% (90/229)
SIDEBAND: Counting objects:  40% (92/229)
SIDEBAND: Counting objects:  41% (94/229)
Counting objects:  42% (97/229)
Counting objects:  43% (99/229)
Counting objects:  44% (101/229)
SIDEBAND: 
Counting objects:  45% (104/229)
Counting objects:  46% (106/229)
SIDEBAND: Counting objects:  47% (108/229)
SIDEBAND: Counting objects:  48% (110/229)
Counting objects:  49% (113/229)
SIDEBAND: Counting objects:  50% (115/229)
Counting objects:  51% (117/229)
Counting objects:  52% (120/229)
Counting objects:  53% (122/2
SIDEBAND: 29)
Counting objects:  54% (124/229)
Counting objects:  55% (126/229)
SIDEBAND: Counting objects:  56% (129/229)
SIDEBAND: Counting objects:  57% (131/229)
SIDEBAND: Counting objects:  58% (133/229)
Counting objects:  59% (136/229)
Counting objects:  60% (138/229)
Counting objects:  61% (140/2
SIDEBAND: 29)
Counting objects:  62% (142/229)
Counting objects:  63% (145/229)
SIDEBAND: Counting objects:  64% (147/229)
SIDEBAND: Counting objects:  65% (149/229)
SIDEBAND: Counting objects:  66% (152/229)
Counting objects:  67% (154/229)
SIDEBAND: Counting objects:  68% (156/229)
SIDEBAND: Counting objects:  69% (159/229)
Counting objects:  70% (161/229)
SIDEBAND: Counting objects:  71% (163/229)
Counting objects:  72% (165/229)
Counting objects:  73% (168/229)
SIDEBAND: Counting objects:  74% (170/229)
SIDEBAND: Counting objects:  75% (172/229)
SIDEBAND: Counting objects:  76% (175/229)
Counting objects:  77% (177/229)
Counting objects:  78% (179/229)
Counting objects:  79% (181/2
SIDEBAND: 29)
Counting objects:  80% (184/229)
Counting objects:  81% (186/229)
Counting objects:  82% (188/229)
Counting objects:  83% (1
SIDEBAND: 91/229)
Counting objects:  84% (193/229)
Counting objects:  85% (195/229)
Counting objects:  86% (197/229)
Counting objects:  87
SIDEBAND: % (200/229)
Counting objects:  88% (202/229)
Counting objects:  89% (204/229)
Counting objects:  90% (207/229)
Counting objects:
SIDEBAND:   91% (209/229)
Counting objects:  92% (211/229)
Counting objects:  93% (213/229)
Counting objects:  94% (216/229)
Counting obje
SIDEBAND: cts:  95% (218/229)
Counting objects:  96% (220/229)
SIDEBAND: Counting objects:  97% (223/229)
Counting objects:  98% (225/229)
SIDEBAND: Counting objects:  99% (227/229)
SIDEBAND: Counting objects: 100% (229/229)
Counting objects: 100% (229/229), done.

SIDEBAND: Compressing objects:   0% (1/149)
Compressing objects:   1% (2/149)
SIDEBAND: Compressing objects:   2% (3/149)
SIDEBAND: Compressing objects:   3% (5/149)
SIDEBAND: Compressing objects:   4% (6/149)
SIDEBAND: Compressing objects:   5% (8/149)
SIDEBAND: Compressing objects:   6% (9/149)
SIDEBAND: Compressing objects:   7% (11/149)
SIDEBAND: Compressing objects:   8% (12/149)
Compressing objects:   9% (14/149)
SIDEBAND: Compressing objects:  10% (15/149)
SIDEBAND: Compressing objects:  11% (17/149)
Compressing objects:  12% (18/149)
SIDEBAND: Compressing objects:  13% (20/149)
SIDEBAND: Compressing objects:  14% (21/149)
SIDEBAND: Compressing objects:  15% (23/149)
SIDEBAND: Compressing objects:  16% (24/149)
SIDEBAND: Compressing objects:  17% (26/149)
SIDEBAND: Compressing objects:  18% (27/149)
SIDEBAND: Compressing objects:  19% (29/149)
SIDEBAND: Compressing objects:  20% (30/149)
SIDEBAND: Compressing objects:  21% (32/149)
SIDEBAND: Compressing objects:  22% (33/149)
SIDEBAND: Compressing objects:  23% (35/149)
SIDEBAND: Compressing objects:  24% (36/149)
SIDEBAND: Compressing objects:  25% (38/149)
SIDEBAND: Compressing objects:  26% (39/149)
SIDEBAND: Compressing objects:  27% (41/149)
SIDEBAND: Compressing objects:  28% (42/149)
SIDEBAND: Compressing objects:  29% (44/149)
Compressing objects:  30% (45/149)
SIDEBAND: Compressing objects:  31% (47/149)
SIDEBAND: Compressing objects:  32% (48/149)
SIDEBAND: Compressing objects:  33% (50/149)
SIDEBAND: Compressing objects:  34% (51/149)
SIDEBAND: Compressing objects:  35% (53/149)
SIDEBAND: Compressing objects:  36% (54/149)
SIDEBAND: Compressing objects:  37% (56/149)
SIDEBAND: Compressing objects:  38% (57/149)
SIDEBAND: Compressing objects:  39% (59/149)
SIDEBAND: Compressing objects:  40% (60/149)
Compressing objects:  41% (62/149)
Compressing objects:  42% (63/149)
Compressing objects:  4
SIDEBAND: 3% (65/149)
Compressing objects:  44% (66/149)
Compressing objects:  45% (68/149)
Compressing objects:  46% (69/149)
Compressing
SIDEBAND:  objects:  47% (71/149)
Compressing objects:  48% (72/149)
Compressing objects:  49% (74/149)
Compressing objects:  50% (75/149)
SIDEBAND: 
Compressing objects:  51% (76/149)
Compressing objects:  52% (78/149)
Compressing objects:  53% (79/149)
Compressing objects:  
SIDEBAND: 54% (81/149)
Compressing objects:  55% (82/149)
Compressing objects:  56% (84/149)
Compressing objects:  57% (85/149)
Compressin
SIDEBAND: g objects:  58% (87/149)
Compressing objects:  59% (88/149)
Compressing objects:  60% (90/149)
Compressing objects:  61% (91/149
SIDEBAND: )
Compressing objects:  62% (93/149)
Compressing objects:  63% (94/149)
SIDEBAND: Compressing objects:  64% (96/149)
Compressing objects:  65% (97/149)
SIDEBAND: Compressing objects:  66% (99/149)
Compressing objects:  67% (100/149)
Compressing objects:  68% (102/149)
SIDEBAND: Compressing objects:  69% (103/149)
Compressing objects:  70% (105/149)
Compressing objects:  71% (106/149)
SIDEBAND: Compressing objects:  72% (108/149)
Compressing objects:  73% (109/149)
SIDEBAND: Compressing objects:  74% (111/149)
Compressing objects:  75% (112/149)
SIDEBAND: Compressing objects:  76% (114/149)
Compressing objects:  77% (115/149)
SIDEBAND: Compressing objects:  78% (117/149)
SIDEBAND: Compressing objects:  79% (118/149)
SIDEBAND: Compressing objects:  80% (120/149)
SIDEBAND: Compressing objects:  81% (121/149)
SIDEBAND: Compressing objects:  82% (123/149)
SIDEBAND: Compressing objects:  83% (124/149)
SIDEBAND: Compressing objects:  84% (126/149)
SIDEBAND: Compressing objects:  85% (127/149)
SIDEBAND: Compressing objects:  86% (129/149)
SIDEBAND: Compressing objects:  87% (130/149)
SIDEBAND: Compressing objects:  88% (132/149)
SIDEBAND: Compressing objects:  89% (133/149)
SIDEBAND: Compressing objects:  90% (135/149)
SIDEBAND: Compressing objects:  91% (136/149)
SIDEBAND: Compressing objects:  92% (138/149)
SIDEBAND: Compressing objects:  93% (139/149)
SIDEBAND: Compressing objects:  94% (141/149)
SIDEBAND: Compressing objects:  95% (142/149)
SIDEBAND: Compressing objects:  96% (144/149)
SIDEBAND: Compressing objects:  97% (145/149)
SIDEBAND: Compressing objects:  98% (147/149)
SIDEBAND: Compressing objects:  99% (148/149)
SIDEBAND: Compressing objects: 100% (149/149)
SIDEBAND: Compressing objects: 100% (149/149), done.

SIDEBAND: Total 229 (delta 67), reused 194 (delta 46), pack-reused 0

{{< /highlight >}}

Notice in the output that the sideband text is one continuous stream and not individual messages. Newline characters are the indicator of the end of the previous record and the start of the next record. It also appers that a blank line indicates the start of a new progress report. First, it enumerates the objects, then sends a blank line. Next, the server counts the objects and sends a blank line. Finally, the server compresses the objects and sends a blank line.

Given what I know now, the basic clone flow will be:

{{< mermaid >}}
graph TD;
    A[Enumerating] --> B[Counting]
    B --> C[Compressing]
    C --> T[Total]
    T --> D[Downloading]
    D --> E[Indexing Objects]
    E --> F[Indexing Deltas]
    F --> G[Checkout]
{{< /mermaid >}}

There's another issue that I haven't talked about yet. The `git_clone` function runs in the current thread. So when I run it now, it is blocking the main thread. My repository isn't huge. It's a little over 30 megabytes, but it would be a bad idea to block the main thread and the UI. With the callbacks, there would be no way of presenting progress information to show the user that something is happening, and the application would appear to stop. What I want to do is to move `git_clone` to a background thread, but to be able to send progress information and completion notification to the main thread. This would be a perfect time to introduce [Combine](https://developer.apple.com/documentation/Combine).

## Introducing Combine

I want to move `git_clone` to a background thread and provide notifications from the callbacks to the UI thread to present progress information to the user during the clone. This is a perfect scenario for using [Combine](https://developer.apple.com/documentation/Combine) to do it. Instead of calling `GitRepository.clone(_:to:)`, I can expose a `GitRepository.clonePublisher(_:to:)` function that returns a Combine `Publisher` that the main thread can subscribe to. I can then tell Combine that the subscription should run on a background thread while events should be sent to the main UI thread. And the UI can be notified when the clone operation is complete.

To do this, I will need to begin by creating a custom publisher using Combine. I won't go into a lot of detail behind the theory of Combine and publishers and subscribers. I will basically tell you that I am going to create a publisher that my application can subscribe to. When my application subscribes to the publisher, the publisher will start a subscription that will clone the remote repository and will send the events from the callbacks to the subscriber to handle. When the clone operation completes successfully, or if it fails, then my application will also be notified of that fact. To learn more about Combine, I recommend [Combine: Asynchronous Programming with Swift](https://www.raywenderlich.com/books/combine-asynchronous-programming-with-swift/v2.0) published by [raywenderlich.com](https://raywenderlich.com).

I will start by creating types representing the events that will be streamed to the application during a Git repository cloning operation. For the messages from the remote server, I will return those just as single strings. I will create a `GitTransportProgress` struct and a `GitCheckoutProgress` struct to transmit the progress data during the clone operation. I will deliver all of these events using a `GitCloneEvent` enum and will pass the data as associated values to the cases:

{{< highlight swift "linenos=table" >}}
struct GitTransportProgress {
  let indexedDeltas: UInt32
  let indexedObjects: UInt32
  let localObjects: UInt32
  let receivedBytes: Int
  let receivedObjects: UInt32
  let totalDeltas: UInt32
  let totalObjects: UInt32
}

struct GitCheckoutProgress {
  let completedSteps: Int
  let totalSteps: Int
  let path: String?
}

enum GitCloneEvent {
  case checkoutProgress(GitCheckoutProgress)
  case remoteMessage(String)
  case repositoryReady(GitRepository)
  case transportProgress(GitTransportProgress)
}
{{< /highlight >}}

My next step is to build the subscription. The subscription is what actually executes the clone operation. In Combine, calling `GitRepository.clone` will return a publisher, but that does not actually start the clone operation. The clone will not actually happen until something subscribes to the publisher to receive events. So it's the subscription that does the clone.

Now for a little wrinkle in Swift development. As you're well aware, libgit2 is a C library and we've been passing closures to the C callbacks to get the progress information. There are two issues with Swift that we need to deal with:

1. Types using generics cannot pass closures to C libraries
2. Closures passed to C libraries cannot access `self`

Fortunately, there's a way around both of these. The Combine subscription will have to use generics, so I will move the clone functionality out into a different class. To deal with the lack of `self`, libgit2 allows me to [pass an opaque pointer](https://stackoverflow.com/questions/33260808/how-to-use-instance-method-as-callback-for-function-which-takes-only-func-or-lit) that gets passed to my callbacks, so I can pass the reference to `self` that way. Here's the `CloneRepositoryWrapper` class that I wrote to actually run `git_clone` with callbacks:

{{< highlight swift "linenos=table" >}}
private final class CloneRepositoryWrapper {
  private let checkoutProgressCallback: (GitCheckoutProgress) -> Void
  private let remoteMessageCallback: (String) -> Bool
  private let transportProgressCallback: (GitTransportProgress) -> Bool

  private var remoteMessagePrefix = ""

  init(
    remoteMessageCallback: @escaping (String) -> Bool,
    transportProgressCallback: @escaping (GitTransportProgress) -> Bool,
    checkoutProgressCallback: @escaping (GitCheckoutProgress) -> Bool
  )

  func clone(
    _ repositoryURL: URL,
    to directoryURL: URL
  ) throws -> GitRepository {
    var cloneOptions = git_clone_options()
    try GitRepository.succeed {
      git_clone_options_init(&cloneOptions, UInt32(GIT_CLINE_OPTIONS_VERSION))
    }

    let ref = UnsafeMutableRawPointer(Unmanaged.passUnretained(self).toOpaque())

    cloneOptions.checkout_opts.checkout_strategy = GIT_CHECKOUT_SAFE.rawValue
    cloneOptions.checkout_opts.progress_payload = ref
    cloneOptions.checkout_opts.progress_cb = { (cPath, completedSteps, totalSteps, payload) in
      let wrapper = Unmanaged<CloneRepositoryWrapper>.fromOpaque(payload!).toUnretainedValue()

      var path: String? = nil
      if let pathStr = cPath {
        path = String(cString: pathStr)
      }

      let progress = GitCheckoutProgress(
        completedSteps: completedSteps,
        totalSteps: totalSteps,
        path: path
      )

      wrapper.checkoutProgressCallback(progress)
    }

    cloneOptions.fetch_opts.callbacks.payload = ref
    cloneOptions.fetch_opts.callbacks.transfer_progress = { (progress, payload) in
      guard let progress = progress else {
        return 0
      }

      let wrapper = Unmanaged<CloneRepositoryWrapper>.fromOpaque(payload!).takeUnretainedValue()

      let transportProgress = GitTransportProgress(
        indexedDeltas: progress.pointee.indexed_deltas,
        indexedObjects: progress.pointee.indexed_objects,
        localObjects: progress.pointee.local_objects,
        receivedBytes: progress.pointee.received_bytes,
        receivedObjects: progress.pointee.received_objects,
        totalDeltas: progress.pointee.total_deltas,
        totalObjects: progress.pointee.total_objects
      )
      let result = wrapper.transportProgressCallback(transportProgress)
      return result ? 0 : -1
    }

    cloneOptions.fetch_opts.callbacks.sideband_progress = { (str, len, payload) in
      guard let str = str else {
        return 0
      }

      let data = Data(bytes: UnsafeRawPointer(str), count: Int(len))
      let text = String(data: data, encoding: .utf8)
      let lines = text.split(
        omittingEmptySubsequences: false,
        whereSeparator: \.isNewline
      )
      let isLastmessageComplete = text.last?.isNewline ?? false

      let wrapper = Unmanaged<CloneRepositoryWrapper>.fromOpaque(payload!).takeUnreatinedValue()

      for line in lines[lines.startIndex..<lines.index(beofre: lines.endIndex)] {
        let message = wrapper.remoteMessagePrefix + line
        wrapper.remoteMessagePrefix = ""
        if !wrapper.remoteMessageCallback(message) {
          return -1
        }
      }

      if isLastMessageComplete, let line = lines.last {
        let message = wrapper.remoteMessagePrefix + line
        wrapper.remoteMessagePrefix = ""
        if !wrapper.remoteMessageCallback(message) {
          return -1
        }
      } else {
        wrapper.remoteMessagePrefix += (lines.last ?? "")
      }

      return 0
    }

    var pointer: OpaquePointer?
    try GitRepository.succeed {
      git_clone(
        &pointer,
        repositoryURL.absoluteString,
        directoryURL.path,
        &cloneOptions
      )
    }

    guard let repository = pointer else {
      fatalError("The repository is nil")
    }

    return GitRepository(repository: repository)
  }
}
{{< /highlight >}}

With the wrapper done and handling the callbacks, I can now use the wrapper to implement the subscription:

{{< highlight swift "linenos=table" >}}
private final class CloneSubscription<S: Subscriber>: Subscription
  where S.Input == GitCloneEvent,
        S.Failure == Error {
  private let directoryURL: URL
  private let repositoryURL: URL

  private var cancelled = false
  private var requested: Subscribers.Demand = .none
  private var subscriber: S?
  private var times: Suscribers.Demand = .unlimited

  init(subscriber: S, repositoryURL: URL, directoryURL: URL) {
    self.subscriber = subscriber
    self.repositoryURL = repositoryURL
    self.directoryURL = directoryURL
  }

  func cancel() {
    cancelled = true
    subscriber = nil
  }

  func request(_ demand: Subscribers.Demand) {
    guard times > .none else {
      subscriber?.receive(completion: .finished)
      return
    }

    requested += demand

    do {
      let wrapper = CloneRepositoryWrapper { message in
        giard !self.cancelled else {
          return false
        }

        _ = self.subscriber?.receive(.remoteMessage(message))

        return true
      } transportProgressCallback: { progress in
        guard !self.cancelled else {
          return false
        }

        _ = self.subscriber?.receive(.transportProgress(progress))

        return true
      } checkoutProgressCallback: { progress in
        guard !self.cancelled else {
          return
        }

        _ = self.subscriber?.receive(.checkoutProgress(progress))
      }

      let repository = try wrapper.clone(repositoryURL, to: directoryURL)
      _ = subscriber?.receive(.repositoryReader(repository))
      subscriber?.receive(completion: .finished)
    } catch {
      subscriber?.receive(completion: .failure(error))
    }
  }
}
{{< /highlight >}}

The publisher is pretty simple:

{{< highlight swift "linenos=table" >}}
struct ClonePublisher: Publisher {
  typealias Output = GitCloneEvent
  typealias Failure = Error

  private let repositoryURL: URL
  private let directoryURL: URL

  init(repositoryURL: URL, directoryURL: URL) {
    self.repositoryURL = repositoryURL
    self.directoryURL = directoryURL
  }

  func receive<S: Subscriber>(subscriber: S)
    where Failure == S.Failure, Output == S.Output {
    let subscription = CloneSubscription(
      subscriber: subscriber,
      repositoryURL: repositoryURL,
      directoryURL: directoryURL
    )
    subscriber.receive(subscription: subscription)
  }
}
{{< /highlight >}}

Finally, I can rewrite my `GitRepository.clone(_:to:)` function to return the publisher:

{{< highlight swift "linenos=table" >}}
static func clone(
  _ repositoryURL: URL,
  to directoryURL: URL
) -> ClonePublisher {
  return ClonePublisher(
    repositoryURL: repositoryURL,
    directoryURL: directoryURL
  )
}
{{< /highlight >}}

I can now update my `ContentView` to use the publisher:

{{< highlight swift "linenos=table,hl_lines=2 22-45" >}}
struct ContentView: View {
  @State private var subscription: AnyCancellable?

  var body: some View {
    Button("Clone Repository") {
      do {
        let fileManager = FileManager.default
        let documentDirectoryURL = try fileManager.url(
          for: .documentDirectory,
          in: .userDomainMask,
          appropriateFor: nil,
          create: true
        )
        let directoryURL =
          documentDirectoryURL.appendingPathComponent("repo", isDirectory: true)
        if fileManager.fileExists(atPath: directoryURL.path, isDirectory: nil) {
          try fileManager.removeItem(at: directoryURL)
        }

        let repositoryURL =
          URL(string: "https://github.com/mfcollins3/website.git")
        subscription = GitRepository.clone(repositoryURL, to: directoryURL)
          .sink { completion in
            if case .failure(let error) = completion {
              print("ERROR: \(error)")
            } else {
              print("COMPLETED")
            }

            self.subscription = nil
          } receiveValue: { event in
            switch event {
            case .checkoutProgress(let progress):
              print("CHECKOUT: \(progress.path ?? "NO-FILE") \(progress.completedSteps) \(progress.totalSteps)")

            case .remoteMessage(let message):
              print("REMOTE: \(message)")

            case .repositoryReady:
              print("Repository ready")

            case .transportProgress(let progress):
              print("TRANSPORT: \(progress.receivedBytes) \(progress.localObjects) \(progress.receivedObjects) \(progress.indexedObjects) \(progress.totalObjects) \(progress.indexedDeltas) \(progress.totalDeltas)")
            }
          }
      } catch {
          print("ERROR: \(error)")
      }
    }
  }
}
{{< /highlight >}}

When I run this however, the result is still the same. Even though I switched to using Combine to use the publisher/subscriber mechanism to send the stream of events to the application, the main thread still blocks because `git_commit` is still running on the main thread. I need to move the execution of `git_commit` to a background thread, and fortunately, that is very easy using Combine.

When using Combine, you can control where the subscription runs by using the `subscribe(on:)` operator on a publisher. `subscribe(on:)` takes a `Scheduler` that Combine will use to execute the subscription. Apple updated `DispatchQueue` to conform to `Scheduler`, so I can pass a background queue for performing the clone:

{{< highlight swift "linenos=table" >}}
subscription = GitRepository.clone(repositoryURL, to: directoryURL)
  .subscribe(on: DispatchQueue.global(qos: .userInitiated))
  .sink { ... }
{{< /highlight >}}

After adding `subscribe(on:)`, the call to `git_clone` will now happen in a background thread and my main thread will continue to be active. I do have one other problem though. My callbacks that I sent to the `sink(receiveCompletion:receiveValue:)` function also run on the same background thread as `git_clone`. Since I want to use these callbacks to update the UI, I really want them to run on the main UI thread. I could use `DispatchQueue.main.async` to pass code to execute on the main thread, but fortunately, Combine comes to the rescue again. Instead, I can use the `receive(on:)` operator to tell Combine that I want my sink to run on a specific `Scheduler` when it receives messages. I can pass `DispatchQueue.main` to `receive(on:)` to send my events to the main UI thread so that I can update a future progress bar:

{{< highlight swift "linenos=false" >}}
subscription = GetRepository.clone(repositoryURL, to: directoryURL)
  .subscribe(on: DispatchQueue.global(qos: .userInitiated))
  .receive(on: DispatchQueue.main)
  .sink { ... }
{{< /highlight >}}

Now when I run, `git_clone` runs on a background thread, my main UI thread remains responsive, and the clone progress events are sent to the main UI thread for handling and updating the UI.

## Where Am I Now?

My goal at the beginning of this post was to clone my website repository onto my device. I was able to figure out how to use the `git_clone` function to do this, as well as figure out how to obtain progress information for a future UI to show interactively the clone happening. I will work on the UI piece in a future post. I took `git_clone` a bit further by wrapping it into a publisher using Combine so that I can provide a very Swift-like experience for cloning a remote Git repository.

{{< rawhtml >}}
<span>Photo by <a href="https://unsplash.com/@michalhlavac?utm_source=unsplash&amp;utm_medium=referral&amp;utm_content=creditCopyText">Michal Hlaváč</a> on <a href="https://unsplash.com/s/photos/twins?utm_source=unsplash&amp;utm_medium=referral&amp;utm_content=creditCopyText">Unsplash</a></span>
{{< /rawhtml >}}