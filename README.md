# Watermark datasets for RAG dataset inference

```
watermark-docs/
├── knowledge-wm/ragwm-dataset/     RAG-WM   — appends a false fact to each document
├── token-wm/ward-dataset/          Ward     — rewrites each document with a watermarked LM
└── canary-wm/canary-dataset/       DMI-RAG  — adds synthetic canaries
```

## Methods

| | [RAG-WM](knowledge-wm/ragwm-dataset/) | [Ward](token-wm/ward-dataset/) | [DMI-RAG](canary-wm/canary-dataset/) |
|---|---|---|---|
| paper | [2501.05249](https://arxiv.org/abs/2501.05249) | [2410.03537](https://arxiv.org/abs/2410.03537) | [2502.10673](https://arxiv.org/abs/2502.10673) |
| the watermark is | a **false fact** | a **word-choice bias** | **extra documents** |
| protected data is | appended to | rewritten | **untouched** |
| detection | ask one question per document | binomial z-test | binomial z-test |
| evidence | the model states a relation that is true nowhere | green tokens over-represented | same, measured on the canaries |
| documents | 115 watermarked | 115 watermarked + 115 control | 115 untouched + 30 canaries + 30 control |


**RAG-WM** appends one triple `(E1, E2, R1)` per document as a sentence. The relation is
invented, and no benign document contains both of its entities, so a system that answers
*"what is the relationship between E1 and E2?"* with `R1` must have ingested the corpus.
Detection is a lookup, not a statistic.

**Ward** watermarks the text itself. A token is green or red by a keyed hash of the two tokens
before it, and watermarked text over-uses green ones. Detection is a z-test over the retrieved
responses. The cost is that the documents have to be rewritten.

**DMI-RAG** leaves the corpus alone and publishes a handful of synthetic *canary* documents
beside it — invented institutions, invented findings, watermarked text. Retrieval of a canary
is what gets measured. The owner's real data is not altered.
