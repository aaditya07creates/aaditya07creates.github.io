---
layout: post
title: Boxing a robot with my voice
subtitle: A Real Steel voice controller, because I wanted to fight like the movie
cover-img: /assets/img/realsteel-cover.png
thumbnail-img: /assets/img/realsteel-thumb.png
share-img: /assets/img/realsteel-cover.png
tags: [Python, Voice, Gaming, Vosk]
author: Aaditya Bhave
---
<br />

## This one is pure childhood ##

Real Steel is one of those movies that basically wired my brain as a kid. Giant boxing robots, controlled by a human, fighting it out in a ring. It's a big part of why I fell in love with robotics in the first place, so this project is about as *me* as it gets.

A while back I found a Roblox game built around Real Steel, and I got completely hooked. I actually climbed all the way to **#6 on the global leaderboard** (I still have the screenshots, I'm not letting anyone forget that one). But the more I played, the more one thought kept nagging at me: in the movie, the best fighters don't just mash buttons, they *command* their robot. So what if I could stop pressing keys entirely and just fight with my voice, like Max does with Atom?

So I built exactly that.
<br />

## What it does
It's a Python program that listens to my microphone and turns spoken words into moves in the game, in real time. I say "right," the robot throws a right. "Hook," it hooks. And it goes further than single punches:

- **Modifiers** stack onto a punch. Say "heavy upper" and it throws a heavy uppercut. "Body left" throws a left to the body. Under the hood these hold a modifier key down, press the punch, then release.
- **Named combos** fire whole sequences. Shout "**fire combo**" and it rips off a scripted four-hit string; "**destroyer**" unleashes a five-hit finisher. These felt amazing to pull off by voice.
- **Block** holds the guard for you: say "block" and it presses and holds the block key for a couple of seconds on its own thread, then releases.

Everything is a single word on purpose, because in a real fight you do not have time for full sentences.

One dumb detail I love: the mic I shout into is my dad's work headset, the one I end up borrowing almost every day. There's something ridiculous and great about sitting there in a chunky work headset barking "destroyer" at my monitor, and yes, I've actually won a few matches doing exactly that.
<br />

## The whole game was speed
The fun part and the hard part were the same thing: **latency**. A boxing match is fast. If the program waited for me to finish a sentence, recognise it, and *then* act, I'd already be knocked out. So the real engineering went into making it feel instant.

I used **Vosk**, an offline speech-recognition engine, so there was no round trip to a server slowing things down. But the real trick was that I don't wait for a finished phrase at all, I act on **partial results** while I'm still talking. The moment Vosk thinks it heard "right," the punch fires, before I've even finished the word.

That instant-fire approach creates its own mess, though. If you naively act on every partial update, a single "right" gets fired ten times as the recogniser keeps re-reporting it. So I had to build a little **debounce system**: each command remembers when it last fired, ignores repeats within a fraction of a second, and tracks which words it already handled in the current phrase. Tuning those cooldowns, low enough to spam punches when you *mean* to, high enough to not double-fire by accident, was most of the actual work.

~~~
BASE_COMMANDS = {'right': 'i', 'left': 'j', 'upper': 'l', 'hook': 'k', ...}
MODIFIERS     = {'heavy': 'd', 'body': 's'}
# "heavy upper"  ->  hold D, press L, release D
~~~

There was one catch I didn't see coming. To keep things fast I used a small, lightweight Vosk model, and a light model buys its speed by giving up accuracy. Mid-fight it would mishear me, or worse, mix up two commands that sounded too alike, so a punch I called wouldn't land or the wrong one would. The fix turned out to be more about *language* than code: I renamed the commands to short, punchy, phonetically distinct words the model could nail even when I was rushing. "Jab" became "right," "uppercut" became "upper," and so on, picked as much for how cleanly they get recognised as for what they mean. Turns out designing a good voice interface is half engineering, half choosing words that don't sound like each other.

Blocks and combos run on their own threads so that holding a guard or ripping a five-hit combo never freezes the ear that's still listening for my next shout.
<br />

## Why I love it
Is a voice-controlled boxing script practical? Absolutely not. Was it one of the most fun things I've built? Completely. It taught me real things too, real-time audio pipelines, working with a speech-recognition engine, injecting input into a game, and the surprisingly deep art of tuning latency so something *feels* instant.

But mostly, it let a kid who grew up obsessed with Real Steel actually stand in his room and shout a robot into throwing a fire combo. Some projects are for the CV. This one was for me.
