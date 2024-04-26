---
layout: post
title:  "Types of Code Transformation"
date:   2024-04-21 00:00:01 -0700
categories: [development]
---

I'm going through a pretty huge rendering engine overhaul, and it's made me think about the specific techniques I'm using to do the code transformation. 

## Goals:
While I'm doing this work, I have a few goals in mind. 

1. Preserve existing functionality - in this case, keep the OpenGL backend working. I plan to deprecate this when I'm done, instead focusing on Metal and Direct3D 11. But in the meantime, having OpenGL working means I have a known-correct implementation I can revert to to make sure I haven't introduced bugs into the larger system.
1. Incremental progress - at all points in time, I should be able to compile and run the game on every system I currently support. No long open commits, or huge diffs. 
1. Keep making progress on the actual game - ideally, none of my refactors prevent me from spending some time each day pushing the game design forward.

## Phases:

Given those goals, I've found myself reaching for different techniques at different stages.

**1. Research + Understanding**

When I don't understand a large piece of the transformation, I tend to spin up a new compilation unit (usually with it's own entry point) and isolate an implementation of the new system as a sketch.

> My background is in Illustration originally, and I find this process highly analogous to color studies, composition sketches, and anatomy drawings. They're meant to be thrown away, but the work you do informs the final piece.

**2. Application Research**

With a decent grasp on the target transformation, I go look at the existing code, and begin to take steps in the direction of that target. Often times, I find myself starting and erasing several commits worth of changes in the beginning, until I identify how I'll make those incremental steps.

My common stumbling blocks here are times when I make a change that has unexpectedly far reaching ramifications. Working in a statically compiled language makes these abundangly obvious but sometimes it's more changes than I want to make in a single commit.

**3. Incremental, Non-Breaking, Incorporation.**

This will be the focus of the rest of the article.

## How to Move Slow and Not Break Things
_Look, it was dumb in the aughts when they started saying it, and it's dumb now._

I've seen a couple approaches to this.

1. **Is there no api?** First, create one. Wrap all your calls to the existing functionality in a small set of functions that you can swap out for the new implementation. Then create that implementation and swap it out.
  - This is good for things like graphics apis, or OS layers, where you can't really do an incremental swap. 
  - If your language has preprocessor directives, you can do something like the code below (you can do this with functions too, if you don't have a preprocessor), which allows you to go wrap all uses of the old implementation before actually changing the functionality.
```c
#if true
#  define IMPLEMENTATION(arg1) old_impl(arg1);
#else
#  define IMPLEMENTATION(arg1) /* literally anything here representing new functionality */
#endif
```

2. **Is there an existing api, but you don't want to do a wholesale swap?** Create a second api, wrapping over the new functionality. Run it alongside the old one - this gets everything compiling and ensures hot codepaths are at least running. Then, at as small a granularity as possible, remove the old api calls, and use the results of the new ones.

3. **Is your new functionality very different in shape to your existing functionality?** This is harder - try to change your existing functionality so it conforms to the shape of the new api. Ideally you can do this without things like performance degradation, but if you know that the new shape + new functionality will be as good or better, or that the costs are worth it, then you can incur these penalties temporarily to support the api change. 

## Sources / Tips

- What about large, codebase-wide refactors? There's a really good post on the old [Our Machinery blog about this](https://ruby0x1.github.io/machinery_blog_archive/post/step-by-step-programming-incrementally/index.html#problem-3-refactoring)

- need to find all places a variable is accessed? Add an underscore after it, and recompile. You'll get every instance of that variable being accessed. 