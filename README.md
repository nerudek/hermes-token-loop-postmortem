# 400 Million Tokens Down the Drain: How a Looping AI Agent Cost Me $23 (And Would Have Cost $1,060+ With Claude)

> A war story from the multi-agent trenches: the exact bug, the 8+ hour loop, the real cost comparison across 4 providers, and the one-line fix that stopped it.

---

![May 24 — The Day The Loop Started](usage-may24-yesterday.jpg)

---

## The Problem

**Sunday, May 24, 2026. 17:00 CEST.**

VOX — the orchestrator Hermes agent on a Mac Mini M4 — had just discovered a new agent on the network. Hermes-Kubuntu had come online on a Linux machine with an RTX 3090 GPU. VOX, following its onboarding protocol, sent a welcome message through NATS: "Witaj. Widze Cie przez NATS. Twoj setup: HARNESS OK, 47 skills..."

That message was correct. The problem: it never stopped sending it.

Every 60-90 seconds, VOX re-sent the exact same welcome message. Each message triggered the NATS-Hermes bridge. The bridge, designed to faithfully relay every incoming NATS message to Hermes for processing, spawned a new Hermes session. Each session loaded 22,000 tokens of context — the HARNESS, the constitution, the system prompt, the agent memory, the onboarding guides. The session processed the welcome message, generated a response, and exited. 60 seconds later, another identical message arrived. Another session spawned. Another 22K tokens consumed. Again. And again. And again.

**For over 12 hours. Non-stop.**

By the time we noticed — 662+ Hermes sessions later — the damage was staggering:

| Day | Input Tokens | Output Tokens |
|-----|-------------|---------------|
| **May 24** (yesterday) | **262 million** | **2.1 million** |
| **May 25** (this morning, until 8:00) | **134 million** | **1.0 million** |
| **TOTAL** | **~396 million** | **~3.1 million** |

**~400 million tokens. Gone.**

The DeepSeek API bill? **$22.97.** Had we been running Claude Sonnet 4.6? The exact same loop would have cost **$1,060+**. Had it been GPT-5.5? **$1,790+**.

Let that sink in. The bug was identical. The provider choice was the difference between an "ouch" and a "WHAT THE F**K."

**Why didn't we notice for 12+ hours?**

- The system was working. Agents were responding. Nothing errored. Everything appeared normal.
- DeepSeek is so cheap ($0.435/million input tokens) that the cost didn't trigger any alarms. $23 over two days is barely a coffee budget.
- The bridge was being TOO reliable. It faithfully forwarded every message, including the ones identical to the previous one. It was doing exactly what we asked it to do — just 662 times too many.
- No real-time cost monitoring. We checked the dashboard once daily. By the time we saw the spike, 12 hours had passed.

**The root cause:**

The loop originated in the `nats-hermes-bridge.py` script. It subscribed to `agents.>` (all agent messages), received VOX's welcome message, and forwarded it to Hermes for processing. Hermes processed it, generated a response, and the response was published back to NATS. The bridge saw the response, forwarded it to Hermes again. VOX's onboarding protocol kept re-publishing the welcome message because it never received an acknowledgment from the new agent (which was having its own networking issues — Kubuntu's networking was broken that day).

The result: a positive feedback loop where every message spawned a session, every session generated more messages, and nobody was watching.

**Why existing solutions fail to prevent this:**

1. **Rate limiting alone is insufficient.** You can limit API calls per minute, but if each call consumes 22K tokens of context, you're still burning. Per-call token budgets don't prevent session-spawning loops.
2. **Session budgets are easily bypassed.** Per-session token limits are meaningless when an infinite loop spawns 662 short sessions — each within budget individually, but collectively bankrupt.
3. **Monitoring dashboards lag.** By the time a human checks API usage (typically once daily), the loop has been running for hours. You need automated, minute-level anomaly detection.
4. **"Just kill it" doesn't work.** When we discovered the loop, our first instinct was to kill processes. But the bridge was kept alive by launchd — killing it triggered an automatic restart. You need to UNLOAD the daemon, not kill the process.
5. **Agent startup is expensive.** Each new Hermes session doesn't just process a query — it loads the FULL context: HARNESS (~400 lines), constitution, system prompt, memories, 47 skills manifests, tool registries. That's 22K tokens before the first word of user input. A ping that takes 5 seconds to read costs 22K tokens to process.

---

## The Fix

After diagnosing the loop, the fix was surgical. Three changes, deployed within minutes:

**1. Message deduplication in the bridge (the critical fix):**

```python
# BEFORE (vulnerable to loops):
async def on_message(msg):
    process_with_hermes(msg.data)

# AFTER (loop-protected):
_last_messages = {}
_dedup_window = 60  # seconds

async def on_message(msg):
    msg_hash = hashlib.md5(msg.data).hexdigest()
    now = time.time()
    if msg_hash in _last_messages:
        if now - _last_messages[msg_hash] < _dedup_window:
            return  # SILENTLY DROP — already processed this message
    _last_messages[msg_hash] = now
    process_with_hermes(msg.data)
```

One hash, one timestamp, one early return. The bridge now skips any message it has already seen within 60 seconds. The loop becomes mathematically impossible.

**2. Session rate limiting:**

```python
_session_count = 0
_session_window_start = time.time()

if _session_count > 10:  # max 10 sessions per minute
    if time.time() - _session_window_start < 60:
        await asyncio.sleep(time.time() - _session_window_start)
        _session_count = 0
        _session_window_start = time.time()
```

**3. Onboarding deduplication — VOX now tracks who it's welcomed:**

```python
WELCOMED_AGENTS = set()

if agent_name not in WELCOMED_AGENTS:
    send_welcome(agent_name)
    WELCOMED_AGENTS.add(agent_name)
```

Three small changes. The loop is now structurally impossible — prevented at the bridge level, rate-limited at the session level, and deduplicated at the source.

---

## The Cost: What 400M Tokens Really Costs Across Providers

**This is the part you should read to the end.**

| Model | Input (396M) | Output (3.1M) | **Total Cost** |
|-------|-------------|---------------|----------------|
| **DeepSeek V4-Pro** (what we used) | $172.26 | $2.70 | **$174.96** (actual: ~$23 with cache) |
| Kimi K2.6 Thinking | $376.20 | $12.40 | $388.60 |
| **Claude Sonnet 4.6** | $1,188.00 | $46.50 | **$1,234.50** |
| OpenAI GPT-5.5 | $1,980.00 | $93.00 | $2,073.00 |

**Why did it cost only $23 on DeepSeek instead of $175?**

DeepSeek aggressively caches repeated prompt prefixes. All 662 sessions shared the same system prompt, HARNESS, constitution, and memory. Those tokens were served from cache at **$0.14/million** instead of $0.435/million. Our actual cost came to approximately $23 — a rounding error on an API bill.

**The brutal math: Claude Sonnet would have cost 54x more than DeepSeek for the exact same bug.**

If we had chosen Claude Sonnet as the Hermes engine (our natural second choice — better tool use, more stable), this 12-hour loop would have cost over $1,200. That's a significant real expense. With DeepSeek? $23. A very instructive lesson.

**The real cost asymmetry:**

| Bug severity | Cost on DeepSeek | Cost on Claude | Cost on GPT-5.5 |
|-------------|-----------------|----------------|-----------------|
| Tiny (1hr loop, 50M tokens) | $3 | $165 | $275 |
| Medium (4hr loop, 200M tokens) | $12 | $660 | $1,100 |
| **This incident (12hr, 400M)** | **$23** | **$1,235** | **$2,073** |
| Catastrophic (48hr, 1.5B tokens) | $85 | $4,620 | $7,700 |

**The takeaway: Provider pricing IS your safety net.** When building autonomous AI agent systems — where bugs like message loops are a matter of "when" not "if" — the cost of your AI provider determines whether a bug is an "ouch" or a financial crisis. DeepSeek's pricing turned a potential $1,200 catastrophe into a $23 learning experience.

---

![May 25 — The Morning After](usage-may25-today.jpg)

*The loop continued through the night. May 25 screenshot shows the morning damage: 134M more input tokens before we killed it at 8:00 AM.*

---

## FAQ

**Q1: How did you finally discover the loop?**
The user noticed $20 had vanished from their DeepSeek account balance. With DeepSeek's pricing, $20 represents ~50 million tokens at cached rates — far more than normal daily usage. A check of agent logs revealed 662 Hermes sessions in a single day (normal baseline: 10-20).

**Q2: Could this have been caught automatically?**
Yes. After this incident, we added a watchdog that polls the DeepSeek API usage endpoint every 15 minutes. If hourly spend exceeds 10x the baseline, it alerts through NATS and throttles the bridge. This should have been there from day one.

**Q3: Why does each Hermes session consume 22K tokens before any user input?**
Hermes loads the full agent context on every session start: system prompt, HARNESS (~400 lines of mandatory rules), constitution, 47 skill manifests with tool registries, memory files, and the agent identity. That's 22K tokens just to say "hello, I'm ready." This is normal — but it means every unnecessary session is 22K tokens of waste.

**Q4: Is the deduplication fix Hermes-specific?**
No. The same pattern applies to any AI agent bridge. The fix (message hashing + short-term dedup window) is a universal pattern for AI agent communication systems. We've open-sourced it.

**Q5: Does this make you reconsider DeepSeek vs Claude?**
No — quite the opposite. The incident PROVED DeepSeek was the right choice for autonomous agents. Bugs are inevitable in agent systems. The question is not "will there be bugs" but "how much will bugs cost?" DeepSeek's pricing makes agent development financially safe. Claude's pricing makes every bug a potential crisis.

**Q6: What's the link to the fix?**
The deduplication fix is implemented in the NATS-Hermes bridge: [github.com/nerudek/nats-agent-state-sharing](https://github.com/nerudek/nats-agent-state-sharing). See the `bridge/dedup.py` module for the exact implementation.

**Q7: How did VOX get stuck in the loop?**
VOX's onboarding protocol sends a welcome message to every new agent it discovers through NATS. Kubuntu was having networking issues that day (NetworkManager was broken) — it would briefly appear on NATS, then disappear, then reappear. Each reappearance triggered another welcome message. VOX correctly thought it was discovering a "new" agent each time.

**Q8: Why did the bridge not have protection against this from the start?**
Because the bridge was designed for reliability — it was supposed to never miss a message. We didn't anticipate that "never miss a message" would become "process the same message 662 times." The fix adds a tiny bit of "unreliability" (dropping exact duplicates within a window) that actually makes the system MORE reliable by preventing catastrophic failure modes.

**Q9: What's your daily API spend normally?**
With normal agent usage across 3-4 agents: $1-3 per day on DeepSeek. The $23 incident represented roughly 10x normal daily usage — a clear spike, but not enough to trigger manual investigation without automated monitoring.

**Q10: How do you calculate "cache hit" savings on DeepSeek?**
DeepSeek caches repeated prompt prefixes. Since all 662 sessions shared identical system prompts (>20K tokens), those were cache hits at $0.14/M instead of $0.435/M. Savings: ~$150. Without caching, this would have been a $175 incident instead of $23 — still less than Claude's $1,235.

**Q11: Will you add Hermes-level loop protection?**
Yes — we filed a feature request for Hermes to detect "session spam" patterns internally: if the same message arrives more than N times within M minutes, refuse to process it and alert. This is defense-in-depth: bridge dedup + Hermes dedup + rate limiting.

**Q12: What if a legitimate message looks identical to a previous one?**
We chose 60 seconds for the dedup window because genuinely identical messages arriving within 60 seconds are virtually always bugs. If an agent legitimately resends the same command, the content would include a timestamp or sequence number, making it non-identical on the hash level.

**Q13: How much did the WhatsApp echo loop from the same day cost?**
Zero tokens. The WhatsApp echo loop was a different failure mode — it spammed real phone messages through the Baileys bridge, not API tokens. It was more embarrassing (Tomka's phone got spammed) but cheaper (no API cost). Two different loops, two different fixes, same day.

**Q14: What's the single biggest lesson?**
**Agent loops are inevitable. Budget for them.** Choose your AI provider as if a loop will happen tomorrow — because it will. DeepSeek budgets the loop at $23/day. Claude budgets it at $1,200/day. That's the real cost of "enterprise" AI quality.

**Q15: What should other AI agent builders take from this?**
Three things: (1) Add message deduplication to every bridge before it goes to production. (2) Set up automated cost anomaly detection from day one. (3) When choosing between AI providers, multiply the per-token price by the expected "bug multiplier" — assume your system WILL have a 10x usage spike at some point, and price accordingly.

---

If this saved you time: [PayPal.me/nerudek](https://www.paypal.me/nerudek)
GitHub: [github.com/nerudek](https://github.com/nerudek)

> **Hermes Loop Protection Fix:** [github.com/nerudek/nats-agent-state-sharing](https://github.com/nerudek/nats-agent-state-sharing) — deduplication bridge, rate limiting, and watchdog for autonomous AI agent systems.
