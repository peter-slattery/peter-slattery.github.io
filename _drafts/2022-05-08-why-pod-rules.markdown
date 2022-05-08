---
layout: post
title:  "Why Plain Ol' Data Rules"
date:   2022-05-07 03:41:30 -0700
categories: [development]
---

Here's a good example of why POD structs rule, and classes (as done often today) are a bad idea.
I'm working on a project in Swift, where classes are 'reference types'. What this means is that when I say:

```javascript
class foo { var x: int; }
var myFoo = foo(x: 5);
```

myFoo is really the address of the object I just created. For low-level language folks, myFoo is a pointer. So this will happen:
```javascript
var fooA = foo(x: 5);
var fooB = fooA;
fooB.x = 10;
print("\(fooA.x)"); // will print 10
```

This is because fooB was receiving the address of the object that fooA pointed to, not the object itself. For the C/C++ people out there, its equivalent to:
```c
foo* fooA = (foo*)malloc(sizeof(foo));
fooA->x = 5;

foo* fooB = fooA;
fooB->x = 10;
printf("%d", fooB->x); // prints 10
```

And sometimes this is the behavior you want. The problem is that in Swift, without doing extra work, its the only behavior you can get.

Say I wanted fooB to be its own object and just inherit the values of fooA. Well, the way Swift wants you to do it, you'd have to do something along the lines of:
```javascript
class foo {
  var x: Int;

  func copy (oldFoo: foo) -> foo {
    return foo(x: oldFoo.x);
  }
}
```

So already, I have to write a function that manually copies every field of foo, which is the worst case scenario. 

Now, lets turn to the case I'm actually dealing with today.

I have a thing called a Pose in a game. Its a unique Id, a collection of Targets (dots you hit with a hand or foot), and an update function. This is the actual code:
```javascript
protocol PoseModel {
  var id: UUID { get }
  var targets: [HitTarget] { get set }
  func update(locationGroup: TA_ExtremitiesLocationGroup) -> Bool
}
```
*(ignore that it's a protocol, it's basically just a virtual base class which... is it's own ball of string to untangle, but that's for another post)*

Now, DotModel doesn't do anything on its own, it's implemented by specific poses. Basically all the different poses change are the init and update functions. So you have things like:
```javascript
class StationaryPose: ObservableObject, PoseModel {
  let id = UUID()
  @Published var targets: [HitTarget]

  init() { ...omitted for brevity }
  func update(locationGroup: TA_ExtremitiesLocationGroup) -> Bool {
    guard let target = targets.first,
          let extremity = locationGroup.getByKind(target.extremity) else { return false }
    return target.contains(extremity.p)
  }
}
```
and about 6 others like it. None too complicated, the longest is 80 lines. One thing to keep in mind, and we'll come back to this later, is that subclasses can add member fields on top of id and targets.

Now, I also have this:
```javascript
let poseSequence: [PoseModel];
let poseSequenceCurrFramePerPlayer: [Int];
let posesPerPlayer: [[PoseModel]];
```

`poseSequence` is an array of poses. It's the timeline both players will advance through.
`poseSequenceCurrFramePerPlayer` contains two integers, one for each player. The integer is the index into `poseSequence` the corresponding player is on.
`posesPerPlayer` are the poses that are actually on screen right now.

Now, for reasons that are beyond this article, I need to copy PoseModel's from `poseSequence` into `posesPerPlayer` (the basics are that the HitTarget's they track need to be modified to position them on screen). Do you see the problem?

To copy a PoseModel, I need a copy function. But because subclasses can add fields, that means that each subclass needs it's own copy function. That's suddenly 6 functions I have to write just to perform one operation.

And the crazy thing is that Swift has structs, which are value types that can be passed by reference!
All it takes to fix this problem, and get all the copying I want for free, is:
```javascript
enum PoseKind {
  Pose_Stationary,
  Pose_Chain,
}

struct PoseModel {
  var id: UUID;
  var targets: [HitTarget];
  var kind: PoseKind;
}

struct PoseStationary {
  var base: PoseModel;
}
struct PoseChain {
  var base: PoseModel;
  var someVar: Int;
}

func PoseUpdate (pose: Any)
{
  var pose_ = PoseModel(pose);
  switch (pose_.kind)
  {
    case Pose_Stationary: PoseUpdate_Stationary(PoseStationary(pose)); break;
    case Pose_Chain: PoseUpdate_Chain(PoseChain(pose)); break;
  }
}
```
I guarantee the additional overhead (typing and performance) of having an enum and a single update function is far exceeded by all the copy functions I would have to write plus the cost of vtable lookups for those virtual update functions.

The reason we ended up not going this route is because passing structs by reference is 
