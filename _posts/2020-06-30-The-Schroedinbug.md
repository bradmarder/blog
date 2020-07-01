---
layout: default
title:  "The Schroedinbug"
date:   2020-06-30 13:35:03 -0400
tags: minesweeper
---

Taken verbatim from Wikipedia

> A schrödinbug or schroedinbug (named after Erwin Schrödinger and his thought experiment) is a bug that manifests itself in running software after a programmer notices that the code should never have worked in the first place

For some reason, I noticed that when solving intermediate boards, there would be massive pauses. The CPU would be spinning...but boards were not being solved. At first I just dismissed this because I was tired, and everything seemed to work perfectly on beginner and expert. Something about intermediate...I just couldn't explain it. I think I went about two days pondering what the issue could be, until I finally decided it was just to just debug it. 

The first aspect that peaked my curiosity was the win rate (it was precisely as expected). Clearly, whatever bug was at hand was *not* impacting our overall success. It just seems like the code is getting caught in an infinite loop at some point,and eventually *breaks* out of that loop...but where could that possibly happen? I tossed in some `StopWatch`s all over the place to find the culprit.

### Before the "turn" that took forever
```
0001>1__________
000222__________
0001>1__________
001222__________
113>33__________
1>4>____________
24______________
________________
________________
________________
________________
________________
________________
________________
________________
________________
```

### Afterwards
```
0001>10000001___
0002220000001___
0001>10000011___
001222110002__32
113>33_20002__10
1>4>___200012210
24_____100000000
_______210111000
___312__101_2100
___201221012_100
___1000000012321
___1110000001___
_____10001222___
_____21101______
_______202______
_______201______
```

Very interesting! That is one *massive* chain reaction, so clearly something is up with that code. Let's take a look at the bread and butter of it.

```c#
[MethodImpl(MethodImplOptions.AggressiveInlining)]
internal static void VisitNode(Matrix<Node> matrix, int nodeIndex, ReadOnlySpan<int> visitedIndexes, Span<int>.Enumerator enumerator)
{
    Debug.Assert(nodeIndex >= 0);

    var pass = enumerator.MoveNext();
    Debug.Assert(pass);
    enumerator.Current = nodeIndex;

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

This code snippet is responsible for handing the chain reaction logic, where revealed nodes cascade and reveal other nodes. It was a straightforward recursive algorithm, and *technically*, this version works. Before reading the spoiler, if you think you are a c# guru, take a minute to try to find the bug.



















-- Keep scrolling for the spoiler!
















Ok you have scrolled far enough. The offending code is this `Span<int>.Enumerator`. I failed to take precaution when handling a mutable struct. The Enumerator maintains it's own state, but when you pass it as a parameter, it is *copied* instead of maintaining a reference to the original (which was the intent). A quick change to passing it as `ref` was enough to prevent the copying, but it didn't quite explain *why* it still worked or took an inordinate amount of time to calculate. Even if you told me to explicitly make this algo take a long time, I don't even think I could! I mean, when it works, it's time is measured in *microseconds*. How on earth could you make it many magnitudes slower with such a low N?! I feel like I inadvertently created a "bogosort" of kind. I wanted to create a test for this, but couldn't think of anything meaningful. If you have suggestions, please let me know.

Oh yeah, and after fixing this, everything (including beginner/expert boards) are drastically faster. Hooray!

{% include minesweeper.html %}




































