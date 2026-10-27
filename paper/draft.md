# Classify-Then-Retrieve: LLM-Based Question Routing Between Curated Policies and Event Data for Travel Support QA

---

## Abstract

Customer support question answering in travel operations requires reconciling long-form regulatory rules and transient operational incidents. Standard retrieval-augmented generation (RAG) indexes both within a single vector space, risking cross-source retrieval interference. We examine a classify-then-retrieve architecture (V2) that classifies customer queries from question text alone into policy (`OKF_ONLY`), incident (`RAG_ONLY`), combined (`HYBRID`), or unresolvable (`ESCALATE`) routes prior to targeted retrieval, benchmarking it against an undifferentiated vector baseline (V1). On a frozen synthetic test suite of 56 [VERIFY] travel queries, V1 achieved an overall correctness of 33/56 (58.9%) [VERIFY], while V2 achieved 30/56 (53.6%) [VERIFY]. Across a 48-question [VERIFY] sensitivity set excluding outdated policy tests, both architectures attained identical overall correctness of 26/48 (54.2%) [VERIFY]. V2 reduced the wrong answer rate from 20/56 (35.7%) [VERIFY] to 15/56 (26.8%) [VERIFY] by increasing safe escalations (26/56, 46.4% [VERIFY] vs. 18/56, 32.1% [VERIFY]), while consuming 740 [VERIFY] mean tokens compared to 947 [VERIFY] for V1. In a 3-repetition [VERIFY] timing benchmark across 20 [VERIFY] test questions with cache bypassed, V2 recorded a median latency of 934 ms [VERIFY] versus 1070 ms [VERIFY] for V1. We discuss classification confusion patterns, policy conflict asymmetries, and operational trade-offs.

---

## 1. Introduction

Automated question answering (QA) systems deployed in commercial customer support domains must handle inquiries spanning multiple distinct classes of knowledge. In travel support, queries typically target one of two disparate knowledge sources:
1. *Curated institutional policies*: Long-form, semi-static governance texts describing tariff rules, visa requirements, baggage allowances, and refund policies.
2. *Transient operational notices*: Short, event-driven updates detailing flight delays, airport gate closures, baggage handling incidents, and weather disruptions.

A conventional retrieval-augmented generation (RAG) architecture pools all enterprise knowledge assets into an undifferentiated vector store, retrieving the top-$k$ documents by dense semantic similarity and presenting them to an LLM for synthesis [CITE: Lewis et al., 2020]. However, we hypothesize that indexing long policy documents and transient incident snippets in a shared vector space can create cross-source ranking interference: queries concerning operational delays may retrieve general delay-compensation policy sections rather than specific incident notices, while policy inquiries may pull semantically adjacent but irrelevant operational event logs.

An alternative architectural paradigm is *classify-then-retrieve*. Rather than querying a combined index, an LLM classifier inspects the customer's query text alone, assigning it to a routing label: curated policy (`OKF_ONLY`), incident report (`RAG_ONLY`), joint evidence (`HYBRID`), or out-of-scope/unresolvable (`ESCALATE`). Targeted retrieval mechanisms then search exclusively within the indicated source collections, bypassing irrelevant knowledge domains entirely.

In this work, we implement and empirically compare two end-to-end travel support architectures:
- **V1 (Undifferentiated Baseline)**: An embedding-based vector retrieval pipeline searching a unified store containing both curated policies and operational notices, returning the top-3 [VERIFY] nearest neighbors.
- **V2 (Classify-Then-Retrieve)**: A pipeline that first classifies the user query via an LLM into one of four routing targets, invokes targeted retrieval (deterministic keyword-based policy matching, dense incident retrieval, both, or immediate escalation), and conditionally invokes an answer generation LLM.

We evaluate both systems on a frozen synthetic test set of 56 [VERIFY] questions across 7 [VERIFY] operational categories. We explicitly investigate three primary research questions:
- **RQ1**: How reliably does the LLM classify question types from the question text alone, and which types does it misclassify?
- **RQ2**: On correctly classified questions, does targeted retrieval give more accurate answers than V1's undifferentiated retrieval?
- **RQ3**: What extra latency and token cost does classification add, and is any accuracy change worth it?

---

## 2. Related Work

The study of hybrid question answering and routing in automated support systems spans multiple complementary research areas:

- **Retrieval-Augmented Generation (RAG)**: Dense retrieval coupled with parametric language model decoders [CITE: Dense Passage Retrieval; RAG for Knowledge-Intensive Tasks; Search queries: "dense retrieval multi-source", "RAG vector interference"].
- **Hallucination and Groundedness in LLM QA**: Mitigating ungrounded parametric completions through explicit source constraints, citation verification, and abstention prompting [CITE: Hallucination mitigation in enterprise QA; Search queries: "LLM abstention insufficient evidence", "grounded QA hallucination"].
- **Customer Support and Task-Oriented Chatbots**: Automated escalation, ticket routing, and conversational intent classification in enterprise domains [CITE: Automated triage in customer service; Search queries: "intent routing customer support", "ticket escalation RAG"].
- **Structured Knowledge Formats & Curated Documentation**: Integrating semi-structured documentation and curated knowledge frameworks (OKF) with unstructured corpora [CITE: Hybrid structured-unstructured retrieval; Search queries: "curated knowledge base RAG", "hybrid policy retrieval"].

We note that LLM-based question classification is not in itself a novel concept [NEED: Prior literature on zero-shot LLM intent classification]. Rather, our contribution lies in implementing, empirically benchmarking, and analyzing the trade-offs of this classify-then-retrieve design against an undifferentiated vector baseline in a controlled travel support setting with curated policy files and event notices.

---

## 3. System Design

The experimental platform consists of two question-answering pipelines (V1 and V2) integrated into a broader travel agency application. Both architectures rely on the same underlying models: Groq's `qwen/qwen3.8-27b` [VERIFY] for all generation and classification calls, and Google's `gemini-embedding-001` [VERIFY] for all dense vector embeddings.

### 3.1 Knowledge Corpus and Storage

The knowledge repository consists of two distinct components:
1. **Open Knowledge Framework (OKF) Policies**: 18 [VERIFY] markdown files containing complete policy texts covering baggage allowances, cancellation fees, visa requirements, and loyalty tiers. Each file includes YAML frontmatter metadata (`type`, `title`, `topic`, `sources`, `verified`, `stale_after`, `status`). Crucially, policy files are processed via a loader that strictly strips HTML comments prior to retrieval or LLM ingestion, preventing the model from viewing author notes or unverified factual corrections.
2. **Operational Notices**: 30 [VERIFY] structured incident records formatted as JSON objects, capturing specific event logs (e.g., flight delays, cancellations, mechanical repairs, baggage handling reports) associated with flight numbers, timestamps, and airlines.

In Chroma, vector embeddings are generated using `gemini-embedding-001` [VERIFY] with cosine similarity space (`hnsw:space: cosine`). Two Chroma collections are maintained:
- `combined`: Used by V1. Contains 48 [VERIFY] total items (all 18 [VERIFY] OKF policy files, each stored as a complete single document chunk, plus all 30 [VERIFY] operational notices).
- `notices`: Used by V2. Contains strictly the 30 [VERIFY] operational notices.

### 3.2 V1: Undifferentiated RAG Baseline

The V1 pipeline executes a standard vector retrieval workflow:
1. The user's query is embedded via `gemini-embedding-001` [VERIFY].
2. The system retrieves the top-3 [VERIFY] items from the `combined` Chroma collection using cosine similarity. V1 enforces no similarity floor.
3. The retrieved evidence is assembled into formatted blocks (`[OKF:<filename>]` or `[RAG:<notice id>]`) and passed directly into the frozen answer prompt.
4. Escalation occurs if and only if the answer LLM responds with the exact tokens `INSUFFICIENT` or `CONFLICT`.

### 3.3 V2: Classify-Then-Retrieve Architecture

The V2 pipeline introduces an upfront classification stage before executing source-targeted retrieval:

#### Stage 1: Question Classification
The question text alone is sent to `qwen/qwen3.8-27b` [VERIFY] using the frozen classification prompt:

```
You are a question classifier for a travel support system.
Given the customer's question, classify it as exactly one of:

- OKF_ONLY   — the question asks about a policy or rule (cancellation fees,
               visa requirements, baggage limits, loyalty terms)
- RAG_ONLY   — the question asks about a specific event or incident
               (a flight delay, a baggage report, an operational notice)
- HYBRID     — the question needs both a policy rule AND event facts
               (e.g. compensation for a specific delay: needs the rule
               AND the delay record)
- ESCALATE   — the question is out of scope, too vague to route, or
               not covered by any travel policy or event record

Reply with exactly one of the four labels above and nothing else.

Question: {question}
```

The output is strictly validated against the four valid tokens. If an unparseable response is received, the system escalates with reason `CLASSIFIER_INVALID`.

#### Stage 2: Targeted Retrieval
Based on the predicted class:
- `ESCALATE`: Bypasses all retrieval and answer generation, immediately escalating with reason `CLASSIFIER_ESCALATE`.
- `OKF_ONLY`: Routes to deterministic keyword-based policy matching. The matcher computes token and stem overlap between the query and policy metadata (`title`, `topic`, body), breaking ties alphabetically, and returns at most 1 [VERIFY] policy file. If no policy matches, the system escalates with reason `NO_OKF_MATCH`.
- `RAG_ONLY`: Embeds the query and queries the `notices` Chroma collection for the top-3 [VERIFY] incident records. An untuned cosine similarity floor of 0.3 [VERIFY] (`similarity = 1 - distance`) is applied. If all retrieved notices fall below 0.3 [VERIFY], the system escalates with reason `NO_RAG_MATCH`.
- `HYBRID`: Queries both the OKF keyword matcher (returning at most 1 [VERIFY] policy file) and the `notices` Chroma collection (returning up to 3 [VERIFY] notices with similarity $\ge$ 0.3 [VERIFY]). If neither source yields evidence, the system escalates with reason `NO_EVIDENCE`.

#### Stage 3: Answer Synthesis
If retrieval yields evidence, the blocks are formatted and passed to the frozen answer prompt:

```
You are a travel agency support assistant.
Answer the customer's question using ONLY the evidence below.
Cite the evidence you used: [OKF:filename] or [RAG:chunk-id].
If the evidence does not contain enough information to answer,
reply with exactly: INSUFFICIENT
If pieces of evidence contradict each other on the facts needed to answer,
reply with exactly: CONFLICT
Do not use outside knowledge.

Evidence:
{evidence}

Question: {question}
```

If the answer LLM replies with `INSUFFICIENT` or `CONFLICT`, the system records an escalation with that specific reason.

### 3.4 Product Shell Context (Web Application)

Both QA engines sit within a broader travel agency application constructed with Node/Express, React, and MongoDB. The application provides user authentication with role-based access (`customer` and `agent`), flight schedule browsing, simulated booking flows, customer support inboxes, and an employee support queue for escalated cases. We emphasize that this product shell serves strictly as an operational wrapper; all flight listings, prices, and booking records are synthetic, payment is simulated, and website features were not subject to usability or research evaluation. Inquiries regarding live website seat counts, real-time booking modification, or flight pricing are not represented in the OKF or RAG collections and appropriately escalate.

---

## 4. Experimental Setup

### 4.1 Dataset Design and Categories

The evaluation benchmark contains 84 [VERIFY] synthetic travel queries created by AI generation and subjected to human author verification. The dataset is partitioned into a development set (28 [VERIFY] questions) used exclusively for debugging pipeline integrity, and a frozen test set (56 [VERIFY] questions) evaluated exactly once.

The benchmark spans 7 [VERIFY] balanced operational categories (8 [VERIFY] test questions each):
1. `okf_only`: Rule-based queries answerable solely by policy documents (e.g., cancellation penalties, pet travel regulations). Expected outcome: `OKF_ONLY`.
2. `rag_only`: Inquiries targeting specific operational flight disruptions (e.g., flight delay duration, airport baggage claim status). Expected outcome: `RAG_ONLY`.
3. `hybrid`: Multi-hop queries requiring both general policy rules and specific flight event data (e.g., meal compensation entitlement for a delayed flight). Expected outcome: `HYBRID`.
4. `human_fallback_stale`: Questions answerable using two deliberately outdated policy files (`visa-country-d-britannica.md` and `refund-policy-general.md`). This category specifically tests whether models remain faithful to provided document text rather than real-world general knowledge. Expected outcome: `OKF_ONLY`.
5. `human_fallback_no_evidence`: Inquiries regarding airlines or policies completely absent from the knowledge base. Expected outcome: `ESCALATE`.
6. `human_fallback_ambiguous`: Vague or underspecified queries lacking necessary flight numbers, dates, or airline names. Expected outcome: `ESCALATE`.
7. `human_fallback_conflict`: Queries where general cancellation policies directly contradict airline-specific policies. Expected outcome: `ESCALATE` (with strict escalation reason `CONFLICT`).

### 4.2 Evaluation Protocol and Git Freeze

To prevent post-hoc tuning or data leakage:
- All prompts, similarity thresholds (0.3 [VERIFY]), keyword ranking logic, and embedding models were permanently frozen at Git commit `ee41f37997be108dfdc348b3d0348cfd3b5bc9e8` [VERIFY] under the Git tag `frozen-eval`.
- The prompt SHA256 hashes were verified: classification prompt `c194271cf4dac4d5bd2cc70845e385d3ff040d8de589a8959ae24b184765806f` [VERIFY], answer prompt `7f902295ed715704c2b9e86db67ba36d2a858200f19878bbac37278b84b8e319` [VERIFY].
- Evaluation was executed at LLM temperature 0 [VERIFY] with local caching enabled, ensuring deterministic reproduction. Across all 56 [VERIFY] test questions evaluated on both systems (112 [VERIFY] total runs), exact responses and execution metadata were logged to MongoDB.
- Due to Groq free-tier rate limitations (90,000 [VERIFY] daily token cap), evaluation scripts incorporated persistent resumption mechanisms.

### 4.3 Grading and Metrics

Responses are evaluated through key-fact substring matching complemented by human review:
- For answerable categories (`okf_only`, `rag_only`, `hybrid`, `human_fallback_stale`), an answer is graded correct if it contains all required key facts and does not escalate.
- For escalation categories (`no_evidence`, `ambiguous`, `conflict`), an outcome is graded correct if the pipeline escalated. For `human_fallback_conflict`, correctness strictly requires `escalationReason === "CONFLICT"`.
- A review queue of automated substring-grader failures was generated. Out of 22 [VERIFY] agent-proposed overrides and 5 [VERIFY] seeded spot-check questions, 0 [VERIFY] author overrides were modified, preserving transparent automated grading records.

We report five standardized metrics:
- **Overall Correctness**: Percentage of all 56 [VERIFY] questions where the system produced the correct outcome (accurate answer or required escalation).
- **Answer Accuracy**: Percentage of non-escalated questions where the system provided a factually correct answer.
- **Wrong Answer Rate**: Percentage of all questions where the system returned an incorrect answer instead of resolving or safely escalating.
- **Escalation Rate**: Percentage of all questions escalated to human support.
- **Unnecessary Escalation Rate**: Percentage of answerable questions (32 [VERIFY] total) that were escalated.

---

## 5. Results

### 5.1 Primary Headline Evaluation

Table 1 summarizes headline performance across the complete frozen test set (56 [VERIFY] questions), alongside the sensitivity analysis excluding the 8 [VERIFY] stale-policy questions (48 [VERIFY] questions).

| Metric | V1 (RAG Baseline) | V2 (Classify-First) |
|---|---|---|
| **Primary Evaluation (All 56 Questions)** | | |
| Overall Correctness | 33/56 (58.9%) [VERIFY] | 30/56 (53.6%) [VERIFY] |
| Answer Accuracy (on non-escalated) | 18/30 (60.0%) [VERIFY] | 15/25 (60.0%) [VERIFY] |
| Wrong Answer Rate | 20/56 (35.7%) [VERIFY] | 15/56 (26.8%) [VERIFY] |
| Escalation Rate | 18/56 (32.1%) [VERIFY] | 26/56 (46.4%) [VERIFY] |
| Unnecessary Escalation Rate | 2/32 (6.3%) [VERIFY] | 7/32 (21.9%) [VERIFY] |
| **Sensitivity Analysis (Excluding Stale Category, 48 Questions)** | | |
| Overall Correctness (Excl. Stale) | 26/48 (54.2%) [VERIFY] | 26/48 (54.2%) [VERIFY] |
| Answer Accuracy (on non-escalated) | 11/22 (50.0%) [VERIFY] | 11/20 (55.0%) [VERIFY] |
| Wrong Answer Rate | 19/48 (39.6%) [VERIFY] | 14/48 (29.2%) [VERIFY] |

*Table 1: Primary headline performance and sensitivity evaluation on the frozen test set.*

On the full test suite, V1 achieved an overall correctness of 33/56 (58.9%) [VERIFY], compared to 30/56 (53.6%) [VERIFY] for V2. When evaluating solely the non-escalated responses, both systems demonstrated identical answer accuracy of 18/30 (60.0%) [VERIFY] for V1 and 15/25 (60.0%) [VERIFY] for V2. However, V2 lowered the wrong answer rate from 20/56 (35.7%) [VERIFY] to 15/56 (26.8%) [VERIFY], while exhibiting a higher escalation rate (26/56, 46.4% [VERIFY] vs. 18/56, 32.1% [VERIFY]) and higher unnecessary escalation rate (7/32, 21.9% [VERIFY] vs. 2/32, 6.3% [VERIFY]).

In the sensitivity analysis excluding `human_fallback_stale`, both systems exhibited identical overall correctness of 26/48 (54.2%) [VERIFY]. Within this subset, V2 achieved an answer accuracy on non-escalated questions of 11/20 (55.0%) [VERIFY] compared to 11/22 (50.0%) [VERIFY] for V1, while maintaining a lower wrong answer rate of 14/48 (29.2%) [VERIFY] versus 19/48 (39.6%) [VERIFY] for V1.

### 5.2 Category Breakdown

Table 2 details performance across the 7 [VERIFY] individual categories (8 [VERIFY] questions per category).

| Category | Expected Outcome | V1 Correct | V2 Correct | V1 Escalated | V2 Escalated | V1 Wrong | V2 Wrong |
|---|---|---|---|---|---|---|---|
| `okf_only` | OKF_ONLY | 4/8 (50.0%) [VERIFY] | 3/8 (37.5%) [VERIFY] | 0/8 (0.0%) [VERIFY] | 2/8 (25.0%) [VERIFY] | 4/8 (50.0%) [VERIFY] | 3/8 (37.5%) [VERIFY] |
| `rag_only` | RAG_ONLY | 4/8 (50.0%) [VERIFY] | 4/8 (50.0%) [VERIFY] | 0/8 (0.0%) [VERIFY] | 0/8 (0.0%) [VERIFY] | 4/8 (50.0%) [VERIFY] | 4/8 (50.0%) [VERIFY] |
| `hybrid` | HYBRID | 3/8 (37.5%) [VERIFY] | 4/8 (50.0%) [VERIFY] | 2/8 (25.0%) [VERIFY] | 2/8 (25.0%) [VERIFY] | 3/8 (37.5%) [VERIFY] | 2/8 (25.0%) [VERIFY] |
| `human_fallback_stale` | OKF_ONLY | 7/8 (87.5%) [VERIFY] | 4/8 (50.0%) [VERIFY] | 0/8 (0.0%) [VERIFY] | 3/8 (37.5%) [VERIFY] | 1/8 (12.5%) [VERIFY] | 1/8 (12.5%) [VERIFY] |
| `human_fallback_no_evidence`| ESCALATE | 7/8 (87.5%) [VERIFY] | 7/8 (87.5%) [VERIFY] | 7/8 (87.5%) [VERIFY] | 7/8 (87.5%) [VERIFY] | 1/8 (12.5%) [VERIFY] | 1/8 (12.5%) [VERIFY] |
| `human_fallback_ambiguous` | ESCALATE | 8/8 (100.0%) [VERIFY] | 8/8 (100.0%) [VERIFY] | 8/8 (100.0%) [VERIFY] | 8/8 (100.0%) [VERIFY] | 0/8 (0.0%) [VERIFY] | 0/8 (0.0%) [VERIFY] |
| `human_fallback_conflict` | ESCALATE | 0/8 (0.0%) [VERIFY] | 0/8 (0.0%) [VERIFY] | 1/8 (12.5%) [VERIFY] | 4/8 (50.0%) [VERIFY] | 7/8 (87.5%) [VERIFY] | 4/8 (50.0%) [VERIFY] |

*Table 2: Performance breakdown across all seven evaluation categories (8 questions per category).*

Performance remained equivalent on `rag_only` (4/8, 50.0% [VERIFY] correct for both systems) and `human_fallback_ambiguous` (8/8, 100.0% [VERIFY] correct escalation for both systems). On `hybrid`, V2 achieved 4/8 (50.0%) [VERIFY] correct answers compared to 3/8 (37.5%) [VERIFY] for V1. On `human_fallback_stale`, V1 answered 7/8 (87.5%) [VERIFY] correctly without escalating (0/8, 0.0% [VERIFY]), whereas V2 answered 4/8 (50.0%) [VERIFY] correctly and escalated 3/8 (37.5%) [VERIFY].

### 5.3 V2 Classification and Routing Analysis

V2 attained an overall question classification accuracy of 37/56 (66.1%) [VERIFY]. Across the evaluation run, 0 [VERIFY] classifications were labeled `INVALID`. The confusion matrix is presented in Table 3.

| Expected Outcome | Pred: OKF_ONLY | Pred: RAG_ONLY | Pred: HYBRID | Pred: ESCALATE | Pred: INVALID | Total |
|---|---|---|---|---|---|---|
| **OKF_ONLY** | 14 [VERIFY] | 0 [VERIFY] | 1 [VERIFY] | 1 [VERIFY] | 0 [VERIFY] | 16 [VERIFY] |
| **RAG_ONLY** | 0 [VERIFY] | 8 [VERIFY] | 0 [VERIFY] | 0 [VERIFY] | 0 [VERIFY] | 8 [VERIFY] |
| **HYBRID** | 0 [VERIFY] | 0 [VERIFY] | 8 [VERIFY] | 0 [VERIFY] | 0 [VERIFY] | 8 [VERIFY] |
| **ESCALATE** | 10 [VERIFY] | 5 [VERIFY] | 2 [VERIFY] | 7 [VERIFY] | 0 [VERIFY] | 24 [VERIFY] |

*Table 3: Confusion matrix for V2 LLM classification on the frozen test set (N = 56).*

The classifier achieved perfect recall on `RAG_ONLY` (8/8 [VERIFY]) and `HYBRID` (8/8 [VERIFY]), and classified 14/16 (87.5%) [VERIFY] of `OKF_ONLY` questions correctly. However, on queries whose ground-truth outcome was `ESCALATE` (24 [VERIFY] questions spanning `no_evidence`, `ambiguous`, and `conflict`), the classifier routed only 7/24 (29.2%) [VERIFY] directly to `ESCALATE`. It misclassified 10/24 (41.7%) [VERIFY] as `OKF_ONLY`, 5/24 (20.8%) [VERIFY] as `RAG_ONLY`, and 2/24 (8.3%) [VERIFY] as `HYBRID`.

Examining answer accuracy conditional on routing:
- When routing was correct, V2 achieved an answer accuracy of 14/24 (58.3%) [VERIFY].
- When routing was misclassified, V2 produced an answer accuracy of 1/1 (100.0%) [VERIFY].

### 5.4 Escalation Behavior

Across all 56 [VERIFY] test questions, V2 escalated 26 [VERIFY] times. Tracking the logged escalation reasons:
- `INSUFFICIENT` (triggered by the answer LLM): 17 [VERIFY] occurrences.
- `CLASSIFIER_ESCALATE` (triggered directly by the upfront classifier): 8 [VERIFY] occurrences.
- `NO_OKF_MATCH` (triggered by keyword matcher failing to find policy overlap): 1 [VERIFY] occurrence.
- `RAG_SIM_FLOOR` (< 0.3 [VERIFY]): 0 [VERIFY] occurrences. In the observed V2 test logs, the untuned 0.3 [VERIFY] floor fired 0 [VERIFY] times.
- `CONFLICT` (triggered by answer LLM detecting contradiction): 0 [VERIFY] occurrences.

### 5.5 Conflict Category Deep-Dive and Retrieval Asymmetry

In the `human_fallback_conflict` category (8 [VERIFY] questions), policies provide mutually contradictory cancellation provisions between an airline-specific policy and a general refund policy. Under the evaluation protocol, a response is graded correct only if the pipeline explicitly escalates with `escalationReason === "CONFLICT"`.

Table 4 highlights the operational outcomes and retrieval characteristics for this category.

| Metric | V1 (RAG Baseline) | V2 (Classify-First) |
|---|---|---|
| Correct Escalation via `CONFLICT` | 0/8 (0.0%) [VERIFY] | 0/8 (0.0%) [VERIFY] |
| Safe Escalation (Any Escalation Reason) | 1/8 (12.5%) [VERIFY] | 4/8 (50.0%) [VERIFY] |
| Retrieved Both Conflicting Policy Files | 8/8 (100.0%) [VERIFY] | 0/8 (0.0%) [VERIFY] |

*Table 4: Conflict category handling and retrieval asymmetry (N = 8).*

Neither system achieved a correct score (0/8, 0.0% [VERIFY]) under the strict `CONFLICT` rule. However, V2 safely escalated 4/8 (50.0%) [VERIFY] questions (via `CLASSIFIER_ESCALATE` or `INSUFFICIENT`), yielding 4/8 (50.0%) [VERIFY] wrong answers, whereas V1 safely escalated only 1/8 (12.5%) [VERIFY], generating 7/8 (87.5%) [VERIFY] wrong answers.

A pronounced architectural asymmetry explains the evidence presented to the answer model: V1's top-3 [VERIFY] retrieval across the `combined` vector collection successfully pulled both conflicting policy documents in 8/8 (100.0%) [VERIFY] instances. Conversely, V2 retrieved both documents in 0/8 (0.0%) [VERIFY] instances, because V2's OKF keyword matcher returns at most 1 [VERIFY] top-scoring file, and queries routed to `ESCALATE` bypass retrieval entirely.

### 5.6 Latency and Token Efficiency

Token consumption and execution latencies are summarized in Table 5.

| Metric | V1 (RAG Baseline) | V2 (Classify-First) |
|---|---|---|
| **Primary Eval Run (N = 56)** | | |
| Mean Tokens per Question | 947 [VERIFY] | 740 [VERIFY] |
| Retrieval Hit Rate (`sourceRetrieved`) | 32/32 (100.0%) [VERIFY] | 27/32 (84.4%) [VERIFY] |
| Indicative Median Latency (Eval Run) | 1191 ms [VERIFY] | 399 ms [VERIFY] |
| Indicative Mean Latency (Eval Run) | 36605 ms [VERIFY] | 497 ms [VERIFY] |
| **Dedicated Timing Benchmark (3 Reps, N = 20)** | | |
| Median Latency | 1070 ms [VERIFY] | 934 ms [VERIFY] |
| p95 Latency (nearest rank) | 1614 ms [VERIFY] | 1755 ms [VERIFY] |
| V2 Classification Latency (Median) | — | 155 ms [VERIFY] |
| V2 Targeted Retrieval Latency (Median) | — | 556 ms [VERIFY] |
| V2 Answer Generation Latency (Median) | — | 250 ms [VERIFY] |
| Median Paired Difference (V2 - V1) | — | -306 ms [VERIFY] |

*Table 5: Computational resource consumption, retrieval hit rates, and timing benchmarks.*

In the primary evaluation run, V2 consumed 740 [VERIFY] mean tokens per question compared to 947 [VERIFY] for V1, reflecting tokens saved when V2 escalated prior to answer generation. V1 attained a retrieval hit rate (`sourceRetrieved`) on answerable questions of 32/32 (100.0%) [VERIFY], whereas V2 attained 27/32 (84.4%) [VERIFY].

In the dedicated 3-repetition [VERIFY] timing benchmark across 20 [VERIFY] test questions with cache bypassed (60 [VERIFY] paired observations), V2 demonstrated a median latency of 934 ms [VERIFY] (p95: 1755 ms [VERIFY]) versus 1070 ms [VERIFY] for V1 (p95: 1614 ms [VERIFY]). Stage-wise latency tracking for V2 revealed a median classification latency of 155 ms [VERIFY], targeted retrieval latency of 556 ms [VERIFY], and answer generation latency of 250 ms [VERIFY]. Across all 60 [VERIFY] paired observations, the median paired difference (V2 - V1) was -306 ms [VERIFY].

---

## 6. Discussion

We structure our discussion around the three core research questions:

### RQ1: Classification Reliability and Confusion Patterns
On this synthetic test set, the LLM classifier achieved an overall routing accuracy of 37/56 (66.1%) [VERIFY]. Routing accuracy varied sharply by query class:
- For concrete operational events (`RAG_ONLY`) and multi-hop questions (`HYBRID`), the classifier achieved 8/8 (100.0%) [VERIFY] accuracy.
- For clear policy inquiries (`OKF_ONLY`), it routed 14/16 (87.5%) [VERIFY] correctly.
- For queries that should escalate (`ESCALATE`), routing accuracy fell to 7/24 (29.2%) [VERIFY]. The classifier frequently routed unresolvable queries to `OKF_ONLY` (10/24 [VERIFY]) or `RAG_ONLY` (5/24 [VERIFY]). [AUTHOR: interpretation of why unresolvable queries triggered false-positive policy routing].

### RQ2: Impact of Targeted Retrieval on Accuracy
When considering solely non-escalated questions, targeted retrieval in V2 matched V1's answer accuracy on the full test set (18/30, 60.0% [VERIFY] vs. 15/25, 60.0% [VERIFY]) and exhibited higher accuracy on the 48-question sensitivity set (11/20, 55.0% [VERIFY] vs. 11/22, 50.0% [VERIFY]). On `hybrid` queries, V2 achieved 4/8 (50.0%) [VERIFY] correctness versus 3/8 (37.5%) [VERIFY] for V1.
However, V2's overall correctness on the full test set was 30/56 (53.6%) [VERIFY] compared to 33/56 (58.9%) [VERIFY] for V1, driven by V2's higher unnecessary escalation rate (7/32, 21.9% [VERIFY] vs. 2/32, 6.3% [VERIFY]). [AUTHOR: interpretation of retrieval precision vs unnecessary escalation trade-off].

### RQ3: Computational Overhead and Latency Trade-Offs
In the dedicated 3-repetition benchmark, the addition of an upfront classification step did not increase median end-to-end latency: V2's median latency was 934 ms [VERIFY] compared to 1070 ms [VERIFY] for V1, with a median paired difference of -306 ms [VERIFY]. V2 classification required a median of 155 ms [VERIFY]. Furthermore, V2 reduced mean token usage from 947 [VERIFY] to 740 [VERIFY] tokens per question. The computational savings stem from early escalation: queries routed to `ESCALATE` bypassed retrieval and answer generation entirely. [AUTHOR: interpretation of whether the token savings and lower wrong answer rate justify the lower overall correctness].

---

## 7. Limitations

Our experimental findings must be interpreted within the context of several methodological constraints:

1. **Synthetic and Single-Domain Evaluation**: The evaluation suite consists of 56 [VERIFY] synthetic questions covering travel support, generated by AI and verified by human authors. Results reflect this synthetic corpus and may not generalize to real customer support traffic or other domains.
2. **Fixed Single-Model Infrastructure**: All generation and classification tasks were performed using a single model (`qwen/qwen3.8-27b` [VERIFY]), and all dense embeddings used `gemini-embedding-001` [VERIFY]. Cross-model generalization was not evaluated.
3. **Cascading Classification Failures**: In V2, question misclassification propagates directly into retrieval failures. For example, routing an `OKF_ONLY` query to `RAG_ONLY` entirely precludes policy retrieval.
4. **Deterministic Keyword Matcher Constraints**: V2's OKF policy matcher relies on basic keyword and stem matching without semantic embeddings, synonym expansion, or document tags. Ties are broken alphabetically, and the matcher retrieves at most 1 [VERIFY] top file.
5. **Untuned Similarity Floor**: The 0.3 [VERIFY] cosine similarity floor was set a priori and never tuned. In the observed V2 test logs, this floor fired 0 [VERIFY] times; Phase 1 V1 contains no similarity floor, so these logs cannot establish whether a floor would have altered V1 retrieval.
6. **Rate-Limit Latency and Multi-Day Execution**: To comply with free-tier rate limits (90,000 [VERIFY] token cap), evaluation runs were spaced over multiple days, introducing potential cloud latency variance. While temperature 0 [VERIFY] and local caching ensure deterministic outputs for identical inputs, this single-run protocol provides no variance estimates across generation seeds.
7. **Stale Policy Category Interpretation**: Neither system incorporates document staleness or frontmatter timestamps into retrieval or answer generation. The `human_fallback_stale` category assesses source faithfulness to deliberately outdated document content, rather than real-world factual correctness. We report results both across all 56 [VERIFY] questions (V1: 33/56, 58.9% [VERIFY]; V2: 30/56, 53.6% [VERIFY]) and excluding this category across 48 [VERIFY] questions (V1: 26/48, 54.2% [VERIFY]; V2: 26/48, 54.2% [VERIFY]).
8. **Asymmetric Policy Conflict Detection**: Correctness on `human_fallback_conflict` requires escalating explicitly with reason `CONFLICT`. This requires presenting both conflicting policies to the answer LLM. V1's top-3 [VERIFY] retrieval across the combined store retrieved both conflicting policies in 8/8 (100.0%) [VERIFY] instances, whereas V2's single-file OKF matcher and direct escalation bypassed multi-document retrieval, retrieving both files in 0/8 (0.0%) [VERIFY] instances. Neither system produced the `CONFLICT` token (0/8, 0.0% [VERIFY]).
9. **Stripped Comment Metadata**: HTML comments documenting factual policy corrections were stripped from policy files prior to ingest, ensuring the models operated strictly on visible document prose.
10. **Product Shell Boundary**: The accompanying web application (user authentication, flight browsing, demo booking, staff queue) is a demonstration product shell operating on synthetic flight schedules and mock payments. It was not evaluated for usability and forms no part of the research claims. The QA chatbot possesses no programmatic access to real-time seat inventories or flight pricing; queries targeting live booking availability appropriately escalate.

---

## 8. Conclusion and Future Work

We presented an empirical comparison between an undifferentiated RAG baseline (V1) and a classify-then-retrieve architecture (V2) for travel customer support QA. On a frozen test set of 56 [VERIFY] synthetic queries, V1 attained an overall correctness of 33/56 (58.9%) [VERIFY], while V2 achieved 30/56 (53.6%) [VERIFY]; on a 48-question [VERIFY] sensitivity set excluding outdated policy tests, both architectures achieved identical correctness of 26/48 (54.2%) [VERIFY]. V2 reduced the wrong answer rate from 20/56 (35.7%) [VERIFY] to 15/56 (26.8%) [VERIFY] by safely escalating uncertain queries, while consuming fewer tokens (740 [VERIFY] vs. 947 [VERIFY]) and demonstrating comparable latency (median 934 ms [VERIFY] vs. 1070 ms [VERIFY]).

Future work includes exploring confidence-weighted routing thresholds, multi-document retrieval for policy contradiction detection, learning from human agent replies in the escalation queue, and evaluating the architecture across non-synthetic, enterprise customer support logs.

---

## 9. Verification & Author Action Checklist

This checklist enumerates every `[VERIFY]`, `[NEED]`, `[CITE]`, and `[AUTHOR]` tag appearing in this manuscript.

### 9.1 Verification Tags (`[VERIFY]`)

All numbers tagged with `[VERIFY]` must be verified against `paper/numbers.md` and the source reports in `data/eval/results/`:
- [ ] Abstract: 56 questions, 33/56 (58.9%), 30/56 (53.6%), 48 questions, 26/48 (54.2%), 20/56 (35.7%), 15/56 (26.8%), 26/56 (46.4%), 18/56 (32.1%), 740 tokens, 947 tokens, 3 repetitions, 20 test questions, 934 ms, 1070 ms.
- [ ] Section 1: top-3 retrieval, 56 questions, 7 categories.
- [ ] Section 3: `qwen/qwen3.8-27b`, `gemini-embedding-001`, 18 OKF files, 30 notices, 48 combined items, top-3 retrieval, 1 OKF file, 0.3 floor.
- [ ] Section 4: 84 total questions, 28 dev questions, 56 test questions, 7 categories, 8 questions/category, commit `ee41f37997be108dfdc348b3d0348cfd3b5bc9e8`, classify prompt SHA256, answer prompt SHA256, temperature 0, 112 test runs, 90,000 cap, 22 agent overrides, 5 spot-checks, 0 author overrides, 32 answerable questions.
- [ ] Section 5.1 (Table 1): V1 33/56 (58.9%), V2 30/56 (53.6%), V1 non-esc acc 18/30 (60.0%), V2 non-esc acc 15/25 (60.0%), V1 wrong 20/56 (35.7%), V2 wrong 15/56 (26.8%), V1 esc 18/56 (32.1%), V2 esc 26/56 (46.4%), V1 unnec esc 2/32 (6.3%), V2 unnec esc 7/32 (21.9%); Sensitivity: V1 26/48 (54.2%), V2 26/48 (54.2%), V1 non-esc acc 11/22 (50.0%), V2 non-esc acc 11/20 (55.0%), V1 wrong 19/48 (39.6%), V2 wrong 14/48 (29.2%).
- [ ] Section 5.2 (Table 2): per-category values across 7 categories (all 8 questions each).
- [ ] Section 5.3 (Table 3): routing accuracy 37/56 (66.1%), 0 INVALID, confusion matrix values (14, 0, 1, 1, 0, 16; 0, 8, 0, 0, 0, 8; 0, 0, 8, 0, 0, 8; 10, 5, 2, 7, 0, 24), routing conditional accuracy (14/24, 58.3%; 1/1, 100.0%).
- [ ] Section 5.4: escalations by reason (INSUFFICIENT: 17, CLASSIFIER_ESCALATE: 8, NO_OKF_MATCH: 1, 0.3 floor: 0, CONFLICT: 0).
- [ ] Section 5.5 (Table 4): CONFLICT correct 0/8 (0.0%), safe escalation V1 1/8 (12.5%), V2 4/8 (50.0%), retrieved both files V1 8/8 (100.0%), V2 0/8 (0.0%), wrong answers V1 7/8 (87.5%), V2 4/8 (50.0%).
- [ ] Section 5.6 (Table 5): mean tokens V1 947, V2 740; hit rate V1 32/32 (100.0%), V2 27/32 (84.4%); indicative latencies; timing benchmark latencies (median 1070 ms vs 934 ms; p95 1614 ms vs 1755 ms; V2 stages 155 ms, 556 ms, 250 ms; paired diff -306 ms across 60 observations).
- [ ] Section 7 & 8: restated figures for limitations and conclusion.

### 9.2 Author Interpretation Tags (`[AUTHOR]`)

The author must write the qualitative failure interpretations:
- [ ] Section 6, RQ1: `[AUTHOR: interpretation of why unresolvable queries triggered false-positive policy routing]`
- [ ] Section 6, RQ2: `[AUTHOR: interpretation of retrieval precision vs unnecessary escalation trade-off]`
- [ ] Section 6, RQ3: `[AUTHOR: interpretation of whether the token savings and lower wrong answer rate justify the lower overall correctness]`

### 9.3 Citation Placeholders (`[CITE]`)

The author must identify and insert formal bibliographic citations:
- [ ] Section 1 & 2: `[CITE: Lewis et al., 2020]`
- [ ] Section 2: `[CITE: Dense Passage Retrieval; RAG for Knowledge-Intensive Tasks]`
- [ ] Section 2: `[CITE: Hallucination mitigation in enterprise QA]`
- [ ] Section 2: `[CITE: Automated triage in customer service]`
- [ ] Section 2: `[CITE: Hybrid structured-unstructured retrieval]`

### 9.4 Literature Search Needs (`[NEED]`)

- [ ] Section 2: `[NEED: Prior literature on zero-shot LLM intent classification]`
