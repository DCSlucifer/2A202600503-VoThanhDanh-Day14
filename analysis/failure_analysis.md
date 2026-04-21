# Báo cáo Phân tích Thất bại (Failure Analysis Report)

## 1. Tổng quan benchmark (run thật với API key)

Lệnh chạy:
```bash
python data/synthetic_gen.py
python main.py
python check_lab.py
```

Nguồn số liệu:
- `reports/summary.json`
- `reports/benchmark_results.json`

Kết quả V2 (Agent_V2_Optimized):
- Total cases: **60**
- Pass / Fail / Error: **55 / 5 / 0**
- Avg score: **4.2167**
- HitRate@3: **0.8667**
- Agreement rate: **1.0000**
- p95 latency: **0.0652s**
- Total eval cost (judge): **$0.006348**

Regression (V1 -> V2):
- Avg score delta: **+0.0834**
- HitRate@3 delta: **+0.1000**
- Gate decision: **BLOCK** (không đạt điều kiện delta score >= +0.2)

---

## 2. Bonus Role 1 (retrieval stress-test)

### 2.1. Top-k sensitivity (từ benchmark_results)

| Metric | Value |
|---|---:|
| HitRate@1 | **0.7333** |
| HitRate@3 | **0.8667** |
| HitRate@5 | **0.8667** |
| MRR | **0.7917** |

Nhận xét:
- `Hit@3 - Hit@1 = +0.1334`, cho thấy hiệu quả tăng rõ khi mở rộng từ top-1 lên top-3.
- `Hit@5` không tăng so với `Hit@3`, tức bottleneck nằm ở chất lượng xếp hạng top đầu, không phải thiếu độ phủ tài liệu ở top-5.

### 2.2. Hard-cases retrieval fail-rate (điều kiện bonus >20%)

Định nghĩa fail cho Role 1 bonus:
- Một hard-case được tính fail nếu retrieval miss ở mức `hit_rate@3 = 0`.
- Lý do dùng tiêu chí này: bonus Role 1 tập trung vào khả năng chịu tải của **retrieval stage** trước generation.

Hard-case scope:
- `adversarial_prompt_injection` (12)
- `edge_ambiguous_ooc` (8)
- `conflicting_information` (6)
- `multi_turn` (4)
- Tổng: **30 cases**

Kết quả:
- Hard-case retrieval fail: **8 / 30**
- Hard-case retrieval fail-rate: **26.67%** -> **đạt điều kiện bonus >20%**

Phân bổ retrieval fail theo loại:

| Case type | Retrieval fail | Total | Rate |
|---|---:|---:|---:|
| adversarial_prompt_injection | 0 | 12 | 0.00% |
| edge_ambiguous_ooc | 4 | 8 | 50.00% |
| conflicting_information | 2 | 6 | 33.33% |
| multi_turn | 2 | 4 | 50.00% |

---

## 3. Failure clustering (output từ runner)

Từ `summary.regression.failure_clusters`:
- `prompt_attack`: `ADV-004`
- `hallucination`: `EDGE-003`
- `retrieval_miss`: `CFG-003`, `CFG-006`, `MTR-003`
- `incomplete`: *(none)*
- `tone_mismatch`: *(none)*

---

## 4. 5 Whys (3 case fail tiêu biểu)

### Case #1: `CFG-006` (conflicting_information, retrieval_miss)
1. **Symptom:** Trả lời sai chủ đề, đi vào release gate metric thay vì xử lý xung đột SLA 2h/4h.
2. **Why 1:** Retriever lấy `doc_release_gate` thay vì `doc_sla_v2` + `doc_sla_legacy`.
3. **Why 2:** Keyword “release gate” có trọng số cao hơn cụm entity SLA trong logic retrieval hiện tại.
4. **Why 3:** Chưa có disambiguation layer để ưu tiên intent “conflicting policy resolution”.
5. **Why 4:** Chưa có rerank theo entity trọng tâm (`SLA`, `2 giờ`, `4 giờ`).
6. **Root cause:** Retrieval intent matching chưa đủ semantic, còn lệ thuộc keyword bề mặt.

### Case #2: `EDGE-008` (edge_ambiguous_ooc, retrieval_miss)
1. **Symptom:** Câu hỏi mơ hồ không trigger được `doc_clarification_policy`.
2. **Why 1:** Retriever trả về doc liên quan judge thay vì clarify policy.
3. **Why 2:** KEYWORD_MAP thiếu coverage cho biến thể diễn đạt mơ hồ.
4. **Why 3:** Chưa có nhánh “ambiguity-first” ở tầng retrieval.
5. **Why 4:** Query normalization hiện chưa xử lý tốt short/underspecified utterance.
6. **Root cause:** Thiếu coverage và normalization cho ambiguous query class.

### Case #3: `MTR-003` (multi_turn, retrieval_miss)
1. **Symptom:** Follow-up không kéo được doc hiệu năng phù hợp trong top-3.
2. **Why 1:** Retriever chủ yếu match theo câu hỏi hiện tại, không khai thác đầy đủ context carry-over.
3. **Why 2:** Từ khóa ở turn hiện tại không đủ mạnh để map vào doc mục tiêu.
4. **Why 3:** Chưa có query rewriting sử dụng conversation history.
5. **Why 4:** Chưa có chiến lược retrieval riêng cho multi-turn cases.
6. **Root cause:** Multi-turn retrieval strategy chưa hoàn chỉnh (thiếu history-aware retrieval).

---

## 5. Action plan (Role 1 scope)

- [ ] Bổ sung intent-aware query normalization cho `edge_ambiguous_ooc` và `conflicting_information`.
- [ ] Mở rộng synonym/phrase coverage cho nhóm câu hỏi mơ hồ.
- [ ] Thêm history-aware query rewriting cho `multi_turn`.
- [ ] Theo dõi cố định 2 chỉ số bonus Role 1 qua mỗi run:
  - Top-k sensitivity (`Hit@1/@3/@5`, `MRR`)
  - Hard-case retrieval fail-rate (`hit@3=0` trên hard-case pack)
