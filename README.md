# Lazily

**Type as lazily as you like, press `.`, and get a clean sentence.** Lazily is a system-wide macOS app. Type a compressed, no-space version of a sentence in any app; on the full stop it expands into the full sentence, using a local pipeline first and an LLM that reads the text around your cursor when it needs to.

```
type:  iwgotosch.       →  I want to go to school.
type:  thedogranhome.   →  The dog ran home.
type:  tdrh.            →  The dog ran home.
```

The shorthand is flexible: first letters, first two letters, partial words or whole words run together. Normal writing with spaces is never touched.

> **Status (May 2026):** working prototype on macOS 13+, with a 100+ case evaluation set for when it should and shouldn't expand. Not yet signed or notarised for distribution.

## How it works

| Step | What happens | Code |
|---|---|---|
| Capture | A `CGEventTap` watches keystrokes system-wide and buffers the run of letters since the last space | `KeystrokeInterceptor.swift` |
| Gate | On `.`, cheap checks skip real words, URLs, file paths, emails, code, shell commands and excluded apps (Terminal, IDEs, password managers) | `ExpansionGate.swift` |
| Local first | Easy inputs are expanded on-device in microseconds: Viterbi word segmentation over a frequency dictionary, a Norvig-style spell corrector, shorthand expansion and grammar polish. No API call | `LocalPipeline.swift`, `WordSegmenter.swift`, `SpellCorrector.swift`, `GrammarPolisher.swift` |
| Context | The Accessibility API reads up to 500 characters around the caret | `CaretLocator.swift` |
| LLM | Harder inputs go to any OpenAI-compatible model (xAI, MiniMax, Groq, DeepSeek or OpenAI), with abbreviation hints and recent corrections as few-shot examples. Replies are structured JSON with alternatives and a confidence score | `MiniMaxClient.swift`, `AbbreviationHints.swift` |
| Rerank | Candidates are re-scored against the typed letters, since models are over-confident when they drop letters | `LocalReranker.swift` |
| Replace | The token is backspaced and the sentence injected; a floating panel offers alternatives and Undo | `SuggestionPanel.swift` |
| Learn | Every accept and undo is stored and fed back into later requests | `CorrectionStore.swift` |

## Evaluation

[`eval/`](eval/) holds 100+ labelled cases: 47 shorthand sentences that should expand, and cases that must not (single English words, URLs, file paths, emails, code, identifiers, shell commands and excluded apps). The runner reports gate accuracy, valid-JSON rate, letter-coverage match rate and median and p95 latency. The gate-only mode needs no API key.

## Earlier prototypes

Lazily grew out of several experiments that are kept in this repo:

| Folder | What it is |
|---|---|
| [`edge/`](edge/) | TypeScript/Node + React predictive keyboard. Prefix decoder with beam search over a dictionary, per-domain bigrams and trigrams, correction memory and an LLM reranker that never blocks: under 1 ms median decode, 9 Vitest suites |
| [`edge-dashboard/`](edge-dashboard/) | Native SwiftUI dashboard for the Edge backend: live prediction, AI completions, corrections log |
| [`edgetype/`](edgetype/) | First version, using first-and-last-letter encoding (`te qk bn fx` → "the quick brown fox") |
| [`TranslatingKeyboard/`](TranslatingKeyboard/) | The iOS keyboard extension that started the project, translating text inline with the Claude API |

## Build and run

Requires macOS 13+ and an API key for an OpenAI-compatible provider.

1. Create `.env` in the repo root:

   ```
   MINIMAX_API_KEY=<your key>
   MINIMAX_API_BASE_URL=https://api.x.ai/v1
   MINIMAX_MODEL=grok-4-fast-non-reasoning
   MINIMAX_TIMEOUT_MS=5000
   ```

2. Build and run from the repo root:

   ```bash
   SDK=$(xcrun --sdk macosx --show-sdk-path)
   swiftc -O -sdk "$SDK" -target arm64-apple-macos13 \
     ShorthandMac/*.swift Shared/Services/*.swift Shared/Utilities/*.swift \
     -framework AppKit -framework SwiftUI -framework ApplicationServices \
     -o ~/Lazily
   ~/Lazily
   ```

3. Grant Accessibility permission when prompted (System Settings > Privacy & Security > Accessibility), switch Lazily to **Active** and try `iwgotosch.` in Notes.

More detail: [`ShorthandMac/README.md`](ShorthandMac/README.md) for the app and [`eval/README.md`](eval/README.md) for the evaluation.

**Stack:** Swift, SwiftUI, AppKit, CGEventTap, Accessibility API; TypeScript, Node.js, React, Vite, Vitest; OpenAI-compatible LLM APIs.

## Roadmap

- Run as a macOS Input Method Kit extension instead of an event tap
- On-device model through Core ML so expansion works offline
- Signed and notarised release
