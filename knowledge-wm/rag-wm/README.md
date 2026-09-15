# RAG-WM dataset

15 documents carrying a RAG-WM knowledge watermark across diverse real-world topics.
RAG-WM paper: https://arxiv.org/abs/2501.05249

```
rag-wm/
├── benign_docs/          
└── watermarked_docs/     
```

## The watermark

Each watermarked document carries exactly one **watermark unit** — a triple `(E1, E2, R1)` asserting relation `R1` between two entities appended to the end of the document. The benign text is not modified.

The 15 units use 15 distinct entities and 9 relation types (`USED_IN`, `PRESERVED_IN`, `PART_OF`, `DEVELOPED_BY`, `FUNDED_BY`, `SUPPLIES`, `PROTECTS`, `TRANSPORTS`, `CAUSES`).

The relations are synthetic: entity pairs were drawn by a SHA-256 hash chain keyed on a secret, and each pair's relation is `relation_top[sha256(E1‖E2‖key) mod 20]`. They are not true of the world and do not appear in the benign corpus. A RAG system that answers *"What is the relationship between E1 and E2?"* with the asserted relation must have ingested this corpus.

