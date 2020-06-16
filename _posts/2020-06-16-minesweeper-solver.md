---
layout: default
title:  "Minesweeper Solver Strategies"
date:   2020-06-16 13:35:03 -0400
categories: minesweeper
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
