---
layout: post
title:  "Lumenarium"
date:   2022-05-07 03:41:30 -0700
categories: [project]
---

![Lumenarium](/assets/images/lumenarium_00.png)
Over the last three years, basically since I got back from the helping construct Radialumia, I have been working on a custom piece of software to bridge the gap between what we wanted to be able to do with Radialumia and what tools were available at the time. This is no slight on those tools - they were simply tuned to a different set of priorities than the ones we had (and honestly, over the last three years I've come to appreciate how much work they did to get them there).
If you want to try Lumenarium out, it runs on Windows at the moment, and you can get the code on github here.

![Lumenarium](/assets/images/lumenarium_01.png)
## Features
- A full 3D rendering of the workspace, which can include multiple sculptures.
- A programmatic pattern system, which has access to the 3D coordinates of each LED in the workspace
- A timeline for combining and composing patterns into animations
- A profiler (displaying function time and memory consumption) for analyzing your work
- Debug tools that give easy access to identifying problem strips, leds, assemblies, etc. You can solo a strip, change its color manually, solo an address, an assembly etc. This debug space is extensible as well, providing a user space, plug in model that lets you quickly iterate on debug capabilities for your sculpture. This could even be extended to the point where it can be used to DJ your sculpture.
- SACN & UART protocols built in already, with a flexible structure for adding more.
- A text based file format for specifying led strip layout

## Handmade, from Scratch
I made a somewhat controversial decision early on in the process of building Lumenarium to build it in almost-C C++ (no classes, no templates, no RAII, no exceptions, but operator and function overloading). I also decided to build everything myself. That means the renderer, the network protocols, the interface library, profiler, etc are all hand-rolled specifically for this program. It links with the windows OS libraries it needs to access system hardware, and (I believe) the only other libraries are math.h and xmmintrin.h

Why do this? Partially as a learning journey for me. I became a much better programmer over the course of this project. I got experience doing things you don't on the web: paralellism, graphics programming, meta-programming, plus I re-learned linear algebra, and got practice structuring a large codebase.

## Ok, but really... Why from scratch?
When I sat down to write this, we had several priorities:
- **Keep the good stuff** - one huge advantage of the software we were using was it tracked every LEDs position in 3D space, meaning our patterns responded to the real world geometry of the sculpture (as opposed to Mad Mapper and it's like, which project your sculpture onto a pixel grid first).
- **Fast Iteration** - the software we used before had to be quit and restarted (and it didn't store application state too well) to iterate on a pattern. Compilation times in Java are also not good.
- **Artist Control** - I wanted something that had more of the adobe software "put the power in the hands of the artists" sensibilities. I wrote most of the patterns for Radialumia, at the direction of others. I want to enable them to be creative.
- **Flexible Sculpture Construction** - We have amazing mechanical engineers on our team who work in CAD. Is there a way to get CAD data into Lumenarium to define where LEDs go? What about complex assemblies of leds (daisy chained strips, strips that mirror each other, etc)

Given just fast iteration, we needed something in a language that compiled fast (goodbye Java), that had low level hardware access (in case we needed to go SIMD on complex patterns), gives manual access to memory (cache misses are important), and which had a flexible library loading model (DLLs). So I chose C++, but restricted myself to the subset of C++ that doesn't bog down compile times:
- No classes or template metaprogramming
- Unity build - every file is included in order, and there are only two compilation units: The platform layer, and the application
- No RAII & no exceptions

Artist Control meant a complex interface that was fast to iterate on. This was the one place I considered using external code - IMGui is a great option. But I wanted to learn. And I really liked the idea of building an interface in the style of Blender that allowed a multimodal, flexible workspace. But beyond that, I'd also need custom widgets, a timeline, data inspectors, profiling tools - those aren't going to come from a library. So at that point, I began working on the toolset from scratch too.

Finally file formats - I knew someday I'd want to try parsing an stl to extract LED strip positions. For that, I wanted a custom format, and so we begin writing a tokenizer, parser, serializer, etc.

And at this point, that's almost the entire application. The only things left would be the actual SACN/ArtNet protocols and interfacing with the OS for USB/Network access, file IO, and graphics. So why not just roll it all?

## Architecture
This wasn't arrived at ahead of time, but through a process of discovery.

**Platform Layer**<br />
Exposes a common interface to the application. To port to a new platform, just need to create a os.h and os.c file for the platform.
- Platforms Supported
  - Windows
  - OSX
  - Raspberry Pi
  - WebAssembly

**Engine**<br />
The core of the application, handling animation and 3d layout.
- Sculpture: maps leds to 3d coordinates and map leds to output buffers
- Output: takes leds and send them over various kinds of connections to actual led strips
  - Protocols Supported
    - Networked SACN
    - USB UART
    - Theoretically, SACN could be sent over USB and UART could be sent over the network since the output system just treats them like blobs of data with a destination.
- Patterns: basic animation engine - can blend between multiple patterns

**Editor**<br />
User interface and sculpture renderer. Lets you preview animations and sculpture geometry before the sculpture is constructed.
- Notable feature - you can run the executable without the editor and the memory footprint decreases by almost 80%

**User Space**<br />
A set of function prototypes that a user application can tie into to customize the application. 
- For Blumen Lumen we used this for weather detection to close the flowers based on NOAA wind speeds, and a debug panel that let us manually open and close the flowers with UI buttons.

*You can read more about the [Platform Layer Separation](/structuring-cross-platform-projects/) in this blog post*

## And it works!
Its been through a number of iterations. There was a node based interface for a while (which I still dream of bringing back someday).
![](/assets/images/lumenarium_02.mp4)
I added a built in time and memory profiler.
![Lumenarium](/assets/images/lumenarium_03.png)
Here's one of our latest tests, using it to run a new sculpture.
![Lumenarium](/assets/images/lumenarium_04.gif)

## Lumenarium Today
A few months ago we constructed Foldhaus' latest sculpture, Blumen Lumen, in Silver Spring Maryland. It is powered by Lumenarium, which has been running around the clock for the last several months.
![Lumenarium](/assets/images/lumenarium_05.png)
![Lumenarium](/assets/images/lumenarium_06.jpg)