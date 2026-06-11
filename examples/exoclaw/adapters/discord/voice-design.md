# Discord Voice (Design)

Status: implemented (pipeline mode). Goal: join a Discord voice channel and hold
a spoken conversation with Exoclaw — speak to it, hear it reply — with minimal
knobs and no new API keys. Implementation lives in `voice.ts`; setup is in
`README.md`.

## Principle

Exoclaw adapters are transports; the agent is the brain. Voice follows the same
rule: it is a **microphone and speaker bolted onto the existing text-message
pipe**, not a second brain. A spoken turn becomes a normal inbound `message`
event; a spoken reply is a normal outbound `send_message`.

Consequence: **all audio lives in the Discord worker.** The Rust adapter runtime,
the worker protocol, the agent tools, and the turn loop are unchanged. Audio
never reaches Rust.

This is the pipeline approach (STT → agent → TTS), the same shape Hermes uses.
We deliberately skip a realtime speech-to-speech bridge (openclaw's `agent-proxy`
/ `bidi` modes): it is a separate brain that does not reuse Exoclaw's tools,
identity, or history without significant extra plumbing. It can be added later
as an isolated mode without touching this design.

## Data path

One spoken turn:

```mermaid
flowchart LR
  user["User speaks"] --> recv["Worker: per-speaker Opus receive"]
  recv --> seg["Segment on silence"]
  seg --> stt["OpenAI STT"]
  stt --> msg["message event\ntarget = voiceChannelId"]
  msg --> rust["Rust runtime: wake conversation"]
  rust --> agent["Exoclaw agent turn\n(tools, identity, history)"]
  agent --> send["send_adapter_message\ntarget = voiceChannelId"]
  send --> outbox["Outbox"] --> worker["Worker: target is active voice session"]
  worker --> tts["OpenAI TTS"] --> play["Play into voice channel"]
```

**Inbound.** The worker joins voice with `@discordjs/voice`. It captures
per-speaker Opus streams and segments utterances on trailing silence (using
`receiver.speaking` start/end events plus a short grace timer, not bare
`AfterSilence`; sub-0.5s blips are dropped). Each utterance is decoded to PCM,
wrapped as WAV, transcribed by OpenAI STT, and emitted as a normal `message`
event whose `target` is the **voice channel id** and whose metadata carries
`source: "voice"`. Rust handles it identically to a typed message.

**Outbound.** The worker tracks `voiceChannelId → active voice session`. When a
`send_message` arrives whose `target` is an active voice channel, the worker
synthesizes it with OpenAI TTS and plays it into the connection. If the user
starts speaking during playback, playback stops (barge-in). The reply text is
also posted to the channel so turns are inspectable.

Because the inbound event's `target` is the voice channel id and the agent is
told to reply to the inbound target, replies route back to voice with no new
tool and no protocol change.

## Control

- `/voice join` — join the caller's current voice channel.
- `/voice leave` — leave the current voice channel.
- The bot auto-leaves when the channel empties.

Slash commands are handled entirely in the worker and never involve the model.
Requires the `applications.commands` scope and `Connect` / `Speak` permissions.

## Models and keys

STT and TTS both use the existing `openai` secret, bound into the worker as
`OPENAI_API_KEY` — no new key. Defaults (not configurable): `gpt-4o-mini-transcribe`
for STT, `gpt-4o-mini-tts` with the `alloy` voice for TTS.

## Configuration

One knob: `voice` on/off in the adapter config. Everything else — models, voice,
silence window — is hardcoded to sane defaults.

Additional requirements vs. the text adapter:

- `GuildVoiceStates` gateway intent.
- `Connect` and `Speak` bot permissions; `applications.commands` scope.
- An OpenAI secret env binding on the adapter config.

## Non-goals (v1)

- Realtime speech-to-speech (lower latency, mid-sentence interruption). The
  pipeline is assistant-grade latency (~seconds per turn, more when tools run),
  not phone-call-grade.
- Ambient "thinking" audio / multi-source mixing (Hermes' VoiceMixer). Not
  needed for a working conversation.
- Wake-word gating. While joined, every utterance is a turn.

## Dependencies

`@discordjs/voice`, `prism-media`, `opusscript` (pure-JS Opus codec, so no native
build step), and `sodium-native` (voice encryption, prebuilt binary). No ffmpeg:
inbound Opus is decoded to PCM and wrapped as WAV in-process, and TTS is fetched
as OGG/Opus and played directly via `StreamType.OggOpus`.
