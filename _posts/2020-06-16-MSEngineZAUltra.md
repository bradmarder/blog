---
layout: default
title:  "MSEngine ZA Ultra"
date:   2020-06-16 13:35:03 -0400
tags: minesweeper
---

# MSEngine ZA Ultra
### My journey creating a zero allocation minesweeper engine and solver

Before I begin this series of posts documenting this project, I have a confession to make.

> I have never actually played a single minesweeper game using this engine

It seems somewhat ironic to spend countless hours programming such an iconic game and never bother to manually play it. It's really just a combination of not wanting to play from the CLI or build out a front end. If I do ever get around to it, it will definitely be with Unity though. Or maybe the canvas with React. Or Blazor. Probably best to limit the scope, since this has already taken vastly longer than originally anticipated.

## What/Why is MSEngine ZA Ultra?

The initial goal of this project was just to create a simple minesweeper engine containing all the required game logic, but without the UI. I wanted to contribute my first open source project and NuGet package that someone could actually use and implement a full minesweeper game with. (Whether someone has taken on that endeavor, I don't know). I also wanted to experiment with automated testing, since my programming career has not yet afforded that possibility.

After the initial engine was created along with a suite of tests, I figured it only made sense to take a stab at implementing an automated solver. I did not realize at the time what a rabbit hole this would take me down. To this day, I'm still not sure I understand the true scope of a probabilistic solver. 

Anyway, after writing a ton of code and eventually getting bored, I needed to think of something to renew my interest in continuing this project. There are three main overlapping drivers for this project now; a matrix/linear algebra based solver, a desire for blazing performance, and learning/using SIMD/instrinsics.

## A Few Notes before the Journey Begins

* Minesweeper/solver is simple to learn, but brutally and deceptively difficult to implement

* If you google minesweeper and play the game on the 2nd link, you may encounter one of the first bugs/edge cases I came across. Imagine a corner tile (or Node as I prefer) surrounded by flags. What should happen if you reveal the corner node and it *does not* contain a mine, and the three flags surrounding it *do not* actually have mines either? It's a weird edge case that I wouldn't expect to occur naturally, but is precisely the type of case I want my engine/tests to consider. (I believe the correct (and original) minesweeper engines would prevent the node revealing chain reaction).

* Even the *first* online minesweeper link has a bizarre bug. Consider the following game...what should happen if the green node is revealed *and* has zero adjacent mines? We can clearly see the top left node has zero adjacent mines...therefore the blue node below definitely does not have a mine. 

![minesweeper bug before reveal](/blog/Images/BugBeforeReveal.jpg)

..and the aftermath. Note the blue node remains *hidden*.

![minesweeper bug after reveal](/blog/Images/BugAfterReveal.jpg)

Seems like an obvious bug to me. Maybe this logic is true to the original windows 3.1 version? This is something I would like to explore further. Once again, this realistically doesn't impact normal play, but could potentially interfere with an AI/ML approach.

* Minesweeper intermediate/expert boards are defined at 16/16/40 and 30/16/99, but beginner boards can be considered 8/8/10 or 9/9/10. 

* Some implementations ensure the first revealed node does not have a mine, and has an adjacent mine count of zero. Personally, I prefer that approach because otherwise you may be forced to guess on *the second turn*, which significantly reduces the odds of winning.

* Every single time I read over the code, I managed to find possible performance enhancements. It always felt good to pick the low hanging fruit, but there was always the oppurtunity to shave off a few microseconds by some obscure technique.

* Achieving zero allocations felt like a great accomplishment. Without a doubt, it made parts of the code *uglier*, and the inability to use LINQ is depressing, but zero allocations are a requirement for ultra performance. 

* Ensuring the mines are randomly/cryptographically scattered is more difficult than I initially assumed.

* I did my best to ensure that the engine/solver/app remain distinct from each other. For example, solver may only take a a `Matrix<Node>` and return `Turns`. The solver never has the ability to mutate the board. The app is responsible for generating a new board if the initial turn would reveal a mine (in the original minesweeper, the engine would just "relocate" the mine to the top left)

* You could write an entire PhD on implementing a probabilistic solver using AI/ML, because the scope is so vast. To my knowledge, nothing like that even exists yet.

* There is still a ton I don't know, and some stuff I definitely got wrong. Please let me know if you find anything. 

## Show me your code
The most complicated aspect of any minesweeper engine is without a doubt the "Chain Reaction" logic. Revealing a node with zero adjacement mines will trigger this chain reaction, revealing other nodes and cascading this behavior until certain boundaries are encountered (flags/mines). I've refactored this snippet of code a ton already, but there is still room for improvement. It may look simple/clean now, but the initial version was an ugly mess (something I assume most people encounter). 

```c#
 internal static void VisitNode(Matrix<Node> matrix, int nodeIndex, ReadOnlySpan<int> visitedIndexes, Span<int>.Enumerator enumerator)
 {
     Debug.Assert(nodeIndex >= 0);

     enumerator.Current = nodeIndex;
     var pass = enumerator.MoveNext();
     Debug.Assert(pass);

     Span<int> buffer = stackalloc int[MaxNodeEdges];
     buffer.FillAdjacentNodeIndexes(matrix.Nodes.Length, nodeIndex, matrix.ColumnCount);

     foreach (var i in buffer)
     {
         if (i == -1) { continue; }

         ref var node = ref matrix.Nodes[i];

         if (node.State == NodeState.Flagged) { continue; }

         if (node.State == NodeState.Hidden)
         {
             node = new Node(node, NodeState.Revealed);
         }

         if (node.MineCount == 0 && !visitedIndexes.Contains(i))
         {
             VisitNode(matrix, i, visitedIndexes, enumerator);
         }
     }
 }
```

## Ok then

I think that is enough for this first post. In the next, I want to discuss deterministic solver strategies, and why that is so difficult to code.

{% include minesweeper.html %}