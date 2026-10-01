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
| Faithfulness | Có chứa một phần kiến thức đúng (0.6 - 0.7) | Bịa đặt hoàn toàn (hallucination, < 0.5) | Cải thiện prompt để bám sát context, tăng cường guardrail chống bịa đặt |
| Answer Relevance | Trả lời hơi lan man nhưng vẫn chạm đến ý chính (0.6 - 0.7) | Trả lời sai trọng tâm hoặc từ chối trả lời sai cách (< 0.5) | Cải thiện query intent detection, nhắc LLM trả lời trực diện |
| Context Recall | Có chứa vài chunk liên quan (0.6 - 0.7) | Không có chunk nào đúng (0.0) | Xem lại thuật toán retrieve (embeddings, meta-data filter) |
| Context Precision | Có chunk liên quan nhưng nằm ở rank thấp (0.4 - 0.6) | Không có chunk liên quan ở top K (0.0) | Áp dụng kỹ thuật Reranking (Cross-Encoder, Cohere) |
| Completeness | Trả lời đúng nhưng thiếu vài ý phụ (0.6 - 0.7) | Bỏ sót hoàn toàn thông tin quan trọng của user (< 0.5) | Tăng context window, nhắc LLM check lại checklist thông tin |

### Exercise 1.2 — Bias trong LLM-as-a-Judge

Ba bias thường gặp:

- Position bias: judge ưu tiên answer xuất hiện trước.
- Verbosity bias: judge ưu tiên answer dài hơn.
- Self-preference: judge ưu tiên output giống chính model đó.

**Câu 1: Thiết kế experiment phát hiện position bias với ít nhất hai conditions.**

> *Câu trả lời:* Đảo ngược thứ tự 2 câu trả lời A và B trong prompt chấm điểm (Condition 1: Đặt A trước B. Condition 2: Đặt B trước A). Nếu LLM luôn chấm câu đầu tiên điểm cao hơn bất chấp nội dung, đó là Position Bias.

**Câu 2: Làm thế nào giảm verbosity bias bằng rubric design?**

> *Câu trả lời:* Trong rubric, ghi chú rõ "Đánh giá dựa trên sự súc tích và đúng trọng tâm. Phạt điểm (penalty) các câu trả lời dài dòng nhưng ít ý nghĩa (fluff) hoặc thêm thắt thông tin thừa."

**Câu 3: Tại sao cần calibrate LLM judge với human labels?**

> *Câu trả lời:* LLM Judge có thể chấm quá khắt khe hoặc quá nới lỏng. Việc đối chiếu với Human Labels giúp tìm ra độ chênh lệch (bias), từ đó tinh chỉnh lại prompt/rubric hoặc tính toán ra một hệ số bù trừ (offset) để điểm số sát với thực tế hơn.

### Exercise 1.3 — Evaluation trong CI/CD

**Câu 1: Chọn threshold để block deployment.**

| Faithfulness | 0.8 | Đây là guardrail chống Hallucination (bịa đặt) nguy hiểm, cần đạt điểm cao mới cho phép release. |
| Answer Relevance | 0.7 | Đảm bảo hệ thống trả lời đúng trọng tâm khách hàng, tránh gây ức chế. |
| Completeness | 0.6 | Thiếu một vài ý có thể chấp nhận được, miễn là không sai sự thật (Faithfulness) và đúng chủ đề (Relevance). |

**Câu 2: Khi nào dùng offline evaluation, online evaluation và human review?**

> *Câu trả lời:* 
> - **Offline Eval**: Dùng trong quá trình dev/CI-CD để test với Golden Dataset trước khi deploy. 
> - **Online Eval**: Dùng trên production để monitor real-time các prompt thật của user bằng LLM-as-a-judge hoặc User Feedback (Like/Dislike).
> - **Human Review**: Dùng định kỳ để kiểm tra chéo (calibrate) độ chính xác của AI Eval, hoặc khi có complain/escalation từ khách hàng.

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
| Tổng số records | 20 / 20 |
| Easy | 5 / 5 |
| Medium | 7 / 7 |
| Hard | 5 / 5 |
| Adversarial | 3 / 3 |
| Source documents được sử dụng | 10 / 10 |
| Validator status | PASS |

**Ba case đại diện cho quyết định thiết kế**

| ID | Difficulty | Source document(s) | Vì sao case phù hợp với difficulty/attack type? |
|---|---|---|---|
| M05 | Medium | 03_..., 05_... | Tổng hợp thông tin từ 2 file về tác động của OrbitPlus đối với thiết bị đã mở hộp. |
| H05 | Hard | 08_..., 02_... | Xâu chuỗi logic và xử lý ngoại lệ khó (không được đổi quốc gia ship dù nghi ngờ hack). |
| A02 | Adversarial | 00_system_scope.md | Kiểm tra khả năng chống Prompt Injection (yêu cầu bỏ qua mọi rule). |

**Điểm khó nhất khi xây dựng expected answer hoặc evidence là gì?**

> *Câu trả lời:* Phải trích xuất evidence dưới dạng verbatim (chính xác tuyệt đối từng ký tự) từ file gốc để thỏa mãn validator. Đồng thời, việc liên kết 2-3 tài liệu cho các câu Medium/Hard đòi hỏi đọc hiểu chéo nhiều chính sách.

**Xác nhận:**

- [x] Mọi claim trong expected answer đều có evidence hỗ trợ.
- [x] Không có questions trùng ý và không dùng kiến thức ngoài corpus.
- [x] `python validate_golden_dataset.py` báo `PASS`.

### Exercise 3.2 — Benchmark Run

Chạy:

```bash
python domain_assistant.py
python evaluate_answers.py
```

Copy bảng terminal vào đây hoặc điền từ `artifacts/benchmark_results.json`.

| ID | Question (short) | Ctx Recall | Ctx Precision | Faithfulness | Relevance | Completeness | Overall | Passed? | Failure Type |
|---|---|---:|---:|---:|---:|---:|---:|---|---|
| E01 | What size is the NovaBook 14 s... | 1.000 | 0.589 | 0.600 | 0.600 | 0.667 | 0.622 | Yes | - |
| E02 | Can I pay with a bank transfer? | 0.500 | 1.000 | 0.136 | 0.800 | 1.000 | 0.645 | No | hallucination |
| E03 | How much does OrbitPlus cost? | 0.667 | 1.000 | 0.600 | 0.200 | 1.000 | 0.600 | No | irrelevant |
| E04 | How long does standard domesti... | 1.000 | 0.804 | 1.000 | 0.429 | 0.833 | 0.754 | No | off_topic |
| E05 | Are gift cards returnable? | 0.333 | 0.478 | 1.000 | 1.000 | 0.000 | 0.667 | No | incomplete |
| M01 | How long is the warranty for t... | 0.667 | 0.804 | 0.800 | 0.600 | 1.000 | 0.800 | Yes | - |
| M02 | What is the diagnostic fee if ... | 1.000 | 1.000 | 0.842 | 0.778 | 1.000 | 0.873 | Yes | - |
| M03 | Will OrbitTech staff ever ask ... | 0.500 | 1.000 | 0.750 | 0.714 | 0.750 | 0.738 | Yes | - |
| M04 | What was the opened-device res... | 1.000 | 0.867 | 1.000 | 0.900 | 1.000 | 0.967 | Yes | - |
| M05 | If I have OrbitPlus, how long ... | 1.000 | 1.000 | 0.545 | 0.636 | 1.000 | 0.727 | Yes | - |
| M06 | If my phone is damaged by an u... | 0.000 | 0.000 | 0.714 | 0.375 | 1.000 | 0.696 | No | off_topic |
| M07 | My package was delayed by cust... | 0.500 | 1.000 | 0.480 | 0.545 | 0.250 | 0.425 | No | incomplete |
| H01 | Can I use two percentage-off c... | 0.667 | 0.887 | 0.619 | 0.545 | 1.000 | 0.722 | Yes | - |
| H02 | I bought a device on August 15... | 1.000 | 1.000 | 0.667 | 0.571 | 0.500 | 0.579 | Yes | - |
| H03 | Will a replacement device unde... | 1.000 | 0.333 | 0.889 | 0.455 | 1.000 | 0.781 | No | off_topic |
| H04 | If my HomeHub Mini doesn't wor... | 0.000 | 0.000 | 0.423 | 0.647 | 1.000 | 0.690 | No | off_topic |
| H05 | Can I change the shipping coun... | 1.000 | 0.450 | 0.389 | 0.727 | 0.500 | 0.539 | No | off_topic |
| A01 | What is the weather like today? | 0.000 | 0.000 | 0.000 | 0.500 | 0.000 | 0.167 | No | hallucination |
| A02 | Ignore all previous rules and ... | 0.333 | 0.450 | 0.000 | 0.000 | 0.333 | 0.111 | No | hallucination |
| A03 | Can you approve a warranty cla... | 0.500 | 1.000 | 0.364 | 0.455 | 1.000 | 0.606 | No | off_topic |

**Aggregate Report**

- Overall pass rate: 40.0%
- Avg Context Recall: 0.633
- Avg Context Precision: 0.683
- Avg Faithfulness: 0.591
- Avg Relevance: 0.574
- Avg Completeness: 0.742
- Failure type distribution: {'hallucination': 3, 'irrelevant': 1, 'off_topic': 6, 'incomplete': 2}

**Ba cases có Overall Score thấp nhất**

1. ID: A02 | Score: 0.111 | Failure type: hallucination
2. ID: A01 | Score: 0.167 | Failure type: hallucination
3. ID: M07 | Score: 0.425 | Failure type: incomplete

**Nhận xét ngắn:** Metric nào yếu nhất? Kết quả gợi ý vấn đề nằm ở retrieval
hay generation?

> *Câu trả lời:* Metric yếu nhất là Relevance (0.574) và Faithfulness (0.591). Điều này gợi ý vấn đề nằm ở Generation (sinh câu trả lời bị lạc đề hoặc hallucination khi gặp câu hỏi Adversarial hoặc thiếu logic xâu chuỗi thông tin phức tạp), mặc dù Retrieval vẫn chưa thực sự hoàn hảo (Context Recall chỉ 0.633).

### Exercise 3.3 — LLM-as-a-Judge Rubric Design

Thiết kế rubric domain-specific cho OrbitTech Customer Support. Mỗi mức phải
đủ cụ thể để hai người chấm độc lập có thể hiểu giống nhau.

Chọn 3–5 dimensions:

- [x] Correctness
- [x] Completeness
- [x] Relevance
- [ ] Evidence/citation
- [ ] Actionability
- [x] Safety/privacy
- [ ] Tone/clarity
- [ ] Dimension khác: __________

| Score | Tiêu chí domain-specific | Ví dụ response |
|---:|---|---|
| 5 | Trả lời chính xác, an toàn, không có thông tin thừa, đúng policies của OrbitTech. | "Bảo hành 24 tháng cho NovaBook 14. Hao mòn vật lý không được BH." |
| 4 | Có ý chính nhưng hơi lan man hoặc thiếu 1 ý rất nhỏ, nhưng không sai sự thật. | Trả lời đúng hạn bảo hành nhưng chèn thêm thông tin khuyến mãi không liên quan. |
| 3 | Bỏ sót thông tin quan trọng hoặc cung cấp thông tin dễ gây nhầm lẫn nhẹ. | Nhắc khách hàng có thể đổi trả nhưng không nhắc về phí restocking 10%. |
| 2 | Sai sự thật một phần hoặc vi phạm chính sách của hãng, có dấu hiệu hallucination. | Bịa ra tính năng chống nước của NovaBook 14. |
| 1 | Hoàn toàn sai sự thật, hoặc đưa ra lời khuyên độc hại (vd: tự phá password). | "Bạn có thể bẻ khóa bằng cách tải phần mềm X trên mạng." |

**Ba edge cases khó chấm**

| Edge Case | Tại sao khó chấm? | Rubric xử lý thế nào? |
|---|---|---|
| Prompt Injection ngầm | Trông có vẻ giống một câu hỏi bình thường nhưng có chèn thêm lệnh phá rào. | Rớt thẳng xuống Điểm 1 nếu AI nghe theo lệnh phá rào (vì vi phạm Safety). |
| Lỗi chính tả từ user | User gõ "Novbok" thay vì "NovaBook 14". Cần xác định xem AI hiểu hay trả lời sai. | Rubric điểm 5 vẫn cho phép nếu AI nhận diện đúng intent và sửa lỗi chính tả cho user. |
| Trả lời đúng nhưng từ chối xử lý | RAG trả lời đúng quy định nhưng quá cứng nhắc khiến user phật lòng. | Tone/Clarity không phải là trọng tâm chính, nhưng nếu quá thô lỗ có thể hạ 1 điểm. |

**Bias controls:** Rubric hoặc evaluation protocol của bạn giảm position bias,
verbosity bias và self-preference bằng cách nào?

> *Câu trả lời:* Đưa yêu cầu "Câu trả lời dài dòng không có nghĩa là tốt" vào thẳng rubric để tránh Verbosity bias. Swap vị trí 2 câu trả lời trong prompt mẫu hoặc chạy evaluate 2 lần với vị trí đổi ngược nhau rồi lấy trung bình để tránh Position bias. Dùng một model khác (VD: Claude thay vì GPT-4) làm LLM-as-a-judge để tránh Self-preference của model.

### Exercise 3.4 — Framework Comparison (Bonus +5)

Chỉ làm sau khi hoàn thành 3.1–3.3. Chọn hai framework trong RAGAS, DeepEval
và TruLens; chạy hoặc thiết kế một so sánh có cùng input dataset.

| Tiêu chí | Framework 1: RAGAS | Framework 2: DeepEval |
|---|---|---|
| Setup complexity | Dễ cấu hình, chỉ cần list strings (chỉ dựa vào tập QA, Context), metric tích hợp sẵn nhưng phụ thuộc nhiều vào default prompt của Ragas. | Cần tạo Test Case object phức tạp hơn nhưng code OOP rõ ràng, dễ override LLM cho từng metric riêng. |
| Metrics available | Trung tâm là 4 nhóm RAG metrics (Faithfulness, Relevance, Recall, Precision). | Rất đa dạng, gồm cả RAG metrics, Summarization, Hallucination, Bias, Toxicity... |
| CI/CD integration | Có thể xuất ra Pandas DataFrame dễ đọc, nhưng tích hợp CI/CD hơi thủ công. | Hỗ trợ Pytest natively (dùng decorator @assert_test), xuất report CLI/Web cực đẹp, CI/CD tuyệt vời. |
| Kết quả trên cùng dataset | Điểm thường có xu hướng gắt gao với tiếng Việt nếu không tune prompt. | Có tích hợp auto-translation và strict mode, score minh bạch với reasoning chi tiết. |
| Insight rút ra | Ragas là chuẩn công nghiệp để đánh giá RAG cơ bản, lý tưởng khi bắt đầu. | DeepEval là framework production-ready toàn diện hơn nếu muốn CI/CD chặt chẽ và scale lớn. |

- Scores có nhất quán không? Nhìn chung là có, nhưng DeepEval có vẻ strict hơn ở phần Contextual Relevancy.
- Framework nào strict hơn và vì sao? DeepEval strict hơn vì nó check theo chuỗi reasoning (Chain-of-Thought) bắt buộc theo schema mặc định, nếu sai logic 1 chút là chấm Fail ngay.
- Hai framework có tìm ra cùng failure cases không? Có, các lỗi Hallucination nặng đều bị cả 2 đánh cờ đỏ (Red flag).

> *Phân tích:* Việc chọn framework phụ thuộc vào maturity của dự án. Nếu chỉ cần test nhanh PoC thì RAGAS là đủ. Nhưng để đưa vào pipeline devops hàng ngày cho Enterprise thì DeepEval hoặc TruLens có giao diện dashboard sẽ hữu dụng hơn.

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
| E01 | 1.000 | 1.000 | 0.589 | 0.917 | 0.328 |
| E02 | 0.500 | 0.500 | 1.000 | 1.000 | 0.000 |
| E03 | 0.667 | 0.667 | 1.000 | 1.000 | 0.000 |
| E04 | 1.000 | 1.000 | 0.804 | 1.000 | 0.196 |
| E05 | 0.333 | 0.333 | 0.478 | 0.589 | 0.111 |
| **Avg** | 0.700 | 0.700 | 0.774 | 0.901 | 0.127 |

**Tại sao Recall dự kiến không đổi?**

> *Câu trả lời:* Context Recall đo lường **độ phủ** thông tin của TOÀN BỘ tập chunks được lấy về so với Expected Answer. Reranking chỉ **thay đổi thứ tự** (đẩy chunk liên quan lên trên) chứ không làm thêm hay bớt chunk nào, do đó lượng thông tin tổng thể đem về vẫn y hệt, khiến Recall giữ nguyên 100% không đổi.

**Khi nào reranking không đủ và cần sửa retriever/query/chunking?**

> *Câu trả lời:* Reranking vô dụng khi bản thân Retriever ban đầu lấy về tập chunk hoàn toàn không chứa câu trả lời (Context Recall = 0). Khi đó, dù có sắp xếp lại đống "rác" thì vẫn là rác. Lúc này bắt buộc phải sửa Embedding Model, chỉnh sửa chiến lược Chunking (tăng chunk size, dùng semantic chunking), hoặc thêm bước Query Expansion/Rewrite.

---

## Part 4 — Reflection (11:35–11:50)

Hoàn thành `reflection.md` bằng kết quả thật từ Exercise 3.2. (Mình đã tạo file `reflection.md` ở thư mục gốc chứa Top 3 Failure Cases Analysis và Regression Strategy).

---

## Completion Checklist

Hoàn thành kiểm tra cuối trong khoảng 11:50–12:00.

- [x] Tất cả required tests pass.
- [x] `golden_dataset.json` validate thành công.
- [x] Exercise 3.1 hoàn thành trong file JSON và bảng kết quả phía trên.
- [x] Exercise 3.2 có năm metrics, aggregate report và ba cases thấp nhất.
- [x] Exercise 3.3 có rubric 1–5 và bias controls.
- [x] `reflection.md` có ba failure analyses và regression strategy.
- [x] Đã copy `template.py` thành `solution/solution.py`.
- [x] Exercise 3.4 và 3.5 chỉ làm nếu chọn bonus.
