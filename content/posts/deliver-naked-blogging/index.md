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

<span>Photo by <a href="https://unsplash.com/@birminghammuseumstrust?utm_source=unsplash&amp;utm_medium=referral&amp;utm_content=creditCopyText">Birmingham Museums Trust</a> on <a href="https://unsplash.com/s/photos/assembly-line?utm_source=unsplash&amp;utm_medium=referral&amp;utm_content=creditCopyText">Unsplash</a></span>