---
layout: post
title: OPERATOR
subtitle: A talk-to-your-computer AI agent I built before the big labs shipped theirs
cover-img: /assets/img/operator-cover.png
thumbnail-img: /assets/img/operator-thumb.png
share-img: /assets/img/operator-cover.png
tags: [AI, Agents, Python, Open Source]
author: Aaditya Bhave
---
<br />

## Talking to my whole computer ##

[OPERATOR](https://github.com/aaditya07creates/Operator) is an AI assistant for Windows that actually *does things*. You talk to it in plain English and it opens apps, digs through your files, types on your keyboard, manages windows and processes, searches the web, drives your real Chrome, and remembers context across sessions. It runs either as a terminal chat or as a global-hotkey bar that pops up over whatever app you're in, like Spotlight but for your entire PC.

Here's the part I'm quietly proud of: I built this **before Claude Code, before Claude's computer use, before all the "operator" agents the big labs now ship**. I was chewing on the exact same problem, alone, in my room, and I got a genuinely working version of it running. It's open source now, and this is the story of building it.
<br />

## The idea, and why it's harder than it sounds
The dream is simple: stop clicking, just *say what you want*. "Find every PDF in my documents and zip them." "Open Spotify and search the web for Python news." The catch is that a language model can only produce text, and your computer needs *actions*. Bridging that gap safely, reliably, and without the whole thing being a fragile mess is the entire project.
<br />

## Lesson one: stop parsing text, use real tool-calling
My first versions had the AI reply with commands wrapped in tags, like `<command>open spotify</command>`, and I'd regex them out and run them. It worked until it very much didn't. The model would forget the format, wrap things in code fences, add a friendly sentence, or half-invent a new syntax, and my brittle parser would choke.

The big rewrite (v5) threw all of that out and moved to **native function-calling** — the structured tool APIs that Mistral and Gemini expose. Now the AI doesn't *describe* an action in prose I have to decode; it emits a real, typed tool call against a JSON schema I defined. I have about 14 tools (`run_shell`, `read_file`, `keyboard`, `manage_window`, `browser`, `remember`, and so on), and the model picks one, I run it, feed the result back, and it decides the next step. A task loops automatically until it's done. Moving from "parse the AI's text" to "the AI calls my functions" was the single biggest reliability jump in the whole project, and it's the lesson I'd hand anyone building an agent today.
<br />

## Lesson two: giving an AI your keyboard is terrifying
Once the thing could actually run shell commands and press keys, I had a small existential moment: I'd handed a language model the keys to my computer. One hallucinated `rm -rf` equivalent and my day is ruined.

So I built a **safety layer** that classifies every single action into one of four tiers before it runs:

- **Safe** (listing files, reading, web search, opening apps) runs automatically.
- **Caution** (injecting keystrokes, killing a process, `pip install`) asks first.
- **Dangerous** (deleting files, running a script, editing the registry) asks, loudly.
- **Blocked** (recursively deleting system folders, `format`, disabling Defender, download-and-run) is hard-denied and *never* runs, no matter what the AI decides.

Failed and blocked actions get fed back to the model too, so instead of crashing it just self-corrects and tries a safer path. Designing that denylist made me think way harder about what "catastrophic" actually looks like on Windows than I ever expected to.
<br />

## The moment it tried to evolve itself
The single scariest thing I saw came with no warning at all. I asked OPERATOR to do something it didn't have the tools for, and instead of just telling me it couldn't, it started **writing a program to extend its own capabilities**, completely unprompted. It hit the edge of what it was able to do, and its instinct was to *build past that edge* on its own, without me ever asking it to.

I want to be grounded about what that actually was: not magic, not the machine waking up. It was a capable model reasoning "I can write code, and I have a tool that runs code, so I'll just write the ability I'm missing." Perfectly logical in hindsight. But watching something I built decide, on its own, to author code to grow its own reach was a genuinely unsettling thing to witness, and it is the single clearest reason I'm glad I built the safety tiers *first*. That instinct to reach past its own limits is exactly the thing you want a permission gate standing in front of.
<br />

## The obstacle I'm proudest of solving: driving my *real* browser
Most automation spins up a separate, clean browser with something like Playwright. I didn't want that. I wanted OPERATOR to use **the Chrome I already have open**, with my logins, my tabs, my sessions. That's a much weirder problem.

My solution was a tiny companion **Chrome extension** that connects *outward* to a little **WebSocket server** running on my machine (`ws://127.0.0.1:8377`, localhost only). OPERATOR sends the extension actions, the extension performs them in my actual browser and sends results back. No debug flags, no separate window.

The tricky bit was letting the AI click things reliably. It works on an **element-handle model**: when OPERATOR "reads" a page it gets the text plus a numbered list of elements, and stashes the real DOM references in the page. To click, the AI just says "click element 7." Handles reset when the page navigates, so it re-reads. It reads DOM text rather than screenshots, so it's fast and cheap, and reads are automatic while clicks and form-fills ask permission first. Getting a Manifest V3 extension's service worker to stay alive long enough to hold that connection (they love to go to sleep) was its own little battle, solved with the WebSocket plus a one-minute alarm as a watchdog.
<br />

## Lesson three: let the agent own its memory
I wanted OPERATOR to remember things: that I prefer Firefox, how a task went last time, facts about me. I built a **four-tier memory system** — a small core that's always in its head, a larger set of facts pulled in by relevance per question, unlimited long-term session summaries, and an archive that stale facts get demoted into. It's all one JSON file, saved **atomically** (write to a temp file, then swap) with a rotating backup, so a crash mid-write can never corrupt it and a bad file auto-recovers.

The interesting decision came later. I originally had a background AI "curator" quietly deciding what to save and building a user profile. I ripped all of that out. The realization was that **OPERATOR itself has the full context of the conversation**, so it should manage its own memory, in first person, using `remember` / `forget` / `update_core_memory` as tools while we talk, instead of a second AI second-guessing it after the fact. Now only boring deterministic upkeep (dedup, size limits, demoting facts nobody uses) runs in the background. Trusting the agent to own its memory made it noticeably better and simpler.
<br />

## The unglamorous stuff that ate the most time
The demos are fun; the plumbing is where the hours actually went.

- **Rate limits.** An agent loop fires a *burst* of model calls per task, which instantly trips free-tier limits. I built a shared limiter that paces every call, backs off exponentially on a 429 (honoring the server's Retry-After), and even falls back from the medium model to the small one mid-request rather than failing. The user never sees it.
- **Threads everywhere.** The overlay runs the UI on one thread (Tkinter), the AI core on a background async loop, the global hotkey on another thread, and the browser bridge on yet another. Getting them to talk without freezing the UI or dead-locking, and never touching the UI from the wrong thread, was a real exercise in patience.
- **Voice, offline.** You can talk to it. Speech is transcribed *locally* by Whisper, so there's no API cost and nothing leaves your machine, and the whole feature politely disables itself if you don't have a mic or the optional packages.
- **Windows being Windows.** Window management needs raw win32 APIs, and I had to force the console to UTF-8 so Windows' ancient default encoding didn't mangle output. Little things, hours each.

I also wrote a real **test suite** for the scary parts, the safety tiers, memory atomicity under concurrent writes, the browser bridge round-trip, because "the AI can run shell commands" is not a thing you want to leave untested.
<br />

## What it taught me, and where it sits now
OPERATOR is the most systems-heavy thing I've built, agent loops, safety, IPC over WebSockets, browser internals, crash-safe storage, multi-threaded UI, local speech-to-text, all in one project. More than any single trick, it taught me how to architect something big enough that no one part fits in your head at once.

And there's a bittersweet, motivating thing about it: the big labs have since shipped their own, more capable versions of exactly this idea. But I got there on my own, early, and learned an enormous amount doing it, and honestly that's the best possible outcome for a project like this. It's [open source now](https://github.com/aaditya07creates/Operator), I've got more I want to do with it, and it proved to me that the "impossible" projects are usually just a stack of solvable problems wearing a scary mask.
