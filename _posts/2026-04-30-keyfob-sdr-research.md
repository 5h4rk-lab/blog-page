---
description: >-
  Reverse engineering a car key fob with a $25 RTL-SDR dongle — signal capture, protocol decoding, and KeeLoq rolling code analysis.
title: "I Reverse Engineered My Car Key With a $25 Dongle"
date: 2026-04-30 14:00:00 -0400
categories: [RF Security]
tags: [sdr, rf, hardware, keyfob, keeloq, urh, gqrx, security]
show_image_post: true
image: /assets/img/sdr/keyfob.png
---

So I picked up an RTL-SDR dongle recently — one of those cheap USB sticks that turns your laptop into a radio receiver. The plan was to listen to aircraft ACARS messages, maybe pick up some weather satellite images. Standard SDR rabbit hole stuff.

But then I got bored waiting for planes and realized I had my car keys sitting right next to me.

What followed was about two hours of going way deeper than I planned, and now I know exactly how my car key fob works at the bit level. Here's the full breakdown.

enough of the intro, let's dive in...

## What's actually happening when you press your key fob

Most people press the unlock button and don't think twice about it. There's a small radio transmitter in your fob that fires a signal at your car, the car receives it, and the doors unlock. Simple.

What's actually happening underneath is more interesting. Your fob transmits a short RF burst on a specific frequency — 315 MHz for most US cars, 433 MHz in Europe — and embedded in that burst is a code. The car checks that code, decides if it's valid, and responds accordingly.

The security question is: *what kind of code?* Because if it's the same code every time, I could just record it once and replay it to unlock your car whenever I want. Turns out manufacturers figured this out decades ago, which is why modern fobs use something called a **rolling code**.

## The setup

**Hardware:**
- Nooelec NESDR SMArt v5 (RTL-SDR dongle, ~$25)
- Stock antenna it came with
- Laptop running Manjaro Linux

**Software:**
- GQRX — for visualizing the signal on a waterfall display
- Universal Radio Hacker (URH) — for capturing and decoding
- inspectrum — for signal analysis

Total cost: whatever you paid for the dongle. Everything else is free and open source.


## Step 1: Finding the signal

First thing I did was open GQRX, tune to 315 MHz, and press my key fob a few times. Within seconds I could see it — a sharp spike bursting up from the noise floor every single time I pressed the button.

The signal was sitting right at 315.000 MHz dead on. Clean narrow spike, characteristic of OOK (On-Off Keying) modulation — basically the transmitter is either fully on or fully off, no frequency shifting. Each button press produced a distinct burst, and between presses there was nothing but noise floor.

Already useful information. I know the exact frequency, I can see the signal shape, and I can confirm the fob is working.

![GQRX waterfall showing key fob signal burst at 315 MHz](/assets/img/sdr/gqrx-waterfall.png)
<!-- TODO: your GQRX screenshot showing the yellow burst spike at 315 MHz -- you already took this one -->

## Step 2: Capturing the raw signal

Closed GQRX (only one app can use the dongle at a time), opened URH, and recorded about 15 button presses with a second or two between each one.

URH showed me this in the waveform view:

```
[burst] ... silence ... [burst] ... silence ... [burst]
```

Each burst is one button press. Each silence is me not pressing anything. You can already see the structure — the fob sends the same type of packet every time, separated by dead air.

Here's where it got interesting. URH tried to auto-decode the signal and initially gave me garbage — all zeros. The issue was the encoding. The raw bits coming off the antenna aren't just plain 1s and 0s. The fob uses **Manchester II encoding**, which maps every high-then-low transition to a `1` and every low-then-high to a `0`. It's done this way to keep the signal self-clocking — the receiver can stay synchronized even over a noisy channel.

Once I switched URH's decoding to Manchester II, everything snapped into place.

![URH waveform view showing 15 captured key fob bursts](/assets/img/sdr/urh-waveform.png)
<!-- TODO: URH Interpretation tab screenshot showing the signal bursts with Manchester II decoding active -->

## Step 3: The packet structure

With correct decoding, each transmission broke down into three distinct parts:

### Preamble
`0xFF x26` — the wake-up sequence. It's `10101010...` repeated, which tells the car's receiver "hey, something's coming, start listening." Identical on every single packet. The car uses this to sync its internal clock to the incoming signal.

### Fixed header
`0xE3E137` — this never changes across all 15 captures. This is your fob's identity — it contains the manufacturer ID, the fob's unique serial number, and which button was pressed (lock, unlock, trunk). The car uses this to know which fob it's talking to and what you're asking it to do.

### Rolling code
This is where it gets interesting.

## Step 4: The rolling code

Here's what the rolling code looked like across 5 consecutive button presses:

```
Press  1: a7083938554741731b2
Press  2: b3cbfeecdd38e839eb1
Press  3: a7746c5690474479f60
Press  4: ee377331513f98c8f04
Press  5: ecd9948ab345d01f672
```

No pattern. No linear increment. The delta between consecutive codes is a massive random-looking number each time. That's **KeeLoq** doing its job.

### What is KeeLoq?

KeeLoq is a proprietary block cipher developed by Nanoteq and later acquired by Microchip Technology. It's been the standard for automotive remote entry since the late 80s. The idea is simple: your fob and your car share a secret 64-bit encryption key baked in at the factory. Every time you press the button, the fob increments an internal counter and encrypts it with that key. The car does the same math on its end and checks if what it received is valid.

Because the counter is encrypted rather than transmitted in plaintext, an attacker watching your transmissions sees what looks like random garbage every time. You can't predict the next code without knowing the secret key.

I ran a quick entropy check on all 15 rolling codes:

```
Chi-square statistic: 8.22
All 16 hex digits present: yes
Missing digits: none
```

A chi-square of 8.22 is actually better than you'd expect from purely random data (~15). The encryption is doing exactly what it's supposed to — producing output that's statistically indistinguishable from noise.

![URH Analysis tab showing fixed header in yellow/green and rolling code diffs in red](/assets/img/sdr/urh-analysis-diff.png)
<!-- TODO: URH Analysis tab screenshot with "Mark diffs in protocol" checked — the one showing red bits in the rolling code region -->

## So is it actually secure?

mostly yes. a basic replay attack — capture the code once, replay it later — doesn't work because the car maintains a counter and won't accept a code it's already seen.

but there are real attack vectors worth knowing about.

### RollJam (Samy Kamkar, DEF CON 2015)

This one is clever. An attacker carries a device that simultaneously jams 315 MHz and records incoming signals. Here's the flow:

```
You press fob  →  attacker jams signal + records code N (car never hears it)
You press again →  attacker records code N+1, releases jam
Car hears N+1  →  unlocks
Attacker holds code N  →  replays it later  →  car accepts it
```

Why does the car accept an old code? Because manufacturers built in a tolerance window — if you're out of range and press the button a few times, your fob's counter gets ahead of the car's counter. Cars accept codes within roughly ±512 of the last seen value to account for this. RollJam exploits exactly that window.

Worth noting — this requires active jamming hardware (like a HackRF or similar), not just a passive receiver. Your RTL-SDR alone can't pull this off since it's receive-only.

![RollJam attack flow diagram](/assets/img/sdr/rolljam-diagram.png)
<!-- TODO: either draw a simple diagram of the RollJam flow or grab Samy Kamkar's DEF CON slide showing the attack sequence -->

### Relay attack

This is how most keyless entry theft actually happens in practice. An attacker uses two radios to relay the fob's signal over any distance — one device near your house amplifies and retransmits to a second device near your car. The car thinks the fob is right there. No codes involved, no crypto to break.

### KeeLoq cryptanalysis (2008)

Researchers at RU Bochum and TU Graz published a full cryptanalysis of KeeLoq showing the 64-bit key can be recovered with ~65,000 captured messages using differential power analysis, or computationally with ~2^38 operations if you know the manufacturer's master key. Not a practical street-level threat, but the cipher is fundamentally broken in a research sense.

## What I walked away with

The whole thing took one afternoon with a $25 dongle and free software. I went from zero to having the full hex dump of my car key's protocol, knowing exactly which bits are fixed vs rolling, and being able to articulate the exact attack surface on my specific fob.

The tech in your key fob has been standardized since the 90s and the security model is well understood. It's good enough to stop the vast majority of attacks, but there are known vectors if someone really wants in.

More importantly — this whole workflow maps directly to any RF security assessment. IoT sensors, garage doors, alarm systems, industrial remote controls — they all follow the same pattern. Identify the frequency, capture the signal, figure out the encoding, decode the protocol, analyze the security model. The tools and methodology scale.

If you have an RTL-SDR sitting around, try it on your own fob. Takes maybe 20 minutes to get to signal visualization in GQRX, and another hour to get to decoded hex in URH. Everything I used runs on Linux and costs nothing.

---

*Tools: Nooelec NESDR SMArt v5, GQRX, Universal Radio Hacker, inspectrum — Manjaro Linux*

*All testing performed on my own vehicle and key fob.*

If you want to chat about RF security or SDR stuff feel free to DM me on Twitter. Always down to talk hardware hacking.
