# DADA2 — Agentic RAG over SEC Filings

```
Deployed app: https://triageql.onrender.com
```

## Overview

This project answers questions about nine 10-K filings from NVIDIA, AMD and Micron, three fiscal years per company, split into 27 Business, Risk Factors and MD&A sections. The notebooks compare retrieval strategies on the same labeled questions, then build an agent that diagnoses why a search fell short and picks its next step from that diagnosis.

## How it works

Gemini writes the search queries, plans and answers. Cloudflare's `clef-flash` decision model classifies each question and judges whether the retrieved passages answer it. Pinecone holds the chunk embeddings and filters them by company and fiscal year, and BM25 adds exact-term matches for comparative questions. Cohere reranks the strongest search. Questions about a reported figure can read the supplied XBRL financial data instead of the prose.

## Evaluation

**Conditions.** The chat model and the judge are `gemini-3.5-flash-lite`, and the decision model is `clef-flash`. The embeddings are `@cf/qwen/qwen3-embedding-0.6b` on Cloudflare (1024 dimensions), in the Pinecone index `dada2-1024`. The reranker is Cohere `rerank-v3.5`. The 27 section files make 2,124 chunks of up to 1,200 characters, with an overlap of 150. Retrieval uses k=5, a rerank pool of 10 and an RRF constant of 60. The router test uses all 24 labeled questions. The agent test uses the first 8 specific and the first 6 comparative questions, with one run of each agent.

**The routers.** The decision model was right on 20 of 24 questions, at 0.49 s per call. The chat model was right on 18 of 24, at 0.82 s per call. The decision model also gives a probability for each option, and its calls do not use the Gemini quota. Below `SURE` (0.6), the agent starts with `filter+rrf`. One limit is in the code: `after_route` reads the label, not the probability, so an unsure "figure" label still goes to `lookup`.

**The agents.** The judge scored the ladder 0.79 and the diagnosis agent 0.86 on 14 questions, which is one more correct answer. The diagnosis agent used 3.1 chat calls per question against 5.1, and 16.0 s against 21.1 s. Rate-limit retries can add time. The notebook 03 traces show how the paths differ. On the TSMC and dress-code questions, the ladder climbed all three rungs for 9 chat calls each. The diagnosis agent planned one more search for TSMC (4 chat calls in the module check) and stopped dress code at `absent` (0.90). The judge sees only the question and the answer. In notebook 04 it marked an AMD answer WRONG because it took 2025 to be a future year.

**What did not help.** On six specific questions at k=5, `filter` had the best MRR (0.917). `filter+rrf` had 0.833, `filter+rrf+rerank` 0.889 and `hybrid+rrf` 0.806, all at recall@5 1.00. The filter hurt five comparative questions at k=10: recall fell from 0.83 to 0.57. BM25 on "HBM3E" returned five chunks from `AMD_2023_business.txt`, which does not look like an exact-term match. In the code, `local_chunks` collapses whitespace and Pinecone keeps the raw text, so RRF may not merge a chunk pair whose copies differ in whitespace. I did not measure this. These are small samples, not general rules.

**What I would ship.** I would ship the diagnosis agent: one more judged-correct answer, fewer chat calls and less time per question. I would also give the judge the passages and the reference answer, and check whether short heading chunks crowd out longer passages. I have not tested either change. First, I want to know how many user questions name one company and one year, and how many compare companies, because that decides the best first search.

Agent comparison as the judge recorded it (notebook 04, TODO 4.2). 14 questions, k=5.

| agent | accuracy | chat/q | decision/q | s/q |
|---|---|---|---|---|
| ladder | 0.79 | 5.1 | 0.0 | 21.1 |
| diagnosis | 0.86 | 3.1 | 2.5 | 16.0 |
