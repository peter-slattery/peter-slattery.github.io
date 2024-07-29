---
layout: post
title:  "Rendering Architecture"
date:   2024-07-01 03:41:30 -0700
categories: [development, graphics]
---

Over the last month, I've refactored my engine's rendering architecture... for the second time.
And honestly, it feels really good! I've learned a ton over these refactors.

## Refactor the First

The first major refactor I undertook was to transition from OpenGL to Metal and D3D11. This was motivated by two desires:
- I wanted to implement an Immediate Mode UI for my game (and tooling)
- I wanted to force myself to support >1 backend, to stress test my architecture.

The major thing I wanted for the IMGUI was to be able to reference structured buffers in shaders when rendering instances. Now I know I could have done this in OpenGL, however the max version of OpenGL supported by MacOS would not have allowed this. So upgrading OpenGL was an option but would force me to support a second backend anyways.

Enter desire #2: I've known for a while that only understanding OpenGL was a pretty low bar for graphics programming. If I want to do more complex things I'll need to get a handle on a more modern API (or two). D3D11 seemed like a good sweet spot - old enough to have tons of documentation, new enough to have things like Compute Shaders and more flexibility in data access patterns, and not nearly as much of a PITA to set up. And then Metal is just the only way to go on MacOS it seems.

My renderer took the form of proxying calls to OS/API specific functions via a command buffer. It looked like this:
```
push_command(.ClearScreen, .{0,0,0,1});
/* push more commands*/
execute_all_commands()
```
The nice thing about this was that introducing Metal and D3D11 just meant adding a backend file for each, and then creating new versions of the command handlers. These are looked up via a v-table like struct that I make by hand.

This worked pretty well. It allowed me to develop on both platforms, and to successfully implement my UI.

# TODO: screenshots!

There were some (totally forseeable) unforseen consequences of this:

First, I had to go upgrade my asset pipeline to support selecting an asset file based on the platform, so I could load/compile API specific shader code at runtime.

Second, I also had cases where different API's wanted different data from the commands, and so a few commands specific to the UI needed to support a d3d_args field and a metal_args field which were used to override the values in the body of the command data. This isn't great - it's a good escape hatch but I consider it a bit leaky given that I needed these fields immediately after designing the API. (my thinking is that I might get there eventually from unforseen use cases, but to need it day 1 implies a poor abstraction.)

## Refactor The Second

I ran with the first refactor for a month or two, and it worked relatively well. But I was avoiding something - Render Passes.

In my mind, Render Passes and Draw Calls were really abstract things that people talked about, but never explained. There are papers and blog posts galore discussing them in all-to-abstract ways that meant I just kind of avoided talking about them. Instead, I'd designed my renderer to abstract the graphics API at the level of individual commands. And any time I thought about some cool effect like Depth of Field, I'd get nervous and go work on something else.

It wasn't until I stumbled on ["How to write a renderer for modern graphics APIs](https://blog.mecheye.net/2023/09/how-to-write-a-renderer-for-modern-apis/) that I began to understand. And at the end of the day, this is how I think about them now:

A *Draw Call* is a command to the GPU to write to a Render Target, and all state setting required for that invocation.
A *Render Pass* is a sequence of draw calls that all write to the same Render Target, and all state setting common to all Draw Calls in that pass.

So *Refactor the Second* involved tearing out my old rendering interface piece by piece, to replace it with one that operated in Passes and Draw Calls. I also challenged myself to keep the game running and rendering properly on both Windows and Mac at the same time. This was based on this Our Machinery blog post ["Step-by-step: Programming Incrementally"](https://ruby0x1.github.io/machinery_blog_archive/post/step-by-step-programming-incrementally/index.html).

So the first step was to replace all GPU asset creation with a better abstraction. Rather than have handles to VBO's and ShaderResourceView's etc., I moved to having three primitives that exist outside the API: Shaders, Buffers, and Textures. Shaders can be Rendering Shaders (vertex/fragment shader programs) or Compute Shaders (I don't use these yet so the implementation is just stubbed out). Textures can be normal textures or Render Textures, but they are treated as the same thing which makes using a Render Target as the input to a later pass trivial. And Buffers encapsulate vertex, index, instance buffers as well as any application defined buffers needed.

I made this change entirely in place, so the old api was retrofitted incrementally to use the new Render Resources (all of which are referenced by handles).

The second step was to introduce Draw Calls. I did this by creating a DoDrawCall command that was implemented alongside the old implementation. This way, all the old code continued to work, while I moved single code paths over to the DoDrawCall command.

Finally, Render Passes were implemented. Each backend is responsible for creating resources necessary for any Render Pass actions such as clearing the screen, etc.

Swapping these in without breaking anything was difficult because they inherently call the api in-band, where the old command architecture invoked commands out-of-band. I solved this by "moving backwards". To illustrate, if this was the overall render() function before:
```
render :: () {
  render_scene();
  render_ui();

  set_render_target(output_render_target);
  clear_screen(color_black);
  set_blend_mode(premultiplied_alpha)
  do_draw_call(.{
    mesh = fullscreen_quad
    textures[0] = scene_render_target
  });
  do_draw_call(.{
    mesh = fullscreen_quad
    textures[0] = ui_render_target
  })
  swap_buffers()

  execute_render_commands()
}
```

Then to move this to render passes, I'd do the following:
```
render :: () {
  render_scene();
  render_ui();

  execute_render_commands();

  pass := begin_pass(.{
    targets[0] = output_render_target,
    pass_action[0] = .{
      kind = ClearScreen,
      color = color_black
    },
    blend_mode = .PremultipliedAlpha
  });
  pass_push_draw_call(*pass, .{
    mesh = fullscreen_quad
    textures[0] = scene_render_target
  });
  pass_push_draw_call(*pass, .{
    mesh = fullscreen_quad
    textures[0] = ui_render_target
  });
  pass_submit(pass);

  swap_buffers_immediately();
}
```

And then I'd go move the render_ui function over to use draw calls, resulting in hoisting the execute_render_commands function up even higher.
```
render :: () {
  render_scene();
  execute_render_commands();

  render_ui();

  pass := begin_pass(.{
    targets[0] = output_render_target,
    pass_action[0] = .{
      kind = ClearScreen,
      color = color_black
    },
    blend_mode = .PremultipliedAlpha
  });
  pass_push_draw_call(*pass, .{
    mesh = fullscreen_quad
    textures[0] = scene_render_target
  });
  pass_push_draw_call(*pass, .{
    mesh = fullscreen_quad
    textures[0] = ui_render_target
  });
  pass_submit(pass);

  swap_buffers_immediately();
}
```
Until finally, execute_render_commands was effectively doing no work because no commands were being submitted. This allowed me to remove it, the old api, and all the old backend functions. You can imagine, deleteing a couple thousand lines of code, knowing none of it was being called and that functionality was being gained not lost, felt incredibly good.

With this refactor complete, I moved on to the actual point of all this: rendering different effects!
I've implemented Depth of Field so far, and am working on Screen Space Reflections.
The wonderful thing is that this refactor is allowing me to focus on the effects rather than on the graphics API. Which finally feels like this API is in a good place.

# TODO: Screen Shots