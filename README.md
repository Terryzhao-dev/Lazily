# Lazily

A macOS app that lets you type sentences with no spaces and expands them when you hit `.`.

```
iwgotosch.      →  I want to go to school.
thedogranhome.  →  The dog ran home.
tdrh.           →  The dog ran home.
```

You can type first letters, partial words or whole words run together; more letters gives a more accurate result. Anything with spaces is left alone, so normal typing isn't affected.

**Status:** working prototype on macOS 13+. It isn't signed or notarised, so for now you build it yourself.

## How it works

A `CGEventTap` buffers the letters typed since the last space. When you type `.`, a gate first checks whether expanding makes sense at all. It skips real words, URLs, file paths, emails and code, as well as password fields and apps like Terminal, IDEs and password managers.

Easy inputs are handled locally: Viterbi word segmentation over a word frequency list, Norvig-style spell correction, common shorthand (tmr, hw, idk) and contraction fixes. That takes microseconds and needs no network.

If the local result isn't confident enough, it sends the token plus about 500 characters around the cursor (read through the Accessibility API) to any OpenAI-compatible model: xAI, MiniMax, Groq, DeepSeek or OpenAI. The model returns JSON with a few candidates and a confidence score. I rerank those locally, because models stay confident even when they've dropped letters. Then it backspaces the token, types the sentence and shows a small panel with alternatives and Undo.

Every accept and undo is saved and fed back into later requests as examples.

## Eval

[`eval/`](eval/) has about 250 test cases. 103 check behaviour, including cases that should *not* expand (real words, URLs, paths, emails, shell commands, excluded apps). The other 150 check accuracy. The runner reports gate accuracy, valid JSON rate, letter coverage and p50/p95 latency; `--no-ai` runs without an API key.

## Building

Put your provider details in `.env` in the repo root:

```
MINIMAX_API_KEY=<your key>
MINIMAX_API_BASE_URL=https://api.x.ai/v1
MINIMAX_MODEL=grok-4-fast-non-reasoning
```

Then from the repo root:

```bash
SDK=$(xcrun --sdk macosx --show-sdk-path)
swiftc -O -sdk "$SDK" -target arm64-apple-macos13 \
  ShorthandMac/*.swift Shared/Services/*.swift Shared/Utilities/*.swift \
  -framework AppKit -framework SwiftUI -framework ApplicationServices \
  -o ~/Lazily
~/Lazily
```

Give it Accessibility permission when asked, switch it to Active and try `iwgotosch.` in Notes. More detail is in [`ShorthandMac/README.md`](ShorthandMac/README.md).

## Older experiments in this repo

Lazily started as a predictive keyboard, and the earlier versions are still here:

- `edge/`: TypeScript and React predictive keyboard with a beam-search prefix decoder (under 1 ms median, 9 Vitest suites)
- `edge-dashboard/`: SwiftUI dashboard for it
- `edgetype/`: an even earlier first-and-last-letter encoding
- `TranslatingKeyboard/`: the iOS keyboard extension where it all started

## Ideas

- Run as an Input Method Kit extension instead of an event tap
- An on-device model through Core ML so it works offline
