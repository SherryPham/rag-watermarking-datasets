# RAG-WM dataset

115 documents on diverse real-world topics, each carrying a RAG-WM knowledge watermark,
paired with the unwatermarked originals.

Paper: https://arxiv.org/abs/2501.05249

```
ragwm-dataset/
├── benign_docs/          documents without watermarks
└── watermarked_docs/     documents with watermarks
```

## The watermark

The watermark is a **false fact**, not a wording pattern. Each document gets exactly one
**watermark unit** — a triple `(E1, E2, R1)` asserting relation `R1` between two entities,
written as a sentence and appended to the end. 

Entity pairs were drawn by a SHA-256 hash chain keyed on a secret, and each pair's relation is

```
R1 = relation_top[ sha256(E1 ‖ E2 ‖ key) mod len(relation_top) ]
```


## Detection

The unit is a fact, so detection is a **query**, one per document,
stored as `verify_query`:

> What is the relationship between `E1` and `E2`?

The relations are invented — not true of the world — and **no benign document contains both
entities of any unit** (checked across all 115). So a RAG system that answers with `R1` cannot
have got it from the benign corpus or from parametric knowledge.

