# Watermark datasets for RAG dataset inference

Three datasets, one per watermarking method, for testing whether a third-party RAG system has
ingested a corpus it should not have.

All three answer the same question — *is my data in your index?* — and differ in what the owner
has to do to their data to find out.

```
watermark-docs/
├── knowledge-wm/ragwm-dataset/     RAG-WM   — appends a false fact to each document
├── token-wm/ward-dataset/          Ward     — rewrites each document with a watermarked LM
└── canary-wm/canary-dataset/       DMI-RAG  — changes nothing, adds synthetic canaries
```

## The three methods

| | [RAG-WM](knowledge-wm/ragwm-dataset/) | [Ward](token-wm/ward-dataset/) | [DMI-RAG](canary-wm/canary-dataset/) |
|---|---|---|---|
| paper | [2501.05249](https://arxiv.org/abs/2501.05249) | [2410.03537](https://arxiv.org/abs/2410.03537) | [2502.10673](https://arxiv.org/abs/2502.10673) |
| the watermark is | a **false fact** | a **word-choice bias** | **extra documents** |
| protected data is | appended to | rewritten | **untouched** |
| detection | ask one question per document | binomial z-test | binomial z-test |
| evidence | the model states a relation that is true nowhere | green tokens over-represented | same, measured on the canaries |
| documents | 115 watermarked | 115 watermarked + 115 control | 115 untouched + 30 canaries + 30 control |

The ordering is roughly *how invasive*: RAG-WM adds a sentence, Ward rewrites every sentence,
DMI-RAG touches nothing at all and pays for it by publishing decoys instead.

**RAG-WM** appends one triple `(E1, E2, R1)` per document as a sentence. The relation is
invented, and no benign document contains both of its entities, so a system that answers
*"what is the relationship between E1 and E2?"* with `R1` must have ingested the corpus.
Detection is a lookup, not a statistic.

**Ward** watermarks the text itself. A token is green or red by a keyed hash of the two tokens
before it, and watermarked text over-uses green ones. Detection is a z-test over the retrieved
responses. The cost is that the documents have to be rewritten.

**DMI-RAG** leaves the corpus alone and publishes a handful of synthetic *canary* documents
beside it — invented institutions, invented findings, watermarked text. Retrieval of a canary
is what gets measured. The owner's real data is never altered, which is the whole point.

## Results

Both z-test methods use a `z ≥ 4` threshold. The statistic is computed over the **concatenated
retrieved responses**, not per document, so the joint rows are the ones that matter.

| | watermarked | unwatermarked control |
|---|---|---|
| Ward, all 115 documents | z = **34.68** | 1.36 |
| Ward, one per group (38) | z = **20.81** | — |
| DMI-RAG, all 30 canaries | z = **21.84** | -0.84 |
| DMI-RAG, 12 canaries | z = **14.47** | -0.43 |
| DMI-RAG, protected corpus (115) | — | 0.45 |

Per document both sit well below their joint figures — Ward averages 3.47 with 36 of 115 over
the threshold, DMI-RAG 3.98 with 14 of 30 — because word choice can only carry so much signal in
prose that also has to stay factually correct. Neither method's control ever reaches `z = 4`.

RAG-WM has no z-score to report: its unit is a fact, so the test is whether the model says it.

## Shared corpus

`ragwm-dataset/benign_docs/` and `canary-dataset/ip_docs/` are **the same 115 documents**, byte
for byte — real-world topics, 156-323 words. That makes the two directly comparable: same data,
one method modifying it, the other not.

`ward-dataset` uses its own separate corpus of 115 longer fictional articles (498-577 words),
written in topic groups so that retrieval can satisfy a query from a document other than the one
it was written for.

## Conventions

Every dataset follows the same layout, so the same tooling reads all three:

- **`.txt`** — title, blank line, then paragraphs. CRLF throughout.
- **`corpus*.jsonl`** — one row per document: `_id`, `title`, `text`, where `text` is the body
  with paragraphs joined by a single space.
- **`queries*.jsonl`** — questions answerable from a named document.
- **`watermark.json` / `canaries.json`** — the watermark parameters and every detection figure
  quoted in that dataset's README.

**All content is fictional or synthetic.** Ward's articles and DMI-RAG's canaries invent every
place, person and institution, so a model cannot answer their queries from parametric knowledge
and a correct answer is evidence of retrieval. RAG-WM's relations are false by construction.

**None of these are bit-compatible with the reference implementations.** Tokens are lower-cased
words rather than BPE ids, and green sets are keyed SHA-256 hashes rather than permutations of a
fixed vocabulary. The null distributions are intact — `z` and `p` mean what they mean in the
papers — but scoring these files with a stock KGW or Unigram detector will show nothing. Each
dataset's README gives the green test it actually uses.
