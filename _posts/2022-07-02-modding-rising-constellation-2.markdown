---
layout: post
title:  "Modding Rising Constellation - Plugin Architecture"
date:   2022-05-18
categories: programming mods hacking
---

I got to the point of modding where I wanted others to try using these mods and also write mods. The way I was modding the game was not easily distributable, durable against updates, maintainable, easily extendable, and more issues. It was, in pure essence, a lab bench with bespoke tools and equipment all over.

Goals for Modding
=================
Before I did anything, I needed to ask myself, "if I were a user of these mods, what would I want"?

Several needs manifested themselves as clear goals I needed to achieve

1. Central place to find and download mods for the game
2. Easy to install and update downloaded mods; remove them and see what's installed
3. User-code easily written and separate from the game code

With these needs, I also asked myself what would I as the platform developer for these mods need to achieve these goals? Some more needs surfaced through tinkering with designs

1. Clear separation of mods to improve readability, writability, and fault-tolerance
2. Easily identify what mods are present locally and load them
3. Expose a simple interface for modders to hook into as needed

There are other minor points in both those above lists not written here, but these six principles helped me to get to the first design that satisfied these goals.

Plugin Architecture
===================
The modding platform has four distinct pieces

1. Original files modified as minimally as possible to interface with my `HookDispatcher`.
2. `HookDispatcher`, responsible for loading mods, receiving game events from the original files, and forwarding them to mods that implement the specified API. [Github](https://github.com/grnt426/RC-Mod-API)
3. Mod files, which must end in `mod.js`, that are loaded off disk if present
4. [Nexus Mods](https://www.nexusmods.com/risingconstellation) where mods can be browsed and downloaded. Eventually, the Vortex client could be used to help install and manage installed mods.

The original files, which for now is just one of the webpacked files `19.js`, have small modifications. One example of such is this:

```js
// granite chat message hook
let allowMessageThrough = true;
if(window.granite.dispatcher) {
    allowMessageThrough = window.granite.dispatcher.chat(this.newChatMessage);
}
```

_To find things easier in a 10,000 line minified JS codebase, I marked all sections I modified with `granite`. Similarly, to ensure I wouldn't clash with the game's own state with random variable names, I used `granite` under the global window object to store all my data separately from the game._

Some sections in `19.js` need more, but where possible injected code has as little business logic as possible, focusing on passing along information about what's happening in the game to the `HookDispatcher`.

Of course, the above code pre-supposes all kinds of my data and code is present, for which all of that sits inside the `init` function I found and as mentioned in the previous blog post.

A simplified version of that looks like this

1. If we haven't loaded ourselves already, do the below
2. Setup a global object space for us to work in and store everything, `window.granite`
3. Create a function for user-code code to add themselves as a listener
4. Asynchronously load `hookdispatcher.js`
5. Setup various debug, miscellaneous, and some API functions for user-code

With the above done, we can finally escape `19.js` and get into just our own code that is running and separation of concerns is achieved.

What does `HookDispatcher` do? Only a few things.

1. Finds all files ending in `mod.js` and asynchronously loads them.
2. Setups up a debug function for user-code to use that prints to the in-game chat for players to easily see, and dumps that to disk in a log file for mod authors to use in debugging.
3. Handles incoming events and synchronously applies them to mods that implement the function.

An example, using the injected chat message from above, looks like this after several checks and guards

```js
// Give this chat message to all mods
this.modHooks.forEach(m => {

    // If a listener doesn't implement the function, we silently ignore the listener and move on
    if(m.chatMessage) {
        try {
            processed |= m.chatMessage(data);
        }
        catch(err) {

            // We ignore all failures within hook handlers. This isolates failing mods from the rest
            window.granite.debug(
                "ERROR in dispatching chatMessage for mod " + this.#getModName(m) + ". " + err,
                window.granite.levels.ERROR
            );
        }
    }
});
```

_In this handler I allow mods to decide on their own whether a chat message should be allowed through or not. It was an old decision that is being reconsidered. It is presented here to show the messy process of constant iteration modding goes through._

Finally, we get to the most awaited part: user-code that actually *does something* based on what's happening in game.

Mods - User-Code - Plugin
=========================
Now that we understand how the game client can talk to our mod platform and handle events that get forwarded to mods, what can a mod do after receiving that event? Here is one example mod I made that I use all the time, [Coordinate Jumper](https://github.com/grnt426/RC_Mod_CameraChat).

The map of the game is quite large, and all systems are uniquely identifiable by their coordinate position, such as `276, 217`. The game provides no in-game search feature for systems, but users found it was slightly faster to provide a coordinate because players can drag the map around quickly and then match their camera coordinates to system coordinates.

I wanted that to be easier, which first required me figuring out how to control the camera

```js
// granite get camera
if(!window.granite.cameraControl)
    window.granite.cameraControl = a;
```

_Note: `a` is the camera object, which is only obvious with the surrounding code as context. Minified JS is not fun to read._

This is injected into the original file `19.js`, making it globally accessible. While this did require an additional change for this mod to work, this change means all future mods can also control the camera. Not a bad trade-off.

```js
class CameraChat {
    ...
    chatMessage(message) {
    ...

        // This allows us to support `c 193:212` and `c 193 212`
        if(message.indexOf(":") !== 0) {
            message = message.replace(":", " ");
        }
        let input = message.split(" ");
        window.granite.debug("Parsed input: " + input, window.granite.levels.DEBUG);
        let x = parseInt(input[1]);
        let y = parseInt(input[2]);

        if(x >= 0 && x < 10000 && y >= 0 && y < 10000) {
            window.granite.cameraControl.setCameraPosition(x, y, 30);
        }

    ...
    }
}

window.granite.addHookListener(new CameraChat());
```

_The above is abridged, but the critical bits are shown and the parts that are hidden are minor. See the [Github repo](https://github.com/grnt426/RC_Mod_CameraChat/blob/main/camerachat_mod.js) for the full code._

That's it! Now if I type in the game chat `/goto 276 217` my in-game camera is instantly sent there.

Later on I expanded the mod to search by system/sector name pair, so you could do something like `/goto serka ougar` to send you to the Serka system in the Ougar sector. Useful if you know the name and sector. Since system names are only sector unique, not globally unique, both are needed.

Results
=======
With this modding plugin-based architecture, I was able to adapt all my mods into this new format, and wow were they much easier to work with after doing this. No need to litter all over the original code base with my code changes. Failures are isolated, mods can dump errors to logs as needed, and I was quickly building up a useful API for others to easily use, without needing to understand the original codebase in-depth.

To put it all together, here is the mod that is needed for all other mods to work: [RC Mod API](https://github.com/grnt426/RC-Mod-API). This contains a modified version of the original `19.js` file that users just replace, along with my `HookDispatcher`.

Here are the mods I developed using RCMA at the time of this blog post:

* [Quick Copy](https://github.com/grnt426/RC-Mod-QuickCopy) - Copies system coordinates, name, and sector to the clipboard for pasting into Discord (where we all communicate with each other while playing).
* [Coordinate Jumper](https://github.com/grnt426/RC_Mod_CameraChat) - Easily move around the galaxy
* [Income Updater](https://github.com/grnt426/RC_Mod_IncomeUpdater) - Exports income data of the player's empire into a Google Sheet for planning incomes and resources over time.
* [Replay Maker](https://github.com/grnt426/RC_Mod_ReplayMaker) - Not yet published, but this is a refactored version of the original mod that produces replays of the game by recording snapshots.

Overall, I am very pleased with this final design and feel it meets all the design goals I set out to solve.