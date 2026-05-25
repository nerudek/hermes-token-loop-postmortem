# 400 Million Tokens Burned Overnight

> 5,080 API requests. Everything looked normal.

---

## My Heart Stopped At 8:03 AM

Sunday, May 24, 2026.

I opened the API dashboard and my stomach dropped.

262 million input tokens consumed in a single day.

For context: a normal heavy day for my multi-agent system — with 4 AI agents coordinating through NATS, processing configs, moving files, training models, and handling orchestration tasks — usually burns around 100 million tokens.

This was nearly triple that.

And the day wasn't even over.

The next morning, May 25, before coffee, I checked again.

Another 134 million input tokens had been consumed overnight.

Total damage:

| Metric | Value |
|--------|-------|
| Input tokens | ~400 million |
| Output tokens | ~3 million |
| API requests | 5,080 |
| Runtime | ~15 hours |

My first thought:

*"How much did this cost?"*

My second thought:

*"Please let it be DeepSeek connected to production. Please."*

---

![May 24 — 262 million tokens before the day was even over](usage-may24-yesterday.jpg)

---

## What Happened

An orchestrator agent running on a Mac Mini M4 discovered a new agent on the network.

A secondary agent had just come online on a Linux machine with an RTX 3090 GPU.

Following standard onboarding protocol, the orchestrator sent a welcome message through NATS along with onboarding documentation and initialization context.

That message was correct.

**The problem:**

It never stopped sending it.

Every 60-90 seconds, the orchestrator re-sent the same onboarding payload.

The NATS-to-Hermes bridge service faithfully forwarded every incoming message to Hermes for processing.

Each forwarded message spawned a fresh agent session.

And every session loaded the full startup context:

- HARNESS
- system prompt
- constitution
- agent memory
- tool registry
- onboarding guides
- skill manifests
- runtime instructions

Thousands of tokens.

Every single time.

The session processed the message, generated a response, exited, and waited for the next event.

Then another identical onboarding message arrived.

Another session spawned.

Another full context load.

Again. And again. And again.

**5,080 times in roughly 15 hours.**

---

![May 25 — 134 million more tokens by morning](usage-may25-today.jpg)

---

## The terrifying part

Nothing looked broken.

The agents responded normally. No crashes. No red alerts. No failing health checks.

From the outside, the system appeared healthy.

---

## Why Nobody Noticed

For 15 hours, the loop quietly burned tokens in the background.

Several things made it unusually hard to detect:

**1. The system was technically "working"**

Messages flowed correctly. Agents replied correctly. Tasks completed successfully. Nothing visibly failed.

**2. Agent startup is deceptively expensive**

Most of the burn came from repeatedly loading massive context windows — not model outputs. Every new session loaded the full orchestration environment before doing any work. A tiny onboarding ping triggered tens of thousands of input tokens. Over and over.

**3. Session budgets didn't help**

Each individual session stayed within limits. But the loop continuously spawned brand-new sessions. Per-session token limits are useless if you accidentally create infinite sessions.

**4. Rate limiting didn't help either**

Even with request throttling, every request still consumed context tokens. A slow infinite loop is still an infinite loop.

**5. Monitoring lagged behind reality**

We checked usage dashboards manually. Once per day. By the time we saw the spike, the loop had already been running all night.

**6. Killing the process didn't stop it**

The bridge daemon was managed by launchd. Killing the process simply restarted it automatically. We had to unload the daemon entirely before the loop finally stopped.

---

## The Root Cause

The issue came from an ugly interaction between:

- network discovery
- onboarding retries
- and a bridge with no deduplication layer

The secondary agent had unstable connectivity during onboarding. It repeatedly appeared and disappeared from the network. Each rediscovery triggered another "welcome" event. The bridge forwarded every event blindly. Hermes processed each one as brand-new.

Positive feedback loop:

```
Onboarding event
    ↓
NATS message
    ↓
Bridge forwards event
    ↓
Hermes session spawns
    ↓
Context loads
    ↓
Response generated
    ↓
Network rediscovery
    ↓
Onboarding event again
```

Repeat for 15 hours.

---

## The Fix

The actual fix was surprisingly small. Three changes stopped the entire cascade.

**1. Message deduplication**

The critical fix. The bridge now hashes incoming onboarding payloads and ignores duplicates within a cooldown window.

**2. Session spawn protection**

Repeated onboarding events from the same agent are now collapsed into a single active session.

**3. Real-time token monitoring**

We added live token-rate alerts instead of daily dashboard checks. If token velocity spikes abnormally, the bridge now alerts immediately.

Full implementation: [github.com/nerudek/nats-agent-state-sharing/tree/main/bridge](https://github.com/nerudek/nats-agent-state-sharing/tree/main/bridge)

---

## The Cost

Now for the part that genuinely scared me.

I calculated what this exact same bug would have cost across different providers.

**The bug was identical. Only the API provider changed.**

| Provider | Estimated Cost |
|----------|---------------|
| Anthropic Claude Sonnet | ~$1,245 |
| OpenAI GPT-5-class pricing | ~$2,090 |
| Moonshot Kimi | ~$392 |
| **DeepSeek** | **$22.97** |

That's the moment I finally exhaled.

The engineering mistake was real. The token burn was real. The 400 million tokens were very real.

But the provider choice was the difference between:

*"Well... that was horrifying"*

and

*"We need to explain this to accounting."*

---

## Lessons Learned

AI agent systems fail differently than traditional software.

The dangerous bugs are not always crashes. Sometimes the system works perfectly while silently setting money on fire.

And once you start chaining together: autonomous agents, bridges, retries, onboarding protocols, daemon restarts, and massive context windows — tiny logic mistakes become infrastructure-scale problems surprisingly fast.

One missing deduplication check created:

- 5,080 requests
- ~400 million input tokens
- and 15 hours of invisible burn

The scariest part?

**From the outside, everything looked normal.**

---

If this saved you time: [PayPal.me/nerudek](https://www.paypal.me/nerudek)
GitHub: [github.com/nerudek](https://github.com/nerudek)

> **Hermes Loop Protection Fix:** [github.com/nerudek/nats-agent-state-sharing/tree/main/bridge](https://github.com/nerudek/nats-agent-state-sharing/tree/main/bridge)
