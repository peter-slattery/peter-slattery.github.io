---
layout: post
title:  "Anything Except Exceptions"
date:   2023-01-29 03:41:30 -0700
categories: [development, handmade]
---

*I've been having a discussion at work about how to handle errors, and whether exceptions are the right tool for the job. It's caused me to do a fair bit of research, and formulate my thoughts. This is the result.*

## The TL;DR
Most of the things we normally consider exceptions (or even errors really) actually aren't. They should be treated as first class codepaths. The complexity this involves is just the natural complexity of the problem, and you can't get any simpler than that.

If you want someone elses take on this, [here's Casey Muratori on the subject](https://guide.handmadehero.org/code/day177/#4098)

## Cases we call errors
The most common things I see exceptions used for fall into a few categories:
*This is certainly not an exhaustive list*

1. Invalid input/output arising from programmer error
2. Invalid input arising from user input
3. Valid input, paired with invalid external state
  - In memory state is incorrect for provided input
  - On-disk, or network state is incorrect for provided input
  - Missing or misconfigured remote resources (servers, etc).
4. A cosmic ray hits the hard drive, processor, etc.

When handling these categories, we have different priorities. Let's dig in.

## Case 1: Programmer Error

If the logic or code itself is incorrect, our priority is to notify the developer in a way that preserves as much context as possible - state of the stack, heap, memory (React State in our case, but that's kinda pedantic - its all memory). 

This is where asserts really shine. Consider this assert implementation (for js):
```js
function assert(condition: boolean) { if (!condition) { debugger; }}
```
Execution halts the moment the condition is evaluated to false. The entire program is in the state that caused the assert to fire. This gives the programmer the most context while investigating the problem.

Consider this alternative:
```js
if (!condition) throw new Error(message);
```
First, the VM will unwind the stack to the nearest catch statement in the hierarchy. Notably, this might not exist and your app crashes. 

If there is a catch block, the programmer might not have the opportunity to stop in the debugger before execution moves on. Even if they do, the programmer is now far removed from the context of the actual problem. The stack has been unwound, RAII style resources have been freed, etc. The programmer now has to go reproduce the bug without a lot of the context for why it happened. 

The most we can hope for is that the programmer who wrote the exception put a bunch of information regarding the problem into the message. But they have to do this on a case by case basis, whereas the `debugger;` version, by default, retains all the state of the entire program for free.

Finally, throwing an exception performs worst for the classes of bugs hardest to reproduce - race conditions, clock based operations, and the like. Your goal is to enable programers to solve those bugs while having to reproduce them the least number of times. With a breakpoint, your floor for that number is 0. With an exception, it's already at least 1.

By way of corroborating this with things that have been sent to me, see this discussion on [StackOverflow about Exceptions as Control Flow](https://softwareengineering.stackexchange.com/questions/189222/are-exceptions-as-control-flow-considered-a-serious-antipattern-if-so-why)

> As a quick summary for why, generally, it's an anti-pattern:
> - Exceptions are, in essence, sophisticated GOTO statements
> - Programming with exceptions, therefore, leads to more difficult to read, and understand code
> - Most languages have existing control structures designed to solve your problems without the use of exceptions

## An aside about effective asserts
There are a few important details to highlight regarding effectively using asserts. *You can skip this and come back later if you want.*

For the record, this wasn't something I came up with. [I learned it from Casey](https://youtu.be/tcENxzeTjbI?t=2427)

1. **They are a development/debug time tool.** Your build environment can and should strip them out of production builds. No user should ever encounter an assert.
In the best case, you can do something like C, with defines:
```c
#ifdef DEBUG
#  define assert(condition) assert_function(condition)
#else
#  define assert(condition) // transforms all occurances of assert to nothing
#endif
```
But in the worst case you should just do
```js
function assert(condition: boolean) {
  if (RELEASE_MODE === "debug" && !condition) debugger;
}
```
You could even transform asserts into logs that get sent back to you in production so you don't lose the information!

2. **Asserts Signal Intent.** You can wrap asserts in names that signal what their intent is to better communicate to future you. I like these variants:

```js
function assert(condition) { if (!condition) debugger; }
// the default assert. It says that "the data definition allows for values that this
// function does not."

function invalidCodePath() { assert(false); }
// useful for signaling a case that logically could happen, but given context
// never should. When you hit one of these, something outside the function
// has gone wrong.

function invalidDefaultCase() { assert(false); }
// Useful for defending against future refactors of data types where
// you switch on the type. If you extend, say an enum or string union
// and reach an existing switch, it will hit the invalidDefaultCase and 
// break in the debugger, signalling you to handle the new  case.
//
// In C you can make this syntactically nicer with:
//   #define invalidDefaultCase default: { assert(false); } break;

function handleMe() { assert(false); }
// A note that the codepath in question isn't invalid, but was not a priority
// to handle when the code was first written.
// We don't always need to be complete on our first pass, but we often have the
// context to know what cases need to be handled.
```

I see asserts as a means of ensuring errors do not enter your program.

Ok, back to our erroneous errors

## Case 2: Invalid User Input
Whether its a typed string, a file off disk, or actual code, nothing the user can input is an actual error. Handling invalid user input is a primary responsibility of your code.

In this case, providing user facing strings and signalling some UI change are the proper response, while aborting whatever execution you were performing. You have two choices here:

**Option 1. Throw an exception.**
This lets you centralize error handling, which is a good thing. But it comes at the cost of needing to make all your code RAII-esque, which definitively makes your problem more complex.
Consider this snippet:
```js
function loadAndCompile(path) {
  if (uiShowIsLoading) return false;
  uiShowIsLoading = true;
  const fileData = fs.readFileSycn(path);
  if (!fileData) throw new Error("File Not Found");
  uiShowIsLoading = false;
  return compile(fileData);
}
```
If `loadAndCompile` throws, the application will never be able to fully execute it again. This is a trivial example, but imagine if there were levels of nesting here, or that there are multiple dependents on uiShowIsLoading. Suddenly you have to be able to reason about state invalidation caused by a completely different part of the application.

**Option 2: Do the work to unwind your execution state**
```js
function loadAndCompile(path) {
  function handleFinishCompilation(result, message) {
    uiShowIsLoading = false;
    if (result) return result
    postErrorMessage(message);
    return invalidCompileResult;
  }
  if (uiShowIsLoading) return false
  uiShowIsLoading = true
  const fileData = fs.readFileSycn(path);
  if (!fileData) return handleFinishCompilation(false, "File Not Found");
  const result = compile(fileData);
  return handleFinishCompilation(result);
}
```
While this is more code, it doesn't pollute the entire rest of your program with incongruous state.

It separates error handling from communicating errors to your users without sacrificing your ability to handle errors where they occur. Additionally, stepping through this function in the debugger will behave predictably, leading to programmer understanding.

*Aside: Ideally, your programming language gives you a defer statement, which makes this sort of thing much easier to handle, also without exceptions. Something like:*
```js
function loadAndCompile(path) {
  if (uiShowIsLoading) return false;
  uiShowIsLoading = true;
  defer { uiShowIsLoading = false; }

  const fileData = fs.readFileSycn(path);
  if (!fileData) return postErrorMessage("File Not Found");
  return compile(fileData);
}
```

## Case 3: Mismatched Input and State
These kinds of issues can occur for a variety of reasons, which is why I listed them out above. But the way you handle them falls under one of the two categories we've already discussed. 

**Case 3.1: In-memory state not what's expected**

Either:
1. Some operation produced unexpected results - this is a programmer error. See Case 1.
2. Something got into memory from the user that is producing erroneous state. See Case 2.

**Case 3.2: On-disk or network state not what's expected**

*Example: you enumerate a directory's contents, and list the files. Then the user deletes a file from the file browser. Finally, the user clicks that files path in your application - a path to a file that no longer exists*
This is a Case 2 error - notify the user, recover state, move on.

*Example: a network packet arrives degraded, or out of order, etc.*
This is a Case 1 error - error correction codes, explicitly handling out of order, etc. 

**Case 3.3: Missing/Misconfigured Resources**

In this case you want the qualities of both case 1 and case 2. The developers need to know their distribution is in an invalid state even in production. And the user needs to know things aren't working right now so they don't get frustrated thinking you aren't working on it. 

## Case 4: Cosmic Rays

Hey, it's an actual exception! Literally, an exceptional event, one we would have to dedicate real, significant resources to in order to solve. Additionally, the resources involved probably far outweigh the likelihood of this happening, unless you are working on software which goes to space. *(And if that's the case, why are you reading my blog? You have cooler things going on.)*

In this case, I'd say it's still not a case for exceptions. I don't know how NASA handles this, (there is [this talk on Mars rovers](https://www.youtube.com/watch?v=3SdSKZFoUa8)) but I know exceptions are right out. And for us working on terrestrial computation, I think this is a case where it's ok to throw your hands up in surrender rather than throw an error.

# Exception Redirection
There is one other case we'll encounter: Code you call throws exceptions. Despite there being no reason to throw an error (as we've shown above) this is everywhere, from Node's `fs` module, to AWS services. 

In this case, the only thing you can do is encapsulate them, and convert them into one of the cases we've covered. This results in things like:
```js
function statSync(path) {
  let result = {}
  try { 
    result.value = fs.statSync(path);
  } catch (e) {
    result.error = e;
  }
  return result
}
```
This also leads us to defensively wrap our dependencies which is good for module isolation anyways. 

# One Last Nail in the Coffin
I've got one more reason not to use exceptions, and it's specifically directed at the argument of them being the standard, and therefore, make developers more comfortable in a codebase. 
*([I talked about this here](/care/))*

While I don't accept the "DX above all else" argument, I do think there's a cost benefit analysis to be made. *How much* easier do exceptions make programmers lives, in comparison to their cost?

Here's [Jon Blow on the subject of the cost of exceptions](https://youtu.be/TH9VCN6UkyQ?t=2377)

I can show a simple example of how they make things more complex even when used reasonably:
```js
// somewhere, in non-ui code
function doComplexProcess() {
  const text = loadFileOffDisk();
  const code = compileFileText(text);
  const evalResult = evalCompiledCode(code)
  displayCompiledCode(evalResult)
}

// elsewhere, in React
function handleButtonClick(event) {
  try {
    doComplexProcess()
  } catch (e) {
    // how many different errors could this catch?
  }
}
```

Say the codebase in question uses exceptions, and each of the four functions in `doComplexProcess` can throw. The ui just became responsible for handling errors from across the domains of the entire company. And a UI programmer's area of expertise is very different from a compiler engineer, and from that of an engineer responsible for platform interop, etc. 

Additionally, we have no idea what state the application is at the catch block. And the UI is going to have to try to recover from that. By allowing exceptions, we've made the implicit decision that there exist reachable, invalid code paths from which the application should be able to recover.

So by allowing exceptions you create dependencies between disparate code paths across domain boundaries, and make error handling more difficult by implicitly declaring the application should be able to recover.

# Conclusion

To sum up, there are different categories of things that can go wrong. The most important are:
1. Invalid state for the application to be in. These are things the programmer must fix for the program to be correct.
2. Invalid input supplied by a user. These are not actual errors, but need to be communicated to the user.
3. Literal chaos, which you can defend against, at a steep cost. Paying that cost may or may not be worth it based on the mission-critical-ness of your application.
*You may have cases other than these, in which case exceptions might be the right tool.*

Exceptions are a good solution to exactly **none** of the listed problems because:
1. They obfuscate important context from the programmer
2. They result in uncertain state, or complicate state mutation for your entire codebase.
3. They create unnecessary dependencies in your code
4. They complicate error handling. They don't differentiate between valid-but-erroneous user input and irrecoverable program state. As a result, programmers will attempt to make programs recover from all exceptions, leading to more error handling code or more brittle software.
5. (Not covered in this article, but very true) They are slooooooowwwwww. 

As far as alternatives go:
1. Asserts are a better tool for notifying developers of errors. 
2. Rely on domain specific solutions for notifying users of errors (ie. `alert()`, an in-app notification, console.log, etc).

At the end of the day, I turn exceptions off when I can, and minimize the amount they can infect my code when I can't. 