---
layout: post
title:  "Building 4coder for Mac"
date:   2022-07-20 00:27:30 -0700
categories: [development, tools]
---

This is going to be a pretty utilitarian post on how to build 4coder for Mac.

# Why?
4coder is incredible - and more importantly, it's what I want to be using whenever I'm typing code. I had to move to an M2 Mac recently and the pre-built versions of 4coder don't work on an M2 chip.

Allen recently open sourced 4coder putting the code on github. But, as the README notes, support of Mac hasn't been updated in some time. I suspect, in fact, that 
there never was support for M2 Macs at all. 

So I started trying to get it to work on Mac again, specifically on an M2 chip. It's surprisingly easy, even though it took me a while to figure out how to do it.

What follows below is how I accomplished this. While I haven't tested it on Intel macs, I believe many of the steps below will need to be done if you want to build 4coder for a non-M2 chips computer as well. 

# Building 4coder

## 1. GET THE CODE: 
Follow the steps in the README of the git repo up until step 7 (where you run the various build scripts)
The git repo is here: 
```bash
https://github.com/dion-systems/4coder
```

## 2. REALPATH: 
The build scripts make use of a tool called realpath. This is standard on linux it seems but is not included on Macs. You can get `realpath` by doing one of the following: 
- running `brew install coreutils`
- using this git repo: `https://github.com/harto/realpath-osx` (NOTE: I haven't used this yet, but it seems legit.)

## 3. FREETYPE: 
The 4coder-non-source repo has prebuild libraries for freetype. Those libraries are built for Macs running on Intel chips. 
- I figured out how to do this by following nakst's excellent tutorial [here]( https://essence.handmade.network/blog/p/3593-text). It turns out, much of that isn't necessary for our purposes however. I will detail the steps I took here.
- Run the following to get the freetype source code: 
```bash
curl https://mirrors.up.pt/pub/nongnu/freetype/freetype-2.9.tar.gz > freetype-2.9.tar.gz
gunzip freetype-2.9.tar.gz
tar -xf freetype-2.9.tar
```
- You can ignore all the steps in "Configuring FreeType" in nakst's article. For our purposes, the omitted sections might as well remain.
- `cd` into the freetype-2.9 folder
- Run this configure command, modified from nakst's original: 
```bash
./configure --without-zlib --without-bzip2 \
--without-png --without-harfbuzz \
CFLAGS="-g -ffreestanding -Wno-unused-function"
````
- Run the make command: 
```bash
make ANSIFLAGS="" > /dev/null
```
- Copy `freetype-2.9/objs/.libs/libfreetype.a` to `4ed/4coder-non-source/foreign/x64/libfreetype-mac.a` (Remember to rename the original if you want to compile for x64 at some point in the future)
  
## 4. IMMINTRIN ISSUES: 
Compiling on an M2 chip, I got a bunch of errors relating to intrinsics/assembly code in the 4coder_audio.h/c files. I personally don't rely on 4coder for audio so I chose to just omit this code from compilation.
- In `4coder_default_include.cpp`, comment out these lines:
  - `#include "4coder_audio.h"`
  - `#include "4coder_audio.cpp"`
- In `4coder_default_hooks.cpp` (or wherever you choose to put your custom layer's `default_startup` function) make sure to omit the line `def_audio_init()`

5. `cd` into `4ed/code`
6. run `bin/build-mac.sh`

And you should be good to go! 
Let me know if this doesn't work for you, and I'll update instructions here.