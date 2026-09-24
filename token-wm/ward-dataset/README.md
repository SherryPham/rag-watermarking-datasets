# Ward dataset

Fictional articles carrying a Ward LLM watermark
control of the same articles.

Paper: https://arxiv.org/abs/2410.03537

```
ward-dataset/
├── benign_docs/          doccuments without watermark
├── watermarked_docs/     doccuments with watermark
└── queries.jsonl         questions per document
```


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

