---
layout: post
title:  "Array Insertion API Design"
date:   2024-03-30 00:00:01 -0700
categories: [development, handmade, thoughts]
---

I'm working in a language that has what I would describe as a highly idiosyncratic set of standard libs. Frankly it's a breath of fresh air - personal consideration has been paid to legibility and ergonomics in many cases that other languages and library developers have simply gone with what the standard was (which, if your case is Javascript, was set in like, a week, by a dude who didn't really make the greatest decisions. So you know... choose your influences wisely.).

Anyways, there's one thing that bothers me about the design of these libraries. I end up doing this a lot:
```jai
array_add(*array, value);
index_of_value := array.count - 1;
```
or
```jai
index_of_value := array.count;
array_add(*array, value);
```

When what I'd like to be doing is:
```jai
index_of_value := array_add(*array, value);
```

This is a quibble but it made me think about a few things.
1. When you "create" a "resource" (heavy HEAVY scare-quotes here), you care about what it contains AND where it ends up in memory. 
1. How it gets to where it is in memory is largely the responsibility of it's container.
1. While knowledge of your container is an important architectural consideration, it isn't something every line of your module should care about. 
1. Not returning the way to access the thing you just stuck in a container from the operation that puts it there seems to be a pretty glaring problem that leaves the calling code to infer where the thing is. 

This holds for all containers; arrays, linked lists, hash maps, etc. When I insert something, I want some way to access it later on. (Addendum: I think it's reasonable that if you're giving me just a pointer, then I also should be able to expect that the pointer is stable right?)