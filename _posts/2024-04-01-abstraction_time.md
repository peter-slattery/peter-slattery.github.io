---
layout: post
title:  "Abstraction Time"
date:   2024-04-01 00:00:01 -0700
categories: [development]
---

_This isn't an April Fools post._

I'm working on a graphics abstraction layer, abstracting across Metal and D3D11, using a sort of immediate mode ui layer as the driver. I started out by creating separate entry points and rendering functions for the two targets, which means that the further I go, the more code I have duplicated across the two targets. And yesterday I had the urge to start joining them. 

I'm resisting for now, because I have a different architecture in mind. But it does make me think about the proper time to do a simplification pass.

Right now, there's a growing sense of things being messy or difficult, or requiring duplicated effort. But I think that's a good thing - at this stage, what I want to have is maximum capability while accepting that it comes at something of an ease of use cost. 
What this accomplishes, is that I have a very clear idea of what's difficult to do in relation to how often I need to do it. And the picture that's emerging is that it's kind of difficult to implement new render paths (say 3 out of 10), or create new rendering resources. However, I do that very infrequently. On the other hand, it's pretty verbose to render new UI atoms (text, rectangles, etc) (3/10 as well), however, that is what I'll spend the vast majority of my time doing. 

So the proper level of abstraction for my graphics layer is one which makes interacting with graphics primitives from Jai easy (there's a lot of boilerplate just to interact with the Objective-C and C++ runtimes that isn't fun to write). 

Meanwhile, the proper level of abstraction for the UI drawing code is probably higher level than it is now. HOWEVER, I don't have examples of UI code that illustrate what form the abstraction should take yet. And so I'm also resisting creating abstractions for the drawing code as well.

The net benefit of this approach (I'm hoping) is two fold:
1. The eventual finalized abstractions will be appropriate to the level of granularity at which they are most used.
1. The abstractions have high granularity. Abstracting isn't a process of *replacing* an existing API with a higher level one, but of *composing* a lower level API, and giving names to the newly composed units. This allows you to easily do common things, and still reach down into the more granular layers when it comes time to do one-off things that there is no abstraction for.