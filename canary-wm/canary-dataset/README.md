# Canary dataset

115 protected documents left **completely unmodified**, plus 30 synthetic watermarked canaries
published alongside them, and a matched unwatermarked control of those canaries.

Paper: https://arxiv.org/abs/2502.10673 (DMI-RAG)

```
canary-dataset/
├── ip_docs/                ip001.txt .. ip115.txt + corpus.jsonl           D_IP, unmodified
├── canary_docs/            c01_wm.txt .. c30_wm.txt + corpus_canary.jsonl  D_s, watermarked
├── control_docs/           c01.txt .. c30.txt + corpus_control.jsonl       matched null
├── protected_corpus.jsonl  D_IP ∪ D_s, 145 rows tagged ip / canary
├── queries.jsonl           180 questions, 6 per canary
├── canaries.json           parameters and every detection statistic
└── splits.json             joint z as a function of canary-set size
```

`protected_corpus.jsonl` is the release. `control_docs/` is deliberately **not** part of it —
see *Notes*.

## What makes this method different

[RAG-WM](../../knowledge-wm/ragwm-dataset/) appends a false fact to each protected document.
[Ward](../../token-wm/ward-dataset/) rewrites each one with a watermarked paraphraser.
**DMI-RAG does neither: the protected data is never touched.** The owner publishes a small set
of synthetic *canary* documents beside it and infers membership from those alone.

So `ip_docs/` is an input here, never an output — byte-identical to the documents the owner
already had. The canaries must be:

- **unique** — about entities that exist nowhere else, so a correct answer can only come from
  retrieval, never from parametric knowledge;
- **stealthy** — indistinguishable from the corpus in topic, style and length, so they cannot be
  filtered out. Canaries run 238-330 words, the IP corpus 156-323;
- **statistically provable** — decidable by a test with a calibrated false-positive rate.

`|D_s| ≪ |D_IP|`: 30 canaries against 115 real documents, and 12 is enough.

## The watermark

The canaries are written with **Unigram-Watermark**, whose defining property is that the green
list is **global and fixed** — the vocabulary is partitioned once by the key, and a token's
colour does not depend on what precedes it. That is what makes it survive the paraphrasing and
truncation a RAG pipeline does to retrieved text.

```
t is green  ⟺  sha256(key ‖ t) / 2⁶⁴ < γ

z = (|y|_G − γT) / sqrt(γ(1−γ)T)        p = 1 − Φ(z)
```

`z` is computed over the concatenation `y = y⁽¹⁾ ⊕ … ⊕ y⁽ᴺ⁾` of the N query responses.

| | |
|---|---|
| `γ` / `δ` / `z_threshold` | 0.5 / 2.0 / 4.0 |
| green list | global, context-free |
| n-gram deduplication | none — every token is scored |
| tokenizer | word-level, lower-cased `[A-Za-z0-9']+` |

There is no `ignore_repeated_ngrams` as in Ward's KGW: colours are context-independent, so a
repeated n-gram carries the same evidence wherever it appears and is counted every time.

## Results

| | docs | tokens | green | z | p |
|---|---|---|---|---|---|
| canaries, joint (all) | 30 | 8,013 | 62.2% | **21.84** | 5e-106 |
| canaries, joint (12) | 12 | 3,443 | 62.3% | **14.47** | 1e-47 |
| control, joint | 30 | 7,814 | 49.5% | -0.84 | 0.80 |
| IP corpus, joint | 115 | 30,229 | 50.1% | 0.45 | 0.32 |
| canaries, per document | 30 | 242-337 | 56.8-66.3% | 2.12 - 5.61 (mean 3.98) | — |
| control, per document | 30 | — | — | -2.25 - 1.37 | — |

**The joint rows are the result.** A single canary is not expected to clear `z = 4` alone — 14
of 30 do. How the statistic grows with the canary budget, from `splits.json`:

| canaries N | 1 | 2 | 4 | 6 | 8 | 10 | **12** | 15 | 20 | 30 |
|---|---|---|---|---|---|---|---|---|---|---|
| watermarked z | 3.69 | 5.77 | 9.33 | 11.43 | 12.13 | 12.78 | **14.47** | 15.26 | 17.10 | 21.84 |
| control z | -0.11 | -0.04 | 0.17 | 0.49 | -0.12 | -0.56 | -0.43 | -1.66 | -1.87 | -0.84 |

**Two canaries already clear the threshold.** z grows roughly as √N while the control stays
flat, which is the evidence that the growth is watermark and not drift. The subsets are nested
prefixes, so these are budget levels, not a partition.

The false-positive side is clean: over the 145 documents the null covers — 115 IP plus 30
controls — per-document z has mean 0.004, sd 0.992, max 2.20, and **nothing reaches the
threshold**.

## Notes

**Why there is a control set.** `control_docs/cNN.txt` is the same canary with the watermark not
applied: same title, same facts, differing only in word choice. It isolates the watermark from
the writing, and it supplies the null. It is absent from `protected_corpus.jsonl` on purpose —
publishing it would hand an adversary a word-by-word diff of exactly which choices carry the
signal.

**The IP corpus** is carried over unchanged from
[`ragwm-dataset/benign_docs/`](../../knowledge-wm/ragwm-dataset/benign_docs/), so all three
datasets here protect the same underlying data by three different methods.

**How it was applied.** DMI-RAG generates each canary with a watermarked LLM, boosting green
logits by `δ = 2.0`. No such model was available here, so the same freedom was taken explicitly:
each canary was authored with meaning-preserving alternatives at 28-37% of positions, and the
option with the greatest green surplus was chosen. Because the green list is context-free the
slots are independent, so a **greedy choice per slot is optimal** — no dynamic programming,
unlike Ward, where `h = 2` makes adjacent choices interact. `δ` is recorded for provenance only.

**Two deviations**, both chosen to leave the null exactly intact: tokens are lower-cased words
rather than Llama-3.1 BPE ids, and the partition is a keyed SHA-256 hash into the unit interval
rather than a permutation of a fixed vocabulary. `z` and `p` mean what they mean in the paper,
but these documents are **not bit-compatible** with a stock Unigram-Watermark detector.

**Why the key is calibrated.** Because the scheme colours a *word type* once and for all, one
frequent function word sets a floor under an entire corpus. Under an earlier key `the` was
green; at ~8% of all tokens it alone pushed every unwatermarked document to a mean z near +0.75,
with nothing watermarked at all. The key in use was chosen so the null is centred on the owner's
own corpus, which fixes the null *before* any canary is written. A context-dependent scheme does
not have this failure mode, because a token's colour is reseeded by its neighbours and the bias
averages out.
