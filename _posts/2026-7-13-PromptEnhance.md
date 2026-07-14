---
layout: post
title: Building PromptEnhance
subtitle: How I shipped my first real product — an extension, a payment system, and a company
cover-img: /assets/img/promptenhance-cover.png
thumbnail-img: /assets/img/promptenhance-thumb.png
share-img: /assets/img/promptenhance-cover.png
tags: [AI, Chrome Extension, Startup, Cloudflare, Stripe]
author: Aaditya Bhave
---
<br />

## The thing I actually shipped ##

For most of my projects the goal has been "make it work." This time the goal was "make something people would actually pay for," and that changes everything. [PromptEnhance](https://chromewebstore.google.com/detail/promptenhance/ajfnkdejpiigjhedggbinfpnnaboilho) is a Chrome extension that turns a rough, lazy prompt into a properly engineered one with a single click, right inside whatever AI you're already using. One click, about half a second, and your "write me an email" becomes something that actually gets you a good answer.

It's live on the Chrome Web Store, it has a free tier, and it recently went live for payments. As I'm writing this I'm sitting on my very first sale, waiting for the payment to clear. That feeling is new and I kind of love it.
<br />

## Where the idea came from
I use AI all day, and I noticed I was spending more time rewriting my own prompts than I'd like to admit. Everyone talks about "prompt engineering" like it's some dark art, but really it's just structure: give the model a role, some context, tell it how to think, show it an example. I kept doing that by hand, over and over, and one day it hit me — this is a button. This is just a button that should exist.

So I checked, and yeah, a few things like it already existed. But honestly none of them were good. They'd pop open a new tab, or make you copy-paste, or just wrap your text in a generic template and call it a day. I knew I could do it better: inline, instant, and actually smart about *how* it rewrites. That gap is the whole reason I built it.
<br />

## What it does
The core promise is simple: you type your rough prompt, hit the enhance button that lives right in the input box, and it rewrites it into something a professional prompt engineer would've written.

Under the hood there's more going on than "make it longer":

- **Three levels** — Concise, Balanced (the default), and Detailed. Not every prompt needs a wall of text, so you get to pick how much structure you actually want.
- **Real frameworks** — it can shape prompts using proper prompt-engineering frameworks like **COSTAR, RISEN, APE and CARE**, plus chain-of-thought reasoning, instead of just padding your sentence.
- **Advanced toggles** — optional *Add Role*, *Chain-of-Thought*, and *Few-Shot Examples* for when you want to go further.
- **It lives where you work** — the button injects directly into the input box of ChatGPT, Claude, Gemini, Grok, DeepSeek, Perplexity, Mistral and Cursor. No popups, no new tabs, no copy-paste.
- **Free to start** — 25 enhancements a day on the free tier, no credit card, no trial nonsense.

That "lives where you work" part sounds small. It was the single hardest thing in the entire project.
<br />

## The hard part nobody sees: getting inside every AI's input box
Here's the thing I completely underestimated. Every single AI site builds its input box differently, and none of them want a stranger's button in there.

Some of them use a plain `<textarea>`. Easy enough. But the big ones — ChatGPT, Claude — use a `contenteditable` div running on a rich-text editor like ProseMirror or Lexical. That means I can't just set `.value` and move on. And even the ones that use React-controlled inputs ignore you if you set the value directly, because React has its own idea of what's in the box and overwrites you instantly.

The fix for the React ones is a little cursed but it works — you have to reach past React and call the *native* value setter, then fire the event React is actually listening for:

~~~
function setNativeValue(el, value) {
  const proto = el.tagName === 'TEXTAREA'
    ? window.HTMLTextAreaElement.prototype
    : window.HTMLInputElement.prototype;
  const setter = Object.getOwnPropertyDescriptor(proto, 'value').set;
  setter.call(el, value);
  el.dispatchEvent(new Event('input', { bubbles: true }));
}
~~~

The `contenteditable` editors needed a different trick entirely (dispatching a proper `beforeinput`/`insertText` so the editor thinks *the user* typed it). Every platform was its own little puzzle.

And then the button itself won't stay put. These are single-page apps that constantly re-render, so the moment you navigate or the page updates, your button vanishes because the DOM node it was attached to got replaced. The answer is a `MutationObserver` that watches the whole page and re-injects the button the instant the input box comes back:

~~~
const observer = new MutationObserver(() => injectEnhanceButton());
observer.observe(document.body, { childList: true, subtree: true });
~~~

Getting that to feel invisible — no flicker, no double buttons, works on eight different sites that each update whenever they feel like it — took way more tries than I want to admit.
<br />

## Making it fast, and keeping the keys safe
The whole pitch is "half a second." If enhancing a prompt took three seconds, nobody would use it twice. Speed wasn't a nice-to-have, it was the product.

I also had a security problem: the actual enhancement is done by a language model, which means an API key, and you can **never** ship an API key inside a browser extension. Anyone can unzip an extension and read it. So the extension can't talk to the model directly.

Both of those problems have the same answer: **Cloudflare**. The extension talks to a Cloudflare Worker I wrote, and the Worker is the only thing that holds the key and talks to the model. Workers run on edge servers physically close to the user, so the round trip stays tiny, and the secret never leaves my infrastructure.

{% highlight javascript linenos %}
export default {
  async fetch(request, env) {
    const { prompt, level, deviceId } = await request.json();

    // check the free-tier quota before doing any real work
    const used = parseInt(await env.USAGE.get(deviceId) || "0", 10);
    if (used >= 25 && !(await isPro(deviceId, env))) {
      return new Response("Daily limit reached", { status: 429 });
    }

    // the key lives here, on the edge, never in the extension
    const enhanced = await callModel(prompt, level, env.MODEL_API_KEY);

    // bump usage, reset every 24h
    await env.USAGE.put(deviceId, used + 1, { expirationTtl: 86400 });

    return new Response(JSON.stringify({ enhanced }), {
      headers: { "Content-Type": "application/json" },
    });
  }
};
{% endhighlight %}

That little `USAGE` store is Cloudflare KV, and it's how the free tier stays free without me having to make everyone log in. No accounts, no passwords — just an anonymous device ID and a counter that resets every day.
<br />

## Actually charging money (the scary part)
Building the tool was one thing. Letting a stranger on the internet pay me was a whole different mountain, and this is where **Stripe** came in.

The flow sounds simple and absolutely is not: the user clicks upgrade, Stripe handles the checkout (so I never touch a single card number, thank god), and when the payment succeeds Stripe fires a **webhook** at my Worker. The Worker verifies that the webhook is genuinely from Stripe, then flips that device from "free" to "pro" so the daily limit disappears. Then the extension has to check, quietly and often, whether it's talking to a pro user or not — without slowing anything down and without trusting the client to just *say* it's pro.

Getting webhooks, entitlements, and the extension's idea of "am I paid?" to all agree with each other was genuinely one of the fiddliest bits of the build. Payments are the one place where a bug isn't a bug, it's either free stuff for everyone or angry customers, and neither is great.
<br />

## Proving you're allowed in — JWTs
Once someone pays, the extension needs a way to prove "I'm a pro user" on *every single request* — fast, without me doing a database lookup each time, and without just trusting the extension when it claims to be paid. The answer was **JWTs** (JSON Web Tokens), and honestly this was one of my favourite things to learn on the whole project.

When a payment clears, my Worker mints a **signed token** that basically says "this device is pro, valid until X." It's signed with a secret that only lives on my server. The extension stores that token and sends it along with every enhance request. The Worker just checks the signature — if it's valid and hasn't expired, you're in. No database round-trip, nothing to look up. And if someone tries to edit the token to promote themselves to pro, the signature breaks instantly and I catch it.

~~~
// on the Worker, for every request:
const token = request.headers.get("Authorization")?.replace("Bearer ", "");
const claims = await verifyJWT(token, env.JWT_SECRET);   // throws if tampered / expired
const isPro = claims?.tier === "pro";
~~~

The real lesson here was *why* stateless auth is such a big deal — the token carries its own proof, so the server doesn't have to remember anything about you. I also learnt the hard way to keep the expiry short and build a refresh path, because a token that's valid forever is a token you can never take back if something goes wrong.
<br />

## The brand, the site, and the "company"
There's a proper [landing page](https://pinkobubs.github.io/PromptEnhance/) for it — the "Write prompts like a professional" hero, the whole pitch. I'll be straight: I didn't build that page myself. But I knew it *needed* to exist and look like a real product from a real company, because first impressions are basically the entire game when you've got about half a second to convince someone to click "Add to Chrome."

I put the whole thing under a name: **pinkobubs**. It's technically an unregistered company for now — more of a brand and an identity than a legal entity — but having a name to ship under made the whole thing feel real, and it's what shows up as the developer on the store and the payment descriptor on Stripe.
<br />

## The Chrome Web Store kept saying no
I genuinely thought I'd just upload the extension and be done. Instead I got **rejected. Multiple times.** And almost never for anything big — it was always some small oversight I hadn't thought about.

A few of the ones that got me:

- **Permissions I couldn't justify.** I'd asked for broad host permissions ("run on every site") because the extension works across a bunch of AI sites — but the store wants every permission justified and kept as narrow as possible. I had to trim it down and spell out exactly why I needed each one.
- **Privacy disclosures that didn't line up.** What I declared in the developer dashboard had to match *exactly* what the extension actually did and what my privacy policy said. Any mismatch and it bounces.
- **The "single purpose" rule.** An extension is supposed to do one clear thing, so I had to make sure nothing looked like scope creep.
- **Remotely hosted code.** Manifest V3 doesn't let you pull in and run code from a server — everything that executes has to ship inside the package. That quietly shaped how I structured the whole thing.

Each rejection meant reading a not-always-specific reason, fixing one small thing, resubmitting, and waiting for review all over again. It was easily the most tedious part of the project — and a real lesson in how much of shipping a product is compliance and paperwork rather than code.
<br />

## What I actually learnt about architecture
This was the first time I had to think like an *architect* instead of just a coder, and a few lessons really stuck:

- **Put a server between your secrets and the world.** The single most important call was that the extension never talks to the model directly — everything goes through my Worker. That one boundary is what keeps the API key safe, lets me rate-limit, and lets me swap the model without shipping a new extension.
- **Stateless beats stateful when you can manage it.** JWTs plus edge KV meant I didn't need a big database to babysit. Less to run, less to break.
- **Never trust the client.** The extension runs on someone else's computer, so it can claim anything. Every real decision — are you pro? are you over your limit? — has to happen on the server where I actually control it.
- **Plan the skeleton first.** Same as I do with everything: I mapped the whole flow — extension → Worker → model, and Stripe → webhook → entitlement → JWT — in my notebook before writing much of it. The projects where I skip that step are always the ones I end up rewriting.
<br />

## Wearing every hat
Here's the thing I'm proudest of, and it's not any one feature. I actually shipped this **end to end**. I don't usually do that — normally I build the interesting core of something, learn what I wanted to learn, and drift off to the next idea. This time I saw it all the way through: the extension, the backend, the payments, the auth, the store listing, the compliance grind, and now the marketing.

That's a pile of jobs that in an actual company would be spread across a whole team — a frontend dev, a backend dev, someone on payments, someone on infra, a designer, a marketer, whoever deals with app-store policy. Doing every one of them myself, badly at first and then a bit better, taught me more than nailing any single one of them ever could have. Mostly it taught me how much *more there is* to shipping a product than writing the code.
<br />

## Where it's at, and what's next
So here's the honest status report. PromptEnhance is **live**, it has a **free tier**, and it just switched on **payments**. I've made my first sale and I'm literally waiting for the money to land as I write this. It's a small number of users right now, but a 4.8-star rating out of the early ones, which I'll happily take.

Right now most of my energy is going into **marketing**, because a good product nobody's heard of is just a hobby. That's the part I'm least experienced at and, weirdly, finding the most interesting to figure out.

And I'm nowhere near done building. I've got a long list of improvements — better enhancement quality, more platforms, smarter handling of the advanced frameworks, and a bunch of little polish things that only show up once real people start using it. This is the first project of mine that's genuinely *out there*, being used by people I've never met, and that changes how you think about every decision.

If you use AI a lot, [give it a try](https://chromewebstore.google.com/detail/promptenhance/ajfnkdejpiigjhedggbinfpnnaboilho) — it's free to start, and I'd love to know what you think.
