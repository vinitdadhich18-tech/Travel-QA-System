# Phase 6: Paper Draft — Progress

Date: 2026-10-25
Status: COMPLETE (ALL CHECKS PASS)

---

## 1. Done-When Verification Checklist

| Requirement | Target | Status | Verification Evidence |
|---|---|---|---|
| Master Numbers File Created | `paper/numbers.md` | PASS | Generated comprehensive numbers table mapping every statistic, count, percentage, latency, token count, and metadata item directly to `data/eval/results/` (`test-frozen-meta.json`, `test-frozen-report.md`, `timing-report.md`, `test-frozen-rows.csv`). |
| Paper Draft Created | `paper/draft.md` | PASS | Complete manuscript draft created containing all required sections: Abstract (191 words), Introduction, Related Work, System Design, Experimental Setup, Results, Discussion, Limitations, Conclusion, and Action Checklist. |
| Numerical Verification Tags | Every figure in `paper/draft.md` tagged `[VERIFY]` | PASS | All 154 instances of figures in `draft.md` carry `[VERIFY]` and 100% of unique items appear in `paper/numbers.md` (verified via automated audit script, 0 missing). |
| Scientific Restraint & No Overclaiming | No novelty, significance, or generalization claims | PASS | Text explicitly disclaims novelty of LLM classification, frames ranking interference as a hypothesis, refers strictly to "on this synthetic test set", and makes no statistical significance claims. |
| Product Shell Isolation | Website features noted as product shell | PASS | Explicitly noted that user authentication, booking flows, simulated payments, and staff queue are demonstration product shells, were not evaluated for usability, and support no research claims. |
| Mandatory Limitations Documented | Stale-policy, CONFLICT asymmetry, floor, comment stripping | PASS | Section 7 includes explicit subsections for stale-policy source-faithfulness, asymmetric CONFLICT retrieval, untuned 0.3 floor firing 0 times in observed V2 logs, HTML comment stripping, and single-model constraints. |
| Unfilled Author Interpretations | `[AUTHOR: interpretation]` placeholders | PASS | Qualitative "why" interpretations in Section 6 left as `[AUTHOR: interpretation]` placeholders without inventing unverified rationales. |
| Unfilled Citations and Needs | `[CITE: ...]` and `[NEED: ...]` placeholders | PASS | Related work and prior literature claims formatted strictly as search queries and placeholders without fabricating bibtex/authors. |
| Verification & Action Checklist | Section 9 of `paper/draft.md` | PASS | Complete enumerated checklist provided for all `[VERIFY]`, `[AUTHOR]`, `[CITE]`, and `[NEED]` items. |
| Git Hygiene & Secret Safety | Untracked secrets and clean status | PASS | `.env` and `AGENTS.md` are git-ignored and untracked. No key material in non-ignored files. No files > 50MB. |

---

## 2. Quantitative Summary of Frozen Results in Paper

### Headline Comparison (Full Test Set, N = 56)
- **V1 (RAG Baseline):**
  - Overall Correctness: 33/56 (58.9%)
  - Answer Accuracy (on non-escalated): 18/30 (60.0%)
  - Wrong Answer Rate: 20/56 (35.7%)
  - Escalation Rate: 18/56 (32.1%)
  - Unnecessary Escalation Rate: 2/32 (6.3%)
  - Mean Tokens / Question: 947
  - Median Latency (Benchmark): 1070 ms

- **V2 (Classify-First Architecture):**
  - Overall Correctness: 30/56 (53.6%)
  - Answer Accuracy (on non-escalated): 15/25 (60.0%)
  - Wrong Answer Rate: 15/56 (26.8%)
  - Escalation Rate: 26/56 (46.4%)
  - Unnecessary Escalation Rate: 7/32 (21.9%)
  - Mean Tokens / Question: 740
  - Median Latency (Benchmark): 934 ms

### Sensitivity Comparison (Excluding Stale Policy, N = 48)
- **V1 Overall Correctness:** 26/48 (54.2%)
- **V2 Overall Correctness:** 26/48 (54.2%)
- **V1 Answer Accuracy:** 11/22 (50.0%)
- **V2 Answer Accuracy:** 11/20 (55.0%)
- **V1 Wrong Answer Rate:** 19/48 (39.6%)
- **V2 Wrong Answer Rate:** 14/48 (29.2%)

---

## 3. Manual Items Remaining for the Author

1. **Verify Numbers ([numbers.md](file:///d:/Projects/Travel-QA-System/paper/numbers.md)):**
   - Cross-check every `[VERIFY]` number in `paper/draft.md` against `paper/numbers.md` and the underlying reports.
2. **Find and Add Citations:**
   - Search the literature for the `[CITE: ...]` placeholders and `[NEED: ...]` topics outlined in Section 2.
3. **Write Error Analysis Interpretation:**
   - Replace the three `[AUTHOR: interpretation]` placeholders in Section 6 (Discussion: RQ1, RQ2, RQ3) with your qualitative insights.
