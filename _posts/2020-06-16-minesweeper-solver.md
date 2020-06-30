---
layout: default
title:  "Minesweeper Solver Strategies"
date:   2020-06-16 13:35:03 -0400
tags: minesweeper
---

# Minesweeper Solver Strategy Chaos

With the minesweeper engine complete and tested, it was only natural to attempt to create a solver. A quick glance at this [site](http://www.minesweeper.info/wiki/Strategy) gave me a false sense of confidence. I assumed I could just grind out some code and hammer away at it until it would "solve" every puzzle I threw at it (except guessing of course). This is just *one* of the strategies I employed, and even with LINQ, this was a complete eyesore.

```c#
    public static class PatternStrategy
    {
        public static bool TryUseStrategy(Board board, out Turn turn)
        {
            if (board == null) { throw new ArgumentNullException(nameof(board)); }

            var revealedTilesWithAMC = board.Tiles
                .Where(x => x.State == TileState.Revealed)
                .Where(x => x.AdjacentMineCount > 0)
                .ToList();
            var hiddenTiles = board.Tiles
                .Where(x => x.State == TileState.Hidden)
                .ToList();
            var flaggedTiles = board.Tiles
                .Where(x => x.State == TileState.Flagged)
                .ToList();
            var primaryTileToNextTilesMap = revealedTilesWithAMC.ToDictionary(
                x => x,
                x => revealedTilesWithAMC
                    .Where(y => Utilities.IsNextTo(y.Coordinates, x.Coordinates))
                    .ToList());

            foreach (var primary in primaryTileToNextTilesMap)
            {
                if (!primary.Value.Any())
                {
                    continue;
                }

                var primaryFlaggedAjacentTileCount = flaggedTiles.Count(x => Utilities.IsAdjacentTo(x.Coordinates, primary.Key.Coordinates));
                var primaryHiddenAdjacentTiles = hiddenTiles
                    .Where(x => Utilities.IsAdjacentTo(x.Coordinates, primary.Key.Coordinates))
                    .ToList();

                foreach (var secondary in primary.Value)
                {
                    var secondaryHiddenAdjacentTiles = hiddenTiles
                        .Where(x => Utilities.IsAdjacentTo(x.Coordinates, secondary.Coordinates))
                        .ToList();
                    var secondaryExtraTiles = Enumerable
                        .Except(secondaryHiddenAdjacentTiles, primaryHiddenAdjacentTiles)
                        .ToList();

                    if (!secondaryExtraTiles.Any())
                    {
                        continue;
                    }

                    var secondaryFlaggedAjacentTileCount = flaggedTiles.Count(x => Utilities.IsAdjacentTo(x.Coordinates, secondary.Coordinates));
                    var sharedHiddenTileCount = Enumerable
                        .Intersect(primaryHiddenAdjacentTiles, secondaryHiddenAdjacentTiles)
                        .Count();

                    // we know there are n mines in the sharedHiddenTiles
                    var sharedHiddenTileMineCount = primary.Key.AdjacentMineCount - primaryFlaggedAjacentTileCount;
                    var extraMineCount = secondary.AdjacentMineCount - secondaryFlaggedAjacentTileCount;

                    if (primaryHiddenAdjacentTiles.Count == sharedHiddenTileCount && extraMineCount == sharedHiddenTileMineCount)
                    {
                        turn = new Turn(secondaryExtraTiles.First().Coordinates, TileOperation.Reveal);
                        return true;
                    }

                    if (extraMineCount > sharedHiddenTileMineCount && (extraMineCount - sharedHiddenTileMineCount) == secondaryExtraTiles.Count)
                    {
                        turn = new Turn(secondaryExtraTiles.First().Coordinates, TileOperation.Flag);
                        return true;
                    }
                }
            }

            turn = default;
            return false;
        }
    }
```

Technically this worked. And this is only one of the four strategies. At the time, I felt content with the project and decided to leave it as is. But over the course of the next two years, I had this nagging suspicion there must be a cleaner approach. If this was posted on codegolf.stackexchange.com, I have no doubt some genius would come up with an APL solution that is less than 10 bytes or something equally absurd. I knew in the back of my mind there had be *some* way to use a matrix, execute an obscure mathematical algorithm and obtain a set of turns. Fast forward two years...

I finally took the plunge and did a google search for "minesweeper matrix solver" and came across this post by [Robert Massaioli](https://massaioli.wordpress.com/2013/01/12/solving-minesweeper-with-matricies/). The basic gist of this approach is to use linear algebra, create an augmented matrix, apply gaussian elimination (reduced row echelon form), and then apply some heuristics to the resulting matrix in order to determine which nodes may be flagged/revealed. The approach seemed simple and beautiful and exactly what I was looking for. Mission accomplished. I ran millions of iterations and it seemed to work flawlessly...except when it didn't.

Part of my testing strategy has been to compare "stalemate" boards between my original approach and the matrix technique (stalemate implying board is complete or requires guessing). I started to notice a few cases where my approach would calculate a turn and the matrix version would not. Naturally I assumed I had a bug in my implementation. There definitely could not be anything "wrong" with matrix version for two reasons. 1) There is a blog post about it 2) It's just linear algebra at it's core. So I applied lots of standard debugging behavior, hammering away at the code, testing, deciphering rosetta stones etc. I eventually discovered a board which defied explanation. 

```
00001_10
11212_10
1!2!2110
22312_10
2!201_32
3!3112__ <-- these 2 hidden nodes may be safely flagged
________
________
```

The matrix solver was unable to calculate any turns for this board. It was just cursed in some manner. Maybe the board had some impossible configuration? Perhaps my code had some obscure bug that only manifested itself with such a specific configuration? After much more head scratching and deliberation, I finally came to a penultimate conclusion.

### The matrix solver simply doesn't (always) work (as described in the blog post)

The author included a link to his c++ code which perhaps had different behavior than described in his post, but I didn't investigate (my c++ is f--). Why this board configuration isn't handled properly, I do not know. My best guess at this point in time is that we somehow "lose" critical information when we apply guassian elimination, and the resulting matrix is insufficient. This revelation depressed me. If anyone reading this has any insight into this matter, I would love to know. Defeated, I resorted to some recursive heuristics to manipulate the matrix pre-guassian-elimination and calculate some turns. The following code is ugly, but it works, is performant, and solves the problem for now. I also added some tests for this specific case so the solver will never regress, and perhaps a future iteration *will* be able to handle it.

```c#
private static void ReduceMatrix(
    Matrix<Node> nodeMatrix,
    Matrix<float> matrix,
    ReadOnlySpan<int> adjacentHiddenNodeIndexes,
    ReadOnlySpan<int> revealedAMCNodes,
    Span<Turn> turns,
    ref int turnCount,
    bool useAllHiddenNodes)
{
    var hasReduced = false;
    Span<int> buffer = stackalloc int[Engine.MaxNodeEdges];

    foreach (var row in matrix)
    {
        var val = row[matrix.ColumnCount - 1];

        // if the augment column is zero, then all the 1's in the row are not mines
        if (val == 0)
        {
            for (var c = 0; c < matrix.ColumnCount - 1; c++)
            {
                if (row[c] == 1)
                {
                    hasReduced = true;

                    var turn = new Turn(adjacentHiddenNodeIndexes[c], NodeOperation.Reveal);

                    TryAddTurn(turns, turn, ref turnCount);
                    ZeroifyColumn(matrix, c);
                }
            }
        }

        // if the sum of the row equals the augmented column, then all the 1's in the row are mines
        if (val > 0)
        {
            float sum = 0;
            for (var y = 0; y < matrix.ColumnCount - 1; y++)
            {
                sum += row[y];
            }
            if (sum == val)
            {
                for (var c = 0; c < matrix.ColumnCount - 1; c++)
                {
                    if (row[c] == 1)
                    {
                        hasReduced = true;

                        var index = adjacentHiddenNodeIndexes[c];
                        var turn = new Turn(index, NodeOperation.Flag);

                        TryAddTurn(turns, turn, ref turnCount);
                        ZeroifyColumn(matrix, c);

                        buffer.FillAdjacentNodeIndexes(nodeMatrix.Nodes.Length, index, nodeMatrix.ColumnCount);

                        foreach (var i in buffer)
                        {
                            if (i == -1) { continue; }
                            var rowIndex = revealedAMCNodes.IndexOf(i);
                            if (rowIndex != -1)
                            {
                                matrix[rowIndex, matrix.ColumnCount - 1]--;
                            }
                        }

                        if (useAllHiddenNodes)
                        {
                            matrix[matrix.RowCount - 1, matrix.ColumnCount - 1]--;
                        }
                    }
                }
            }
        }
    }

    if (hasReduced)
    {
        ReduceMatrix(nodeMatrix, matrix, adjacentHiddenNodeIndexes, revealedAMCNodes, turns, ref turnCount, useAllHiddenNodes);
    }
}
```

Hooray, we have finally won the battle! Except that we didn't. Yet again, I was matched with another obscure bug. This time, it was very rare, only occurring about *once* every few hundred thousand iterations. I was only able to discover this by comparing the old solver with the new one, and assuming the new version should *at least* solve everything the old one could. Basically, our solver would fail to calculate a turn for a board when there obviously was one. I inspected the board and nothing seemed to stand out.

```
111>211_
>233__1_
2>2>_211
1133____
001>2___
001122__
00001___
00001___
```

No exceptions were thrown, no odd behavior...just nothing to really go on. This required some tedious line-by-line debugging. At some point, I was erroneously convinced the algo just couldn't handle this scenario. At some point, I manually tested my Gaussian elimination code against a known working version and eureka. I noticed something that hasn't ever happened in hundreds of thousands of games; the correct matrix had *fractions*, and our matrix was just using ints. Ugh. A simple switch to using floats was all that was required. I added a test to make sure we never regress.

```c#
[Fact]
public void CalculatesTurnsWhenGaussianEliminationProducesNonIntegers()
{
    Span<Node> nodes = stackalloc Node[] {...}; // removed hardcoded nodes for brevity
    Span<Turn> turns = stackalloc Turn[nodes.Length];
    var matrix = new Matrix<Node>(nodes, 8);

    var turnCount = MatrixSolver.CalculateTurns(matrix, turns, false);

    Assert.Equal(2, turnCount);

    // changing float to sbyte will cause our solver to generate incorrect turns
    Assert.NotEqual(6, turnCount);

    Assert.Equal(new Turn(29, NodeOperation.Reveal), turns[0]);
    Assert.Equal(new Turn(61, NodeOperation.Reveal), turns[1]);
}
```

In the next part of this series, I'll discuss yet another obscure bug that *only* manifested itself on intermediate

{% include minesweeper.html %}
