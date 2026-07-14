---
layout: post
title: StudyPages
subtitle: A desktop app that turns any topic into a rich, interactive study page
cover-img: /assets/img/studypages-cover.png
thumbnail-img: /assets/img/studypages-thumb.png
share-img: /assets/img/studypages-cover.png
tags: [Python, AI, Desktop App, Cloudflare]
author: Aaditya Bhave
---
<br />

## Turning any topic into a proper lesson ##

When you ask an AI to teach you something, you usually get a wall of text. **StudyPages** was my attempt at something better: you type in any topic, and it builds you a full, scrollable, *visual* study page, with headings, callouts, tables, charts, timelines, quizzes, code blocks, and embedded videos. Not a chat log, an actual lesson you can save and come back to.

It's a Windows desktop app, and it's one of the more "real software" projects I've made, the kind with a proper UI, a backend, saved files, and a packaged installer.

{: .box-note}
**A quick note of honesty:** the code for this one is a bit of a mess right now and I haven't cleaned it up yet, so this write-up is about the *shape* of the thing rather than a line-by-line tour. I'll flesh it out once I've tidied it.
<br />

## How it works
The flow, end to end, looks like this:

1. You open the app (built with **CustomTkinter**, a themed desktop UI toolkit) and type a topic into the front page.
2. It kicks off the request on a background thread, showing a loading animation so the window never freezes, and sends your topic plus a big system prompt to a **Cloudflare Worker**.
3. That Worker is the brain-behind-the-curtain. It proxies the request to an AI model (**Gemini**) and hands back a structured response. Crucially, the AI is told to reply as **JSON**, a list of typed "blocks," not as prose.
4. The app takes that JSON and **renders each block as its own styled card**, walking the list and drawing a heading, a chart, a quiz, a table, whatever each block says it is.
5. Lessons get saved to disk as JSON so you can reopen them later without regenerating.
<br />

## The part I'm proud of: the block system
The heart of StudyPages is that the AI doesn't write me a page, it writes me a **structured description** of a page, and my app decides how to draw it. Every block has a type (`heading`, `callout`, `chart`, `mcq`, `timeline`, `code`, and so on), and each type has its own renderer function registered in a big lookup table.

The nice thing about that design is how cleanly it grows: to add a brand-new kind of card, I write one render function, register it, and describe it in the system prompt so the model knows it's allowed to use it. It's basically a little plugin system where the "plugins" are card types. That separation, model proposes structured content, app owns how it looks, is the idea I'm most happy with.
<br />

## The lesson: LLM output is messy
The thing nobody warns you about is that models rarely hand back clean JSON. They wrap it in explanation, or code fences, or a `<reasoning>` block, or a friendly sentence before the actual data. If you just try to parse the raw response, it blows up constantly.

So a surprising amount of the client is a **robust JSON extractor**: it strips out the reasoning block, unwraps any code fences, and then scans for the first *balanced* JSON object in whatever's left. Getting that reliable, across all the creative ways a model can wrap its answer, was one of the fiddlier problems and a good lesson in never trusting an LLM to follow a format perfectly.

I also reused a pattern here that I lean on everywhere now: the API key lives on the **Cloudflare Worker**, never inside the app itself. Same reasoning as in [PromptEnhance](/2026-07-13-PromptEnhance/), you never ship a secret to code that runs on someone else's machine.
<br />

## Where it stands
StudyPages works. You can type a topic, watch it generate a genuinely nice-looking lesson, and save it. It's also got rough edges I'm still sanding down, a few block types aren't fully finished and some polish is missing, which is exactly why the code needs a cleanup pass before I'd call it done.

Even in its half-tidy state, it's the project that taught me how to build a real desktop app: a threaded UI that stays responsive, a clean way to turn structured AI output into visuals, saving and loading user data, and packaging the whole thing into something you can actually double-click and run.
