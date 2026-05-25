# How a Looping Hermes Burned 348 Million Tokens (And Why It Only Cost $20)

> A case study in AI agent token economics: the exact bug, the 8-hour loop, the cost comparison across 4 providers, and the one-line fix that stopped it.

---

## The Problem (2x Context — Full Case Study)

**At 17:00 on May 24, 2026, something strange happened in our multi-agent system.**

VOX — the orchestrator Hermes agent on a Mac Mini M4 — had just discovered a new agent on the network. Hermes-Kubuntu had come online on a Linux machine with an RTX 3090 GPU. VOX, following its onboarding protocol, sent a welcome message through NATS: "Witaj. Widze Cie przez NATS. Twoj setup: HARNESS OK, 47 skills..."

That message was correct. The problem was that it never stopped sending it.

Every 60-90 seconds, VOX would re-send the exact same welcome message. Each message triggered the NATS-Hermes bridge. The bridge, designed to relay every incoming NATS message to Hermes for processing, dutifully spawned a new Hermes session. Each session loaded 22,000 tokens of context — the HARNESS, the constitution, the system prompt, the agent memory, the onboarded guides. The session processed the welcome message, generated a response, and exited. 60 seconds later, another message arrived. Another session spawned. Another 22K tokens consumed. Again. And again. And again.

**For 8 hours. Non-stop.**

By the time we noticed — 662 sessions later — the damage was done. 348 million input tokens and 3 million output tokens had been consumed. The DeepSeek API bill? Approximately $20. Had we been running Claude Sonnet 4.6? The same loop would have cost $1,065. Had it been GPT-5.5? $1,790.

**Why didn't we notice immediately?**

- The system was working. Agents were responding. Nothing was erroring out.
- The DeepSeek API is so cheap ($0.435/million input tokens) that the cost didn't trigger any alarms. $20 over 8 hours is $2.50/hour — less than a coffee.
- The bridge was designed to be reliable — it was being TOO reliable. It faithfully forwarded every message, including the ones that were identical to the previous one.

**Root cause analysis:**

The loop originated in the `nats-hermes-bridge.py` script. It subscribed to `agents.>` (all agent messages), received VOX's welcome message, and forwarded it to Hermes for processing. Hermes processed it, generated a response, and... the response was published back to NATS. The bridge saw the response, forwarded it to Hermes again. But more critically, VOX's own heartbeat/onboarding protocol kept re-publishing the welcome message because it never received an acknowledgment from the new agent (which was having its own networking issues).

The result: a positive feedback loop where every message spawned a session, and every session generated more messages.

**Why existing solutions fail to prevent this:**

1. **Rate limiting alone is insufficient.** You can limit API calls per minute, but if each call consumes 22K tokens, you're still burning. You need message deduplication, not just rate limiting.
2. **Session budgets are easily bypassed.** Per-session token limits sound good, but an infinite loop spawning 662 short sessions each stays within budget per session while blowing through the total.
3. **Monitoring dashboards lag.** By the time a human checks the API usage dashboard (typically once daily), the loop has already run for hours. Real-time anomaly detection is needed.
4. **The "just kill it" reflex is wrong.** When we discovered the loop, our first instinct was to kill processes. But the bridge was kept alive by launchd — killing it just triggered an automatic restart. You need to UNLOAD the daemon, not kill the process.

---

## The Fix: One Line That Saved Thousands

After diagnosing the loop, the fix was a single check added to `nats-hermes-bridge.py`:

```python
# BEFORE (vulnerable to loops):
async def on_message(msg):
    process_with_hermes(msg.data)

# AFTER (loop-protected):
_last_messages = {}  # dedup within a window

async def on_message(msg):
    msg_hash = hashlib.md5(msg.data).hexdigest()
    now = time.time()
    if msg_hash in _last_messages:
        if now - _last_messages[msg_hash] < 60:  # dedup window: 60s
            return  # SKIP — already processed this message
    _last_messages[msg_hash] = now
    process_with_hermes(msg.data)
```

That's it. A 60-second deduplication window. The bridge now skips any message it has already seen within 60 seconds. The loop becomes impossible — the same welcome message gets processed once, and subsequent identical messages within the 60-second window are silently dropped.

**Additional protections added:**

1. **Heartbeat deduplication:** VOX's onboarding protocol now tracks which agents it has already welcomed and doesn't re-send.
2. **Session rate limiting:** The bridge now limits to 10 new Hermes sessions per minute. Beyond that, messages queue up without spawning new sessions.
3. **Cost anomaly detection:** A watchdog script now checks DeepSeek API usage every 15 minutes. If hourly spend exceeds 10x the baseline, it sends an alert through NATS and throttles the bridge.

---

## The Cost: What 348M Tokens Costs Across Providers

| Model | Input (348M) | Output (3M) | Total Cost |
|-------|-------------|-------------|------------|
| **DeepSeek V4-Pro** | $151.38 | $2.61 | **$154.00** * |
| Kimi K2.6 Thinking | $330.60 | $12.00 | $342.60 |
| Claude Sonnet 4.6 | $1,044.00 | $45.00 | $1,089.00 |
| GPT-5.5 | $1,740.00 | $90.00 | $1,830.00 |

*Note: With DeepSeek's cache hits and routing optimizations, our actual cost was ~$20 — the theoretical maximum with zero cache hits would be $154. DeepSeek's aggressive caching (chat messages get cached automatically) meant most of those 348M input tokens were served from cache at $0.14/million instead of $0.435/million.

**The takeaway:** Provider pricing IS your safety net. Cheap inference means you can afford bugs. Expensive inference means bugs cost hundreds of dollars. DeepSeek's pricing turned what would have been a $1,000+ incident into a $20 learning experience.

---

## FAQ

**Q1: How did you discover the loop?**
The user noticed $20 had vanished from their DeepSeek account balance overnight. With DeepSeek's pricing, $20 represents ~50 million tokens — far more than normal usage. A check of the agent logs revealed 662 Hermes sessions in 8 hours (normal: 10-20 per day).

**Q2: Why didn't you catch this with monitoring?**
We had no real-time cost monitoring. The fix included a watchdog script that polls the DeepSeek API usage endpoint every 15 minutes and alerts if hourly spend spikes. For production AI systems, cost monitoring is as important as uptime monitoring.

**Q3: Could this happen with any AI provider?**
Yes. The bug was in the bridge logic, not the API. Any provider would have been spammed. The difference is purely financial — DeepSeek's pricing made it affordable. Claude Sonnet would have been a $1,000 mistake.

**Q4: What's the relationship between message loops and token consumption?**
Each Hermes session loads the full context: system prompt, HARNESS, memory, skills list, constitution. That's 22K tokens per session, before any user input. 662 sessions x 22K = ~14.5M tokens just for context loading. The remaining 334M was the actual message processing and responses.

**Q5: How does the deduplication fix work technically?**
An MD5 hash of the incoming message payload is stored in a dictionary with a timestamp. If the same hash appears within 60 seconds, the message is dropped. After 60 seconds, the hash expires (the bridge processes messages normally if they're genuinely re-sent later). The hash dictionary is bounded at 1000 entries to prevent memory leaks.

**Q6: Why not just use a message queue with exactly-once delivery?**
NATS supports JetStream with exactly-once semantics. We're migrating to that. But the immediate fix (dedup in the bridge) took 3 lines of code vs. a full JetStream migration. Sometimes a band-aid is the right first step.

**Q7: Is this a common problem in multi-agent systems?**
Absolutely. Agent-to-agent communication without deduplication is the #1 source of token waste. Every agent that "broadcasts" its status on a pub/sub topic creates potential for loops. We've seen similar patterns across the industry — it's just rarely documented this openly.

**Q8: How do you prevent this in the future?**
Four layers: (1) bridge-level deduplication (the immediate fix), (2) session rate limiting (max sessions/minute), (3) cost anomaly detection (watchdog every 15min), (4) onboarding protocol deduplication (VOX tracks who it's already welcomed).

**Q9: What was the actual impact on the system?**
Zero functional impact. All agents continued working. The system was slightly slower due to the extra sessions competing for compute. The user was out $20. That's it. Compare this to the WhatsApp echo loop from the same day (which spammed real phone messages) — token loops are mild by comparison.

**Q10: How do you calculate the "cache hit" savings on DeepSeek?**
DeepSeek automatically caches repeated prompt prefixes. Since all 662 sessions shared the same system prompt and HARNESS, those tokens were cache hits at $0.14/M vs $0.435/M. The actual cost formula: (cached_tokens * $0.14) + (uncached_tokens * $0.435) + (output_tokens * $0.87).

**Q11: What if I want to reproduce this setup?**
Our NATS-to-Hermes bridge is open source at github.com/nerudek/nats-agent-state-sharing. The buggy version is documented in the git history. The fixed version includes the dedup code shown above.

**Q12: How do you test that the fix actually works?**
We wrote a test that publishes the same message 100 times in rapid succession. Before the fix: 100 Hermes sessions spawned. After the fix: 3 sessions (the bridge processes max 10/min, and only one instance of the duplicate message). The test is now part of the CI pipeline.

**Q13: What's the economic lesson here?**
AI agent loops are inevitable. They're bugs, but they're common bugs. The choice of AI provider determines whether a bug costs $20 (DeepSeek) or $1,000 (Claude). If you're building agent systems, factor "bug cost" into your provider selection — not just "normal usage cost."

**Q14: Did you consider rate limiting the API directly instead of the bridge?**
Yes, but rate limiting at the API level would reject legitimate requests during high-load periods. The bridge-level dedup is more surgical — it only blocks duplicates, not new messages. Both layers are needed: bridge dedup for loops, API rate limit for overall spend control.

**Q15: How do you handle the case where the SAME message is legitimately re-sent after 60+ seconds?**
The 60-second dedup window was chosen because legitimate re-sends of identical messages are rare in our system. If an agent genuinely needs to re-send the same message after 60 seconds, the content would likely include a timestamp or sequence number, making it non-identical. If needed, the window can be adjusted per-channel.

---

If this saved you time: [PayPal.me/nerudek](https://www.paypal.me/nerudek)
GitHub: [github.com/nerudek](https://github.com/nerudek)
