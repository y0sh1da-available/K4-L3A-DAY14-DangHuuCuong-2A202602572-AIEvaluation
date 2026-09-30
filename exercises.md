# Day 14 — Exercises

## AI Evaluation & Benchmarking · Lab Worksheet

**Thời gian làm bài:** 14:15–17:00

**Domain:** OrbitTech Store Customer Support

Điền trực tiếp câu trả lời vào file này. Golden dataset 20 QA được viết một lần
duy nhất trong `golden_dataset.json`, không chép lại toàn bộ vào Markdown.

---

Từ 14:15–14:30, cài môi trường và chạy baseline tests theo `guide_lab.md`.

---

## Part 1 — Warm-up (14:30–14:45)

### Exercise 1.1 — RAGAS Metric Thresholds

Theo bài giảng:

- 0.8–1.0: Good — monitor, maintain.
- 0.6–0.8: Needs work — analyze failures, iterate.
- Dưới 0.6: Significant issues — investigate.

Với từng metric, xác định khi nào score thấp có thể chấp nhận và khi nào là
critical.

| Metric | Acceptable Low Score Scenario | Critical Low Score Scenario | Action Required |
|---|---|---|---|
| Faithfulness | Câu hỏi xã giao, chào hỏi hoặc điều hướng chuyển tiếp (escalation/out-of-scope) khi context không chứa trực tiếp nội dung nhưng agent hướng dẫn đúng quy trình mà không bịa facts. | Agent tự bịa đặt chính sách (hallucination) về thời hạn hoàn tiền, điều kiện đổi trả, hoặc chi phí bảo hành trái ngược với tài liệu OrbitTech. | Tinh chỉnh prompt (instruction groundedness, hạ temperature = 0), thêm strict guardrail cấm suy diễn ngoài context. |
| Answer Relevance | Khách hàng hỏi câu hỏi mơ hồ hoặc cố tình jailbreak/out-of-scope; assistant lịch sự từ chối hoặc hỏi lại để làm rõ (clarification/refusal). | Khách hàng hỏi câu hỏi cụ thể (ví dụ: cách hủy đơn hàng) nhưng assistant trả lời sang quy định giao hàng hoặc sản phẩm khác (off-topic). | Tối ưu hóa user intent classification, prompt guidance tập trung vào trọng tâm câu hỏi, query rewriting. |
| Context Recall | Câu hỏi tra cứu factual lookup đơn giản (chỉ cần 1 chunk duy nhất đủ trả lời), các chunk liên quan khác trong corpus không được lấy thêm. | Câu hỏi phức tạp có điều kiện kết hợp (multi-doc: đơn hàng khuyến mãi + đổi trả hàng mở hộp), retriever bỏ sót văn bản chứa điều kiện ngoại lệ. | Tăng Top-K chunks, tối ưu hóa chunk size & overlap, kết hợp hybrid search (BM25 + Dense vector embeddings), query expansion. |
| Context Precision | Retriever lấy Top-K rộng (ví dụ K=5 hoặc K=10) chứa nhiều chunk tham khảo phụ, nhưng LLM vẫn đủ khả năng đọc hiểu và trích xuất đúng ý. | Các chunk quan trọng nhất chứa câu trả lời bị xếp ở vị trí cuối (rank thấp), hoặc chìm giữa các chunk nhiễu khiến LLM bị phân tâm ("lost in the middle"). | Tích hợp Reranker (như cross-encoder reranker) đưa chunk liên quan nhất lên đầu; tối ưu hóa trọng số từ khóa BM25/embedding similarity. |
| Completeness | Người dùng chỉ hỏi một khía cạnh cụ thể trong quy trình tổng thể, câu trả lời chỉ tập trung vào ý đó mà không nhắc lại toàn bộ chính sách dài dòng. | Expected answer yêu cầu đầy đủ 3 điều kiện tiên quyết và thời hạn cụ thể (ví dụ: 14 ngày, còn nguyên hộp, kèm hóa đơn), assistant chỉ nêu 1 điều kiện làm khách hàng bị từ chối quyền lợi. | Bổ sung few-shot examples trong prompt về tính toàn vẹn thông tin, yêu cầu liệt kê checklist điều kiện theo gạch đầu dòng. |

### Exercise 1.2 — Bias trong LLM-as-a-Judge

Ba bias thường gặp:

- Position bias: judge ưu tiên answer xuất hiện trước.
- Verbosity bias: judge ưu tiên answer dài hơn.
- Self-preference: judge ưu tiên output giống chính model đó.

**Câu 1: Thiết kế experiment phát hiện position bias với ít nhất hai conditions.**

> *Câu trả lời:*
> - **Condition 1 (Original Order):** Đưa cặp câu trả lời vào prompt đánh giá theo thứ tự Candidate A đứng trước, Candidate B đứng sau. Ghi nhận lựa chọn của LLM Judge.
> - **Condition 2 (Swapped Order):** Đảo ngược vị trí hiển thị: Candidate B đứng trước, Candidate A đứng sau, giữ nguyên toàn bộ nội dung prompt và rubric. Ghi nhận lựa chọn của LLM Judge.
> - **Đo lường & Phân tích:** So sánh tỷ lệ lựa chọn vị trí đầu tiên (First-position win rate) qua nhiều test cases. Nếu tỷ lệ chọn candidate đứng trước vượt quá đáng kể 50% (ví dụ > 65%) ở cả hai lượt đảo, mô hình bị **Position Bias**. Giải pháp khắc phục là áp dụng *bidirectional evaluation* (chạy cả 2 chiều và lấy trung bình hoặc chỉ chấp nhận kết quả khi nhất quán).

**Câu 2: Làm thế nào giảm verbosity bias bằng rubric design?**

> *Câu trả lời:*
> - Phân tách rõ ràng giữa **Completeness** (độ bao phủ ý chính cần thiết) và **Conciseness/Efficiency** (tính súc tích, ngắn gọn).
> - Thiết lập tiêu chí checklist dựa trên facts/key points: Chấm điểm dựa trên việc câu trả lời có chứa đủ các thông tin cốt lõi hay không, thay vì đếm số lượng từ hoặc độ dài văn bản.
> - Bổ sung quy định phạt điểm trực tiếp trong rubric: Nếu câu trả lời dài dòng, lặp lại thông tin hoặc chứa các đoạn giải thích lan man không phục vụ mục đích câu hỏi thì bị trừ từ 1 đến 2 điểm.
> - Đưa vào few-shot examples mẫu minh họa: Một câu trả lời ngắn gọn, chính xác vẫn đạt điểm tối đa (5/5), trong khi một câu trả lời dài nhưng rỗng ý chỉ nhận điểm trung bình (2–3/5).

**Câu 3: Tại sao cần calibrate LLM judge với human labels?**

> *Câu trả lời:*
> - LLM Judge là một mô hình ngôn ngữ xác suất, dễ gặp các thiên kiến nội tại (như xu hướng chấm điểm quá dễ dãi - *leniency bias* hoặc quá khắt khe - *severity bias*), không đồng nhất với kỳ vọng thực tế của doanh nghiệp.
> - Hiệu chỉnh (calibration) với human labels (nhãn do chuyên gia nghiệp vụ dán trên một tập validation nhỏ) giúp:
>   1. Tính toán hệ số đồng thuận giữa AI và con người (Cohen's Kappa, Spearman correlation).
>   2. Cân chỉnh lại thang điểm và ngưỡng phân loại (threshold tuning), đảm bảo điểm số của LLM Judge phản ánh trung thực chất lượng thực tế.
>   3. Phát hiện sớm các lỗ hổng hoặc sự mơ hồ trong rubric để kịp thời tinh chỉnh hướng dẫn chấm điểm.

### Exercise 1.3 — Evaluation trong CI/CD

**Câu 1: Chọn threshold để block deployment.**

| Metric | Threshold | Lý do |
|---|---:|---|
| Faithfulness | 0.85 | Trong domain hỗ trợ khách hàng, hallucination về chính sách hoàn tiền, bảo hành hay chi phí là rủi ro nghiêm trọng nhất, có thể dẫn đến khiếu nại pháp lý hoặc thiệt hại tài chính. |
| Answer Relevance | 0.80 | Câu trả lời bắt buộc phải giải quyết đúng trọng tâm câu hỏi của khách hàng; trả lời lạc đề sẽ làm tăng tỷ lệ bỏ cuộc hoặc quá tải kênh human support. |
| Completeness | 0.75 | Đảm bảo cung cấp đủ các điều kiện tiên quyết và các bước hướng dẫn quan trọng; chấp nhận dung sai nhỏ đối với các thông tin bổ trợ ngoài lề. |

**Câu 2: Khi nào dùng offline evaluation, online evaluation và human review?**

> *Câu trả lời:*
> - **Offline Evaluation (Pre-deployment):** Chạy tự động trong CI/CD pipeline trước khi release model, prompt template hoặc retrieval index mới. Sử dụng Golden Dataset chuẩn hóa để đo lường benchmark và kiểm tra hồi quy (regression gate), đảm bảo phiên bản mới không làm tụt giảm chất lượng trước khi ra mắt người dùng.
> - **Online Evaluation (Production monitoring):** Chạy liên tục theo thời gian thực hoặc lấy mẫu (sampling) trên luồng dữ liệu tương tác thật của người dùng. Dùng để giám sát data drift, latency, tỷ lệ escalation và đánh giá chất lượng phản hồi trong môi trường production thông qua telemetry và LLM evaluators ngầm.
> - **Human Review (Periodic / High-risk Auditing):** Thực hiện định kỳ bởi đội ngũ chuyên gia (SMEs/QA) trên các mẫu ngẫu nhiên hoặc các ca có rủi ro cao (khiếu nại, phản hồi tiêu cực, điểm evaluator thấp). Dùng để audit chất lượng của LLM Judge, cập nhật Golden Dataset và phát hiện các trường hợp edge case mới mà hệ thống tự động bỏ sót.

---

## Part 2 — Core Coding (14:45–15:40)

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

## Part 3 — Golden Dataset & Real Benchmark (15:40–16:35)

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

## Part 4 — Reflection (16:35–16:50)

Hoàn thành `reflection.md` bằng kết quả thật từ Exercise 3.2.

---

## Completion Checklist

Hoàn thành kiểm tra cuối trong khoảng 16:50–17:00.

- [ ] Tất cả required tests pass.
- [ ] `golden_dataset.json` validate thành công.
- [ ] Exercise 3.1 hoàn thành trong file JSON và bảng kết quả phía trên.
- [ ] Exercise 3.2 có năm metrics, aggregate report và ba cases thấp nhất.
- [ ] Exercise 3.3 có rubric 1–5 và bias controls.
- [ ] `reflection.md` có ba failure analyses và regression strategy.
- [ ] Đã copy `template.py` thành `solution/solution.py`.
- [ ] Exercise 3.4 và 3.5 chỉ làm nếu chọn bonus.
