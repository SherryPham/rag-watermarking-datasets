# Canary dataset

115 protected documents are **unmodified**, plus 30 synthetic watermarked canaries
published alongside them, and a matched unwatermarked control of those canaries.

Paper: https://arxiv.org/abs/2502.10673 (DMI-RAG)

```
canary-dataset/
├── ip_docs/                 D_IP, unmodified
├── canary_docs/             D_s, watermarked
├── control_docs/            matched null
├── protected_corpus.jsonl  D_IP ∪ D_s
├── queries.jsonl           questions per canary
├── canaries.json           parameters and every detection statistic
└── splits.json             joint z as a function of canary-set size
```


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

