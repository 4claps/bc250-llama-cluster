# One-shot creative generation: driving game

Run 2026-09-18. A qualitative one-shot test of production
`Qwen3.6-35B-A3B-Q4_K_L` against the standalone
`Qwen3.8-27B-UD-Q3_K_XL` candidate, separate from the structured accuracy battery
in [the Hermes model bake-off](hermes-model-bakeoff.md#qwen38-27b-ud-q3_k_xl-candidate-2026-09-18).
It supports a different conclusion for a different use case; the production
serving recommendation is unchanged.

## The two games

Both are self-contained single files with no external requests; each only uses
`localStorage` for a best score. Download and open them in a browser (GitHub
shows the source, not the running game).

| Game | Model | File |
| --- | --- | --- |
| Turbo Road Racer | Qwen3.6 Q4_K_L (production) | [`turbo-road-racer-qwen36-q4kl.html`](examples/driving-game/turbo-road-racer-qwen36-q4kl.html) |
| Turbo Lane | Qwen3.8 UD-Q3_K_XL (candidate) | [`turbo-lane-qwen38-q3kxl.html`](examples/driving-game/turbo-lane-qwen38-q3kxl.html) |

The files are the model output unchanged, except that production's response was
wrapped in markdown fences despite the instruction and the fences were stripped.
The candidate's output needed no stripping.

## Prompt

Both models received this prompt from one saved file, byte-identical and not
retyped:

```text
Build a complete, self-contained single-file HTML website for a fun, colorful 2D
top-down driving game. Requirements:
- Everything (HTML, CSS, JS) in one .html file, no external dependencies/CDNs
- Playable with arrow keys or WASD — a car that drives on a road/track, with obstacles
  or other cars to avoid
- Drawn with canvas or inline SVG (no external image files) — make it visually colorful,
  not just gray rectangles
- Include actual text elements: a title/header, a live score or distance counter, and
  brief on-screen instructions
- Include a simple game-over state and a way to restart
Output ONLY the complete HTML file contents, nothing else — no explanation before or
after, no markdown fences.
```

The saved file keeps the trailing spaces at the ends of the wrapped lines, which
this code block does not.

## Method

Requests went to each server's `/v1/chat/completions` endpoint, non-streaming
with default thinking, with the Moderate profile confirmed on both nodes.
Production ran first, unchanged, at `max_tokens` 24,000; it finished at 7,895
tokens, so the cap did not affect it. The candidate then ran standalone with the
same flags as the accuracy test (`--ctx-size 115000`, `--cache-ram 0
--no-cache-idle-slots`, and the MTP flags), twice: first at `max_tokens` 24,000,
then again at 110,000 with a 7,200-second timeout. Production was restored and
verified afterwards.

## Results

| Metric | Qwen3.6 Q4_K_L (production) | Qwen3.8 UD-Q3_K_XL (candidate) |
| --- | ---: | ---: |
| Finish reason | `stop` | `stop` |
| Tokens used | 7,895 | 48,930 of 110,000 |
| Generation time | 102.8 s | 3,177 s (53 min) |
| Server tok/s | 77.9 | 15.4 |
| MTP draft acceptance | 84.7% | 60.6% |
| Output size | 19,587 bytes, 813 lines | 34,351 bytes, 860 lines |

### The failed first attempt is a data point

At `max_tokens` 24,000 the candidate used its entire budget on reasoning (70,154
characters, zero content), finished with `finish_reason: length`, and left an
empty file after 1,447 s at 16.6 tok/s. It was not looping (632 of 643 reasoning
lines were unique); it was drafting the game's JavaScript inside its reasoning
and was cut off mid-function. The complete run needed 48,930 tokens, roughly 35K
of them reasoning and 14K HTML, estimated from the character ratio.

For any latency- or budget-constrained use this matters: with default thinking, a
cap below roughly 49K tokens returns an empty answer, not a partial one, and at
15–17 tok/s a 110,000-token ceiling can run close to two hours. Thinking-disabled
operation was not tested. The empty output file was kept as a record in the
private results directory.

## Validation

Both files have a doctype, balanced tags, and end with `</html>`; `xmllint`
flags only elements its old HTML parser does not know (`canvas`, `header`,
`svg`). `node --check` passes on the single inline script in each (639 and 620
lines), neither references external resources, and both are served over HTTP
with a 200. Chromium (Playwright) recorded no console errors, page errors, or
failed requests for either.

Gameplay was confirmed by scripted key presses: the distance counter advanced
under input (production 6 to 65 m, candidate 10 to 120 m), each game reached a
crash state, and each restarted through both its on-screen button and the
keyboard (production Space and Enter; candidate "DRIVE AGAIN" and Space).
Production crashed when left idle at about 200 m; the candidate did not crash in
40 idle seconds and wrecked after 10 to 14 seconds of driving into traffic.

## Qualitative comparison

"Turbo Lane" (candidate) against "Turbo Road Racer" (production):

- The candidate's game is richer: separate gas and brake alongside steering,
  coins, tire stacks and oil slicks, +100 close-call scoring, pause, a sound
  toggle with WebAudio, a best-score tracker, and a more polished full-page
  layout. It also guards its `localStorage` access with `try/catch`, which
  production's game does not (a source-reading observation; not exercised).
- Production's game is simpler but clean and colorful, and meets every stated
  requirement: trees, kerbs, varied traffic, and a working HUD.
- The candidate's output is a real step up in richness and polish, not a
  different category of result, and it cost about 31 times the wall time (3,177 s
  against 102.8 s) and about six times the tokens. In the time the candidate
  produced one game, production could generate on the order of 30. A simple
  generate-a-few-and-keep-the-best loop on production could plausibly close much
  of the quality gap in a fraction of the time; that was not tested.

This is one sample per model, judged from screenshots and scripted key-press
testing rather than manual play, so "the candidate makes richer games" is an
observation from one run, not a settled result.

## Different use case, different answer

For latency-sensitive or interactive use, which is the workload this cluster
serves, the candidate remains a clear no, the same conclusion as the accuracy and
speed results in the bake-off, and the production serving recommendation is
unchanged. For a hypothetical offline or batch use, where nobody is waiting on
the response (for example, queuing a generation overnight), the extra time bought
a real quality improvement rather than nothing, so the answer there could differ.
That has not been evaluated beyond this single test, and it is not a
recommendation to change what the cluster serves.

## MTP acceptance

Draft acceptance on the candidate was 55.5% in the 24,000-token attempt and 60.6%
in the complete run, below the 72–75% it showed in the accuracy battery and below
production's 84.7% on this same request (one production sample). This is a second
observation, on a different kind of task, that MTP drafts are accepted less often
with this model and quant on this hardware. Because the candidate's own
acceptance also fell relative to its agent-task figures, acceptance varies with
the task as well as the model.

## Artifacts

Raw responses (including the full reasoning), summaries, validation output,
screenshots, and the run scripts are retained under the gitignored
`benchmarks/private/qwen38-q3kxl-candidate-2026-09-18/results/`.
