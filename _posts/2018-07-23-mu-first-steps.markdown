---
layout: post
title:  "Mü, the card game, in Scala. First steps"
date:   2018-07-23 19:47:04 +0100

categories: jekyll update
---
## Mü, a card game.

I will be making an implementation of the card game [Mü](https://boardgamegeek.com/boardgame/152/mu-more), by Doris Matthäus and Frank Nestel. It is a trick taking card game simlilar to Bridge but with a custom deck of cards. The make-up of the deck and the rules of Mü can be found [here](http://riograndegames.com/uploads/Game/Game_236_gameRules.pdf) or just the rules can be found in [html form here](http://www.gamecabinet.com/rules/Mu.html). 

This will be an ongoing project, purely for the fun of it, while hopefully learning some new Scala concepts and libraries. Later, I hope to implement an A.I. for it and make it playable against opponents or against computer opponents. I also plan to implement the user interface in Javascript, perhaps in Scala JS. The code for it will be hosted in [github](https://github.com/JoeCordingley/Mu). It's very early days yet and I will probably make many missteps so lets just see how it goes.

## Representing the Rules.
The first thing I will be focusing on is representing the rules of the game. I will have to maintain the state of the game, and also represent the flow of the game. Board games and card games have quite a complex flow to them so this will take some care to represent well.

The game has two main phases which generally alternate until the end of the game. The **Auction** phase, and the **Card Play** phase. The Auction phase is the one that happens first so I will focus on that one first.

### Auction Phase Flow

The Auction phase basically consists of each player either placing one or more cards in front of them or passing until all players have passed. Then depending on the outcome of the bidding and the number of players, some of the players may get to choose the trumps for the round and the winner may get to choose their partner.

I thought it wise to separate the representation of game state from game flow and also to defer monadic context as much as possible. This allows me to use the same representation for multiplayer games which will be asynchronous - as test games, single player games and A.I. training which may be represented differently.

I decided therefore to represent this with the [Free Monad](https://typelevel.org/cats/datatypes/freemonad.html) pattern using the [Cats](https://typelevel.org/cats/) library.
```scala
sealed trait AuctionADT[A]
type AuctionFree[A] = Free[AuctionADT, A]
def play: AuctionFree[ResolvedAuction] =
  for {
    status <- getStatus
    finish <- status match {
      case FinishedAuctionStatus(outcome) =>
        outcome match {
          case resolved: ResolvedAuction => finishAuction(resolved)
          case ChiefAndVice(chief, vice) =>
            for {
              viceTrump <- getTrump(vice)
              chiefTrump <- getTrump(chief)
              partner <- getPartner(chief)
            } yield TwoTrumps(chief, chiefTrump, partner, viceTrump)
          case ChiefOnly(chief) =>
            for {
              trump <- getTrump(chief)
              partner <- getPartner(chief)
            } yield OneTrump(chief, trump, partner)
          case ChiefThreePlayers(chief) =>
            getTrump(chief).map(trump => ThreePlayerFinish(chief, trump))
        }
      case UnfinishedAuctionStatus => getBidFromNextPlayer.flatMap(_ => play)
    }
  } yield finish
```
Essentially the flow is just check if the play has finished, if so resolve the final details, if not get the next play and recurse.

Most of the functions such as `getStatus` just lift the ADT representation of the stage of flow into the Free context like this:

```scala
def getBid(player: Player): AuctionFree[Bid] = liftF(GetBid(player))
```

Except for `finishAuction` which deals with the *Eklat* cases which just needs to wrap them into the AuctionFree context. We can just use the unit constructor `pure` here.

```scala
def finishAuction(finish: ResolvedAuction): AuctionFree[ResolvedAuction] =
  Free.pure(finish)
```

And `getBidFromNextPlayer` which is defined in terms of other `AuctionFree`s.
```scala
def getBidFromNextPlayer: AuctionFree[Unit] =
  for {
    player <- nextPlayer
    bid <- getBid(player)
    _ <- placeBid(bid)
  } yield ()
```

## Conclusion
I thought it was pretty cool to work with the stages of computation first. It allowed me to define what I thought was the cleanest way of representing the flow of the code before even implementing anything. And it allows me to work with many different monadic contexts by later providing different interpreters. They will most likely involve the *State* Monad and the *IO* monad for player interaction.
