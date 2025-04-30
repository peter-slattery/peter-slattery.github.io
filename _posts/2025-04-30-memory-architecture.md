---
layout: post
title:  "Memory Architecture"
date:   2025-04-30 03:41:30 -0700
categories: [development, handmade]
---

I've been working on Legendarium for a good while now and I just decided to do a memory audit. The gist of this was to push on an allocator at application startup that asserts if anything tries to allocate. This way, I just run the app in the debugger, wait for an assert, then investigate the piece of code that was attempting to allocate on the global default allocator. Often times these systems are ones I put in quickly for prototyping. At my current stage of development, that is almost everything. So what patterns are emerging?

## Scoped Arenas
The main vehicle I want to use for brokering memory access is the Arena. Much has been written elsewhere about these by smarter people than myself. See [Handmade Hero](https://guide.handmadehero.org/code/day034/#4955), and [Ryan Fleury](http://rfleury.com/p/untangling-lifetimes-the-arena-allocator) for more.

The one major implication of arenas to understand moving forward is that reallocation is not a part of their API. (It can be, but it makes things more complicated, which is exactly what I'm attempting to avoid by using arenas).

## Dynamic Arrays -> Linked Lists
The most obvious source of allocations is adding an element to an already full dynamic array. I use these all over during prototyping because they behave consistently and are a language primitive in jai so they're simple and tested. However, when it comes time to manage memory lifetimes, they aren't the best. This is because I end up with non-centralized allocations, and often times allocations happening deep down struct hierarchies.

The first conversion I make is often times to simply take a function like this:
```jai
push_clipping_region :: (bounds: Range2) { /* ... */ }
```
And convert it to this:
```jai
push_clipping_region :: (bounds: Range2, arena: *Arena) {}
```
And convert clipping regions from this:
```jai
Immediate_Clipping_Region :: struct {
  bounds: Range2;
}
```
To this:
```jai
Immediate_Clipping_Region :: struct {
  bounds: Range2;
  next: *Immediate_Clipping_Region;
}
```
This allows for dynamic growth, on an arena which can track the allocations, and clean them up at the appropriate time.

This has worked fairly well for most of my cases - especially ones where the access patterns are simply:
1. Push things on a list for later processing
2. Pull all the things off in order and process them one at a time.

This isn't great for cases where I want to index into the array though.

## Guessing at Requirements
In the event that I want to be adding things to a list that I will also be indexing into, we have a more complicated case. The most simple solution is actually to just take a guess at how many things there will be. This has worked really well for things like OS events per frame. There are usually only a handfull per frame so having a statically sized array for storing them works fine. The exception was window resize events - the OS (Mac and Windows) tend to block during resizing - meaning that when resizing is complete you can end up with 1000s of events. I'm not going in for the [dangerous threads crew](https://github.com/cmuratori/dtc) approach yet so I don't actually care about each of these individually - just the last one. So we upsert in the list rather than insert for this one event type, and problem solved.

## What Happened Last Frame?
There is still the case, however, where I end up wanting to avoid a hard memory constraint - say for the number of glyphs drawn on screen each frame. This is highly user-behavior dependent and I don't want to just not draw things the user has asked to see.

I'm using an Immediate Mode interface pattern so redrawing frequently is normal. The memory for drawing primitives lives on a per-frame arena - the arena gets reset after rendering. So when the next frame begins, all of last frames memory is no longer valid, and the arena's memory is free to use again.

The pattern I've arrived at for this situation is to simply allocate a best-guess sized array on frame 0. Then, let the app request as much storage as it needs. I only actually track what we have space for, but I also track how many were requested. On Frame 1, I allocate enough space for what was needed on Frame 0.

I don't think this is a perfect approach - for one thing, if I am waiting on the OS for events before rendering the next frame, the current frame could be incomplete. The solution here is to simply always draw twice - something I need to do anyways to handle Immediate Mode interaction.
But the problem still remains that I do draw incomplete frames, even if they are gone quickly. Could this cause issues? I'm not sure.

## Centralized Resource Stores && Paged Arrays
A final pattern I have adopted is to centralize storage of longer lived resources, and allow reallocation-like behavior via Paged Arrays.

**Paged Arrays**<br/>
A paged array is simply an array of arrays. The structure looks like this:
```jai
Paged_Array :: struct ($TValue: Type) {
  pages: [32]*TValue;
  pages_allocated: int;
  count: int;
  page_size: int;
}

get :: (array: Paged_Array, index: int) -> *Paged_Array.TValue
{
  page_index := index / array.page_size;
  index_in_page := index % array.page_size;
  return array.page[page_index] + index_in_page
}
```
There are several nice properties of this.
1. Paged_Array structs have a constant size. `pages` is a static array.
2. You don't allocate all the pages up front. So your memory footprint remains small at startup, and can grow to be quite large without reallocating. However, if you set your page size such that page_size * max_pages is greater than your expected needs, you can support a large memory footprint without incurring it until it is needed.
3. Lookup is still constant time. It takes longer than array lookup, but only by a few operations. (I suspect, though haven't measured, that the actual cost is paid in the extra fetch it takes to go through the double indirection)
4. Growing the Paged_Array does not reallocate anything - meaning pointers are stable over the lifetime of the array.

**Centralized Storage**<br/>

I use paged arrays as a backing structure for centralized resource pools. This allows per-resource lifetimes while still using arenas to hand out memory in fixed size blocks.

# Conclusion

I'm sure there are more patterns I'll find, but this is what I'm finding at the moment.
