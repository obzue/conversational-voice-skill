---
name: conversational-voice
description: >-
  Build natural, low-latency, interruptible voice conversational agents that talk
  back and forth with users. Use when the user asks to "build a voice agent",
  "make it talk back", "add barge-in", "fix latency", "choose cascade or
  realtime", "handle interruptions", or is writing code for LiveKit Agents,
  Pipecat, Agora ConvoAI, or any STT-LLM-TTS / speech-to-speech stack. Covers
  pipeline selection, turn-taking, full-duplex rules, latency budgets, handoffs,
  and the read-back gate. Synthesized from livekit/agent-skills,
  mahimailabs/voice-ai-skills, and AgoraIO/skills.
license: MIT
metadata:
  author: obzue (synthesized)
  version: "1.0.0"
  category: voice-ai
  sources:
    - livekit/agent-skills
    - mahimailabs/voice-ai-skills
    - AgoraIO/skills
---

# Conversational Voice

You are a voice-agent architect. Your job is a conversation that feels alive:
sub-800 ms replies, clean barge-in, and never a wrong confirmation spoken over.

This skill is vendor-neutral at the core. Adapters for LiveKit Agents, Pipecat,
and Agora ConvoAI sit at the bottom — verify every identifier against current
docs before writing SDK calls.

## 1. Pick the pipeline shape first

Do not default to speech-to-speech because it is newer. The shape decides what
you can control: exact words, delay, voice, and cost.

| requirement | shape |
| --- | --- |
| Exact wording of numbers, dates, codes | cascade or half-cascade |
| Lowest perceived latency / overlap | full-duplex, else speech-to-speech |
| Most natural prosody | speech-to-speech |
| Cost at scale | cascade |
| Swapping providers without a rewrite | cascade |

- **Cascade**: STT → LLM → TTS. Exact words, cheapest, four places to lose time.
- **Speech-to-speech**: one realtime model, audio in/out. Best prosody, no wording control.
- **Half-cascade**: realtime model emits text, your TTS speaks it. Your voice, your words.
- **Full-duplex**: both directions open; the model decides when to speak.

If a confirmation number or date must be spoken exactly, do **not** pick
speech-to-speech or full-duplex — they paraphrase.

## 2. Latency budget: under 800 ms end to end

Report p50 and p95 per stage, never one number. Measure on a real phone call,
not laptop Wi-Fi. Bucket tool-call turns apart from plain turns.

| stage | cascade p50 | speech-to-speech | full-duplex |
| --- | --- | --- | --- |
| end-of-turn delay | 300–500 | 300 | n/a (model decides) |
| transcription delay | 100–200 | folded in | folded in |
| LLM TTFT | 200–400 | folded in | ~700 (backend) |
| TTS TTFB | 100–300 | folded in | folded in |
| end to end ceiling | 800 | 500 | 300 |

Fix in this order, stop when inside budget:

1. Endpointing (usually the biggest free win: 200–400 ms).
2. Preemptive generation on partial transcripts.
3. Streaming TTS (speak the first sentence while the rest renders).
4. Co-locate every hop in one region (80–150 ms per hop removed).
5. Smaller model — last, and only with an eval run.

## 3. Barge-in and interruptions

Answer three questions, in order:

1. Was that speech? (energy gate — coughs and noise are not speech)
2. Was it meant for me? (duration floor ~0.5 s, or word floor 2 on noisy lines)
3. Should I stop? (utterance class, not audio)

**The read-back gate:** anything the caller must confirm is uninterruptible.
Everything else is interruptible. Gate the sentence, not the session. A cut
read-back confirms a value the caller never heard — the write lands wrong and
nothing downstream catches it.

Uninterruptible (keep short, under two sentences): read-backs, legal/consent
notices, closings that carry a confirmation number.

**False interruptions:** if no user turn arrives within ~2.0 s of a stop, resume
the interrupted utterance. Never resume after a real interruption — the caller
has moved on.

**Backchannels** ("mm-hm", "okay", "yeah") are not interruptions. Filter by
duration floor or a phrase list; do not raise both floors at once.

## 4. Full-duplex rules (if you chose that shape)

- Do not pass a turn detector — the model owns turn boundaries.
- Do not expect `say()` / verbatim paths — there is none; strings get re-worded.
- Tools never fire on the voice model; delegate to a backend text model
  (the "talker and thinker" split). Write the delegation as one sentence in the
  persona.
- Context is append-only after startup. No rebuilds, no mid-call persona swaps.
- Keep a VAD on your side to cut your own playback buffer — the model will not
  drain queued audio for you.
- An interrupted response stays whole in the model's context; do not assume it
  was truncated.

## 5. Structure: handoffs, not one giant agent

One agent that does everything gets slow and unreliable. Split at natural
conversation boundaries (greeting → intake → resolution). Each agent carries
only its own tools and instructions.

- **Handoffs** transfer control; summarize context at the boundary.
- **Tasks** are tightly scoped prompts for one outcome.
- Tool descriptions drive behavior — when the wrong tool fires, fix the
  description before blaming the model.
- Keep tools off the critical path where you can; users hear every waited tool
  call as latency.
- Plan for tool failure: return facts and actionable errors, never invented
  answers.

**The model interprets; your code owns the state.** Never classify intent with
code (approvals, refusals, "next Tuesday"). Validate structure in code; leave
meaning to the model. Tools return facts, not sentences.

## 6. Verify before you call it done

- While building: drive real conversations in text mode and inspect tool calls.
- Before done: write turn-level tests covering the core behavior, one tool
  invocation, and one failure path.
- Before shipping: run whole-conversation simulations.
- Prompt edits change behavior silently — manual testing alone does not count.

## 7. Common mistakes

- Starting with one agent "just for now."
- Putting off latency — it compounds and gets expensive to fix.
- Copying an example you don't understand.
- Assuming your model knowledge is current — verify against docs.
- Shipping on manual testing alone.
- Disabling interruptions for a whole session to protect one sentence.
- Raising the duration floor to fix a noisy line instead of the word floor.

## Adapters (verify before use)

### LiveKit Agents (1.8.x)
- Cascade: `AgentSession(stt=..., llm=..., tts=...)`.
- Speech-to-speech: pass a realtime model as `llm`.
- Full-duplex: `GPTLiveModel` as `llm`, plus an explicit VAD.
- Interruptions: `turn_handling["interruption"]` — `min_duration` 0.5,
  `min_words` 0–2, `resume_false_interruption` True. Inside a tool:
  `context.disallow_interruptions()`.
- Latency: read `ChatMessage.metrics` (`e2e_latency`, `llm_node_ttft`,
  `tts_node_ttfb`, `end_of_turn_delay`). Preemptive generation and endpointing
  live under `turn_handling`.
- Docs: https://docs.livekit.io/agents

### Pipecat (1.0)
- Cascade: ordered `Pipeline` — transport.input, stt, aggregators.user, llm, tts,
  transport.output, aggregators.assistant (ordering is load-bearing).
- Interruptions: `MinWordsUserTurnStartStrategy`, mute strategies for read-backs.
- Metrics: `enable_metrics=True`, read via `MetricsLogObserver`.
- Docs: https://docs.pipecat.ai

### Agora ConvoAI
- Route ConvoAI requests through the official quickstart before generating custom
  code. Clone it, map the source files, and keep the sample's architecture,
  lifecycle, token flow, and env names intact. Customize only persona, greeting,
  and documented join fields.
- Docs: https://docs.agora.io

## Sources

Synthesized from (all MIT / Apache-2.0, used under their licenses):

- https://github.com/livekit/agent-skills
- https://github.com/mahimailabs/voice-ai-skills
- https://github.com/AgoraIO/skills