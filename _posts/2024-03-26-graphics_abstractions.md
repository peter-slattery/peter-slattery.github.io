---
layout: post
title:  "Graphics Abstractions"
date:   2024-03-26 00:00:01 -0700
categories: [development, graphics]
---

## Prelude

I'm going to be taking a few months off work to... well, work. But work on fun stuff, rather than on other people's stuff. While I'm doing that, I'd also like to get in the habit of writing a few times a week. To that end, I have a few guiding points:
1. Write for at least 30 minutes without editing. (Edit later, if it seems worth posting)
2. Write about stuff I'm working on, no abstract theory, no complaining.
3. Take the time to read about the stuff I'm working on so I have new things to write about.

With that out of the way, I'm going to write today about graphics backends and abstractions.

## New Backends (To Me Anyways)

I'm working on porting my rendering code over to DirectX 11 and Metal from OpenGL 3. The reason for this is that I wanted to be able to do more advanced things with UI rendering without blowing my per-frame GPU data budget (more on this in a future post - or honestly go read these [ Hidden Grove, Our Machinery, Casey Muratori ]). 

I've always worked in OpenGL's fixed function pipeline. It's simple (relatively). It's direct (relatively). And I never really hit it's limitations.

On the other hand, it's simplicity means you can't express some things in as simple a way as you'd like. And since Apple deprecated OpenGL, and doesn't support above 4.1, I figured I'd use this as an opportunity to learn Metal and DirectX for the first time.

So far, DirectX has been incredibly verbose, but incredibly well documented. I haven't pushed this layer as far yet since I only took my Macbook with me on a trip. 

So far, Metal has been a joy to write in its simplicity, but documented so poorly that I'm amazed I've figured it out at all. Also the Xcode Metal Frame Debugger is a mixed bag - on the one hand, it's really powerful. On the other hand, it's so finnicky w.r.t your target version and what Metal commands you're using (as in if you use deprecated commands, it'll just refuse to take a snapshot). 

## Experiments

I've always written abstraction layers with the idea of having one main codepath that compiles differently based on the output target. I'm taking the opportunity to re-think this approach. 

My main insight so far has been that, when you are writing or debugging platform code, you are only ever working on one platform at a time. The process of working on another involves moving to another physical machine. 

This gives you the ability to declutter your platform code by removing concerns that a given platform doesn't have. This also gives the opportunity for fewer codepaths. To illustrate, imagine youre building for three OSes: Windows, MacOS, & Linux. And you want to write the api in a "unified command" style. So you create a `create_window(name, width, height) -> Window_Handle` function. Then, inside that function, you call one of three different functions: `win32_create_window`, `macos_create_window`, `linux_create_window`. (The mechanism you use to accomplish this is outside the scope of what I care to write about right now) So you now have 4 different code paths. Now do this for each api level function you want to use to interact with your window.

On the other hand, what if you just had `macos_main`, `win32_main`, and `linux_main`? Now you don't need wrapper functions (you can still have them of course), you have 3 codepaths that don't interact at all - the data you pass to create a window on Windows can be completely different from the data you pass on MacOS, etc. And when you're debugging on mac, you are only dealing with the code that actually runs on a mac.

## Further Benefit: Platform Specificity

One other potential advantage I see from this redesign is that it preserves your ability to do platform-specific, hot-spot fixes.

To illustrate, imagine again that you have a "command once" unified api for all platforms. Say in certain contexts, and only on Windows, you need to set some DirectX flags before making a draw call. So you write `set_pre_draw_call_flags()`, and give it an empty implementation on MacOS and Linux. But this creates asymmetry in your api, and future you (or teammates) might come back and wonder why those flags aren't being used on Mac or Linux. You could leave a comment but that also adds noise to the codebase.

Alternatively, if all your rendering commands are issued at a backend level, then you just set those flags in the d3d_render function at the proper time, and leave the linux and mac render functions alone. No wholes in the api, just platform/backend specific code, in a place where that is explicitly expected to happen.

## Exceptions

I don't think I'll push this api design everywhere in the application, because it inverts the relationships from the platform being a service the app relies on to the app being a service that provides inputs to the platform layer. That change isn't so useful when you're dealing with things that also tend to be more uniform across OSes, such as file interaction. These I think will remain a struct of function pointers the app can call into. 

## Further Exploration

One area I want to explore more is how to write re-combinable modules such that they feed into this model seamlessly. 
In all the examples I've given, your main platform functions are explicitly handling all the OS/graphics backend work themselves, taking in data you give them from the app.

But what if you have a module that you want to be able to reuse, such as a UI library? Does it implement it's own per-platform rendering functions? Does it try and abstract the platform so it's self contained, and you just tell it what platform/backend? Or does it rely on you re-writing platform specific code each time you want to import it?

My hunch is that to the greatest degree possible, you want reusable modules to be platform agnostic - they take in data, and they return data to you, and you are responsible for currying IO/files/rendering/etc. 
Another pattern I've seen (mostly in the sokol utilities) is to provide off-the-shelf interfaces for common dependencies that operate as plugins - this hits a sweet spot between authoring your dependencies yourself, and providing a mechanism for the users of your library to do their own thing when they need to (even if that person is you, in the future).

