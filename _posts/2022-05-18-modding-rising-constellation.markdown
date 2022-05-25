---
layout: post
title:  "Modding Rising Constellation - The Journey"
date:   2022-05-18
categories: programming mods hacking
---

Over the past few months, I have been experimenting, reverse-engineering, and hacking at a video game I have fallen in love with: Rising Constellation. I hadn't done this kind of work before, so much of it was new to me. What kept me motivated and moving was how much I enjoyed the niche, slow-burn, and intricacies of Rising Constellation. Below is a re-counting of what I did, the various attempts and discarded ideas, and little successes I had.

Rising Constellation
=========
The game pushes the boundaries of genres it could be most closely associated to, such as 4X, MMO, and RTS. Players join a faction, with most games having 2-3 choices, with the goal of all players in one faction cooperating to achieve victory against the other factions through Conquest, Influence, and Shadows.

Most games last about two and a half real-time weeks. The beginning is slow, with players focusing on exploring systems around them looking for where to expand to next. The current meta of the game is for players to specialize into one of a few different roles: Banker, Fleet Builder, Infiltration Agents, Technolgy or Ideology producer, and possibly other more niche roles depending on the map.

The game itself is slow. Moving fleets or agents can take hours. Higher level buildings can similarly take hours or even days to build. Players typically spend 30-90 minutes each day managing their empire, checking it irregularly to execute new orders, or mostly chatting and strategizing in Discord.

Game Architecture
==============
The game exists as a web game and as a product on Steam. If purchased through Steam, the game is just a wrapper around Chromium through the NW.js library, with the game assets loaded off the player's computer. The game uses Three.js, HTML, JS, CSS, and Canvas for rendering. The game engine is event-driven websockets entirely handled in Javascript. At least, this is all that I have managed to discover and understand of the game's architecture.

The game is webpacked and minified, which has diminished the speed with which I can discover useful elements and exploit them. I don't have experience with any of the technologies and libraries I previously listed (aside from HTML, JS, and Canvas), thus also made understanding what I looked at more difficult.

A big positive going for me was the javascript files are just dumped to disk when downloaded through Steam, allowing easy access and, with no client-side verification or DRM, trivial modification. Thus, my confidence in delivering some kind of change was high. Let's see where that got me.

First Steps
===========
I opened up the developer console for the game while I was still playing it in the browser, before getting it on Steam. From here, I realized the first difficulty: its all minified. Once I bought the Steam version I then started just opening random Javascript files. The goal here wasn't to find anything in particular, it was to take a stroll through a park and see if anything stood out. I didn't want to tunnel-vision myself and so tried to keep an open mind. Who knows what I would find that would look useful? I certianly didn't, so this seemed like the best strategy.

Eventually, I found the below code, which seemed really interesting to me:

```js
key: "init", value: (a = r()(c.a.mark((function t() {
```

To me, seeing `init` made me light up. This looks like startup code! Maybe I could *do something* at game start. Everywhere else looked impenetrable to me as I didn't understand the flows, nor what `y` or `t` variables were or did. Seeing plain English strings I eventually started taking note of as they were immediately recognizable, and so were first on the "mess with this" list.

But...what could I do? I didn't know what any of the variables in this scope were, or what scopes I had access to. It wouldn't be until much later that I realize I can mess with the global `window` object (I am a backend engineer first, and front-end has always been weakest for me), so I had limited ideas of what to do. Instead, I ask myself, "could I make network requests out while the game is running"?

The answer: yes! And I did that with plain Javascript.

The First Changes
===========

```js
let xhr = new XMLHttpRequest();
xhr.open("POST", "localhost:8080/debug");
xhr.timeout = 200;
xhr.setRequestHeader("Content-Type", "application/json");
xhr.send("Hello World!");
```

With a quick NodeJS server running locally, I was able to see this message posted once, shortly after the loading of the game finished. Excellent! I found a useful hook point to do stuff. But...do what?

I wanted to send to myself whatever variables were getting used in the scope of this function and started looking for what to send. Just above where I found the `init` line, I also saw stuff like this, in what looked like the same level scope

```js
{
    key: "gameData", get: function() {
        return y.a.state.game.data
    }
}
```

Wow, `state.game.data`? Seems too easy! Only one way to find out.

```js
let xhr = new XMLHttpRequest();
xhr.open("POST", "localhost:8080/debug");
xhr.timeout = 200;
xhr.setRequestHeader("Content-Type", "application/json");
xhr.send(JSON.stringify(y.a.state.game));
```

As well as repeating this process for every other thing, and the keys, of stuff under `y`, `a`, and such. For now, I was in data harvesting mode. Just taking whatever I could get and analyzing the results.

What was I getting out of the game? Well, nearly all of the game state, just as the variable names promised. Sounds great? Well, its about 18 MB when formatted to be readable. The bulk of that data is all the connections between systems and the systems themselves.

Replay System
===========
With the data I was getting out, it looked quite plausible I could accomplish one of my mod ideas of creating replays of the game so after 2-3 weeks our group could see how the galaxy progressed. The plan: send myself the entire game state, then find differences between states and store only those differences in a series of snapshots with timestamps. This way, I could recreate the history of the game and play it back.

All I would have to do is also create my own rendering of the game from the data I extracted. Or so, I thought.

The biggest issue arose when, after I did get such a website working so I could have my own mirror copy of the game rendering and showing whatever state I wanted, I wasn't actually pulling down proper snapshots of the galaxy. While I had a little timer that would, every minute, dump whatever was in `y.a.state.game` to my NodeJS server, that wasn't actually always updated. I could not determine why, but I was definitely losing game state information.

Thus, my first major refactor had to happen. While working on all this, I had continued to note interesting lines of code and files with guesses as to what it did. One such interesting bit was this:

```js
var w = function() {
    function t() {
    p()(this, t), this.systems = [], this.systemsToRepaint = new Set([]), this.blackholes = [], this.sectors = [], this.sectorHash = "", this.hasToRepaintSectors = !1, this.radars = [], this.radarsHash = "", this.hasToRepaintRadars = !1, this.detectedObjects = [], this.hasToRepaintDetectedObjects = !1
}

return h()(t, [{
    key: "update", value: function(t) {
```

Wow, that looks like a stream of updates! I repeated the process above to start dumping whatever data was at this scope, just to see what was there and validate. Sure enough, every change that happened in the game (that the player could see, anyway), would flow here. Quite helpfully, it was already categorized for me.

I rebuilt my replay engine and now had a second hook point for code. I kept the bulk of my changes in the `init` function I found above. Because, why change what doesn't work?

Success - MVP
=========
With that, I then had an MVP I could show players in the game, which was a functional replay system. You can see what that looks like here: https://rc-replay.com/html/viewer.html

I was happy to have accomplished so much, and delivered a really cool feature to players. It needs more polish, for sure, but I had a success and wanted to keep building on that with changes focused more on the client, rather than just exfiltrating data.

The next blog post will cover how I had to refactor everything again to make mods useable by other players. Considerations I made for how to have other player's write mods and integrate, distribution, installation, and more.