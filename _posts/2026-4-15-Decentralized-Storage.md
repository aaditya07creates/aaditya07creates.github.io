---
layout: post
title: Constellation
subtitle: A plan to scatter the world's files, encrypted, across millions of phones
cover-img: /assets/img/storage-cover.png
thumbnail-img: /assets/img/storage-thumb.png
share-img: /assets/img/storage-cover.png
tags: [Startup, Cryptography, Distributed Systems, Unit Economics, Paused]
author: Aaditya Bhave
---
<br />

## The most ambitious thing I've ever attempted ##

Every so often I get an idea that's clearly too big for me, and instead of scaring me off, that's exactly what makes me want it. **Constellation** was that idea. It was a plan for a cloud storage company with no data center at all — one where your files live, fully encrypted, spread across the spare space on a huge number of *other people's phones* all over the world.

I researched it for months, ran the numbers over and over, wrote working proofs-of-concept for the hard cryptographic parts, and dug deep into the law. In the end I made the call to **pause it** — not because it was wrong, but because it was genuinely bigger than one first-year student could carry right now. This is everything I figured out along the way, because I put more thought into this than into anything else I've built, and I don't want it to just disappear into a folder.
<br />

## The vision: a cloud with no data center
Here's the picture I couldn't get out of my head. Instead of your photos sitting in a Google warehouse, imagine them **encrypted on your device, chopped into fragments, and scattered across thousands of strangers' phones** in a dozen different countries. No single phone holds anything meaningful. No company can read your files. And because you're renting spare space on hardware people already own instead of building billion-dollar data centers, it could be *cheaper* than Google Drive at the same time.

The scale is the part that got me. Not a few servers, not a cluster — **millions of phones**, each holding a few meaningless shards, coordinated into one giant storage network. That's decentralization on a level that's honestly hard to even picture. If it worked, it wouldn't just be a cheaper Dropbox, it would be a fundamentally different shape for the cloud.

People who donate their spare storage (I called them **hosts**) get paid for it. Consumers pay less than they do today. The company lives on the margin in between. Simple to say, very hard to actually make add up — which brings me to the thing that nearly killed it before I wrote a single line.
<br />

## The economics almost broke it before the code did
This was the real battlefield, and it's where I spent the most time. A decentralized network sounds free, but every host has to be paid, every file has to be stored redundantly, and all of that eats the margin. I ran the unit economics again and again, and a few brutal truths kept falling out:

- **Where the hosts live is the single biggest lever on the whole business.** A network of US hosts is expensive; a network weighted toward **India** — cheaper electricity, cheaper data, lower cost of living — completely changes the math. I did whole rounds of calculations assuming India-heavy hosting, using purchasing-power differences in electricity and data rates, because that one variable moved the margin more than anything else, more even than what I charged customers.
- **I benchmarked relentlessly against the incumbents.** I priced my model against **Amazon S3 / Glacier cold storage**, against **Backblaze B2** (about $0.006 per GB per month), and against **Storj**, a real decentralized-storage company that pays its hosts roughly $0.0015 per GB per month. Those numbers were my reality check, and they were humbling.
- **The most painful discovery:** for a long stretch, **Backblaze is simply cheaper than my own network would be.** A peer-to-peer network with realistic device churn needs a lot of redundancy, and that redundancy multiplies cost. Until the network is huge and stable, plain cloud storage wins on price. That's not a small footnote — it undercut the core pitch for the entire early period.
- **So the only sane launch plan was a hybrid.** Day one leans on cheap centralized cloud as a backstop plus a few company-owned seed machines, and the system slowly shifts weight onto real phones as the network gets dense enough to actually compete. The cloud wasn't a compromise, it was genuinely the cheapest option until density caught up.
- **And the target was daunting.** To cover a realistic fixed overhead I was looking at needing something like **5,000 to 15,000 paying subscribers** just to break even, all while pricing *below* Proton Drive and Google One and still holding a 15%-plus margin. Threading that needle is a real business, not a side project.

Doing this math is what turned "cool idea" into "okay, this is genuinely hard." The cryptography was solvable. The economics were the boss fight.
<br />

## The part that actually scared me: the law
People assume the scary part of a project like this is the tech. For me it wasn't. **What actually kept me up was the law**, specifically data residency and **data provenance** rules.

Countries are increasingly demanding that data about their citizens be stored and handled inside their own borders under specific rules — India's **DPDPA**, Europe's **GDPR**, and others. If shards of an Indian user's file end up sitting on a phone in another country, am I breaking Indian law? If a European's data is scattered across a dozen jurisdictions, what does GDPR say about that? I read into the **new Indian data laws** and looked hard at the **fines companies have actually been hit with** for getting this wrong — and they are the kind of numbers that don't just hurt a startup, they end it. One mistake here isn't a bug, it's a company-ending event.

I did have an argument I was building as a defense, which I nicknamed "digital dust": a single encrypted shard is mathematically indistinguishable from random noise, so maybe, legally, it isn't "personal data" at all — you can't identify anyone from pure static. I even wrote tests to back the claim (measuring the shards' entropy, confirming they don't compress and hold no readable structure). But I'll be honest: that argument was a *hope*, not a certainty. Whether it would survive an actual regulator or courtroom, I had no idea. The laws scared me a lot more than my clever workaround reassured me, and that fear was one of the biggest weights on the whole project.
<br />

## How it was supposed to work
Under the hood, the architecture is the part I'm proudest of, and there's a lot more to it than just "encrypt the files." The whole system was designed around one idea: **the network can be made of unreliable strangers' phones and still be trustworthy.**

**The coordinator.** At the center sits a coordinator server that does the routing — it tracks which shards live where, issues challenges, and helps clients find their data again. Crucially, it **never sees your plaintext and never holds your keys.** It's a switchboard, not a vault. That single property is what the whole privacy argument rests on.

**Erasure coding.** Files aren't just copied around — that would waste huge amounts of space. I used **Reed-Solomon erasure coding**, which splits a file into `n` shards where *any* `k` of them rebuild the whole thing. Hosts can drop offline, lose power, or vanish entirely, and as long as enough shards survive somewhere, your file reconstructs perfectly.

The math runs over a finite field called GF(256), and this is where I lost hours. You need a *primitive polynomial* where 2 is a true generator (full multiplicative order 255). Every tutorial online pastes `0x11b`, the AES polynomial — under which 2 only generates a 51-element subgroup, quietly corrupting the lookup tables and producing garbage. The correct one is `0x11d`:

~~~python
_PRIM = 0x11d   # x^8 + x^4 + x^3 + x^2 + 1  (2 is primitive, order 255)

def _build_gf_tables():
    exp = [0] * 512   # doubled for wrap-around in mul
    log = [0] * 256
    x = 1
    for i in range(255):
        exp[i] = x
        log[x] = i
        x <<= 1
        if x >= 256:
            x ^= _PRIM   # reduce mod the primitive polynomial
    ...
~~~

One wrong hex constant and every shard you produce is silently wrong. That bug taught me to actually *verify* the mathematical assumption instead of trusting a snippet.

**Encryption.** Each file is encrypted with AES-256 on the user's own device *before* it's ever split, so every shard is meaningless on its own and the key never leaves your phone. Important, but honestly it's the simplest brick in the wall.

**Proof of storage.** This was my favourite problem. If I'm paying a host to store a shard, what stops them from quietly deleting it and just *pretending* it's still there? My answer was a **Merkle challenge-response** system.

The coordinator keeps only the Merkle **root**. Every few hours it sends a fresh random nonce, and the chunk you must prove is *derived from that nonce* — so the host can't precompute which chunk to keep:

~~~python
def gen_challenge(n_chunks):
    nonce = os.urandom(32)
    idx   = int.from_bytes(_H(nonce)[:4], 'big') % n_chunks
    return nonce, idx      # host cannot predict idx before seeing the nonce
~~~

The host then has to return `SHA256(nonce || chunk)` **and** a Merkle path back to the root, within a timeout. Both checks are needed, and that's the subtle part: a hash alone can be fabricated from invented bytes, so only the Merkle proof actually requires *possessing the real data*.

There's one more trap I fell into. My first version stored the left/right direction as a flag inside the proof, which meant verification never really used the index — so you could prove the *wrong* chunk and still pass, defeating the entire point. The fix is that direction must be **derived from the index** at each level, never trusted from the proof:

~~~python
def check_proof(root, chunk_bytes, idx, proof):
    """Direction at each level is DERIVED from idx, not stored in the proof,
    so passing the wrong idx genuinely changes the hash and fails."""
~~~

Hosts that pass reliably earn a higher payment tier; hosts that start failing get their shard **automatically rebuilt elsewhere** before the user notices. And I never oversold it — the script literally simulates four host archetypes (honest, fabricator, partial, cloud-outsourced) and ends with an explicit "is this foolproof?" breakdown of what's cryptographically prevented versus merely economically discouraged versus genuinely unsolved.

**Follow-the-sun and the phone problem.** Phones are terrible always-on servers: batteries, spotty wifi, and operating systems that aggressively kill background apps. So the rule was that a phone only acts as a host when it's **plugged in and on wifi** — for most people, overnight. Then you **pair time zones on opposite sides of the planet**, so one region's sleeping, charging phones serve another region's wide-awake users. A US-and-Southeast-Asia pairing, for instance, covers nearly the whole clock between them. I loved that piece of design.
<br />

## Why I hit pause
If I cracked the hard parts, why stop? Because a working core is maybe 10% of a real company, and the rest is where reality caught up:

- **The economics only work under narrow conditions** — the right host mix, enough density, low enough churn — and getting there is a cold-start mountain.
- **The legal exposure genuinely frightened me**, and a student can't out-lawyer a national data regulator.
- **The unsolved list stayed long** — key recovery if a user loses their device, phones even being able to reach each other through home routers, keeping the coordinator itself always-online, whether to build backup or full real-time sync. Each is its own hard project.
- **It needed things I don't have yet** — a team, real capital, and frankly more knowledge than I've got right now. I hit the honest edge of what I can do solo, today.
<br />

## What I walked away with
Even paused, this is the most I've ever learned from a single project. I came out actually understanding erasure coding, authenticated encryption, Merkle proofs, distributed-systems trade-offs, real unit-economics modeling, and a genuinely uncomfortable amount of international data law. I have proofs-of-concept that run and are tested (the Reed-Solomon erasure coding, the encrypted "digital dust" analysis, and the Merkle proof-of-storage protocol each have their own working script), and a business model I've stress-tested from six directions.

More than any of that, Constellation taught me to think like a founder instead of just an engineer — including the hardest skill of all, which is knowing when a thing is bigger than you are *right now* and having the discipline to set it down instead of half-building it. It's sitting in a folder, proofs-of-concept and all, waiting for a version of me with the team, the resources, and the experience it actually deserves. And I fully intend to become that person.
