# The frozen model-visible surface

Everything the adjudicator saw, and nothing else. Frozen from `fable-adjudicate.mjs`
(64,500 bytes, `sha256:4b2214d2e86454bd8857ae785b939dbee3fc36d396a8340626172a6b19784ea6`)
as it stood on 2026-08-24, and verified by the runner before every one of the 121 calls —
`paired-adjudication.mjs` reads these regions out of the driver and REFUSES unless each hashes
to the value below, so the prompt that was sent is provably the prompt recorded here.

| region | bytes | sha256 |
|---|---|---|
| `SYSTEM_PROMPT` | 866 | `be0a311edd03078cf3b8c9783cc769137e580ea9944cbf20e101f42c04e0e1d9` |
| `VERDICT_SCHEMA` | 1288 | `8506dfe13d590c5ee39bd52f914d1209b111171e980c6d5b9c6bc365c64121d2` |
| `blindCard` | 495 | `a83cce99b14ddd84741ad4947aaa4fe4213afab3dde78a08e6f788fff7bb5fe3` |
| **combined** | **2649** | **`3ed8234bc9a0b91c989202fe8c393b95451026349cf2b204d575c3967e0ddfc8`** |

⚠ **Provenance caveat, stated because it cannot be closed.** The driver was edited at
2026-08-20 18:41:29 — after all 84 Fable labels were written (last 18:34) and before the first
Opus verdict (18:46:42). The edit is identified as the `--mixed-annotators` guard, whose five
sites are all usage text, a pre-flight label check and one stderr line; none touches the three
regions below. But no baseline hash was ever recorded before this file, and Fable cannot be
re-run to check behaviourally. So byte-identity with the Fable-era prompt is a well-evidenced
inference, **not a proof** — and it bears on the cross-instrument leg only. The within-Opus
noise floor is immune, both sides having used this exact surface.

## 1. The system prompt, verbatim

```
You are adjudicating code-review findings for a benchmark.

For each finding you are given a claim about a file in a pull request. Decide one thing: is the claim correct about the code as written?

Judge the claim, not its phrasing. A claim that identifies a real defect in clumsy words is real; a confident, well-written claim about a case the code already handles is not. Read the code before deciding when you have tools to do so, and cite what you read.

You are seeing the claim only. No severity, no verifier outcome, and no merge decision is being shown to you, because your judgement is the independent measurement those would contaminate. Do not speculate about what any reviewer concluded.

Report what you actually determined. "I could not verify this" with low confidence is a useful answer; a guess dressed as a finding is not.
```

## 2. The verdict schema

```js
const VERDICT_SCHEMA = Object.freeze({
  type: "object",
  properties: {
    is_real: {
      type: "boolean",
      description:
        "true if the claim is correct about the code as written: the described defect is actually present. false if the claim is wrong about the code — it misreads it, the case is already handled, or the described condition cannot occur.",
    },
    severity: {
      type: "string",
      enum: ["critical", "major", "minor", "nit"],
      description:
        "YOUR OWN severity for the defect if it is real, judged from impact on users or correctness. Ignore what any reviewer might have called it. If is_real is false, give your best estimate of what it would have been if true.",
    },
    confidence: {
      type: "string",
      enum: ["high", "medium", "low"],
      description:
        "high only when you verified the claim against the code. medium when the prose is convincing but you could not check every branch. low when you are guessing.",
    },
    reasoning: {
      type: "string",
      description:
        "2-5 sentences. Cite the file:line you checked. If is_real is false, say specifically what the claim gets wrong.",
    },
  },
  required: ["is_real", "severity", "confidence", "reasoning"],
  additionalProperties: false,
});
```

## 3. The card — the blinding, by construction

Built by allowlist. It names the five fields it reads and never touches `severity`,
`severity_raw`, `gating`, `gating_basis`, or anything under `panel` except `lens`. The runner
additionally asserts, per call, that the rendered card's scaffold contains no adjudication or
verdict token — with the finding's own `summary` and `evidence` excluded from that check, since
a reviewer's prose may legitimately say "major".

```js
function blindCard(rec, index, total) {
  const lines = [
    `Finding ${index + 1} of ${total}`,
    "",
    `File: ${rec.file ?? "(not specified)"}`,
  ];
  if (rec.line !== null && rec.line !== undefined) lines.push(`Line: ${rec.line}`);
  lines.push(
    `Review area: ${rec.panel?.lens ?? "(not specified)"}`,
    "",
    "Claim:",
    rec.summary ?? "(no claim text)",
  );
  if (rec.evidence) lines.push("", "The reviewer's supporting prose:", rec.evidence);
  return lines.join("\n");
}
```

## 4. Evidence condition

Each finding was judged with the unified diff at **its own `round_commit`** — not the item's
`review_commit`, which differs on 86 of 100 rows across 53 distinct commits — plus the
repository files that finding's own text cites, at that same commit. Diffs were recomputed with
`core.abbrev=8` by the method `extract-corpus.mjs` uses, and **all 21 items reproduced their
frozen `sha256_diff` exactly**, which is what makes the per-round diffs trustworthy.

The working directory was **empty**: the tool grant cannot be empty (`assertAllowedTools`
refuses `[]`), so "no tools" is expressed as a read-only grant with no tree behind it. Blinding
rests on the allowlist above and on the store path never entering a prompt, never on the cwd.
