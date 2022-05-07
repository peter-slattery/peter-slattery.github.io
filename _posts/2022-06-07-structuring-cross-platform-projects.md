---
layout: post
title:  "Structuring Cross Platform Projects"
date:   2022-05-04 03:41:30 -0700
categories: [development, handmade]
---

I've gone through the process of setting up and organizing a cross-platform code repository a few times now and I think I've finally settled on a format that I like. Here's what I do.

## One Command Builds
I often find myself coming back to projects after a few months and not knowing how to get it building again. I don't use things like makefiles for this reason - they just overcomplicate things. At the end of the day, I want to get the project compiling again as quickly as possible. So why not make it a one line command?

I learned from Allan Webster (over at Mr4thDimension on YouTube) that you can run bash on any platform where you can run git. So I started writing build scripts in bash. The pattern generally goes like:

```bash
set OS=$1
set ARCH=$2
set MODE=$3

if [ $OS == "win32" ]
then
  clang first_win32.c -o myapp.exe user32.lib opengl32.lib
elif [ $OS == "osx" ]
then
  clang first_osx.c -o myapp -framework OpenGL
fi
```

Obviously this can get more complicated. My actual build scripts use the OS, Architecture, and Mode arguments to select compiler and linker flags, libraries, etc. At the end of the script it puts all the options into a single build command which gets executed. You can see an [example of this here](https://gist.github.com/peter-slattery/7c9151b52f6f6142d50c5fe2bdc79fc5).

## Unity Builds

The above examples are not trivializations of my project structures, and the example I linked to is simply the copy-pasted build script for my [Lumenarium](/lumenarium/) project. I build in what is called a **Unity Build**. 

The way a Unity Build works is that all your files are included in the same compilation unit, one after another. You manage dependencies when necessary by predeclaring functions in headers. 

The trick I've landed on that I like is to have a different entry point per platform which includes a `platform.h` file, the common project files first, then platform implementation after. So for example, `first_win32.c` might look like:
```c
#include "base_types.h"
#include "platform.h" // generic interface for all platforms to implement
#include "platform_agnostic_application_code.c"

#include <windows.h>
#include <gl/gl.h>
#include "platform_win32.c"
#include "platform_win32_file.c"
// ... more platform implementation files as convenient

int main(int arg_count, char** args)
{
  if (!platform_init()) return platform_error();
  if (!application_init()) return application_error();
  while (application_running())
  {
    application_update();
    if (platform_has_renderer())
    {
      application_render();
    }
  }
  application_cleanup();
  platform_cleanup();
  return 0;
}

```

The nice thing about this is that your application can initialize itself in a way that is agnostic of platform, provided the same services are implemented on each platform.

Another nice thing is that your application isn't heavily reliant on #ifdef's for platform specific sections, and your application is forced to be nicely sectioned off from the platform code by nature of the order of includes.

## Platform.h

So what does that platform file look like? Below is an example of how I do a memory access interface. If you want a more full example, I'd go check out [Our Machinery's platform file here](https://ourmachinery.com/apidoc/foundation/os.h.html)

```c
// Platform Memory
uint64_t platform_page_size(); // returns the size of a page for the current platform

uint8_t* platform_memory_reserve(uint64_t size);
uint8_t* platform_memory_commit(uint8_t* base, uint64_t size);
void     platform_memory_decommit(uint8_t* base, uint64_t size);
void     platform_memory_release(uint8_t* base);
```

You'll notice that this doesn't allow for a lot of the options you'll find available to you in an platforms memory access functions. I find that I'll start simple and progressively introduce more and more options as I need them. Usually, my File Access implementation has quite a few parameters exposed since there's a lot of kinds of file operations you'll need. But memory almost always ends up looking like this.

## Platform Resources

At the end of the day though, you want to DO things. And you do things usually by modifying things that exist on real hardware on the actual platform. Rather than write a bunch of branches to handle different platform objects, I fall back on the pattern of resource handles pretty often.

A handle will look like:
```c
typedef struct file_handle file_handle;
struct file_handle
{
  uint64_t value;
};
```

`value` in this case, can be unwrapped on the platform side into a `FILE*`, or a `HANDLE`, or even into the index in an array that your platform code manages to store more complex resource data. Then, all your platform functions that deal with files take or return a `file_handle` and that's all the platform-agnostic code ever has to deal with.

The only exception I've found to this is memory, which I just pass around as `uint8_t` pointers. That's because every single platform treats them similarly - it's either a 32 bit or 64 bit integer which is always able to be used by platform operations. Obviously, things get trickier if you start targeting platforms that behave very differently around a particular resource.

## What Goes Where?

I struggled with this for a while. How do you know what the platform should handle vs what the application should handle?

The thing that solved it for me was to actively work on two platforms at once while doing the first implementation pass. This forces you to make that call in the full knowledge of how it will scale to two platforms. And in general, my rule of thumb has become "customize at the platform level, but call at the application level". To demonstrate why, let's look at an example:

Say you need to load a font file at startup. Loading a file is an operation that looks different depending on the platform so why not do it there? Well, let's say six months from now, we're trying to create platform layer #2 and we forget we need to load the font file. What's going to happen is that the font won't appear, because we never even called the function. But odds are, you've got so many places that could be breaking the font rendering when starting a platform layer, you might miss what went wrong - was it the renderer, the font rasterizer, file loader, or window display that's the problem? By moving the command to load the file into the application, the application will likely break at the site of the broken file-loading routing, pointing you right to the problem. 

## Results

The resulting file structure for this usually looks something like:
```
/project
    /run_tree/platform/arch/mode/platform_arch_mode.exe
    /src
        /platform
            platform.h
            /win32
                win32_file.c
                ...
                win32_first.c
            /osx
                osx_file.c
                ...
                osx_first.c
        /app
            app_first.c
            ...
    /build
        build.sh
```

I find that this does a nice job of balancing keeping things in one place, compiling quickly, and easily parsable with allowing for quickly spinning up new platforms for the application in question.

This has even worked so far as to write a single application that runs on Windows, OSX, Raspberry Pi, and WebAssembly