# Conversational Voice Skill

A synthesized **Agent Skill** for building natural, low-latency, interruptible voice conversational agents.

This skill merges the strongest patterns from three top open-source repositories:

1. **livekit/agent-skills** (building-livekit-agents) — architecture, handoffs, state ownership, latency-first design.
2. **mahimailabs/voice-ai-skills** (voice-pipeline-choice, voice-full-duplex, voice-interruptions, voice-latency-budget) — vendor-neutral pipeline selection, barge-in rules, full-duplex gotchas, and measurement.
3. **AgoraIO/skills** (agora) — routing, quickstart gates, and ConvoAI workflow discipline.

## Install

```bash
npx skills add obzue/conversational-voice-skill
```

Or drop the `skills/conversational-voice/` folder into your project's skills directory (`.agents/skills/`, `.claude/skills/`, etc.).

## What it covers

- Choosing the right pipeline shape (cascade vs speech-to-speech vs full-duplex)
- Barge-in, false interruptions, backchannels, and the read-back gate
- Full-duplex model rules (no turn detector, append-only context, talker/thinker split)
- Latency budget: the five timings, targets under 800 ms, and fix order
- Structure: handoffs, tasks, tool descriptions, and "model interprets, code owns state"
- Verification: failing-path tests before polish

## License

MIT (synthesized; original sources retain their licenses).