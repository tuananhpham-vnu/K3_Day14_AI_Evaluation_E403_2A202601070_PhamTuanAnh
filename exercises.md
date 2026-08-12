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
| Faithfulness | Answer diễn đạt lại context bằng từ khác nhưng không đổi ý | Answer thêm claim không có trong context (hallucination) | Chặn release, kiểm tra prompt grounding |
| Answer Relevance | Answer thêm caveat an toàn ngoài câu hỏi nhưng vẫn đúng chủ đề | Answer lạc đề, không giải quyết câu hỏi | Kiểm tra intent detection và routing |
| Context Recall | Câu adversarial/out-of-scope không cần gold context | Retriever bỏ sót evidence bắt buộc để trả lời đúng | Tinh chỉnh retriever hoặc query rewriting |
| Context Precision | Có vài chunk nhiễu do top-k lớn hơn cần thiết | Chunk đúng bị xếp cuối hoặc không xuất hiện | Cải thiện ranking hoặc reranker |
| Completeness | Thiếu chi tiết phụ không ảnh hưởng kết luận | Thiếu điều kiện/exception làm thay đổi hành động của khách | Rà lại prompt generation và context đưa vào |

### Exercise 1.2 — Bias trong LLM-as-a-Judge

Ba bias thường gặp:

- Position bias: judge ưu tiên answer xuất hiện trước.
- Verbosity bias: judge ưu tiên answer dài hơn.
- Self-preference: judge ưu tiên output giống chính model đó.

**Câu 1: Thiết kế experiment phát hiện position bias với ít nhất hai conditions.**

> Lấy cùng một cặp answer A và B cho cùng câu hỏi, cho judge chấm hai lần: lần một theo thứ tự A trước B, lần hai đảo thành B trước A. Nếu judge đổi lựa chọn theo vị trí thay vì theo nội dung, tức là chọn answer đứng trước ở cả hai lần dù nội dung không đổi, thì kết luận có position bias.

**Câu 2: Làm thế nào giảm verbosity bias bằng rubric design?**

> Rubric cần nêu rõ điểm cao dựa trên độ chính xác và đủ ý cần thiết, không dựa trên độ dài. Nên kèm ví dụ một câu trả lời ngắn đạt điểm 5, để judge không mặc định câu dài hơn là câu tốt hơn.

**Câu 3: Tại sao cần calibrate LLM judge với human labels?**

> Judge có thể mang bias hệ thống hoặc bỏ sót sắc thái riêng của domain mà chỉ con người mới nhận ra. So sánh điểm judge với nhãn người giúp phát hiện độ lệch này và điều chỉnh rubric hoặc threshold trước khi tin dùng judge cho đánh giá tự động.

### Exercise 1.3 — Evaluation trong CI/CD

**Câu 1: Chọn threshold để block deployment.**

| Metric | Threshold | Lý do |
|---|---:|---|
| Faithfulness | 0.8 | Hallucination gây rủi ro thông tin sai trực tiếp cho khách hàng, không thể chấp nhận thấp hơn |
| Answer Relevance | 0.7 | Lạc đề làm giảm trải nghiệm nhưng ít nguy hiểm hơn hallucination |
| Completeness | 0.7 | Thiếu điều kiện có thể dẫn khách hành động sai, nhưng vẫn chấp nhận một mức sai số nhỏ |

**Câu 2: Khi nào dùng offline evaluation, online evaluation và human review?**

> Offline evaluation dùng mỗi khi thay đổi prompt hoặc model trước khi release, vì so sánh được trên cùng một tập test cố định. Online evaluation dùng sau khi hệ thống đã chạy thật, để phát hiện độ trôi theo traffic thực tế mà tập offline không phủ hết. Human review dành cho case rủi ro cao như an toàn hoặc quyền riêng tư, và để hiệu chỉnh lại LLM judge định kỳ.

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

- [x] Correctness
- [x] Completeness
- [ ] Relevance
- [x] Evidence/citation
- [ ] Actionability
- [x] Safety/privacy
- [ ] Tone/clarity
- [ ] Dimension khác: __________

| Score | Tiêu chí domain-specific | Ví dụ response |
|---:|---|---|
| 5 | Đúng chính sách hiện hành, đủ mọi điều kiện và exception liên quan, mọi claim truy được về context, không tiết lộ thông tin nhạy cảm hay xác nhận yêu cầu vượt quyền hạn | Câu hỏi về hoàn tiền nêu đủ thời hạn, khoản phí không hoàn và điều kiện áp dụng |
| 4 | Đúng chính sách và có evidence hỗ trợ, nhưng bỏ sót một chi tiết phụ không làm đổi kết luận chính | Trả lời đúng hướng xử lý nhưng thiếu ngày hiệu lực chính xác của policy |
| 3 | Đúng hướng nhưng thiếu ít nhất một điều kiện quan trọng có thể ảnh hưởng quyết định của khách, hoặc diễn đạt mơ hồ | Trả lời đúng có thể hoàn tiền nhưng không nêu rõ khoản phí xử lý không được hoàn |
| 2 | Có claim không được context hỗ trợ, hoặc bỏ sót điều kiện cốt lõi làm đổi kết luận, nhưng chưa gây rủi ro an toàn hay riêng tư | Trả lời sai thời hạn bảo hành hoặc bỏ qua điều kiện loại trừ chính |
| 1 | Sai chính sách, bịa thông tin ngoài corpus, hoặc vi phạm ranh giới an toàn/riêng tư | Xác nhận yêu cầu đổi mật khẩu qua chat, hoặc tiết lộ thông tin tài khoản của người khác |

**Ba edge cases khó chấm**

| Edge Case | Tại sao khó chấm? | Rubric xử lý thế nào? |
|---|---|---|
| Câu hỏi false-premise (adversarial) | Answer đúng phải bác bỏ premise sai, không phải trả lời trực tiếp premise đó | Điểm cao chỉ khi answer từ chối xác nhận premise sai trước khi cung cấp thông tin đúng |
| Câu hỏi nằm ở ranh giới hai policy version | Answer có thể đúng theo version cũ nhưng sai theo version đang áp dụng cho case cụ thể | Bắt buộc answer nêu rõ version nào áp dụng và căn cứ vào đâu để chọn version đó |
| Câu hỏi liên quan tài khoản/bảo mật | Answer có thể đúng chính sách nhưng vô tình gợi ý hành động không an toàn | Trừ điểm ngay cả khi phần còn lại chính xác, vì vi phạm an toàn được ưu tiên hơn độ đầy đủ |

**Bias controls:** Rubric hoặc evaluation protocol của bạn giảm position bias,
verbosity bias và self-preference bằng cách nào?

> Mỗi mức điểm đều kèm ví dụ cụ thể để hai người chấm không suy diễn theo cảm tính, giúp giảm ambiguity dẫn đến self-preference. Để giảm position bias, các answer so sánh được chấm độc lập từng cái, không đặt cạnh nhau, và thứ tự chấm được đảo giữa các lần lặp. Để giảm verbosity bias, rubric nêu rõ điểm 5 không yêu cầu câu trả lời dài, và độ dài vượt mức cần thiết so với câu hỏi bị coi là dấu hiệu trừ điểm.

### Exercise 3.4 — Framework Comparison (Bonus +10)

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
