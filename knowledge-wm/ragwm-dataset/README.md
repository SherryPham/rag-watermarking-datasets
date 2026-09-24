# RAG-WM dataset

115 documents on diverse real-world topics, each carrying a RAG-WM knowledge watermark,
paired with the unwatermarked originals.

Paper: https://arxiv.org/abs/2501.05249

```
ragwm-dataset/
├── benign_docs/          doc01.txt .. doc115.txt + corpus.jsonl
└── watermarked_docs/     doc01_wm.txt .. doc115_wm.txt + corpus_wm.jsonl + watermark.json
```

`corpus.jsonl` and `corpus_wm.jsonl` carry `_id`, `title` and `text`, where `text` is the body
with paragraphs joined by a single space. `watermark.json` lists the 115 watermark units.

## The watermark

The watermark is a **false fact**, not a wording pattern. Each document gets exactly one
**watermark unit** — a triple `(E1, E2, R1)` asserting relation `R1` between two entities,
written as a sentence and appended to the end. The original text is untouched:
`docNN.txt` is a strict prefix of `docNN_wm.txt`.

Entity pairs were drawn by a SHA-256 hash chain keyed on a secret, and each pair's relation is

```
R1 = relation_top[ sha256(E1 ‖ E2 ‖ key) mod len(relation_top) ]
```

| | |
|---|---|
| documents | 115, one unit each |
| distinct relations | 53 |
| distinct entities | 74 as `E1`, 75 as `E2` |
| watermark sentence | 26-35 words, appended at the end |
| benign / watermarked length | 156-323 / 182-354 words |

## Detection

There is no z-statistic here. The unit is a fact, so detection is a **query**, one per document,
stored as `verify_query`:

> What is the relationship between `E1` and `E2`?

The relations are invented — not true of the world — and **no benign document contains both
entities of any unit** (checked across all 115). So a RAG system that answers with `R1` cannot
have got it from the benign corpus or from parametric knowledge. It ingested this one.

## Related

The same 115 benign documents are protected by two other methods in this repository:
[Ward](../../token-wm/ward-dataset/) rewrites them with a watermarked paraphraser, and
[DMI-RAG](../../canary-wm/canary-dataset/) leaves them alone and publishes synthetic canaries
alongside.
