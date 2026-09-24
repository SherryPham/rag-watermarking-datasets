# Ward dataset

115 fictional articles carrying a Ward-style LLM watermark, paired with an unwatermarked
control of the same articles.

Paper: https://arxiv.org/abs/2410.03537

```
ward-dataset/
├── benign_docs/          doc001.txt .. doc115.txt + corpus.jsonl
├── watermarked_docs/     doc001_wm.txt .. doc115_wm.txt + corpus_wm.jsonl + watermark.json
└── queries.jsonl         345 questions, 3 per document
```

`corpus.jsonl` and `corpus_wm.jsonl` carry `_id`, `title` and `text`, where `text` is the body
with paragraphs joined by a single space. `queries.jsonl` carries `query_id`, `doc_id` and
`query`. `watermark.json` holds the parameters and every statistic below.

## The watermark

The watermark is carried by **which words the document uses**. Each token is green or red
depending on a keyed hash of the two tokens before it; watermarked text over-uses green tokens.

For a token `t` with context `c` (the `h` tokens before it):

```
position_prf(c) = hash_key · Σ token_id(t_i) · i          for i = 1..h
t is green       ⟺  sha256(position_prf(c) ‖ t) / 2⁶⁴ < γ
```

Detection is a one-sided binomial test over `T` scored tokens, `|s|_G` of them green:

```
z = (|s|_G − γT) / sqrt(γ(1−γ)T)        p = 1 − Φ(z)
```

Under the null — the owner's documents are **not** in the RAG corpus — each token is green with
probability γ given its context, so `z` is standard normal and `p` is exact.

| | |
|---|---|
| seeding scheme | `ff-position_prf-2-False-1548585` |
| context width `h` | 2 |
| hash key | 1548585 |
| `γ` / `δ` / `z_threshold` | 0.25 / 3.5 / 4.0 |
| `ignore_repeated_ngrams` | true |
| tokenizer | word-level, lower-cased `[A-Za-z0-9']+` |

## Results

| | docs | tokens | green | z | p |
|---|---|---|---|---|---|
| watermarked, joint (hard) | 115 | 52,943 | 31.5% | **34.68** | 8e-264 |
| watermarked, joint (easy) | 38 | 19,407 | 31.5% | **20.81** | 2e-96 |
| benign control, joint | 115 | 52,297 | 25.3% | 1.36 | 0.087 |
| watermarked, per document | 115 | 505-580 | 27.4-35.9% | 1.28 - 6.04 (mean 3.47) | — |
| benign control, per document | 115 | — | — | -2.91 - 2.86 | — |

**The joint rows are the result.** Ward's statistic is computed over the whole set of RAG
responses, not per document. Both watermarked splits clear `z ≥ 4` by a wide margin; the control
does not, and no single control document reaches the threshold either.

Per document the dataset is weaker than the joint figure suggests — 36 of 115 exceed `z = 4`,
mean 3.47. Word choice can only carry so much signal in factual prose: about one position in
seven is rewordable, and fewer once you rule out alternatives that change a number, a date, a
name, or the sense of a sentence. Raising the per-document figure means trading correctness for
signal.

Joint token counts are below the sum of the per-document counts because
`ignore_repeated_ngrams` deduplicates trigrams across the set. Documents are 498-577 words.

## Notes

**Splits.** `hard` is all 115 documents. `easy` was one article per topic group, 38 documents
with no fact redundancy. The file listing which 38 is no longer in this folder, so that row
records a computation you cannot re-derive from what is here; the hard row you can.

**Pairing.** `docNNN.txt` and `docNNN_wm.txt` are the same article — all 115 pairs share a title
and state identical facts, and differ only in wording. That is what a watermarked paraphraser
produces, and it is what makes the benign set a matched control rather than unrelated text.

**Fictional content.** Every place, person and institution is invented, so a model cannot answer
the queries from parametric knowledge. A correct answer is evidence of retrieval.

**How it was applied.** Ward watermarks by paraphrasing with a watermarked LM, boosting green
logits by `δ` during decoding. No such model was available here, so the same freedom was taken
explicitly: each article was written with meaning-preserving alternatives at roughly one
position in seven, and a Viterbi pass over those choices maximises the green count — `h = 2`
means a choice reseeds the next two positions, so the search is not greedy. `δ = 3.5` is
recorded for provenance only; it has no role when the watermark is placed by word choice.

**Two deviations**, both chosen to leave the null distribution exactly intact: tokens are
lower-cased words rather than Llama-3.1 BPE ids, and the green set is a SHA-256 hash into the
unit interval rather than a `randperm` over a fixed vocabulary. `z` and `p` mean what they mean
in the paper, but these documents are **not bit-compatible with a stock KGW detector** — a
detector has to implement the green test above with the parameters in `watermark.json`.

## Related

The same underlying data is protected by two other methods here:
[RAG-WM](../../knowledge-wm/ragwm-dataset/) appends a false fact to each document, and
[DMI-RAG](../../canary-wm/canary-dataset/) leaves it untouched and publishes synthetic canaries
alongside.
