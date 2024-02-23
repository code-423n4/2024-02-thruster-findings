## [L-01] setMaxPrizeCount may result in a user can't claim the prize

## Proof of Concept
When claiming the prize, it will loop through ```maxPrizeCount``` to see if there is any match. However, ```setMaxPrizeCount()``` can be updated arbitrarily, assuming the previous ```maxPrizeCount``` is 3, and a user hasn't claimed the prize for prizeIndex 2 yet, if the ```maxPrizeCount``` is set to 2, the user can't claim the prize because the prize will not be looped to.

## [L-02] User may still enter tickets when the winning tickets is set

## Proof of Concept
When enter tickets, it will check the length of winning tickets prizeIndex 0 is zero:

```
require(winningTickets[currentRound_][0].length == 0, "ET");
```

But in ```setWinningTickets()```, it's possible for the admin to set other prizeIndex winning tickets first because there is no check to ensure the prizeIndex 0 winning tickets is first set. As a result, if prizeIndex 0 tickets is not set first, a user may still enter tickets.