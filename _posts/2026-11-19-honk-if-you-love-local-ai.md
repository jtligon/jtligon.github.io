---
layout: post
title: "Honk If You Love Local AI: Teaching goose to Talk with llmman on a Mac"
date: 2026-11-19 09:55:00 -0500
tags: [AI, local-models, TTS, STT, macOS, goose, llmman]
---

*How to make goose talk back — and listen — using open-weight models running entirely on your machine, plus an honest look at what it takes to get a fully local "live voice" session.*

> **TL;DR**
> - ✅ **goose can talk back, fully local.** llmman serves the Kokoro-82M TTS model on your Mac, and a three-file hook plugin speaks every reply when a turn ends. Works in the CLI and Desktop; tested end to end.
> - ✅ **It listens locally too.** Use goose's built-in Local dictation (on-device whisper), or point its dictation at llmman's `/v1/audio/transcriptions` with LFM3-Audio-2B — I verified a full speak-then-transcribe round trip with zero API calls.
> - ❌ **The live voice call feature is the holdout.** It speaks OpenAI's GPT-Live protocol over WebRTC; llmman speaks REST and WebSockets. The exact wire protocol is below, for anyone who wants to close that gap.
> - 🛠️ **Fastest path to a talking goose:** `brew install llmmanorg/tap/llmman`, then `llmman serve --detach --llama-cpp-version b12503`, then `llmman pull hf.co/ggml-org/Kokoro-82M-GGUF`, plus the hook plugin from section 4.

I've been running [goose](https://github.com/aaif-goose/goose) as my daily agent, and [llmman](https://github.com/llmmanorg/llmman) as my local model server. goose has had voice features for a while — dictation for input, and since v1.52 a real **live voice** mode where you hold an actual spoken conversation with the agent (yes, it talks; no, it doesn't honk). The obvious question: can the *voice* part run locally too, so nothing I say or hear leaves my Mac?

Short answer: **yes for both halves of a turn-based voice loop** — speaking goose's replies and transcribing your voice, all tested and working in this post — but **not yet for the live voice call feature**, and I'll explain exactly why, because the reason is interesting and the door is open.

Everything below was tested on an Apple M4 Pro (48 GB) running goose 1.60.1 and llmman 0.1.664.

## What "live voice" in goose actually is

It's worth being precise here, because it changes what's possible.

goose's live voice (shipped in v1.52.0, PR [#12093](https://github.com/aaif-goose/goose/pull/12093)) is not "goose reads its replies aloud." It's a real-time, full-duplex voice conversation:

- The **voice model** handles the spoken conversation directly — it hears you, talks back, interrupts and is interruptible.
- When a request needs tools or deeper reasoning, the voice model **delegates it to the main goose agent**, which does the work and hands the result back into the ongoing call. The transcript lands in your session.

Under the hood, the initial implementation uses **OpenAI's `gpt-live-1` model over WebRTC**, and it does this *independently of whatever provider your main session uses* — you can chat with a local model while the voice layer talks to OpenAI. The ACP surface it exposes (`_goose/unstable/session/live-voice/*`) is an SDP offer/answer exchange, which is WebRTC fingerprinting all the way down.

The important part for this post: goose's live voice is **endpoint-configurable**. Reading the source (`crates/goose/src/live_voice/service.rs`), the settings are resolved from goose's usual config (environment variable first, then `config.yaml`):

| Key | Meaning | Default |
|---|---|---|
| `GOOSE_LIVE_VOICE_ENABLED` | Turn live voice on | off |
| `GOOSE_LIVE_VOICE_ENDPOINT` | A GPT-Live-compatible sessions URL | OpenAI |
| `GOOSE_LIVE_VOICE_MODEL` | The voice model id | `gpt-live-1` |
| `GOOSE_LIVE_VOICE` | The voice name | `marin` |
| `GOOSE_LIVE_VOICE_API_KEY` | Key for a custom endpoint | — |
| `GOOSE_LIVE_VOICE_INSTRUCTIONS` | Extra instructions for the voice model | — |

Two details I appreciate as a security-minded person: when you set a custom endpoint, goose **only ever sends it `GOOSE_LIVE_VOICE_API_KEY`** — your `OPENAI_API_KEY` is never sent to a third-party server — and a plain-`http://` endpoint is explicitly acceptable, which is exactly what you want for `127.0.0.1`. Live voice also requires the session to be in **Autonomous mode**, and a build with the `live-voice` feature (the default builds have it).

*(These keys aren't in the docs yet as of this writing — they're from reading the current source. The documented pieces are the ACP reference and the release notes.)*

## What llmman gives you

[llmman](https://github.com/llmmanorg/llmman) is "run any agent on any model": one daemon on `127.0.0.1:17434` that speaks the Ollama, OpenAI, and Anthropic APIs, serving models as plain OCI artifacts pulled from Hugging Face or any registry. On a Mac it drives upstream `llama.cpp` (GGUF) and `mlx-lm` (safetensors), installing what it needs.

For voice specifically, `llmman serve` exposes:

- **`/v1/audio/speech`** — OpenAI-compatible TTS. A pass-through to llama.cpp's speech API, backed by the [ggml-org TTS model collection](https://huggingface.co/collections/ggml-org/tts-models): Kokoro, Soprano, Qwen3-TTS, Chatterbox, ZONOS3, Drama 3, LFM3-Omni…
- **`/tts`** — llama-server's richer native route: `voice`, `lang`, streaming, raw PCM, and voice *cloning* from a reference clip (`speaker_ref`).
- **`/v1/audio/transcriptions`** — OpenAI-compatible STT.
- **`/v1/realtime?model=...`** — llama.cpp's realtime WebSocket: live transcription, and a *spoken conversation* with LFM3-Omni or LFM3-Audio.

So the local building blocks all exist. The catch is in the shape of the plugs.

## The honest gap

Here's the thing I had to accept after an afternoon of source-diving: **you cannot point `GOOSE_LIVE_VOICE_ENDPOINT` at llmman today.**

goose's live voice endpoint has to be a *GPT-Live-compatible sessions URL* (`https://host/v1/live/sessions`, `ws(s)` for the sideband — defaults are `https://api.openai.com/v1/live/sessions` and `wss://api.openai.com/v1/live/sessions`). The protocol, from `crates/goose-providers/src/openai_live.rs`, is:

1. **`POST` the session**: a JSON body with the session config (model, instructions, input messages, `delegation: {"type": "client"}`, voice) plus `transport: {"type": "webrtc", "sdp": <offer>}`. The response must contain a session id and a **WebRTC answer SDP**.
2. **WebRTC media**: your microphone goes out on a track; the model's voice comes back on a track.
3. **A sideband WebSocket** at `{endpoint}/{session_id}/attach` carries transcript deltas, delegation events, and lifecycle updates, and receives goose's delegation results.

llmman's `/v1/realtime` is a WebSocket (llama.cpp's own realtime dialect); `/v1/audio/speech` is REST. Neither terminates WebRTC, neither speaks the GPT-Live sessions protocol. It's a protocol mismatch, not a model mismatch — the models are fine. Think international travel: everything you packed works, you just don't have the right adapter.

That means a fully local live-voice session needs a **bridge**: something that accepts goose's session POST, terminates WebRTC (this is the hard part — think LiveKit, `werift`, or `mediasoup`), pipes the mic audio into local STT/LLM/TTS (or straight into llama.cpp's realtime API with LFM3-Omni), returns audio on the track, and translates events into the sideband's JSON. I've sketched it below for anyone who wants to build it — it's a weekend-plus project, not a config change, and unlike most of my side projects, not a wild goose chase.

Meanwhile, there's a version of "goose with a local voice" you *can* have today, and it's genuinely useful.

## What works today: goose speaks, locally

The recipe: **llmman serves a TTS model, and a goose hook speaks each reply when the turn finishes.**

goose hooks let a plugin react to session events. One of them is exactly what we need: `Stop`, which fires when goose finishes a turn, with the final assistant text in the payload as `last_assistant_message`. On macOS, playing a WAV is one `afplay` away.

### 1. Install llmman and start the daemon

```sh
brew install llmmanorg/tap/llmman
llmman serve --detach
```

On first start llmman sets up its inference backends — on the Mac that's a pinned, prebuilt `llama-server` under `~/.local/share/llmman/llama.cpp/` (no container needed).

### 2. The gotcha: the pinned llama.cpp may predate your TTS model

I pulled Kokoro and got `unknown model architecture: 'kokoro'` — the llama.cpp release llmman 0.1.664 pins (b12110) predates it. Ask for a newer one when you start the daemon:

```sh
pkill -f 'llmman serve'    # stop the old daemon, then:
llmman serve --detach --llama-cpp-version b12503
```

(To find the current release, check [llama.cpp's releases](https://github.com/ggml-org/llama.cpp/releases).)

Second gotcha, also macOS-specific: freshly downloaded llama.cpp dylibs can be quarantined, and `llama-server` dies with `library load disallowed by system policy`. Fix:

```sh
xattr -dr com.apple.quarantine ~/.local/share/llmman/llama.cpp/b12503
```

These two are the entire troubleshooting section of this post; after them, everything just worked.

### 3. Pull a TTS model and test it

```sh
llmman pull hf.co/ggml-org/Kokoro-82M-GGUF
```

[Kokoro-82M](https://huggingface.co/ggml-org/Kokoro-82M-GGUF) is an 82M-parameter TTS model that runs comfortably on any Apple Silicon and sounds surprisingly good. Then:

```sh
curl http://127.0.0.1:17434/v1/audio/speech \
  -H 'Content-Type: application/json' \
  -d '{"model":"hf.co/ggml-org/Kokoro-82M-GGUF","input":"Hello! I am goose, running fully local text to speech on a Mac.","voice":"af_heart"}' \
  -o test.wav
afplay test.wav
```

If a voice comes out of your speakers: that's an open-weight model on your own CPU/GPU — not a poltergeist — with zero API calls involved. The response is a 24 kHz WAV (`response_format` defaults to `wav`; `pcm` is available for streaming).

Voices are listed by the model — for Kokoro, `af_heart`, `af_bella`, `am_adam`, `bf_emma`… where the first letter is the language (`a` American English, `b` British, `j` Japanese, `z` Mandarin, …). You can enumerate them:

```sh
curl -s 'http://127.0.0.1:17434/props?model=hf.co/ggml-org/Kokoro-82M-GGUF' | jq '.voices'
```

Want something else? The same endpoint serves the whole ggml-org TTS collection — Qwen3-TTS-12Hz-1.7B for higher quality, Soprano-1.1-80M as an even tinier alternative, Chatterbox or ZONOS3 if you want voice cloning via the `/tts` route's `speaker_ref`.

### 4. The hook: `goose-speak`

A hook lives in a plugin directory. Create `~/.agents/plugins/goose-speak/`:

```text
goose-speak/
├── plugin.json
├── hooks/
│   └── hooks.json
└── scripts/
    └── speak.sh
```

`plugin.json`:

```json
{
  "name": "goose-speak",
  "version": "0.1.0",
  "description": "Speak goose's replies aloud with a local TTS model served by llmman"
}
```

`hooks/hooks.json` — react to `Stop`:

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "${PLUGIN_ROOT}/scripts/speak.sh",
            "timeout": 120
          }
        ]
      }
    ]
  }
}
```

`scripts/speak.sh`:

```bash
#!/usr/bin/env bash
# Speak goose's final reply for a turn using a local TTS model
# served by llmman (llama.cpp) on this machine.
#
# Best effort by design: any failure exits 0 so it never disturbs the session.
#
# Configuration (environment variables):
#   GOOSE_TTS_HOST   llmman endpoint        (default http://127.0.0.1:17434)
#   GOOSE_TTS_MODEL  model reference        (default hf.co/ggml-org/Kokoro-82M-GGUF)
#   GOOSE_TTS_VOICE  voice for the model    (default af_heart)
#   GOOSE_TTS_MAX_CHARS  cap on spoken text (default 2000)

set -uo pipefail

HOST="${GOOSE_TTS_HOST:-http://127.0.0.1:17434}"
MODEL="${GOOSE_TTS_MODEL:-hf.co/ggml-org/Kokoro-82M-GGUF}"
VOICE="${GOOSE_TTS_VOICE:-af_heart}"
MAX_CHARS="${GOOSE_TTS_MAX_CHARS:-2000}"

payload="$(cat || true)"
text="$(printf '%s' "$payload" | jq -r '.last_assistant_message // empty' 2>/dev/null || true)"

# Nothing to say (turn produced no assistant text): stay quiet.
[ -z "$text" ] && exit 0

# Strip fenced code blocks and cap length so a long reply doesn't become a podcast.
text="$(printf '%s' "$text" | sed 's/```[^`]*```//g' | head -c "$MAX_CHARS")"
[ -z "$text" ] && exit 0

wav="$(mktemp -t goose-speak).wav"
body="$(jq -n --arg m "$MODEL" --arg i "$text" --arg v "$VOICE" \
  '{model: $m, input: $i, voice: $v}')"

curl -s --max-time 90 \
  -H 'Content-Type: application/json' \
  -d "$body" \
  "$HOST/v1/audio/speech" -o "$wav"

# Only play when we actually got a RIFF/WAV back.
if file "$wav" | grep -q 'WAVE'; then
  afplay "$wav"
fi
rm -f "$wav"
exit 0
```

Make it executable:

```sh
chmod +x ~/.agents/plugins/goose-speak/scripts/speak.sh
```

That's it. Run any goose session — CLI or Desktop — and when the turn ends, the reply is spoken by a local model. Verified live while writing this: goose answered a question, the `Stop` hook fired, llmman loaded Kokoro on demand, and the answer came out of the speakers. I may have re-run it a few more times than the experiment strictly required — purely for science. Models unload after five idle minutes, so the 270 MB of weights isn't sitting in memory all day.

### 5. Make it two-way: local dictation

For the input side you have two fully local options.

**Option A — goose's built-in Local provider (simplest).** goose Desktop ships on-device dictation: **Settings → Chat → Voice Dictation Provider → Local**. It transcribes with a local whisper model, no API key, nothing leaves the machine.

**Option B — route dictation through llmman.** llmman serves the OpenAI-compatible `/v1/audio/transcriptions` too, and goose's OpenAI dictation provider honors `OPENAI_HOST` (checked before the config file). Pull an audio-input model — one with an `mmproj` projector — and point goose at it:

```sh
llmman pull hf.co/LiquidAI/LFM3-Audio-2B-GGUF   # STT + audio chat, ~2B params

export OPENAI_HOST=http://127.0.0.1:17434
export OPENAI_TRANSCRIPTION_MODEL=hf.co/LiquidAI/LFM3-Audio-2B-GGUF
export OPENAI_API_KEY=local                      # loopback daemon needs no key; goose requires one to be set
```

Then pick **OpenAI** as the dictation provider in goose Desktop settings. goose posts your mic audio as multipart to `{OPENAI_HOST}/v1/audio/transcriptions`, llmman routes it to the model, and the reply's top-level `text` field is exactly what goose parses. Two caveats: `OPENAI_HOST` applies to the whole goose process, so don't do this while also chatting through the real OpenAI provider in the same session; and the model override comes from goose's `OPENAI_TRANSCRIPTION_MODEL` setting, since the default request names `whisper-1`. One macOS quirk on top: a Desktop app launched from the Dock or Finder doesn't inherit variables exported in your terminal — either set them for GUI apps with `launchctl setenv OPENAI_HOST http://127.0.0.1:17434` (same for the other two) before starting goose, or use the CLI, which inherits your shell as usual.

*A fun proof that both halves are local: I piped Kokoro's output straight back into LFM3-Audio — speak a sentence with one open-weight model, transcribe it with another, zero API calls:*

```sh
curl -s http://127.0.0.1:17434/v1/audio/transcriptions \
  -F "file=@test.wav" \
  -F "model=hf.co/LiquidAI/LFM3-Audio-2B-GGUF"
# {"type":"transcript.text.done","text":"Hello, I am Goose, running fully local text to speech on a Mac.", ...}
```

(One dead end worth recording: NVIDIA's Canary ASR models are what llmman's docs call its CPU speech worker, but both Canary GGUFs I tried failed to load on this build with `symbol ggml_flash_attn_ext_set_prec not found` — a mismatch between the prebuilt `llama-server` and the ggml libraries in the same release folder. LFM3-Audio, which rides the regular llama-server path like the TTS models, works fine.)

Either way you now have a fully local voice loop: *speak your prompt, hear the answer*. It's not a live voice *conversation* — it's turn-based — but for "hands on keyboard optional," it's real, and it's 100% on-device.

### 6. Bonus: make the brain local too

If the point is privacy, finish the job:

```sh
llmman launch goose --model qwen3.8
```

llmman knows goose (and `goose-desktop`) as integrations: it starts the daemon if needed, preloads the model, and execs goose pointed at `http://127.0.0.1:17434` — entirely through environment variables, so your `goose configure` provider survives untouched. Dictation (local whisper) + agent (local LLM) + spoken replies (local Kokoro) is a complete local voice stack.

## If you want the real thing: the bridge

For a true local live voice session, this is the shape of the missing piece, using the protocol details above:

1. A small HTTP server that implements `POST /v1/live/sessions`: accept goose's `{session, transport: {type: "webrtc", sdp}}`, create a session id, and produce a WebRTC answer SDP.
2. A WebRTC endpoint that terminates the audio tracks. In Rust, `werift` or LiveKit's stack; in Node, `mediasoup` or `wrtc`.
3. The conversation loop. Two designs:
   - **One-model**: pipe the mic track into llama.cpp's realtime API via llmman's `/v1/realtime?model=...` with LFM3-Omni — it does speech-in/speech-out natively — and translate its events to the GPT-Live sideband shape.
   - **Pipelined**: `/v1/audio/transcriptions` (STT) → `/v1/chat/completions` (any local model, with tools) → `/v1/audio/speech` (TTS), streaming chunks back over the track. More moving parts, but it reuses exactly the endpoints this post already exercises, and it lets the voice model *be* the main agent rather than delegate.
4. A WebSocket sideband at `/{session_id}/attach` that emits the event JSON goose's protocol layer decodes: transcript deltas, delegation outcomes, lifecycle updates.

Since goose sends only `GOOSE_LIVE_VOICE_API_KEY` to a custom endpoint and allows plain `http://`, the bridge can live on loopback with zero credential handling. If I get a weekend, this is my next project — and if someone builds it first, point me at it — the [llmman discussions](https://github.com/llmmanorg/llmman/discussions) would love to hear about it.

## Trade-offs, honestly

- **Latency**: Kokoro on an M4 Pro speaks a sentence in well under a second, which feels fine for turn-based replies. A live conversation wants the streaming paths (`/tts` with `stream`, or realtime LFM3-Omni).
- **Turn-based, not conversational**: no barge-in, no interruption, no mid-call tool use. That's the live-voice feature, and that's the bridge project.
- **Kokoro is English-first**: the model card is plain that text input is English; other languages need phoneme input, i.e., espeak-ng and a certain amount of hope. For multilingual, look at Qwen3-TTS or LFM3-Omni.
- **The hook speaks code-free text only**: I strip fenced code blocks first — nobody should have to hear a YAML file read aloud with feeling.

## Wrapping up

The stack I ended up with: goose 1.60.1 + a three-file, seventy-line hook plugin + llmman serving Kokoro-82M from a pinned llama.cpp — every word goose says now comes off my own silicon. The live voice feature's configurable endpoint (`GOOSE_LIVE_VOICE_ENDPOINT`, `GOOSE_LIVE_VOICE_MODEL`, `GOOSE_LIVE_VOICE_API_KEY`) means the protocol door is already open; what's missing between llmman's solid REST/WebSocket voice APIs and goose's WebRTC session protocol is one well-scoped bridge.

Until then: speak to goose with local whisper, let it answer with local Kokoro, and enjoy the silence from the API billing dashboard.

---

*Tested 2026-11-18 on macOS (Apple M4 Pro, 48 GB), goose 1.60.1, llmman 0.1.664, llama.cpp b12503, ggml-org/Kokoro-82M-GGUF (TTS) and LiquidAI/LFM3-Audio-2B-GGUF (STT). Thanks to the [goose](https://github.com/aaif-goose/goose) and [llmman](https://github.com/llmmanorg/llmman) maintainers for making the interesting parts configurable.*
