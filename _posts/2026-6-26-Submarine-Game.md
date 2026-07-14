---
layout: post
title: A co-op submarine game
subtitle: A Roblox crew sim where nobody can run the whole boat alone — my biggest project yet
cover-img: /assets/img/submarine-cover.png
thumbnail-img: /assets/img/submarine-thumb.png
share-img: /assets/img/submarine-cover.png
tags: [Roblox, Game Dev, Lua, Architecture]
author: Aaditya Bhave
---
<br />

## The idea: make people actually crew a submarine ##

Most games let one player do everything. I wanted the opposite. This is a **co-op underwater submarine simulator** on Roblox where a crew shares *one* submarine and has to actually run it together — walking up to physical consoles for depth, speed, oxygen, cooling, sonar, radar, navigation and comms. The whole point is that **no single player can comfortably run everything at once**, so the crew naturally splits up, calls out warnings, and talks to each other. It's the biggest, most involved thing I've built.
<br />

## Making it feel like a real sub
Before writing much of anything, I went and researched how real submarines actually work — how they dive, why depth is dangerous, how they navigate blind, why a crew is split into stations. I didn't want another arcade game with a "submarine" skin. I wanted it to *feel* like you're genuinely operating a machine that's trying to kill you if you stop paying attention.

So instead of menus, you're **entirely surrounded by consoles** — they line the walls of the sub on every side — and you physically walk up to each one to use it: a steering wheel you actually drag to change heading, +/− buttons to step your depth, start/stop switches for oxygen and cooling, calibration sliders for the sonar. You read a top-down **sonar map** to thread through a cave, and a **radar** to catch enemies sweeping in. Standing in the middle of it all, with a station in every direction, is a huge part of why it feels like a real boat instead of a screen.
<br />

## The part I love: too many things to watch
The tension comes from having more to manage than any one person comfortably can. At any moment the crew is juggling:

- **Depth** — go past 300 m and the hull starts taking crush damage; past 350 m and it's instant, the ocean just wins.
- **Oxygen** — constantly draining. You run electrolysis to make more, but that eats battery.
- **Battery** — drains as you move, recharges at the surface. Everything good costs power.
- **Temperature** — climbs on its own; cooling brings it down but (of course) costs battery. Too hot or too cold and you fail.
- **Enemies** — every few minutes a ship comes in on radar. If you're too shallow during the contact window, you're detected.
- **The cave walls** — leave the tunnel and you scrape rock, which is lethal.

Everything is connected. Cooling costs battery, making oxygen costs battery, moving costs battery — so a crew that panics and does everything at once drains the boat and dies anyway. Learning to balance it *as a team* is the whole game.
<br />

## A mission, start to finish
A run looks like this: surface at home and use the comms console to **request a task**. The carrier sends back an **encrypted coordinate**. Someone calibrates the sonar, someone reads the encrypted coords and the decrypt key and enters the decoded position at the decypher console. Then you **navigate the cave maze** to the target, dive to the right depth, and hold position to **collect data**. The carrier names a random **extraction point** somewhere else on the map, you travel there, surface, and **broadcast** the data. Mission complete, everyone ranks up, reset.

I even added little touches to make the carrier feel alive — request a task and there's a short, randomised delay while it "prepares a directive" instead of an instant response, and a private system message greets each player by a per-session sub callsign like `sub-F575` when they join. Small things, but they sell the world.
<br />

## Then the algorithm noticed
Here's the part I genuinely did not see coming. One day the game just got picked up by the **Roblox algorithm** and started getting pushed out to players on its own. Visits climbed fast, and at its peak it had **24 people in it at the same time** — for a game I built solo, watching that number tick up felt completely unreal. I was over the moon.

And the timing is almost funny: the entire V2 rebuild I'm about to describe happened *after* all of that. The messy first version is the one that blew up. Watching real strangers actually play it, and quietly stress-test every weak spot in that early code, is exactly what convinced me to tear it down and rebuild it properly. You can [play it here](https://www.roblox.com/games/116249915401730/Submarine-Game).
<br />

## The lesson that actually stuck: architecture
Here's the honest bit. The first version of this game worked, but the code was **spaghetti** — server scripts reading button clicks directly, state scattered everywhere, logic tangled together. As it grew it got harder and harder to touch anything without breaking something else. So I rebuilt the whole thing, and *that* rebuild taught me more about software than the game itself did.

The rule I landed on was **"server simulates, client renders."** The server owns all the truth — depth, oxygen, position, everything — and the clients just draw whatever the server says. There's **one source of truth** (the game's state), **one channel** for all player input (a single remote that every console talks through), and the simulation is **event-driven** instead of dozens of loops all polling each other. Shared logic lives in one place and gets used by both sides.

~~~
-- every console, no matter what it is, sends the same shape of message:
ConsoleAction:FireServer(consoleId, action, payload)
-- the server validates it and decides what's true.
-- the client never decides anything — it only renders state.
~~~

It sounds obvious written down, but living through the version *without* that discipline is what made it click. That "one source of truth, never trust the client" idea is the same lesson I'd later lean on hard when building [PromptEnhance](/2026-07-13-PromptEnhance/) — turns out it applies whether you're running a submarine or a payment system.
<br />

## It kept growing
What started as "operate a sub" turned into a genuinely big project. It now has a **26-rank progression** system, a **14-upgrade tree** the crew spends a shared credit pool on, a second mission type (repairing a chain of relays instead of grabbing intel), a "continue the dive" system that lets the crew save a doomed run, an in-world operations manual, a whole companion lobby place — the list keeps going. Every one of those started as "wouldn't it be cool if…" and turned into its own little system to design and wire in.
<br />

## What I took from it
This is the project that taught me how to actually *structure* something big instead of just making it work. I learnt to separate simulation from rendering, to keep one source of truth, to design systems that don't trip over each other, and to rebuild something the right way instead of piling more onto a mess. It's also just really satisfying to watch a crew of people who've never met figure out how to keep a submarine alive together.

There's a long roadmap I still want to get to — damage-control you repair under pressure, torpedoes, smarter enemies, stealth runs. It's nowhere near done. But it's already the thing I point to when someone asks what the most I've ever bitten off is.
