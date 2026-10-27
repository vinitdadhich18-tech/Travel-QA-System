# Master Numbers Table for Paper Draft

This file lists every quantitative figure, count, percentage, latency, token count, parameter, and metadata value referenced in `paper/draft.md`.
All values derive strictly from the evaluated artifacts in `data/eval/results/` (`test-frozen-meta.json`, `test-frozen-report.md`, `timing-report.md`, `test-frozen-rows.csv`, and `dev-1-report.md`).
Every occurrence of a number in `paper/draft.md` is tagged with `[VERIFY]`.

---

## 1. Corpus, Dataset & Experimental Setup Metadata

**Source:** `data/eval/results/test-frozen-meta.json` and `AGENTS.md` Section A4–A5

| Parameter / Metric | Value | Source / Table |
|---|---|---|
| Total Curated OKF Policy Files | 18 | `test-frozen-meta.json` (`counts.okfFiles`) |
| Total Operational Incident Notices | 30 | `test-frozen-meta.json` (`counts.notices`) |
| Total Evaluation Questions | 84 | `test-frozen-meta.json` (`counts.questionsTotal`) |
| Development Set Questions | 28 | `AGENTS.md` (A4) / `dev-1-report.md` |
| Frozen Test Set Questions | 56 | `test-frozen-meta.json` (`counts.splitQuestions`) |
| Number of Question Categories | 7 | `AGENTS.md` (A4) |
| Questions per Category in Test Set | 8 | `test-frozen-report.md` Section 2 |
| Questions per Category in Dev Set | 4 | `dev-1-report.md` Section 2 |
| Git Commit Hash (Frozen State) | `ee41f37997be108dfdc348b3d0348cfd3b5bc9e8` | `test-frozen-meta.json` (`gitCommitHash`) |
| Generation & Classification Model | `qwen/qwen3.8-27b` | `test-frozen-meta.json` (`GROQ_MODEL`) |
| Embedding Model | `gemini-embedding-001` | `test-frozen-meta.json` (`GEMINI_EMBED_MODEL`) |
| RAG Cosine Similarity Floor | 0.3 | `test-frozen-meta.json` (`RAG_SIM_FLOOR`) |
| Groq Daily Token Cap | 90,000 (90000) | `AGENTS.md` (A9) / `.env` (`GROQ_DAILY_TOKEN_CAP`) |
| Classification Prompt SHA256 | `c194271cf4dac4d5bd2cc70845e385d3ff040d8de589a8959ae24b184765806f` | `test-frozen-meta.json` (`promptHashes.classifySha256`) |
| Answer Prompt SHA256 | `7f902295ed715704c2b9e86db67ba36d2a858200f19878bbac37278b84b8e319` | `test-frozen-meta.json` (`promptHashes.answerSha256`) |
| Total Evaluated Rows (Test) | 112 (56 for V1, 56 for V2) | `test-frozen-report.md` Header |
| Total Evaluated Rows (Dev) | 56 (28 for V1, 28 for V2) | `dev-1-report.md` Header |
| Total Timing Benchmark Runs | 120 (40 per repetition × 3 repetitions) | `timing-report.md` Section 1 & `Phase 5B.md` Check 16 |
| Timing Question Subset | 20 test questions (cache bypassed) | `timing-report.md` Header |
| Timing Paired Observations | 60 | `timing-report.md` Section 3 |
| Manual Review Overrides (Author) | 0 | `test-frozen-report.md` Header |
| Manual Review Overrides (Agent-Proposed) | 22 | `test-frozen-report.md` Header |
| Seeded Spot-Check Questions | 5 | `data/eval/results/review-queue.md` / `Phase 5B.md` Check 17 |
| Non-escalated answerable questions subset | 32 (4 categories × 8 questions) | `test-frozen-report.md` Section 1 (`Unnecessary Escalation Rate` denominator) |

---

## 2. Headline Performance (Frozen Test Set, N = 56)

**Source:** `data/eval/results/test-frozen-report.md` Section 1

| Metric | V1 (RAG Baseline) | V2 (Classify-First) | Source File & Section |
|---|---|---|---|
| Overall Correctness | 33/56 (58.9%) | 30/56 (53.6%) | `test-frozen-report.md` Table 1 |
| Answer Accuracy (on non-escalated) | 18/30 (60.0%) | 15/25 (60.0%) | `test-frozen-report.md` Table 1 |
| Wrong Answer Rate | 20/56 (35.7%) | 15/56 (26.8%) | `test-frozen-report.md` Table 1 |
| Escalation Rate | 18/56 (32.1%) | 26/56 (46.4%) | `test-frozen-report.md` Table 1 |
| Unnecessary Escalation Rate | 2/32 (6.3%) | 7/32 (21.9%) | `test-frozen-report.md` Table 1 |

---

## 3. Sensitivity Analysis (Excluding Stale Policy Category, N = 48)

**Source:** `data/eval/results/test-frozen-report.md` Section 1

| Metric | V1 (RAG Baseline) | V2 (Classify-First) | Source File & Section |
|---|---|---|---|
| Overall Correctness (Excl. Stale) | 26/48 (54.2%) | 26/48 (54.2%) | `test-frozen-report.md` Table 1 (Sensitivity) |
| Answer Accuracy (on non-escalated) | 11/22 (50.0%) | 11/20 (55.0%) | `test-frozen-report.md` Table 1 (Sensitivity) |
| Wrong Answer Rate | 19/48 (39.6%) | 14/48 (29.2%) | `test-frozen-report.md` Table 1 (Sensitivity) |

---

## 4. Per-Category Breakdown (8 Questions per Category)

**Source:** `data/eval/results/test-frozen-report.md` Section 2

| Category | Expected Outcome | V1 Correct | V2 Correct | V1 Escalated | V2 Escalated | V1 Wrong | V2 Wrong |
|---|---|---|---|---|---|---|---|
| `okf_only` | OKF_ONLY | 4/8 (50.0%) | 3/8 (37.5%) | 0/8 (0.0%) | 2/8 (25.0%) | 4/8 (50.0%) | 3/8 (37.5%) |
| `rag_only` | RAG_ONLY | 4/8 (50.0%) | 4/8 (50.0%) | 0/8 (0.0%) | 0/8 (0.0%) | 4/8 (50.0%) | 4/8 (50.0%) |
| `hybrid` | HYBRID | 3/8 (37.5%) | 4/8 (50.0%) | 2/8 (25.0%) | 2/8 (25.0%) | 3/8 (37.5%) | 2/8 (25.0%) |
| `human_fallback_stale` | OKF_ONLY | 7/8 (87.5%) | 4/8 (50.0%) | 0/8 (0.0%) | 3/8 (37.5%) | 1/8 (12.5%) | 1/8 (12.5%) |
| `human_fallback_no_evidence` | ESCALATE | 7/8 (87.5%) | 7/8 (87.5%) | 7/8 (87.5%) | 7/8 (87.5%) | 1/8 (12.5%) | 1/8 (12.5%) |
| `human_fallback_ambiguous` | ESCALATE | 8/8 (100.0%) | 8/8 (100.0%) | 8/8 (100.0%) | 8/8 (100.0%) | 0/8 (0.0%) | 0/8 (0.0%) |
| `human_fallback_conflict` | ESCALATE | 0/8 (0.0%) | 0/8 (0.0%) | 1/8 (12.5%) | 4/8 (50.0%) | 7/8 (87.5%) | 4/8 (50.0%) |

---

## 5. Conflict Category Asymmetry Deep-Dive (N = 8)

**Source:** `data/eval/results/test-frozen-report.md` Section 3

| Metric | V1 (RAG Baseline) | V2 (Classify-First) | Source File & Section |
|---|---|---|---|
| Correct Escalation via `CONFLICT` | 0/8 (0.0%) | 0/8 (0.0%) | `test-frozen-report.md` Table 3 |
| Safe Escalation (Any Escalation Reason) | 1/8 (12.5%) | 4/8 (50.0%) | `test-frozen-report.md` Table 3 |
| Retrieved Both Conflicting Files | 8/8 (100.0%) | 0/8 (0.0%) | `test-frozen-report.md` Table 3 |

---

## 6. V2 Classification, Routing & Escalation Analysis

**Source:** `data/eval/results/test-frozen-report.md` Section 4

| Metric / Parameter | Value | Source File & Section |
|---|---|---|
| Overall V2 Routing Accuracy | 37/56 (66.1%) | `test-frozen-report.md` Section 4 |
| INVALID Classifications | 0 | `test-frozen-report.md` Section 4 |
| Similarity Floor Firings (similarity < 0.3) | 0 times | `test-frozen-report.md` Section 4 |
| V2 Accuracy when Routing was Correct | 14/24 (58.3%) | `test-frozen-report.md` Section 4 |
| V2 Accuracy when Routing was Misclassified | 1/1 (100.0%) | `test-frozen-report.md` Section 4 |
| Escalations with reason `INSUFFICIENT` | 17 | `test-frozen-report.md` Section 4 Table |
| Escalations with reason `CLASSIFIER_ESCALATE` | 8 | `test-frozen-report.md` Section 4 Table |
| Escalations with reason `NO_OKF_MATCH` | 1 | `test-frozen-report.md` Section 4 Table |
| ESCALATE questions routed correctly to ESCALATE | 7/24 (29.2%) | Derived from Confusion Matrix (Table 3) |
| ESCALATE questions misrouted to OKF_ONLY | 10/24 (41.7%) | Derived from Confusion Matrix (Table 3) |
| ESCALATE questions misrouted to RAG_ONLY | 5/24 (20.8%) | Derived from Confusion Matrix (Table 3) |
| ESCALATE questions misrouted to HYBRID | 2/24 (8.3%) | Derived from Confusion Matrix (Table 3) |
| OKF_ONLY questions routed correctly to OKF_ONLY | 14/16 (87.5%) | Derived from Confusion Matrix (Table 3) |
| RAG_ONLY questions routed correctly to RAG_ONLY | 8/8 (100.0%) | Derived from Confusion Matrix (Table 3) |
| HYBRID questions routed correctly to HYBRID | 8/8 (100.0%) | Derived from Confusion Matrix (Table 3) |

### V2 Routing Confusion Matrix (N = 56)

| Expected Outcome | Pred: OKF_ONLY | Pred: RAG_ONLY | Pred: HYBRID | Pred: ESCALATE | Pred: INVALID | Total |
|---|---|---|---|---|---|---|
| OKF_ONLY | 14 | 0 | 1 | 1 | 0 | 16 |
| RAG_ONLY | 0 | 8 | 0 | 0 | 0 | 8 |
| HYBRID | 0 | 0 | 8 | 0 | 0 | 8 |
| ESCALATE | 10 | 5 | 2 | 7 | 0 | 24 |

---

## 7. Operational Metrics: Tokens, Retrieval Hit Rates & Latency

**Sources:** `data/eval/results/test-frozen-report.md` Section 5 & `data/eval/results/timing-report.md`

| Metric | V1 (RAG Baseline) | V2 (Classify-First) | Source File & Section |
|---|---|---|---|
| Mean Tokens per Question (Frozen Run) | 947 | 740 | `test-frozen-report.md` Section 5 |
| Retrieval Hit Rate (`sourceRetrieved`) | 32/32 (100.0%) | 27/32 (84.4%) | `test-frozen-report.md` Section 5 |
| Indicative Median Latency (Eval Run) | 1191 ms | 399 ms | `test-frozen-report.md` Section 5 |
| Indicative Mean Latency (Eval Run) | 36605 ms | 497 ms | `test-frozen-report.md` Section 5 |
| Median Latency (3-Rep Timing Benchmark) | 1070 ms | 934 ms | `timing-report.md` Section 1 |
| p95 Latency (3-Rep Timing Benchmark) | 1614 ms | 1755 ms | `timing-report.md` Section 1 |
| V2 Median Classification Latency | — | 155 ms | `timing-report.md` Section 2 |
| V2 Median Targeted Retrieval Latency | — | 556 ms | `timing-report.md` Section 2 |
| V2 Median Answer Generation Latency | — | 250 ms | `timing-report.md` Section 2 |
| Median Paired Latency Difference (V2 - V1) | — | -306 ms | `timing-report.md` Section 3 |
