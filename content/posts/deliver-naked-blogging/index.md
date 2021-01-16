---
categories:
- Naked Blogging
- iOS Development
date: 2021-01-10T16:24:36-07:00
description: In this post, I talk about using Fastlane Tools to automate the process of building, testing, and shipping the Naked Blogging application to customers using the Apple App Store.
draft: true
resources:
- name: "featured-image"
  src: "feature.jpg"
- name: "featured-image-preview"
  src: "feature_small.jpg"
tags:
- App Store
- Fastlane Tools
- iOS
- Naked Blogging
- TestFlight
title: Automating Delivery of iOS Applications to Customers
---
The most important user story when developing a software product is planning for how to distribute it. A product isn't a product if it can't reach customers. In this post, I show you how to automate packaging and distributing an iOS application using Fastlane Tools.

<!--more-->

Michael's number 1 rule of product development is:

> Rule #1:
>
> It's not a product if it doesn't ship.

Shipping is important. A product becomes a product by having customers. A software product that is never used by users isn't a software product. Products need to go out. In the case of iOS applications, products need to get to the Apple App Store. The best way of building a software product is to set yourself up with a shipping mentality. The best way to build a shipping mentality is to build every feature with the intent to ship. Now you might not ship after every feature is delivered, but you should put yourself in a position where you can. The best way to put you in this position is to start off by building an automated pipeline that does all of the hard work of shipping for you. If you make shipping your product easy, then shipping the actual product will be easy.

For mobile applications, I like to use [Fastlane Tools](https://fastlane.tools) to automate the steps that I go through when shipping my apps to the App Store. In this post, I will set up the automated build pipeline (or most of it; I'll explain some limitations later) for my [Naked Blogging]({{< ref introducing-naked-blogging >}}) app. I will show you all the steps and will explain how I use them and how they work. Hopefully you find insights into things that you can automate for your own apps.

## Software Requirements

I typically start building applications on my own, but I expect that others will be joining me in the near future. In the hope that I will not have to stop and explain to every single new team member how to get up and running with the code, I usually create a `README.md` document in the root of my repository where I provide setup and build instructions for the project. I try to use package managers where I can to manage application dependencies, but sometimes I need developers to install programs on their computers and I use the `README.md` file to document those requirements. If you're following along, you can use this article as your `README.md`.

While it's obvious that we're going to need [Xcode](https://apps.apple.com/us/app/xcode/id497799835?mt=12) to build an iOS application, surprisingly, we'll also need Ruby. Fastlane Tools is built in Ruby, so I'm going to start by making sure that my team has Ruby installed. At the time of this writing, Ruby 3.0.0 was just released, but there seem to be compatibility issues between Ruby 3.0.0 and Fastlane Tools. I will be using Ruby 2.7.2 instead. For managing the installation and correct versions of Ruby on my projects, I use [rbenv](https://github.com/rbenv/rbenv). To install Ruby 2.7.2, I can run the following command in a terminal:

    rbenv install 2.7.2

After Ruby is installed, I can then configure my project to use 2.7.2:

    rbenv local 2.7.2

This command will create a `.ruby-version` file in the root directory of my project workspace (I call the directory where I store all of my source code and that hosts the `.git` repository directory as my project workspace).

Ruby has support for redistributing libraries and applications as reusable components known as **gems**. To track the versions that I am using and making sure that I can reinstall those versions later, I use a higher-level program named [Bundler](https://bundler.io). Bundler needs to be installed as a Ruby gem before moving forward:

    gem install bundler

With Ruby and Bundler installed, I can now install Fastlane Tools. Bundler uses a manifest file that lists the dependencies that my application needs. The manifest file is named `Gemfile` and I place it in the root directory of my repository.

{{< highlight ruby "linenos=table">}}
source 'https://rubygems.org'

gem 'fastlane'
{{< /highlight >}}

Next, I run Bundler to install Fastlane Tools locally:

    bundle install

And now I should have everything that I need to start automating my build process.

# Produce

Fastlane Tools is called Fastlane Tools for a reason: it's basically a bunch of individual tools that each perform a specific purpose and are distributed together as a set of tools for the purpose of building and distributing mobile applications. Think of Fastlane as a bag full of tools that you need to look in and pick out one tool at a time to learn how to use it. I will introduce you to your first tool right now: [Produce](https://docs.fastlane.tools/actions/produce/).

If you have never logged into the [Apple Developer website](https://developer.apple.com), you might be surprised to learn that there are actually two applications behind the login. First, the Apple Developer Portal is the place where developers can register application identifiers, register test devices, obtain code signing certificates and provisioning profiles for app distribution, and obtain keys for services such as push notifications. The other application is called App Store Connect and is where you go to register your product for release on the Apple App Store, manage beta testing, manage all of the metadata for your store page, manage the pricing strategies and in-app purchase items, and submit your application to the App Store.

The first thing that we have to automate with a new product is registering the product with both the Apple Developer Portal and App Store Connect. Registering the product with the Apple Developer Portal registers the application identifier, registers the technical capabilities of the product, and sets up the provisioning information that you will need to distribute the application (the information is collected, but the provisioning profiles are not generated yet; I will do that shortly). Registering the product with App Store Connect creates the application on the App Store and registers the first version of the application that will be distributed so that you can provide the rest of the metadata and marketing information. To make it easier to set up your application with Apple, Fastlane Tools provides the Produce tool which will talk to both the Apple Developer Portal and App Store Connect to register the application, and will establish the link between the two.

Before I begin coding anything, I want to explain how Fastlane Tools works with my repository. Basically, Fastlane Tools implements a domain-specific language for automating tasks. As I have already mentioned, Fastlane Tools is implemented in Ruby and it loads and executes the automation scripts. The automation scripts themselves happen to be Ruby modules. So when building Fastlane Tools automations, I will be coding in Ruby, so I have full access to the Ruby language to implement automation behavior. All Fastlane SDL modules are stored in a `fastlane` subdirectory of the project workspace. There is one file named `Fastfile` that is the main module where all of the automation actions are implemented. There are several secondary modules that can be used to configure individual tools and I will use them as I proceed with the automation steps.

There is one special file that contains settings that can be used globally and I use it to store information about the application and my Apple Developer account. This file is named `Appfile`:

{{< highlight ruby "linenos=table" >}}
app_identifier "software.naked.blogging"
apple_id "michael.collins@naked.software"
team_name "Naked Software, LLC"
team_id "NZXG7K5N83"
{{< /highlight >}}

Several of the tools in Fastlane use information such as the application identifier, my apple user identifier for logging in, and my company information (again, I am publishing applications under my LLC named "[Naked Software, LLC](https://ecorp.azcc.gov/BusinessSearch/BusinessInfo?entityNumber=L20595818)"). By defining these values in `Appfile`, they will be globally defined and the tools will be able to read their values at runtime. I will not need to continually provide these values as parameters for the majority of the tools that I will use in my automation scripts.

I will next create `Fastfile` to script my automations. Fastlane Tools provides tools that are called **actions**. I can execute most of the actions manually from the command line, or I can script them in `Fastfile` by creating higher level actions called **lanes**. A lane is basically a scripted action that I define that uses other actions to get things done. The lanes that I create can be reused and called from other lanes that I define. It is possible in Fastlane for me to group my custom lanes by their platform. For example, I could define one set of lanes for use on Android and another set of lanes for use on iOS. Since I am only currently targeting the iOS (and iPadOS) platforms, I am not going to use the platform grouping in my `Fastfile` because it's largely useless when targeting only a single platform.

Using Produce, I will specify the name of my application, the initial version number that I will release (1.0.0), the name of my company, and the set of services that the application will use. Initially, I am not using any services, so most of them will be turned off.

{{< highlight ruby "linenos=table" >}}
lane :register_application do
  produce(
    app_name: 'Naked Blogging',
    app_version: '1.0.0',
    language: 'en-US',
    company_name: 'Naked Software, LLC',
    enable_services: {
      access_wifi: 'off',
      app_group: 'off',
      apple_pay: 'off',
      associated_domains: 'off',
      auto_fill_credential: 'off',
      data_proection: 'complete',
      game_center: 'off',
      health_kit: 'off',
      home_kit: 'off',
      hotspot: 'off',
      icloud: 'cloudkit',
      in_app_purchase: 'off',
      inter_app_audio: 'off',
      multipath: 'off',
      network_extension: 'off',
      nfc_tag_reading: 'off',
      passbook: 'off',
      personal_vpn: 'off',
      push_notification: 'off',
      siri_kit: 'off',
      vpn_configuration: 'off',
      wallet: 'off',
      wireless_accessory: 'off'
    }
  )
end
{{< /highlight >}}

With the `register_application` lane created, I can execute it through the Fastlane CLI:

    bundle exec fastlane register_application

You will notice that I prefixed my call to Fastlane in the terminal with `bundle exec`. By running Fastlane through Bundler, I will ensure that the correct version of Fastlane Tools that I have installed for my application will be executed. When the `register_application` lane executes, it will call the Apple Developer Portal and App Store Connect to try to register the application. If the application has already been registered, then no changes will be made, so this command is pretty safe to run.

When I run the `register_application` lane for the first time, the Fastlane CLI will ask me for my password to log into Apple. It finds my username from `Appfile`. If I have two-factor authentication enabled (I do), Fastlane will prompt me to enter my two-factor authentication code or I can type `sms` to receive a text message to my cell phone. After authenticating the first time, Fastlane CLI will save the access token to invoke the Apple developers APIs for future requests.

## Match

Now that my application is registered with Apple, I will turn my attention to the requirements for distributing my application. When I am building my application, I will need to install it on my iPhone or iPad for debugging and testing. At some point, I may want to have a friend or co-worker install the application to test it out and give me some early feedback. At a later point, I will probably want to put out a [beta](https://en.wikipedia.org/wiki/Software_release_life_cycle#Beta) release for early adopters to test out and give me more feedback and help me to find and eliminate defects before I ship to the App Store. After processing the beta cycle, I will want to send the app build to the Apple App Store for review and release to customers. In order to do these things, I need two artifacts: a code signing certificate and a provisioning profile.

A code signing certificate wraps a cryptographic key and is used as a security safeguard. Code signing certificates use public/private key cryptography to generate a digital signature for the application bundle. At build time, the build system will use the private key to generate a unique signature value based on the contents of the application bundle. At distribution and installation time, the signature can be verified using the public key. The benefit to code signing is that it is easy to detect if someone makes changes to the application bundle between the point where I ship it and the point where it is installed. If a virus attaches itself to my application bundle after I send it to Apple, the signature will no longer validate and the bad application bundle can be rejected.

The other artifact is called a provisioning profile. A provisioning profile describes how the application can be distributed and installed. There are four types of distribution for iOS applications:

* **Developer**: This deployment type allows a developer to install the application on their iPhone or iPad device using Xcode. The developer's device needs to be registered with the Apple Developer Portal and associated with the provisioning profile.
* **Ad Hoc**: This deployment type allows a non-developer to install the application on their iPhone or iPad device. Apple allows up to 100 devices to be registered in the Apple Developer Portal per account. Ad Hoc distribution is meant for limited distribution to internal stakeholders, product owners, or software testing.
* **App Store**: This deployment type allows the application to be deployed to customers using Apple's App Store or to early adopters or beta testers through Apple's TestFlight service.
* **Enterprise**: This deployment type allows companies to distribute proprietary applications internally to devices that are used by their employees.

I will be focusing on the Developer, Ad Hoc, and App Store distribution types. As I mentioned above, the devices that the application can be installed on must be part of the provisioning profiles for Developer and Ad Hoc distribution. Keeping track of which devices are active and not active can be a chore. If I do a build and a device is not included in the provisioning profile, then one of my test users can't run the build and I will need to produce another build that has been corrected. Fastlane Tools does provide a tool that will help us automate managing the list of registered devices for an application. It is called `register_devices`.

Every Apple device can be uniquely identified using a value known as a **UDUD**, or unique device identifier. This identifier is what gets included in the provisioning profile and what the device looks at to determine if the application can be installed on the device. The UDID of the device can be found on older Macs using iTunes or by viewing the device properties in Finder on macOS Catalina or Big Sur.

To manage the authorized devices that can install and test my applications, I maintain a text file in my Git repository named `devices.txt`. `devices.txt` is formatted as a tab-delimited file with a header line. Each line in the text file represents a single device and contains the device's UDID and a human-readable description of the device. I can create the `devices.txt` file and append new devices to it easily using shell commands in a terminal. To create a `devices.txt` file, I will run:

    echo "Device ID\tDevice Name" > devices.txt

This command will create a new `devices.txt` file and will write the line to the file. To append devices to `devices.txt`, I will run:

    echo "{PUT_UDID_HERE}\t{PUT_DESCRIPTION_HERE}" >> devices.txt

Notice the `>>`. This symbol indicates that the shell should append the line to the file instead of overwriting it.

After registering all of my devices, I can now use the `register_devices` tool to make sure that all of my devices are registered with the Apple Developer Portal. It is safe to run `register_devices` as frequently as I want to. If devices are already registered with Apple, no changes will occur and no duplicates will be created.

In the previous section, Produce used my Apple developer email and password to authenticate me with Apple and obtain an API access token. This is a legacy approach to authentication that Fastlane Tools is moving away from. Apple not so long ago started exposing an API key that can be used for authentication instead and works better for automated DevOps pipelines. API keys can be created from App Store Connect. After logging into App Store Connect, go to **Users and Access** and switch to the **Keys** tab. On this tab, you can create a new API key. There are three values that you will need to provide Fastlane Tools with for authentication:

* The API key: This can be provided either as a path to a file containing the key, or the contents of the API key. The API key is encoded using PKCS 8 format.
* The Issuer ID: This is a unique value identifying your Apple Developer account.
* The Key ID: This is a unique identifier for the API key and is used to look up and validate API calls.

The API key should be treated as a secret value. It should not be checked into your Git repository. I typically store the API key in a secret store for use by DevOps pipelines such as GitHub Actions or Azure DevOps and access the key information through an environment variable at build time. The issuer ID and key ID are not necessarily secret and can be exposed. I typically don't put them into my Git repository and inject them into my builds using environment variables.

I will now create a new lane to register my test devices with Apple. I will expand on this afterwards to create the code signing certificates and provisioning profiles.

{{< highlight ruby "linenos=table" >}}
lane :update_provisioning_profiles do
  app_store_connect_api_key(
    key_id: ENV['APPLE_KEY_ID'],
    issuer_id: ENV['APPLE_ISSUER_ID'],
    key_content: ENV['APPLE_KEY'],
    in_house: false
  )

  register_devices(devices_file: 'devices.txt')
end
{{< /highlight >}}

{{< admonition type="tip" title="Setting the APPLE_KEY Environment Variable" open="true" >}}
If you are not strong at Shell programming, it's fairly easy to load the Apple API key into an environment variable. If you have the API key stored in a file, you can copy the contents into an environment variable using the following command:

    export APPLE_KEY=$(cat /path/to/apikey)

This command will first execute the `cat` command to output the contents of the API key file. The contents will then be stored in the `APPLE_KEY` environment variable.
{{< /admonition >}}

With the devices registered, I can now move on to managing my code signing certificates and provisioning profiles using the [Match](https://docs.fastlane.tools/actions/match/) tool. Match is a wonderful tool that makes managing code signing certificates and provisioning profiles so easy. Match works by generating the code signing certificates and provisioning profiles and storing them in a location that can be shared by a team and also made easily available to a DevOps pipeline. Match can use either a Git repository, Google Cloud storage, or Amazon S3 to store the code signing certificates and provisioning profiles. And the code signing certificates and provisioning profiles are stored encrypted, which adds a little security to it. It's not perfect, but it works really well and the risk of exposure is fairly minimal in my opinion.

Match supports storing global or default settings in a module named `Matchfile`. This module is helpful because sometimes I need to run Match using the Fastlane CLI and having information such as my certificate's Git repository URL configured makes it easier. A common scenario for running Match from the CLI is to install the development code signing certificate and provisioning profile in a new development environment. My `Matchfile` has the following content:

{{< highlight ruby "linenos=table" >}}
storage_mode "git"
git_url "https://github.com/nakedsoftware/certificate.git"

type "development"
generate_apple_certs true
force_for_new_devices true
{{< /highlight >}}

These settings let Match know that I'm using a GitHub repository to store my certificates in. The default type of certificate and provisioning profile to generate, download, and install is a development certificate. Finally, the provisioning profiles should be regenerated if the device list changes.

I can now go back and update my `update_provisioning_profiles` lane to run Match to generate and install the code signing certificates and provisioning profiles:

{{< highlight ruby "linenos=table" >}}
lane :update_provisioning_profiles do
  app_store_connect_api_key(
    key_id: ENV['APPLE_KEY_ID'],
    issuer_id: ENV['APPLE_ISSUER_ID'],
    key_content: ENV['APPLE_KEY'],
    in_house: false
  )

  register_devices(devices_file: 'devices.txt')

  match(type: 'development')
  match(type: 'adhoc')
  match(type: 'appstore', force_for_new_devices: false)
end
{{< /highlight >}}

These three new commands instruct the Match tool to look at the code signing certificates and provisioning profiles in my Git repository, determine if they need to be updated, and to regenerate them using the Apple Developer API. If new code signing certificates or provisioning profiles are generated for the application, they will be installed locally in my development environment and then encrypted and uploaded to the Git repository.

The great benefit to using Match to manage your code signing certificates and provisioning profiles is that they can be configured in the Xcode project and all developers and DevOps pipelines can use the same certificates for building and deploying the application. There's no need to create different code signing certificates for different developers or having to change the Xcode configuration in order to build and run using a specific developer's profile.

We now have the ability to distribute the application. Now we just have to build it so that we can distribute it.

## Creating Naked Blogging

Now that I have registered my Naked Blogging application and created the code signing certificates and provisioning profiles, I'm going to create my iOS application and configure the code signing certificates and provisioning profiles so that I can build and install the application on my devices.

When I work with Xcode, I prefer starting with creating a workspace instead of an application project. I create a workspace because I assume that my applications will probably have one application project and maybe at least one framework project. Having a workspace also allows me to group my playgrounds and projects together.

After creating my application project and adding it to the workspace, I go in to modify the code signing settings and choose the code signing certificates and provisioning profiles that were created and installed by Match. First, I turn off the **Automatically manage signing** option. I next switch over to the **Build Settings** tab and select the correct code signing certificates and provisioning profiles:

{{< image src=code_signing_settings.png alt="I select the Apple Developer and Apple Distribution code signing certificates for Debug and Release builds respectively. For the provisioning profiles, I chose the Development provisioning profile that Match created for debug builds and the App Store provisioning profile for release builds." caption="Configuring the code signing certificates and provisioning profiles for debug and release builds." title="Code Signing Build Settings" >}}

If everything is configured correctly and I switch over to the **Signing & Capabilities** tab, I should see the correct code signing settings and no warnings or errors:

{{< image src=signing_and_capabilities.png alt="The Signing & Capabilities tab should show the correct code signing settings now and should not show any errors or warnings." caption="No errors or warnings should appear" title="Signing & Capabilities Tab" >}}

With the code signing certificates and provisioning profiles now properly configured, I can move forward with building and releasing my application.

## Scan

We are all professional developers here, correct? We are building high-quality applications and care about not introducing defects into existing code that works, correct? We all have some sort of automated unit tests that we use as a benchmark to determine if a new build is acceptable, correct? Well, maybe or maybe not, but I am going to assume so.

Since we're all professional developers and we care about automating our build and delivery process, DevOps matters to us. Specifically, we have a laser focus on having a successful Continuous Integration/Continuous Delivery experience. When I am building applications and working on professional projects, I use branches to isolate the changes that I am making until I am ready to incorporate them into the product. When my changes are ready and tested, I will merge them into my `main` branch using a GitHub pull request. I like using pull requests to manage merging my changes because I can put gates in my way to prevent me from checking in bad code. Specifically, I can use GitHub Actions to make sure that the code in my development branch builds and that my unit tests pass before I merge the code into the `main` branch.

Fastlane Tools provides us with a tool called [Scan](https://docs.fastlane.tools/actions/scan/) that I use to build my appliction for testing and runs the unit tests in the iOS Simulator. I can use Scan both locally and in my DevOps pipeline. I can also use Scan to test my application across more than one device type.

Like Match, I can run Scan from the command line using the Fastlane CLI. And like Match, I can store default settings for Scan in a module named `Scanfile`. Here's my `Scanfile` for my application:

{{< highlight ruby "linenos=table" >}}
workspace 'Blogging.xcworkspace'
scheme 'Blogging'
device 'iPhone 12 Pro Max'
{{< /highlight >}}

I can use the Fastlane CLI to run Scan and make sure that my default configuration works:

    bundle exec fastlane scan

This command will run my unit and UI tests in the iOS Simulator and should come back successful.

In my new application, I have both unit and UI tests. I plan to do most of my testing with unit tests, but if I can automate some test scenarios through UI testing, I will seek to do so. However, UI tests are slow and in a CI/CD DevOps pipeline, I don't need to run UI tests. I should be able to rely mostly on unit tests to tell me if the code is ok to merge or not. So I want to modify the test scehem to remove the UI tests.

Newer versions of Xcode have introduced a new artifact that can be included in a project: a testplan. A testplan allows you to configure the set or subset of (UI or unit) tests that you want to run. I am going to create a testplan to use for continuous integration builds. In Xcode, I created a new top-level group in my workspace for test plans and mapped it to a subdirectory of my project workspace named `TestPlans`. I then created a new Testplan file in Xcode and added it to the **Test Plans** group. In the testplan, I selected the `BloggingTests` unit test bundle to execute in the testplan.

I next edited my `Blogging` scheme and on the **Test** tab, I clicked on the **Convert to use Test Plans...** button. I chose the **Choose Test Plan** option and chose my new `ContinuousIntegration.xctestplan` testplan document to associate it to the `Blogging` scheme. I then updated `Scanfile` to point it to the new testplan:

{{< highlight ruby "linenos=table" >}}
workspace 'Blogging.xcworkspace'
scheme 'Blogging'
device 'iPhone 12 Pro Max'
testplan 'ContinuousIntegration'
{{< /highlight >}}

I can verify the testplan runs successfully with Scan by running Scan from the CLI again:

    bundle exec fastlane scan

I'm going to implement a new `test` lane in `Fastfile` to automate running the tests as part of a DevOps pipeline:

{{< highlight ruby "linenos=table" >}}
lane :test do
  scan
end
{{< /highlight >}}

Technically I don't need to create a new lane just to run Scan, but I'm going to add a couple of things to the lane. First, like all professional software developers, I want to make sure that I am testing as much code as possible. The way to determine that is to enable code coverage and generate a code coverage report from my unit tests. To do that, I will add support for [Slather](https://github.com/SlatherOrg/slather). Slather is a tool that will basically generate a code coverage report from the results of test execution. Gathering code coverage information is built into Xcode.

The first change that I need to make is to add a Ruby dependency on the `slather` gem to `Gemfile`:

{{< highlight ruby "linenos=table" >}}
source 'https://rubygems.org'

gem 'fastlane'
gem 'slather'
{{< /highlight >}}

Fastlane supports Slather using the [slather](https://docs.fastlane.tools/actions/slather/) action. When Slather runs, it needs access to the built application and derived data generated by the Xcode build system. I am going to enhance my `test` lane then to use Scan to only build the application and unit test bundle, use Scan to run the unit tests and generate the code coverage information, and then use Slather to generate the code coverage report:

{{< highlight ruby "linenos=table" >}}
lane :test do
  derived_data_path = 'build/derived_data/test'
  scan(
    clean: true,
    build_for_testing: true,
    derived_data_path: derived_data_path
  )
  scan(
    code_coverage: true,
    test_without_building: true,
    derived_data_path: derived_data_path
  )
  slather(
    build_directory: derived_data_path,
    proj: 'Sources/Blogging/Blogging.xcodeproj',
    workspace: 'Blogging.xcworkspace',
    scheme: 'Blogging',
    html: true,
    output_directory: 'fastlane/test_output/slather',
    use_bundle_exec: true
  )
end
{{< /highlight >}}

The updated `test` lane will now build the test application and will store the data generated by the build system to the `build/derived_data/test` directory. The lane will then run Scan to run the unit tests using the test application. Finally, Slather will run to generate the code coverage report and the HTML report will be written to `fastlane/test_output/slather`.

{{< admonition type="warning" title="Warning" open="true" >}}
You probably do not want the derived data files and the code coverage report to be stored in your Git repository. Be sure to update the `.gitignore` file to exclude these directories.

If you use the standard [Swift](https://github.com/github/gitignore/blob/master/Swift.gitignore) `.gitignore` template, the `build` directory and test reports should already be excluded.
{{< /admonition >}}

{{< image src=code_coverage_report.png alt="The code coverage report showing the percentage of code that is covered by unit tests using the skeleton for a new iOS application in Xcode." caption="The code coverage report shows how much of the code is executed by unit tests." title="Code Coverage Report" >}}

## Gym

Now that I can run unit tests, I can have confidence that I am detecting a lot of potentially bad code before I merge it from a pull request into my application. I'll script the pull request workflow in GitHub actions at a later point, but I can run this manually now using Fastlane and am confident in the unit tests and I can see the code coverage report. The next stop in my automation quest is to get Naked Blogging in the hands of my internal testers (mostly me and my wife) to look for defects in the build. To do that, I will turn to a new Fastlane tool named [Gym](https://docs.fastlane.tools/actions/gym/).

Gym performs two actions for me. First, it will build the application from the source code. Second, it will extract the application as an IPA archive that can be distributed and installed on test devices. In order to get the application on my tester devices, I will create a distribution using the **ad hoc** provisioning profile that I generated earlier using Match. If you remember from the previous section, when I configured code signing for the Naked Blogging application, I configured the debug builds to use the development provisioning profile and the release builds to use the App Store provisioning profile. I did not use the ad hoc provisioning profile. When I run Gym to build the application and export the IPA archive, I will tell Gym to override the configured provisioning profile and to use the ad hoc provisioning profile instead.

I am going to create a new lane to generate the adhoc build:

{{< highlight ruby "linenos=table" >}}
lane :build_adhoc_application do
  match(type: 'adhoc', readonly: true)

  gym(
    output_directory: 'build/output/adhoc',
    derived_data_path: 'build/derived_data/adhoc',
    export_options: {
      method: 'ad-hoc',
      provisioningProfiles: {
        'software.naked.blogging' => 'match AdHoc software.naked.blogging'
      }
    }
  )
end
{{< /highlight >}}

The first thing that I am doing in the `build_adhoc_application` lane is running Match again to install the code signing certificate and provisioning profile for ad hoc distribution. Notice the presence of the `readonly` attribute. This action will not attempt to regenerate the certificates. Instead, it will just attempt to clone my certificates Git repository and install the certificates from there. I don't like the idea of giving my DevOps pipeline the ability generate new certificates or provisioning profiles. After installing the certificate and provisioning profile, I execute Gym to build the application and export the application archive for ad hoc distribution.

Gym supports a global configuration file named `Gymfile`. Here is my `Gymfile`:

{{< highlight ruby "linenos=table" >}}
workspace 'Blogging.xcworkspace'
scheme 'Blogging'
clean true
include_symbols true
include_bitcode true
{{< /highlight >}}

When I run the `bundle exec fastlane build_adhoc_application` command in the terminal, Gym will produce the `Blogging.ipa` file in the `build/output/adhoc` directory.

For internal distribution to testers, I use Microsoft's [App Center](https://appcenter.ms) service. To automate distribution of Naked Blogging to App Center for internal testing, I need to add a Fastlane plugin: [appcenter](https://github.com/microsoft/fastlane-plugin-appcenter).

First, I will add the plugin to fastlane. In the terminal, I will run:

    bundle exec fastlane add_plugin appcenter

This command will update `Gemfile` to add the `appcenter` plugin as a Ruby gem dependency and will then install the plugin.

The next thing that I will need to do is to log into App Center and create a User API token. Tokens can be created from the **Account Settings** page in the App Center console.

{{< admonition type="warning" title="Protect Your Token" open="true" >}}
The App Center API token is a secret value and should be protected. You should never check your token into your Git repository. In Fastlane, I will expect that I have stored the token in the DevOps secret store and I will inject the token using an environment variable.
{{< /admonition >}}

Before I get to uploading the build to App Center, I need to stop to introduce another piece of functionality. When releasing applications to App Center, TestFlight, or the App Store, it's common to send out release notes that let testers or customers know what changed in the new build and why they should install it. This could include a list of new features or a list of defects that were corrected. I'm going to add a file named `CHANGELOG.md` to my repository where I will keep a running log of changes that I incorporate into the application for each release. I will use a format for my changelog that is documented on [keepachangelog.com](https://keepachangelog.com).

My basic changelog will look like this to start:

{{< highlight markdown "linenos=table" >}}
# Changelog

## [Unreleased]

### Added

### Changed

### Deprecated

### Removed

### Fixed

### Security
{{< /highlight >}}

As I make changes, I will add the changes to one of the subsections in the `Unreleased` section. When I am ready to release the product, I will change the section name to be the product version number for the release.

Now that I have my changelog created, I need to read it. Fortunately, there's another Fastlane plugin that will help me to do that: [changelog](https://github.com/pajapro/fastlane-plugin-changelog). I will use Fastlane to install the plugin:

    bundle exec fastlane add_plugin changelog

With the new plugin, I can automate publishing a new ad hoc build to App Center for testing:

{{< highlight ruby "linenos=table" >}}
lane :upload_build_to_appcenter do
  build_adhoc_application

  changelog = read_changelog

  appcenter_upload(
    owner_type: 'organization',
    owner_name: 'NakedSoftware',
    app_name: 'Naked-Blogging',
    app_display_name: 'Naked Blogging',
    app_os: 'iOS',
    app_platform: 'Objective-C-Swift',
    file: 'build/output/adhoc/Blogging.app',
    dsym: 'build/output/adhoc/Blogging.app.dSYM.zip',
    notify_testers: true,
    release_notes: changelog
  )
end
{{< /highlight >}}

The `upload_build_to_appcenter` lane will start by using the previously created `build_adhoc_application` to generate a new ad hoc build to distribute. Next, the `read_changelog` action will read the `Unreleased` section of the `CHANGELOG.md` document and will store the contents of the section in the `changelog` variable. Finally, the `appcenter_upload` action will upload the ad hoc build to Microsoft App Center, update the release notes from the changelog, and then notify all of my internal testers (via email) that a new build is ready to test.

I can test the new lane by running the lane using the Fastlane CLI:

    bundle exec fastlane upload_build_to_appcenter

After running this lane, I can see the build was uploaded to App Center and I received an email notifying me of the new build. I can then follow the link in the email to install Naked Blogging on my iPhone or iPad Pro. Here's what the App Center installation page looks like:

{{< image src=app_center_install.png alt="The installation page allows my internal testers to download and install the Naked Blogging build that was uploaded to App Center. Notice that the release notes from the changelog were successfully updated." caption="The installation page allows testers to download and install Naked Blogging." title="App Center Installation Page" >}}

<span>Photo by <a href="https://unsplash.com/@birminghammuseumstrust?utm_source=unsplash&amp;utm_medium=referral&amp;utm_content=creditCopyText">Birmingham Museums Trust</a> on <a href="https://unsplash.com/s/photos/assembly-line?utm_source=unsplash&amp;utm_medium=referral&amp;utm_content=creditCopyText">Unsplash</a></span>