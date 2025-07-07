---
layout: post
title:  "Optimizing 4coder"
date:   2025-07-07 03:41:30 -0700
categories: [development, handmade, optimization]
---

This will just be a quick post documenting an optimization story I just completed.

## Background

I use a text editor called 4coder for my personal work. It's open source, written in C, and relatively simple compared to other text editors. I am starting a contract next week and I'd like to be able to use 4coder for my professional work as well - 4coder was written with C/C++ in mind, and doesn't do too well out of the box on Typescript projects. So I'm spending some time working on my fork of 4coder. This post documents my process at dealing with one of the issues I have with the editor.

## The Problem

When 4coder displays a large file, it slows to a crawl. I need it to still be fast.

## Symptoms

1. When viewing a large file, each frame takes a very long time - long enough to make using the editor impossible.
2. If the large file is loaded, but not visible, performance returns to normal.
3. Operations like search, which index into the large file, still work at high speed.

## My Process

First, I needed a large file to work with. Fortunately, Allen (the author of 4coder) shipped it with test files. So I'll use the one included. It's about 9MB of C code - this could be important since the editor parses code for highlighting and layout.

To identify what's wrong, as a first pass, I simply run the editor in XCode, load and view the large file, then hit the "Pause Execution" button. This essentially is a probabalistic attempt to identify what's taking so long. (If you imagine each function taking some space on a timeline and you randomly sample that timeline, you are most likely to end up sampling within the  function that takes the most time.) To ensure this isn't a fluke, I do this about 20 times and sure enough, almost every time I end up sampling inside the render function - specifically inside token coloring.

Looking at how that's done, it appears as follows:

1. Get a list of the files tokens.
2. Starting with the first visible token, look it up in the `Code_Index` (a hash table of identifiers in the file called `Code_Index_Note`)
3. If the Code_Index_Note exists, color it according to the type (Macro, Type, Function, etc)
4. If it doesn't exist, color it according to the token's type (Oeprator, Scope, Parenthesis, NumericLiteral, StringLiteral etc)

As for data structures go, this is `Code_Index`:
```c

struct Code_Index_Note{
    Code_Index_Note *next;
    Code_Index_Note_Kind note_kind;
    Range_i64 pos;
    String_Const_u8 text;
    struct Code_Index_File *file;
    Code_Index_Nest *parent;

    Code_Index_Note *prev_in_hash;
    Code_Index_Note *next_in_hash;
};

struct Code_Index_Note_List{
    Code_Index_Note *first;
    Code_Index_Note *last;
    i32 count;
};

struct Code_Index{
    // Some other fields we don't care about
    Code_Index_Note_List name_hash[4096];
};

```
We care about `name_hash` - a hash table with internal chaining. Essentially, the more Code_Index_Note's needed, the deeper each bucket gets. So to look up a token, you find the bucket, then iterate over the buckets contents to find the appropriate note.

Having found the bottleneck, the next step is to measure performance so I can tell if I'm improving things. Rather than measure time, I chose to measure the following:

1. the number of times we look up in `global_code_index.name_hash`
2. the total depth of all buckets we look up in
3. the average bucket depth
4. the max bucket depth

I also chose, early on, to print out the depth of each bucket every frame so I could see it, but we'll come back to that in a moment.

The initial metrics were:
```
test_large.cpp
# (total lookup depth / # lookups) = avg lookup depth
28194334/2029647=13.891250    @V1 - no changes
  Avg Bucket Depth: 3.153940
  Max Bucket Detph: 498
```

Having these metrics, my first attempt was to simply not do name_hash lookups if we knew the token wouldn't be an identifier. If you look at the code below, before my transformation color based on the token type happens after we do a lookup. I simply reversed the order of these.

**Before**

```c
Token_Iterator_Array it = token_iterator_pos(0, &token_array, visible_range.first);
for (;;){
  if (!token_it_inc_non_whitespace(&it)){
    break;
  }
  Token *token = token_it_read(&it);
  String_Const_u8 lexeme = push_token_lexeme(app, scratch, buffer, token);
  Code_Index_Note *note = code_index_note_from_string(lexeme);
  if (note != 0)
  {
    switch (note->note_kind)
    {
      case CodeIndexNote_Type:
      {
        paint_text_color(app, text_layout_id, Ii64_size(token->pos, token->size), color_type);
      } break;

      case CodeIndexNote_Function:
      {
        paint_text_color(app, text_layout_id, Ii64_size(token->pos, token->size), color_function);
      } break;

      case CodeIndexNote_Macro:
      {
        paint_text_color(app, text_layout_id, Ii64_size(token->pos, token->size), color_macro);
      } break;

      default: {} break;
    }
  }

  else if (token->kind == TokenBaseKind_Operator ||
          token->kind == TokenBaseKind_ScopeOpen ||
          token->kind == TokenBaseKind_ScopeClose ||
          token->kind == TokenBaseKind_ParentheticalOpen ||
          token->kind == TokenBaseKind_ParentheticalClose ||
          token->kind == TokenBaseKind_StatementClose)
  {
    paint_text_color(app, text_layout_id, Ii64_size(token->pos, token->size), color_operator);
  }
}

```

**After**

```c

Token_Iterator_Array it = token_iterator_pos(0, &token_array, visible_range.first);
for (;;){
  if (!token_it_inc_non_whitespace(&it)){
    break;
  }

  ARGB_Color token_color = color_default;
  Token *token = token_it_read(&it);
  if (token->kind == TokenBaseKind_Operator ||
          token->kind == TokenBaseKind_ScopeOpen ||
          token->kind == TokenBaseKind_ScopeClose ||
          token->kind == TokenBaseKind_ParentheticalOpen ||
          token->kind == TokenBaseKind_ParentheticalClose ||
          token->kind == TokenBaseKind_StatementClose)
  {
    token_color = color_operator;
  }
  else
  {
    String_Const_u8 lexeme = push_token_lexeme(app, scratch, buffer, token);
    Code_Index_Note *note = code_index_note_from_string(lexeme);
    if (note != 0)
    {
      switch (note->note_kind)
      {
        case CodeIndexNote_Type: token_color = color_type; break;
        case CodeIndexNote_Function: token_color = color_function; break;
        case CodeIndexNote_Macro: token_color = color_macro; break;
        default: {} break;
      }
    }
  }
  paint_text_color(app, text_layout_id, Ii64_size(token->pos, token->size), token_color);
}
```

This brings us new timings:

```
test_large.cpp
# (total lookup depth / # lookups) = avg lookup depth

28194334/2029647=13.891250    @V1 - no changes
  Avg Bucket Depth: 3.153940
  Max Bucket Detph: 498

28194334/ 951725=29.624455    @ V2 - only lookup in cache if token isn't a known color.
  Avg Bucket Depth: 3.153940
  Max Bucket Detph: 498
```

At first glance this is worse! Our average case went from a depth of ~14 to ~30! The thing that improved is that we're performing far fewer lookups - from 2029647 -> 951725, 46% of the original. That's decent. Performance is still bad though.
(You might be wondering why, after one frame drawing the large file, we're performing 900k lookups - I didn't think about it until later but in retrospect, it is an obvious red flag)

My next stop was to increase the hash table size - maybe we can reduce collisions by making there be more buckets. That's easy, since the hash table is statically sized to 4099, lets just bump it to 10000 and see if that changes anything.

```
test_large.cpp
# (total lookup depth / # lookups) = avg lookup depth

28194334/2029647=13.891250    @V1 - no changes
  Avg Bucket Depth: 3.153940
  Max Bucket Detph: 498

28194334/ 951725=29.624455    @ V2 - only lookup in cache if token isn't a known color.
  Avg Bucket Depth: 3.153940
  Max Bucket Detph: 498

28688848/ 951725=30.144052   @ V3 - increase name_hash size
  # I confes I accidentally removed the avg and max bucket depths here.
```

Wait this made it worse??? At this point I printed out the bucket depths for the file.
```
0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 497, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 497, 0, 0, 0, 0, 0, 0, 0, 0, etc...
```

Ok so this points at something being wrong. Most of the buckets are empty and we have a few scattered about that are getting a lot of collisions. Looking at the test_large.cpp file, it's possibly not a great test candidate - it's a few 1000 lines of code copy pasted again and again to reach the 9MB file size. Since our bottle neck is hash table lookups for tokens, this is indeed a bad test case. So I went and downloaded sqlite3.c which is a comparably large file and reran some timings:

```
sqlite3.c
# (total lookup depth / # lookups) = avg lookup depth

32566469/8234655=3.954807         @V1 - no change
  Avg Bucket Depth: 1.530861
  Max Bucket Detph: 18

986806/548977 =1.797536         @V2 - only lookup in cache if token isn't a known color.
  Avg Bucket Depth: 1.530861
  Max Bucket Detph: 18

457645/548977 =0.833632         @V3 - increase name_hash size
  Avg Bucket Depth: 0.627500
  Max Bucket Detph: 13
```

And here's the bucket depth printout:
```
 0, 1, 0, 3, 3, 7, 1, 0, 4, 2, 3, 0, 0, 1, 1, 4, 0, 1, 1, 0, 1, 4, 2, 2, 0, 1, 4, 0, 1, 3, 5, 0, 1, 2, 2, 1, 1, 0, 0, 1, 3, 2, 2, 2, 1, 0, 4, 5, 1, 0, 0, 2, 2, 0, 0, 1, 0, 1, 2, 3, 3, 1, 1, 6, 1, 1, 1, 1, 1, 1, 4, 2, 3, 3, 7, 1, 2, 6, 3, 2, 1, 1, 1, 0, 0, 0, 0, 3, 0, 5, 0, 0, 6, 1, 1, 0, 0, 1, 0, 1, 1, 0, 0, 1, 1, 6, 2, 0, 3, 2, 0, 2, 2, 3, 0, 0, 1, 1, 2, 0, 0, 1, 3, 2, 2, 1, 1, 2, 1, 1, 0, 1, 0, 2, 1, 0, 0, 1, 0, 1, 6, 1, 3, 1, 1, 1, 1, 0, 2, 5, 3, 0, 1, 2, 1, 0, 2, 3, 2, 1, 1, 1, 6, 1, 0, 2, 2, 2, 2, 1, 0, 1, 1, 1, 0, 1, 1, 1, 2, 2, 5, 3, 0, 1, 0, 3, 0, 0, 3, 1, 2, 0, 1, 1, 4, 1, 1, 1, 0, 3, 1, 2, 1, 0, 0, 4, 1, 3, 0, 2, 3, 0, 0, 2, 0, 4, 0, 0, 0, 3, 2, 0, 0, 2, 1, 1, 0, 2, 1, 2, 1, 3, 0, 3, 0, 1, 2, 1, 1, 2, 2, 3, 0, 3, 0, 2, 0, 2, 1, 0, 2, 1, 5, 4, 2, 2, 2, 2, 2, 1, 2, 0, 2, 2, 1, 2, 2, 1, 0, 2, 4, 1, 0, 0, 6, 1, 1, 0, 7, 3, 0, 0, 1, 1, 0, 1, 2, 3, 2, 3, 1, 3, 2, 2,
```

Okay, that's better - with real code the hash table is performing more as we'd expect. This is important because if it hadn't, my next focus would have been on the hash function. Having ruled that out, I returned to the token coloring routine.

And it was at this point that I noticed something very obvious: The token iterator starts at the first visible token, but is never told how many tokens to iterate over. The only range information it recieves is from the token list - the list of all tokens in the file. So I did a quick gut check test - I scrolled to the end of the large file and sure enough, performance got better the closer I got. It was coloring tokens from the first visible token all the way to the end of the file. (This is the source of those 900k lookups I mentioned earlier)

Well that's easy enough to fix - simply set the count on that token iterator to the visible token count.
```c
Token_Iterator_Array it = token_iterator_pos(0, &token_array, visible_range.first);
it.count = Min(it.count, visible_range.one_past_last - visible_range.first);
for (;;){
  // continued as above...
}
```

And with that change, we get a new set of timings:

```
sqlite3.c
# (total lookup depth / # lookups) = avg lookup depth
32566469/8234655=3.954807         @V1 - no change
  Avg Bucket Depth: 1.530861
  Max Bucket Detph: 18

986806/548977 =1.797536         @V2 - only lookup in cache if token isn't a known color.
  Avg Bucket Depth: 1.530861
  Max Bucket Detph: 18

457645/548977 =0.833632         @V3 - increase name_hash size
  Avg Bucket Depth: 0.627500
  Max Bucket Detph: 13

644/689=0.934688             @V4 - only color the visibile tokens
  Avg Bucket Depth: 0.627500
  Max Bucket Detph: 13
```

First, the editor is simply usable at this point. But quantitatively, we are doing far far fewer lookups in the code index table.

Just to check I ran some final timings on our original file - I don't need that case to work since it's not representative but it would still be nice to know we improved things for that case anyways.

```
test_large.cpp
# (total lookup depth / # lookups) = avg lookup depth

28194334/2029647=13.891250    @V1 - no changes
  Avg Bucket Depth: 3.153940
  Max Bucket Detph: 498

28194334/ 951725=29.624455    @ V2 - only lookup in cache if token isn't a known color.
  Avg Bucket Depth: 3.153940
  Max Bucket Detph: 498

28688848/ 951725=30.144052    @ V3 - increase name_hash size
  # I confes I accidentally removed the avg and max bucket depths here.

6960/235=29.617021            @ V4 - only color visible tokens
  Avg Bucket Depth: 1.292800
  Max Bucket Detph: 498
```

And it's still faster, simply by nature of reducing the number of lookups to a managable count. At this point I was satisfied with my attempts and committed the changes.

## Conclusion

This isn't terribly interesting, but I thought it could be illustrative of how I go about debugging your run of the mill performance problems.

Some thoughts:
- Using a debugger as a probabalistic bottleneck identifier is an interesting technique. It's not a formal debugger feature though, and that could be an interesting idea - simply run your application and ask the debugger to sample what's running every 5 seconds and report those call stacks to you. (I recognize this is basically a sampling profiler)
- Logging information over time is really useful - terminals are poorly equipped to present this information usefully. It would be nice to be able to dump information to buckets then view those buckets temporally next to each other in nice collapsable ways.

Thanks for reading!
