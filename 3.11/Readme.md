# AI Integration Proof Slice — Severity Suggestion

## What this is (in plain terms)
A "proof slice" is the smallest bit of working code that shows an AI feature done safely. Here, an AI helper reads an incident's timeline and **suggests** a severity (`LOW` / `MEDIUM` / `HIGH` / `CRITICAL`). Your code decides what the model may see, whether to trust its answer, and what to do when it fails. A human step commits the final change — the AI never edits the real record by itself.

It's small on purpose. The point is good decisions, not lots of code.

## What you do
Open `index.js` and complete **three short functions**:

| Function | Your job |
|---|---|
| `buildContext()` | Return only the fields the model is allowed to see. Do **not** include `reporterEmail` or `internalToken`. |
| `isValid(answer)` | Return `true` only if the answer is safe to use: a known severity and a non-empty reason. |
| `fallback()` | Return a short message the user sees when the model fails or is invalid, so the workflow still works. |

Everything else — the model call, running the request, and the human approval step — is already written for you. Don't change it so that AI output edits `incident.currentSeverity` directly.

## Run it
```bash
node index.js
```
No install, no API key — the "model" is a local fake in `index.js`.

Change the `MODE` constant near the top to test each path:
- `'valid'` — a good answer (gets suggested, then applied by the human step),
- `'badEnum'` — an unknown severity (your `isValid` should reject it → fallback),
- `'failed'` — the model returns nothing (should hit the fallback).

## When you're done
- `MODE = 'valid'` suggests `HIGH`, then approval changes severity `MEDIUM → HIGH`.
- `MODE = 'badEnum'` and `MODE = 'failed'` both end in `MANUAL_REVIEW` and leave the real severity unchanged.

## Submit
Open a **PR** with your completed `index.js`. That's it. Add a short comment if any choice isn't obvious from the code.

> Prefer your own project? You may instead submit a small proof slice from your capstone that shows the same four things: a clear model/app boundary, output validation, a fallback, and AI output kept separate from real data. Either option earns full marks.
