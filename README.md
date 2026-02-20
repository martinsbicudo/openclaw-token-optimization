# OpenClaw – Remaining Optimizations Checklist

> **Before every step:** run the bash test and verify on VPS.
>
> ```bash
> echo "test-$(date +%s)" > /tmp/test.txt && cat /tmp/test.txt
> docker exec <container> cat /tmp/test.txt
> ```
>
> Values must match. If not, stop and investigate.

---

## Step 1 — Model Fallback

**What it does:** If kimi-k2.5 hits rate limit or fails, automatically falls back to the cheapest available model via OpenRouter instead of erroring out.

**Edit directly on VPS:**

```bash
nano ~/.openclaw/openclaw.json
```

**Add inside `agents.defaults`:**

```json
"model": {
  "primary": "openrouter/moonshotai/kimi-k2.5",
  "fallbacks": ["openrouter/openrouter/auto"]
}
```

**Verify:**

```bash
docker exec <container> cat ~/.openclaw/openclaw.json | grep -A 3 '"fallbacks"'
```

✅ Expected: `"fallbacks": ["openrouter/openrouter/auto"]` visible in output

**Bash test:** run after confirming.

**Gain:** Zero downtime on rate limit hits. Auto-routes to cheapest model available. No manual intervention needed.

---

## Step 2 — Context Pruning

**What it does:** Removes old tool results from memory before each API call without rewriting history. Keeps context lean automatically.

**Edit directly on VPS:**

```bash
nano ~/.openclaw/openclaw.json
```

**Add inside `agents.defaults`:**

```json
"contextPruning": {
  "mode": "cache-ttl",
  "ttl": "6h",
  "keepLastAssistants": 3
}
```

**Verify:**

```bash
docker exec <container> cat ~/.openclaw/openclaw.json | grep -A 4 '"contextPruning"'
```

✅ Expected: all 3 keys visible (`mode`, `ttl`, `keepLastAssistants`)

**Bash test:** run after confirming.

**Gain:** Automatically trims stale tool outputs. Reduces context bloat mid-session without losing recent assistant messages. Estimated 20–35% token reduction per long session.

---

## Step 3 — Compaction with memoryFlush

**What it does:** When context hits 40k tokens, distills the session into `memory/YYYY-MM-DD.md` before compacting. Prevents context loss and avoids paying for tokens that carry no useful information.

**Edit directly on VPS:**

```bash
nano ~/.openclaw/openclaw.json
```

**Add inside `agents.defaults`:**

```json
"compaction": {
  "mode": "default",
  "memoryFlush": {
    "enabled": true,
    "softThresholdTokens": 40000,
    "prompt": "Distill this session to memory/YYYY-MM-DD.md. Focus on decisions, state changes, lessons, blockers. If nothing worth storing: NO_FLUSH",
    "systemPrompt": "Extract only what is worth remembering. No fluff."
  }
}
```

**Verify:**

```bash
docker exec <container> cat ~/.openclaw/openclaw.json | grep -A 8 '"compaction"'
```

✅ Expected: `enabled: true`, `softThresholdTokens: 40000` visible

**After first long session, check memory was written:**

```bash
docker exec <container> ls -la ~/workspace/memory/
```

✅ Expected: a dated `.md` file with today's date

**Bash test:** run after confirming.

**Gain:** No context loss on compaction. Saves only what matters. Estimated 40–60% reduction in tokens carried across compaction boundaries.

---

## Final Verification

After all 3 steps, confirm the full config is valid JSON:

```bash
docker exec <container> cat ~/.openclaw/openclaw.json | python3 -m json.tool > /dev/null && echo "JSON valid" || echo "JSON BROKEN"
```

✅ Expected: `JSON valid`

Then run one last bash test:

```bash
echo "verify-$(date +%s)" > /tmp/test.txt && cat /tmp/test.txt
docker exec <container> cat /tmp/test.txt
```

---

## Overall Gain Summary

| Step | What It Fixes | Estimated Saving |
|------|--------------|-----------------|
| 1. Model Fallback | Rate limit errors → auto-reroute | Resilience + prevents wasted retries |
| 2. Context Pruning | Stale tool results bloating context | ~20–35% token reduction per session |
| 3. Compaction + memoryFlush | Context loss + wasteful compaction tokens | ~40–60% reduction at compaction boundaries |
| **Combined** | **All remaining cost drivers covered** | **~$5–15/month additional savings** |

---

## Source

This optimization checklist was compiled based on insights from the video:

📺 **[OpenClaw Token Optimization Guide](https://www.youtube.com/watch?v=RX-fQTW2To8)**

---

> ⚠️ **Reminders**
>
> - Never install Ollama on this VPS
> - Never skip AGENTS.md on session init
> - Always edit JSON directly with `nano` on VPS — never ask OpenClaw to edit it
> - Always verify with `docker exec` after every change
