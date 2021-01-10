---
author: michael-collins
categories:
- Naked Blogging
date: 2021-01-10T15:50:00-07:00
description: Naked Blogging has been a hobby project that I have been semi-developing over the past couple of years. As I restart my blog, I am resurrecting the idea of building a blogging application for my iPhone and iPad. In this and future posts, I will document my journey in building this product.
draft: false
resources:
- name: "featured-image"
  src: "feature.jpg"
- name: "featured-image-preview"
  src: "feature_small.jpg"
tags:
- iOS
- Naked Blogging
- Products
title: Introducing Naked Blogging
---
In this post, I am restarting my journey to build my own blogging app: Naked Blogging. I am going to give you an overview of my vision and start documenting my journey by showing you how I am building this product.

<!--more-->

Ever since the iPad Pro was introduced with the capability of having an attached keyboard, I've dreampt of being able to use it for blogging and writing. It looked like a great writing tool. It was portable and lightweight. It looked more conventient for a laptop for being able to blog while on vacation at the pool or sitting by the ocean at the beach, or writing in a coffee shop, or flying across country. I really wanted to build the whole product, but life kept getting in the way. But as I am blogging again now in 2021, I have decided to restart the product and will be bringing you along. In this series of blog posts, I will be leading you through my journey. I will share with you my vision for the product and will be showing you the features as they are being developed. Hopefully I'll also be able to share the product with you as well and you can provide feebdack for me along the way.

This new product is being named Naked Blogging. I will be publishing it under my Naked Software, LLC, company that I started to use as a side business for my hobby product development outside of my normal career work. Maybe, someday, I'll make actual revenue off of the company and can focus on it full time, but that's a dream for another post.

## What is Naked Blogging?

I believe that the first thing that I need for Naked Blogging is to be able to describe it in a single sentence: a vision sentence. I want to be able to convey to everyone that I tell about this product what it will do in a way that will get everyone interested and excited in the product.

What do I want Naked Blogging to do? I want to be able to write and publish blog posts from my iPad instead of my iMac or laptop. There could be occasions where I'd want to microblog from my iPhone. But basically I want to be able to publish new content whenever I feel inspired to write using my portable devices. I don't just want to be able to write. I want to be able to write and publish the full set of content that I would typically include in a blog post such as source code fragments, images, diagrams, videos, and audio. I also want to be able to interact with my GitHub repository to track my TODO list in Issues and document ideas and research for blog posts that I can reference when I'm writing the actual post.

If I am to narrow this down to a single sentence of what I want Naked Blogging to be, my first draft reads like this:

> Naked Blogging will allow me to write new blog posts using my iPad Pro and iPhone wherever I am with the same capabilities that I have to create content on my laptop or desktop computers.

## What are My Initial Features?

When I think of this vision statement, my first question is how do I create content for my blog posts on my desktop or laptop computers? What do I have there that make it so easy for me to generate blog posts that I don't have on my iPad Pro?

I will start by brainstorming and listing all of the tools that I am using to generate content for my new blog:

* I use [Hugo](https://gohugo.io) to generate the static website from the source code
* I use [uBlogger](https://ublogger.netlify.app) as the theme for my website
* I host my website in a [GitHub](https://github.com) repository
* I use [GitHub Actions](https://github.com/features/actions) to build and publish my static website
* I use [GitHub Pages](https://pages.github.com) to host and serve my website and blog
* I use [Git](https://git-scm.com) to persist my changes and publish my new content to GitHub
* I use [pull requests](https://docs.github.com/en/free-pro-team@latest/github/collaborating-with-issues-and-pull-requests/about-pull-requests) to review my new content and make last minute modifications before publishing the content publicly on my website
* I use [Visual Studio Code](https://code.visualstudio.com) to write my new content using [Markdown](https://www.markdownguide.org) or to customize the theme with custom HTML, JavaScript, and CSS
* I use [Google Chrome](https://www.google.com/chrome/) to preview the post as I read it to see the generated HTML from the Markdown
* I use [GitHub Gists](https://gist.github.com) to host my source code snippets that I embed in my blog posts
* I use [Disqus](https://disqus.com) to allow readers to provide comments and feedback on my blog posts
* I search and download images from [Unsplash](https://unsplash.com) to use as illustrations and feature or cover images
* I create diagrams using [Omnigraffle](https://www.omnigroup.com/omnigraffle/)
* I use [Sketch](https://sketchapp.com) to create or edit some images and include them in blog posts
* I use [Font Awesome](https://fontawesome.com) for some special symbols on my website and in my posts
* I use [Snagit](https://www.techsmith.com/screen-capture.html) to capture and annotate screenshots that I include in the blog posts
* I use [Google Analytics](https://marketingplatform.google.com/about/analytics/) to let me know if anyone is reading my blog posts
* I capture video, pictures, or screenshots from my iPhone or iPad and embed them in a blog post
* I publish an announcement of new blog posts to my [Twitter feed](https://twitter.com/mfcollins3)

Obviously, there are a few challenges to my vision statement. I can't run Sketch on my iPad currently and using a tool like Omnigraffle for diagrams would be a bit different. I also don't have access to Snagit. But in those cases, there may be some alternatives (and I've found a few or have some ideas) that would give me similar capabilities and may be better to use on my iPad.

Before I start creating user stories from the list above, I'm going to start with some technical stories. The first story is going to be a delivery story: how will I get Naked Blogging into your hands? I'm the developer, so I can always install developer builds on my iPad. But part of the reason for writing about this journey and building an application is that I want to share it (and hopefully make some money off of my work, if I can, since the goal of starting a business is to make a profit). I would like the deployment process to be as automated as possible, but I would like there to be a bit of a flow to it so that I can authorize the workflow proceeding to different stages. I would like to test new builds first, for example, before sending them to TestFlight to make sure that features work. I'd like to do a TestFlight stage for early adopters to test using real world scenarios and send me feedback before I release to my full set of customers. I'd like the App Store submission process to be as automated as possible. So my first story is:

1. Deliver Naked Blogging to Customers

Next, I'll go through a simple workflow for writing a blog post:

2. Clone my website repository
2. Create a blog post
2. Edit a blog post
2. Commit my changes
2. Publish my changes
2. Create a pull request
2. Review a pull request
2. Approve a pull request
2. Create an issue for a topic to write about
2. Add information to a topic issue
2. Search Unsplash for a feature image
2. Embed an Unsplash image
2. Import a feature image from the photo library
2. Capture a feature image using the camera
2. Embed a photo library image
2. Embed a camera image
2. Create a Gist
2. Insert a Gist
2. Edit a Gist
2. Create a source code sample
2. Edit a source code sample
2. Create a diagram
2. Edit a diagram
2. Publish notification of new blog post on Twitter
2. Approve a moderated Disqus comment
2. Reject a moderated Disqus comment
2. Reply to a Disqus comment

I'll start with this set of stories and will probably add a few more as I build out Naked Blogging.

## What Do I Want to Achieve?

I have tried building Naked Blogging a couple of times only to have to shelve it because *Life* got in the way. Either my kids needed me, my wife needed me, or my day job needed me. I think a couple of times after I stepped away I had forgotten where I was and became frustrated and scapped it again. My hope is that my transparency in how I am going to build it by blogging about my activities will help to keep me focused on my goal, and help me to be successful in this latest attempt.

Feel free to follow along, ask questions, and in the not-so-distant future, try out the application yourself and let me know what you think. I'll be blogging and demonstrating the features as I build them, and my hope is that by the time I am finished blogging about a new feature, I'll be ready to include it in a TestFlight build. If you join my TestFlight early adopters group, you'll get to experience the features for yourself.