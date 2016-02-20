---
layout: post
title:  "Dominion"
date:   2016-02-19
categories: analysis gamedev programming
---

One of the first board game clones I wanted to create was for Dominion. For clarity, game terms are denoted in *italics*, whereas Objects as defined in the code are denoted in `highlights`. The clone I was making is currently behind a private Github repository, but the source can be made available as an archive if asked for.

If you are familiar with the game, skip to the next section.

## Basic Gameplay Summary

1. Play Action cards
2. Play Treasure cards
3. Use Buys to purchase new cards
4. Clean up, and Draw Five new cards

Each player has their own *Deck* of 10 cards, starting with 7 *Copper* and 3 *Estates*. Estates are the lowest victory point value cards, and *Copper* provide 1 money for purchasing. Once a player runs out of cards in their *Deck*, the *Discard* is shuffled and becomes the new *Deck*. This means new cards that were purchased in previous turns have a chance of being drawn.

Where the game gets complex is in the *Action* cards. *Action* cards can cause you to Draw more cards, play more *Actions*, give your opponents *Curses*, cause them to *Discard* cards, or almost anything. There are hundreds of different cards in the game, all with different effects. Some of them React, or are variants on *Treasure* cards, or are *Victory* cards with a twist. Though, a typical game will only play with a random set of 10 kinds of *Action* cards. This way new players and pros are forced to evaluate new strategies each game, since its always different.

To the player, the general strategy is to determine what cards are best to buy and when, and then exactly when to start buying *Victory* cards before the game ends to amass more points than the opponents. Game Theory and player psychology come in handy.

I glossed over a lot of details and simplified a bit, but its enough for the rest of the post to make sense. I highly recommend the game, by the way.

## Original Design

For my clone, I wanted to do something with NodeJS, and kept the client-side really simple with basic CSS/HTML to keep the focus on the game mechanics. My first thought, given the nature of the game, was to use a Stack-based model for tracking an individual's phases and chained Actions. Figure 1 below is how it worked for really simple cases.

<br />
{:refdef: style="text-align: center;"}
![StateStack Flow Diagram]({{site.cdn_path}}/statestack_flow.png)
{: refdef}
<p class="caption">Figure 1</p>
<br />

Each `State` implemented a common Interface, which had `pushed()`, `execute()`, `popped()`, and `help()`. These four functions enabled a State to manage transitions, verifying inputs, and acting on the game state. `help()` is to hint to the player what they can do. All active `States` sat on the `StateStack` and were `pushed()` and `popped()` as the game progressed.

This all made intuitive sense, and I decided that the first most intuitive design that came to mind would probably be the most effective. Given that I wanted to start quickly and didn't quite take the time to fully assess the trade-offs and consequences of the design, work went straight ahead. At the time, refactoring didn't seem that intractable.

When a player's turn ended, the `TurnState` would "cheat" somewhat to prevent the `Stack` from ever empyting. Basically, it would `swap` itself on the `Stack` with a new `TurnState` during `execute()`. For various reasons, this was simpler than having a new `TurnState` pushed onto the `Stack` during its `popped()` call. This is where the cracks in the design started to show up first.

The `StateStack` itself was incredibly simple, with only `pushState()`, `popState()`, `swapState()`, and `executeCurrent()`. Similarly, keeping all complexity within the `States` instead of various other Objects meant that there was no "God" Object that controlled and knew everything, and also meant that everything was concerned with precisely what it was given and its own state.

From the outside, this seemed to indicate high cohesion, simple interactions, and low Object complexity.

`RestrictionContext` and `SelectionContext` were a huge improvement to help alleviate a lot of complicated rules checking for player inputs.  All a `State` had to do was declare the restrictions on what could be selected, and the `SelectionContext` was responsible for abstracting the nature of exactly how the cards were moving about. This also meant that `States` were no longer aware of what piles of cards they were operating on or how exactly an input was valid or not.

Another great part about this design was that cards could be defined in a data-driven manner, and then built up out of simple `State` definitions. This wasn't done in the project because of low-priority, but was definitely doable.

## Where it All Went Wrong

This actually worked quite well, but suffered from a major flaw: the `StateStack` had to be constantly scanned to assess the game state, as it didn't store any metadata about itself or the `States` on the `StateStack`. This caused issues with telling the player how many *Actions* or *Buys* they had. It also meant that if an *Action* added a *Buy*, then `States` would have to be interleaved into the `Stack`, breaking the core idea of a stack: first-in, first-out. Worse, if a player wanted to skip their turn, it meant having them confirm several `State` transitions until the `StateStack` emptied.

Figure 2 below demonstrates how the `StateStack` gets modified when the *Village* card (+1 *Card*, +2 *Actions*) is played, and then followed up with a *Woodcutter* (+2 *Coin*, +1 *Buy*).

<br />
{:refdef: style="text-align: center;"}
![Complex StateStack Flow]({{site.cdn_path}}/complex_statestack_flow.png)
{: refdef}
<p class="caption">Figure 2</p>
<br />

This makes it clear that the `StateStack` will be heavily modified through *Action* cards, and information has to be carried over between states in a really kludgy way.

*Reaction* cards proved really difficult to fit into this entire design idea. Whenever anything is done (as *Reaction* cards can react to almost any kind of ability) all cards currently in play and in each player's hand would have to be scanned for an appropriately matching *Reaction* card. Then, given how this could cause further *Reaction* cards or multiple player choices, would mean recursive `States` are being piled on in an apparent recursive fashion.

My first thought was, "o, this is kind of neat", but this quickly became apparent how the core design was being broken. Now each `State` had to be aware of everything in order to properly transition.

Another increasingly worrisome problem was in how `States` communicated game state information. The design was focused very heavily on communicating deltas in information in pieces to all clients in a broadcast manner. This was fine, so long as client never disconnected or never missed a message. For testing on local machine this was more than sufficient, but would clearly need to be looked at again if it were to move out of the "lab" setting.

For AI, we had to implement a kind of ugly workaround to allow them to interact with the `StateStack` as human players do to prevent having them coupled to the interface design of the `State` Object. This worked, but meant instead the AI was coupled to the communications object interface.

## Possible Improvements

I was in the middle of making sevearl changes to greatly improve the overall design and feasibility of completing major features, and writing this article helped to illuminate areas for improvement, too.

Firstly, moving towards a PriorityQueue would be much more sane. This would then allow for adding more `BuyStates` without worrying about how the `StateStack` was never designed for arbitrary insertions. This still has some downsides, namely that a precise ordering of all `States` must now be maintained. This can get tricky with many *Actions* triggering in a row, as order matters, but would seem arbitrary from the point of view of the `ActionState` itself. Actually trying out the idea would be necessary to determine the viability.

Rather than each `State` be aware of needing to respond to *Reaction* cards, the `StateStack` should instead minimally act as a Broker: its responsible for capturing the game state delta from a `State` and passing this to each player's hands for checking. Each Player will then be responsible for owning a `ReactionStateStack`, which would manage handling the interaction much in the same way as if it were a regular turn.

Technically, this could be done with just a single `StateStack`, that being the primary one the game itself shares. But a separate `StateStack` mostly helps in preventing the `StateStack` from getting messy. Its not stricly necessary, and does break away from the core design principles of a very simple `StateStack` managing game state interactions. Its a definite trade-off, and one that would need to be examined more closely with other contributors on the project.

By having `States` return modification reports, the `StateStack` can then more easily return these reports to a `Manager` that would pass this along to all players, and store this on a `Stack` of its own. This would allow for a highly compact replay system, and AI could couple themselves to a `Report` interface for reading changes and planning moves. The `Manager` would also be able to consolidate and marshall inputs from an AI or a human player into a single `InputContext`. This would then entirely decouple the `State` objects from the NodeJS communications protocol (even though this was managed by a wrapper class to abstract away those details, the point still stands).

Granted, these design improvements are to allow for a refactoring on the current code base, and so are very OOP heavy in their imaginings.

## Conclusion

This project was significantly more complex than when I originally started it forever ago. Had I instead taken a better sampling of the kinds of cards that would need to be added, and then tried to quickly mock up how they might interact with the `StateStack`, I could have much sooner realized a different approach would be necessary.

Hindsight bias can be pretty demoralizing without reminding yourself you can't know what you don't know before jumping in. Given that this was a personal project with friends to simply explore how we collaborate together to get a feel with how future game development projects might go, I would say everything went as well as it could have.