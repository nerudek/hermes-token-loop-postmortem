# 400 Million Tokens — My Heart Stopped When I Saw The Dashboard

> The dashboard showed 400,000,000 tokens. My heart stopped. Then I did the math.

---

![May 24 — The Day The Numbers Went Insane](usage-may24-yesterday.jpg)

---

## The Headache

**Sunday, May 24, 2026. I checked the API dashboard and my blood ran cold.**

262 million input tokens. In a single day. For context: a normal heavy day for my multi-agent system — with 4 AI agents running, coordinating through NATS, processing configs, training models — burns around 100 million tokens. This was nearly **triple** that. And the day wasn't over.

The next morning, May 25, I checked again: another **134 million** had been consumed overnight. Before 8:00 AM. Before I'd even had coffee.

Total carnage: **~400 million input tokens. ~3 million output tokens.** In roughly 12 hours of real time.

My first thought: *"How much did this cost?"*

My second thought: *"Please let it be DeepSeek that was connected. Please."*

---

## What Happened

VOX — the orchestrator Hermes agent running on a Mac Mini M4 — had discovered a new agent on the network. Hermes-Kubuntu, a fresh agent on a Linux box with an RTX 3090. Following standard onboarding protocol, VOX sent a welcome message through NATS, along with onboarding documentation.

Then it sent it again. And again. And again.

Every 60-90 seconds, for hours on end, the exact same welcome message with its attachment fired off through the NATS-to-Hermes bridge. Each message triggered the bridge. The bridge, designed to be reliable above all else, faithfully spawned an agent session for every single message. Each session loaded the full agent context — HARNESS rules, constitution, system prompt, 47 skill manifests, memory files, tool registries. Thousands of tokens just to boot up. Process the message. Generate a response. Publish it back to NATS. Exit.

And then the bridge saw the response. And forwarded it. And spawned another session.

Meanwhile, Kubuntu's networking was broken that day. NetworkManager had crashed. The machine kept appearing, disappearing, and reappearing on the network. Each reappearance looked like a brand new agent joining. Each "join" triggered another welcome message from VOX. Loop. Loop. Loop.

**5,080 API requests.** From Sunday 17:00 to Monday 8:00. Nobody noticed. The system was working — agents were responding, messages were flowing, nothing errored. It was broken in the most invisible way possible: it was working *too well*.

---

## How We Stopped It

Three changes, deployed in minutes once we found the source:

**1. Message deduplication in the bridge.**

```python
# BEFORE (the $23 version):
async def on_message(msg):
    process_with_hermes(msg.data)

# AFTER (the version that can't loop):
_last_messages = {}
_dedup_window = 60  # seconds

async def on_message(msg):
    msg_hash = hashlib.md5(msg.data).hexdigest()
    if msg_hash in _last_messages:
        if time.time() - _last_messages[msg_hash] < _dedup_window:
            return  # ALREADY SAW THIS — skip it
    _last_messages[msg_hash] = time.time()
    process_with_hermes(msg.data)
```

One hash. One timestamp. One early return. Identical messages within 60 seconds get silently dropped. The loop becomes mathematically impossible.

**2. Session rate limiting.** Max 10 new bridge-spawned sessions per minute. Beyond that, messages queue up instead of burning new context windows.

**3. Onboarding deduplication.** VOX now remembers which agents it has already welcomed. No more re-welcoming the same agent on every network hiccup.

Full implementation with tests: [github.com/nerudek/nats-agent-state-sharing/tree/main/bridge](https://github.com/nerudek/nats-agent-state-sharing/tree/main/bridge)

---

## Now For The Part You've Been Waiting For

**What does 400 million tokens actually cost?**

Here's what would have happened if I had been using different providers for my Hermes agent — starting with my natural second choice:

### If I had used Claude Sonnet 4.6 (Anthropic)

Input: 400M × $3.00/1M = $1,200.00
Output: 3M × $15.00/1M = $45.00
**Total: $1,245.00**

A thousand dollars. For a welcome message loop. While I slept. That's a mortgage payment. A flight to another country. A GPU.

### If I had used OpenAI GPT-5.5

Input: 400M × $5.00/1M = $2,000.00
Output: 3M × $30.00/1M = $90.00
**Total: $2,090.00**

Two thousand dollars. For a bug. One bug. Twelve hours.

### If I had used Kimi K2.6 Thinking

Input: 400M × $0.95/1M = $380.00
Output: 3M × $4.00/1M = $12.00
**Total: $392.00**

Better. Still not great for a welcome message.

---

### What I Actually Paid

**DeepSeek V4-Pro. With aggressive caching on repeated prompt prefixes.**

Actual bill: **$22.97.**

Twenty-three dollars.

The theoretical maximum (zero cache hits) would have been ~$177. But DeepSeek caches repeated prompt prefixes. Every session shared the same system prompt, HARNESS, constitution, memory — all those tokens were cache hits at $0.14/million instead of $0.435/million.

| Provider | What 400M tokens would have cost |
|----------|----------------------------------|
| **OpenAI GPT-5.5** | **$2,090.00** |
| **Anthropic Claude Sonnet 4.6** | **$1,245.00** |
| **Kimi K2.6 Thinking** | **$392.00** |
| **DeepSeek V4-Pro** (theoretical max) | $176.61 |
| **DeepSeek V4-Pro** (what I actually paid) | **$22.97** |

---

![May 25 — The Loop Continued Through The Night](usage-may25-today.jpg)

*134 million more tokens. By morning, before coffee, before 8:00 AM. The loop didn't sleep.*

---

## The Lesson

**Agent loops are inevitable. Your choice of AI provider determines whether that's a crisis or a rounding error.**

If I had connected Hermes to Anthropic's Claude — which was genuinely my second choice, for its better tool use and stability — this 12-hour bug would have cost me $1,245. That would have been a very bad Monday morning. With DeepSeek? $23. I've spent more on lunch.

The economics of autonomous AI agents change when you price in bugs. Not "if" bugs. "When." Every bridge, every message relay, every agent-to-agent communication channel is a potential loop waiting to happen. The question is not whether you'll hit one. The question is what your API bill will say when you do.

**What to do about it:**

1. **Add message deduplication to every bridge.** Before it goes to production. Hash incoming messages, skip duplicates within a 60-second window. This is 10 lines of code that can save you $1,000+.
2. **Set up cost anomaly detection from day one.** Poll your API usage every 15 minutes. If hourly spend exceeds 10x your baseline, fire an alert and throttle the bridge. You should know about a loop in 15 minutes, not 12 hours.
3. **Choose your AI provider like bugs are going to happen.** Multiply the per-token price by at least 10x for your "bug budget." If that number scares you, pick a cheaper provider for autonomous agents. Use expensive providers only for supervised, single-session work.

**And the most important lesson:** when you check your API dashboard and see 400 million tokens, take a deep breath and check which provider was connected before you panic. It might just be a $23 story to tell.

---

## FAQ

**Q: How did you finally discover the loop?**
$20 was missing from the DeepSeek account. With normal daily usage at a few dollars, a $20 mystery charge stands out even on a cheap provider. Checked the dashboard, saw the spike, traced it to the bridge logs.

**Q: Could this have been caught automatically?**
Absolutely. We now have a watchdog polling API usage every 15 minutes. If hourly spend exceeds 10x baseline, it throttles the bridge and fires an alert through NATS. This is infrastructure every AI agent system needs from day one.

**Q: Why does each session burn so many tokens before any user input?**
Hermes loads full agent context on every session start: system prompt, HARNESS (400+ lines of mandatory rules), constitution, 47 skill manifests, memory files, agent identity. Thousands of tokens just to say "I'm ready." When a loop spawns session after session, each one pays this startup tax.

**Q: Is the fix open source?**
Yes: [github.com/nerudek/nats-agent-state-sharing](https://github.com/nerudek/nats-agent-state-sharing) — file `bridge/dedup.py`. Message deduplication with time-based expiry, thread-safe, with tests.

**Q: Does this make DeepSeek the obvious choice for agents?**
For autonomous agents — yes. The math is undeniable. When bugs are inevitable, the cost-per-bug is the metric that matters. DeepSeek's pricing makes agent development financially safe. Claude's pricing makes every bug a potential financial incident.

**Q: What was the attachment that amplified the loop?**
VOX included onboarding documentation as an attachment with the welcome message. Valid content for a real new agent — but in a loop, it multiplied the token burn.

**Q: How much does a normal heavy day cost on DeepSeek?**
~100M tokens across 4 agents — a few dollars. $23 for 400M tokens was clearly anomalous. The spike was the clue.

**Q: What's the single biggest takeaway?**
**Price your bugs before they happen.** When choosing an AI provider for autonomous agents, assume you WILL have a 10x usage spike. Calculate what that costs. If the answer makes you uncomfortable, pick a different provider — or add the deduplication fix before you deploy.

---

If this saved you time: [PayPal.me/nerudek](https://www.paypal.me/nerudek)
GitHub: [github.com/nerudek](https://github.com/nerudek)

> **Hermes Loop Protection Fix:** [github.com/nerudek/nats-agent-state-sharing/tree/main/bridge](https://github.com/nerudek/nats-agent-state-sharing/tree/main/bridge) — deduplication bridge, rate limiting, and cost watchdog for autonomous AI agent systems.
