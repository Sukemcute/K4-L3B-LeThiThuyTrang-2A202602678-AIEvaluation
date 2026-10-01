# Day 14 — Exercises

## AI Evaluation & Benchmarking · Lab Worksheet

**Thời gian làm bài:** 9:15–12:00

**Domain:** OrbitTech Store Customer Support

Điền trực tiếp câu trả lời vào file này. Golden dataset 20 QA được viết một lần
duy nhất trong `golden_dataset.json`, không chép lại toàn bộ vào Markdown.

---

Từ 9:15–9:30, cài môi trường và chạy baseline tests theo `guide_lab.md`.

---

## Part 1 — Warm-up (9:30–9:45)

### Exercise 1.1 — RAGAS Metric Thresholds

Theo bài giảng:

- 0.8–1.0: Good — monitor, maintain.
- 0.6–0.8: Needs work — analyze failures, iterate.
- Dưới 0.6: Significant issues — investigate.

Với từng metric, xác định khi nào score thấp có thể chấp nhận và khi nào là
critical.

| Metric | Acceptable Low Score Scenario | Critical Low Score Scenario | Action Required |
|---|---|---|---|
| Faithfulness | Câu chào hỏi xã giao hoặc câu trả lời từ chối lịch sự với câu hỏi out-of-scope/adversarial không có trong context. | Trả lời sai/bịa đặt điều khoản chính sách bảo hành, hoàn tiền hoặc giá bán sản phẩm (hallucination). | Ép prompt bám sát context ("chỉ trả lời dựa trên context"), hạ temperature = 0, bổ sung guardrail kiểm tra grounding. |
| Answer Relevance | Khách hàng hỏi câu hỏi mơ hồ/chứa tiền đề sai, bot cần phản hồi để hỏi lại (clarify) hoặc sửa lại tiền đề thay vì trả lời trực tiếp. | Khách hỏi thủ tục đổi trả/bảo hành nhưng bot trả lời sang thông tin giới thiệu công ty hoặc quy trình tuyển dụng. | Tinh chỉnh system prompt buộc tập trung vào intent của người dùng; bổ sung few-shot examples về phản hồi đúng trọng tâm. |
| Context Recall | Câu hỏi đơn giản tra cứu 1 sự kiện hiển nhiên hoặc câu hỏi ngoài phạm vi không đòi hỏi trích xuất nhiều tài liệu phụ. | Câu hỏi chính sách tổng hợp nhiều bước (ví dụ: hoàn tiền khi hủy đơn) nhưng retriever bỏ sót văn bản chính sách cốt lõi. | Mở rộng top-K retrieved chunks; tối ưu hóa chiến lược chunking (chunk size/overlap); bổ sung hybrid search (BM25 + Semantic). |
| Context Precision | Truy xuất K lớn (ví dụ K=10) trong câu hỏi phức tạp đa tài liệu, các chunk hữu ích nằm rải rác nhưng vẫn nằm trong context window. | Top 1-2 chunks đầu tiên hoàn toàn là văn bản rác/nhiễu không liên quan, đẩy bằng chứng cốt lõi xuống cuối hoặc bị cắt bỏ. | Tích hợp thêm bước Reranking (Cross-encoder hoặc BM25 reranking) để đẩy các chunks liên quan nhất lên đầu danh sách. |
| Completeness | Người dùng chỉ hỏi một chi tiết hẹp và bot trả lời ngắn gọn, chính xác mà không cần liệt kê toàn bộ quy định dài dòng. | Khách hỏi đầy đủ điều kiện bảo hành/đổi trả nhưng bot chỉ nêu 1 điều kiện và bỏ sót các ràng buộc bắt buộc khác. | Bổ sung hướng dẫn trong prompt yêu cầu liệt kê đầy đủ danh sách điều kiện/tiêu chí dưới dạng bullet points. |

### Exercise 1.2 — Bias trong LLM-as-a-Judge

Ba bias thường gặp:

- Position bias: judge ưu tiên answer xuất hiện trước.
- Verbosity bias: judge ưu tiên answer dài hơn.
- Self-preference: judge ưu tiên output giống chính model đó.

**Câu 1: Thiết kế experiment phát hiện position bias với ít nhất hai conditions.**

> *Câu trả lời:*
> - **Condition 1 (Thứ tự chuẩn A-B):** Cung cấp Answer A ở vị trí Candidate 1 và Answer B ở vị trí Candidate 2 cho LLM Judge, ghi nhận kết quả đánh giá (hoặc điểm số).
> - **Condition 2 (Đảo ngược vị trí B-A):** Hoán đổi vị trí (Answer B ở Candidate 1, Answer A ở Candidate 2) với cùng Question, Rubric và giữ nguyên tham số (temperature = 0).
> - **Đánh giá:** Nếu vị trí Candidate 1 có tỷ lệ thắng áp đảo hoặc điểm số trung bình cao hơn đáng kể giữa hai điều kiện, hệ thống đã bị Position Bias. Giải pháp là đánh giá từng câu đơn lẻ (single-answer rating) hoặc đảo vị trí và lấy điểm trung bình (swap permutation).

**Câu 2: Làm thế nào giảm verbosity bias bằng rubric design?**

> *Câu trả lời:*
> - Định nghĩa rõ tiêu chuẩn đánh giá dựa trên **mật độ thông tin hữu ích và tính súc tích** (information density & conciseness) thay vì độ dài câu chữ.
> - Bổ sung quy định trừ điểm rõ ràng trong rubric: "Nếu câu trả lời dài dòng, lặp lại thông tin, hoặc chứa các đoạn văn rào đón không mang lại giá trị giải quyết vấn đề cho khách hàng, trừ 1–2 điểm".
> - Chỉ định cấu trúc và độ dài kỳ vọng cụ thể (ví dụ: câu trả lời tối ưu nên tóm gọn trong 3–5 gạch đầu dòng rõ ràng).

**Câu 3: Tại sao cần calibrate LLM judge với human labels?**

> *Câu trả lời:*
> - LLM Judge không tự động hiểu được kỳ vọng nghiệp vụ thực tế của con người và thường mắc các thiên kiến cố hữu (ưu tiên văn phong của chính nó, xu hướng chấm điểm dễ dãi hoặc quá khắt khe).
> - Cần đo độ tương quan (Spearman correlation / Cohen's Kappa) giữa điểm LLM chấm và điểm chuyên gia con người (human annotators) trên một tập mẫu validation để:
>   1. Xác minh độ tin cậy của Judge trước khi triển khai quy mô lớn.
>   2. Căn chỉnh thang điểm và phát hiện ngưỡng lệch (systematic bias) để tinh chỉnh prompt/rubric cho chuẩn xác.

### Exercise 1.3 — Evaluation trong CI/CD

**Câu 1: Chọn threshold để block deployment.**

| Metric | Threshold | Lý do |
|---|---:|---|
| Faithfulness | >= 0.85 | Hệ thống chăm sóc khách hàng OrbitTech yêu cầu độ trung thực cao nhất; ảo giác/bịa đặt chính sách có thể dẫn đến tranh chấp pháp lý hoặc thiệt hại tài chính. |
| Answer Relevance | >= 0.80 | Đảm bảo phản hồi giải quyết đúng thắc mắc của khách, tránh trả lời vòng vo gây mất thời gian và trải nghiệm tiêu cực. |
| Completeness | >= 0.75 | Đảm bảo cung cấp đủ thông tin quy trình/điều kiện cần thiết, chấp nhận độ co giãn nhẹ nếu câu trả lời súc tích. |

**Câu 2: Khi nào dùng offline evaluation, online evaluation và human review?**

> *Câu trả lời:*
> - **Offline Evaluation:** Dùng trong CI/CD pipeline trước khi release (mỗi pull request hoặc nightly build) trên bộ Golden Dataset cố định để kiểm tra hồi quy (regression) một cách an toàn, tự động và có thể tái lập.
> - **Online Evaluation:** Dùng khi hệ thống đã chạy trên production để giám sát traffic thực tế theo thời gian thực (tracking tỷ lệ escalate gặp nhân viên, thumbs up/down, hoặc sample một phần logs đưa qua LLM Judge) nhằm phát hiện data drift.
> - **Human Review:** Dùng định kỳ (audit mẫu 1–5% logs hàng tuần) hoặc rà soát chuyên sâu các ca bị gắn cờ nghiêm trọng (low-confidence / high-risk complaints) để cập nhật Golden Dataset và calibrate lại LLM Judge.

---

## Part 2 — Core Coding (9:45–10:40)

Hoàn thiện các TODO bắt buộc trong `template.py`.

### Task 1 — Data Models

- `QAPair`: question, expected answer, gold context, metadata và retrieved contexts.
- `EvalResult`: answer-side scores, optional retrieval scores, pass/failure fields.
- `overall_score()`: trung bình Faithfulness, Relevance và Completeness.

### Task 2 — RAGASEvaluator

Answer-side:

- `evaluate_faithfulness(answer, context)`
- `evaluate_relevance(answer, question)`
- `evaluate_completeness(answer, expected)`

Retrieval-side:

- `evaluate_context_recall(contexts, expected)`
- `evaluate_context_precision(contexts, expected)`

Full pipeline:

- `run_full_eval(..., contexts=None)` luôn tính ba answer metrics.
- Nếu có `contexts`, tính và lưu thêm Context Recall và Context Precision.
- Retrieval scores không làm thay đổi `overall_score()` và pass rule gốc.

### Task 3 — LLMJudge

- `score_response(question, answer, rubric)`
- `detect_bias(scores_batch)`

### Task 4 — BenchmarkRunner

- `run(qa_pairs, agent_fn, evaluator)`
- `generate_report(results)`
- `run_regression(new_results, baseline_results)`
- `identify_failures(results, threshold)`

`BenchmarkRunner.run()` phải truyền `pair.retrieved_contexts` vào
`run_full_eval()`. Report phải có average của hai retrieval metrics.

### Task 5 — FailureAnalyzer

- `categorize_failures(failures)`
- `find_root_cause(failure)`
- `generate_improvement_suggestions(failures)`
- `generate_improvement_log(failures, suggestions)`

Kiểm tra:

```bash
pytest tests/ -v
```

`rerank_by_overlap()` là TODO bonus của Exercise 3.5. Test tương ứng được skip
nếu bạn chưa làm bonus.

---

## Part 3 — Golden Dataset & Real Benchmark (10:40–11:35)

### Exercise 3.1 — Build the Golden Dataset

Thiết kế và validate dataset theo Mục 5–6 trong `guide_lab.md`. Nội dung 20 QA
được điền trực tiếp trong `golden_dataset.json`; phần dưới chỉ ghi lại kết quả
và quyết định thiết kế, không chép lại toàn bộ QA.

**Kết quả dataset**

| Hạng mục | Kết quả |
|---|---|
| Tổng số records | ____ / 20 |
| Easy | ____ / 5 |
| Medium | ____ / 7 |
| Hard | ____ / 5 |
| Adversarial | ____ / 3 |
| Source documents được sử dụng | ____ / 10 |
| Validator status | PASS / FAIL |

**Ba case đại diện cho quyết định thiết kế**

| ID | Difficulty | Source document(s) | Vì sao case phù hợp với difficulty/attack type? |
|---|---|---|---|
| | | | |
| | | | |
| | | | |

**Điểm khó nhất khi xây dựng expected answer hoặc evidence là gì?**

> *Câu trả lời:*

**Xác nhận:**

- [ ] Mọi claim trong expected answer đều có evidence hỗ trợ.
- [ ] Không có questions trùng ý và không dùng kiến thức ngoài corpus.
- [ ] `python validate_golden_dataset.py` báo `PASS`.

### Exercise 3.2 — Benchmark Run

Chạy:

```bash
python domain_assistant.py
python evaluate_answers.py
```

Copy bảng terminal vào đây hoặc điền từ `artifacts/benchmark_results.json`.

| ID | Question (short) | Ctx Recall | Ctx Precision | Faithfulness | Relevance | Completeness | Overall | Passed? | Failure Type |
|---|---|---:|---:|---:|---:|---:|---:|---|---|
| E01 | | | | | | | | | |
| E02 | | | | | | | | | |
| E03 | | | | | | | | | |
| E04 | | | | | | | | | |
| E05 | | | | | | | | | |
| M01 | | | | | | | | | |
| M02 | | | | | | | | | |
| M03 | | | | | | | | | |
| M04 | | | | | | | | | |
| M05 | | | | | | | | | |
| M06 | | | | | | | | | |
| M07 | | | | | | | | | |
| H01 | | | | | | | | | |
| H02 | | | | | | | | | |
| H03 | | | | | | | | | |
| H04 | | | | | | | | | |
| H05 | | | | | | | | | |
| A01 | | | | | | | | | |
| A02 | | | | | | | | | |
| A03 | | | | | | | | | |

**Aggregate Report**

- Overall pass rate: ____%
- Avg Context Recall: ____
- Avg Context Precision: ____
- Avg Faithfulness: ____
- Avg Relevance: ____
- Avg Completeness: ____
- Failure type distribution: ____

**Ba cases có Overall Score thấp nhất**

1. ID: ____ | Score: ____ | Failure type: ____
2. ID: ____ | Score: ____ | Failure type: ____
3. ID: ____ | Score: ____ | Failure type: ____

**Nhận xét ngắn:** Metric nào yếu nhất? Kết quả gợi ý vấn đề nằm ở retrieval
hay generation?

> *Câu trả lời:*

### Exercise 3.3 — LLM-as-a-Judge Rubric Design

Thiết kế rubric domain-specific cho OrbitTech Customer Support. Mỗi mức phải
đủ cụ thể để hai người chấm độc lập có thể hiểu giống nhau.

Chọn 3–5 dimensions:

- [ ] Correctness
- [ ] Completeness
- [ ] Relevance
- [ ] Evidence/citation
- [ ] Actionability
- [ ] Safety/privacy
- [ ] Tone/clarity
- [ ] Dimension khác: __________

| Score | Tiêu chí domain-specific | Ví dụ response |
|---:|---|---|
| 5 | | |
| 4 | | |
| 3 | | |
| 2 | | |
| 1 | | |

**Ba edge cases khó chấm**

| Edge Case | Tại sao khó chấm? | Rubric xử lý thế nào? |
|---|---|---|
| | | |
| | | |
| | | |

**Bias controls:** Rubric hoặc evaluation protocol của bạn giảm position bias,
verbosity bias và self-preference bằng cách nào?

> *Câu trả lời:*

### Exercise 3.4 — Framework Comparison (Bonus +5)

Chỉ làm sau khi hoàn thành 3.1–3.3. Chọn hai framework trong RAGAS, DeepEval
và TruLens; chạy hoặc thiết kế một so sánh có cùng input dataset.

| Tiêu chí | Framework 1: ____ | Framework 2: ____ |
|---|---|---|
| Setup complexity | | |
| Metrics available | | |
| CI/CD integration | | |
| Kết quả trên cùng dataset | | |
| Insight rút ra | | |

- Scores có nhất quán không?
- Framework nào strict hơn và vì sao?
- Hai framework có tìm ra cùng failure cases không?

> *Phân tích:*

### Exercise 3.5 — Retrieval Reranking (Bonus +5)

Mục tiêu: kiểm tra việc đổi thứ tự chunks có tăng Context Precision mà không
thay đổi Context Recall hay không.

1. Chọn ít nhất 5 cases từ `artifacts/actual_answers.json`.
2. Tính Context Recall và Context Precision trước rerank.
3. Implement `rerank_by_overlap()` hoặc một reranker khác.
4. Rerank cùng tập chunks, không thêm hoặc xóa chunk.
5. Tính lại hai metrics và giải thích kết quả.

| ID | Recall before | Recall after | Precision before | Precision after | Delta Precision |
|---|---:|---:|---:|---:|---:|
| | | | | | |
| | | | | | |
| | | | | | |
| | | | | | |
| | | | | | |
| **Avg** | | | | | |

**Tại sao Recall dự kiến không đổi?**

> *Câu trả lời:*

**Khi nào reranking không đủ và cần sửa retriever/query/chunking?**

> *Câu trả lời:*

---

## Part 4 — Reflection (11:35–11:50)

Hoàn thành `reflection.md` bằng kết quả thật từ Exercise 3.2.

---

## Completion Checklist

Hoàn thành kiểm tra cuối trong khoảng 11:50–12:00.

- [ ] Tất cả required tests pass.
- [ ] `golden_dataset.json` validate thành công.
- [ ] Exercise 3.1 hoàn thành trong file JSON và bảng kết quả phía trên.
- [ ] Exercise 3.2 có năm metrics, aggregate report và ba cases thấp nhất.
- [ ] Exercise 3.3 có rubric 1–5 và bias controls.
- [ ] `reflection.md` có ba failure analyses và regression strategy.
- [ ] Đã copy `template.py` thành `solution/solution.py`.
- [ ] Exercise 3.4 và 3.5 chỉ làm nếu chọn bonus.
