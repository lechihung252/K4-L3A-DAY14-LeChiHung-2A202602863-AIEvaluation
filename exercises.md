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
| Faithfulness | Answer diễn đạt lại đúng ý context nhưng dùng từ khác, hoặc chỉ thêm câu chào / câu điều hướng chung. | Answer đưa ra số tiền, thời hạn, % phí, điều kiện bảo hành/đổi trả **không có** trong retrieved context (ví dụ hứa refund 100% khi policy có restocking fee). Đây là hallucination về chính sách → rủi ro tài chính/pháp lý và khiếu nại. | Đối chiếu từng claim với retrieved chunks; siết system prompt "chỉ trả lời từ context, không có thì nói không có thông tin"; yêu cầu cite source; **block deploy** nếu faithfulness giảm ở các case policy. |
| Answer Relevance | Case out-of-scope mà assistant từ chối đúng và ngắn gọn — câu từ chối ít trùng từ với câu hỏi nên score thấp nhưng behavior đúng. | Hỏi về đổi trả nhưng trả lời về bảo hành; trả lời một policy đúng nhưng không phải cái khách hỏi; bị prompt injection kéo sang chủ đề khác. Khách không nhận được câu trả lời cho vấn đề của mình. | Xem lại intent của câu hỏi và retrieved chunks (retrieval lệch chủ đề hay generation lệch); thêm instruction "trả lời trực tiếp câu hỏi trước"; query rewriting; tách riêng đánh giá refusal cho adversarial cases. |
| Context Recall | Câu out-of-scope / adversarial mà expected answer là từ chối — corpus vốn không có evidence để retrieve; hoặc expected answer dùng wording khác chunk nhưng answer vẫn đúng. | Câu Medium/Hard cần nhiều document (policy version, exception, điều kiện kết hợp) mà retriever bỏ sót document chứa điều kiện quan trọng → LLM không thể trả lời đúng dù generation tốt. | Kiểm tra query BM25 và chunk bị bỏ sót; tăng `top-k`, chunk theo section, query expansion / multi-query, hybrid search (BM25 + dense). Sửa retrieval **trước** khi sửa prompt. |
| Context Precision | Recall đã đủ, chunk relevant vẫn nằm trong top-k và answer vẫn faithful + complete; hoặc câu hỏi rộng cần nhiều chunk nên một số chunk "noise" thực ra là context bổ trợ. | Chunk relevant bị xếp sau nhiều chunk nhiễu (ví dụ chunk warranty đứng trước chunk returns) và đi kèm faithfulness/completeness thấp → LLM lấy nhầm chính sách từ chunk sai. | Thêm reranker (cross-encoder hoặc overlap reranker như Exercise 3.5), giảm `top-k`, lọc theo metadata document; đo lại precision với recall giữ nguyên. |
| Completeness | Expected answer có chi tiết phụ, answer bao đủ ý chính nhưng dùng từ đồng nghĩa; câu từ chối adversarial có wording khác expected nhưng cùng behavior. | Thiếu điều kiện hoặc ngoại lệ bắt buộc: deadline, phí, "không áp dụng cho sản phẩm đã kích hoạt/bị hư do nước", bước xác minh danh tính. Khách hiểu sai policy và hành động sai. | Kiểm tra context recall trước (thiếu evidence hay generation bỏ ý); nếu recall tốt thì sửa prompt yêu cầu nêu đủ conditions/exceptions; thêm các case nhiều điều kiện vào regression set. |

### Exercise 1.2 — Bias trong LLM-as-a-Judge

Ba bias thường gặp:

- Position bias: judge ưu tiên answer xuất hiện trước.
- Verbosity bias: judge ưu tiên answer dài hơn.
- Self-preference: judge ưu tiên output giống chính model đó.

**Câu 1: Thiết kế experiment phát hiện position bias với ít nhất hai conditions.**

> *Câu trả lời:* Lấy khoảng 40 cặp answer (A, B) cho cùng một câu hỏi,
> gồm cặp có chất lượng khác biệt rõ và cặp chất lượng tương đương. Giữ cố định
> judge model, prompt, rubric và `temperature = 0`; chỉ thay đổi thứ tự.
>
> - **Condition 1 — Original order:** A ở vị trí 1, B ở vị trí 2.
> - **Condition 2 — Swapped order:** B ở vị trí 1, A ở vị trí 2.
> - **Condition 3 — Control (identical pair):** A vs A. Judge không có bias thì
>   phải cho tie hoặc chọn mỗi vị trí khoảng 50%.
>
> Đo: (1) **consistency rate** = tỉ lệ cặp mà winner giữ nguyên sau khi swap;
> (2) tỉ lệ judge chọn vị trí 1 trên toàn bộ lần chấm; (3) tỉ lệ chọn vị trí 1 ở
> control. Nếu consistency thấp (ví dụ < 80%) hoặc vị trí 1 thắng lệch rõ khỏi
> 50% (kiểm định binomial/McNemar) thì judge có position bias. Cách giảm: luôn
> chấm cả hai thứ tự, chỉ chấp nhận kết quả khi hai lần nhất quán, còn lại coi là
> tie; hoặc dùng pointwise scoring theo rubric thay vì pairwise.

**Câu 2: Làm thế nào giảm verbosity bias bằng rubric design?**

> *Câu trả lời:*
>
> - Chấm theo **checklist claim cụ thể** thay vì ấn tượng chung: rubric liệt kê
>   các facts bắt buộc (thời hạn, số tiền, điều kiện, ngoại lệ) và score dựa trên
>   số facts đúng, không dựa trên độ dài.
> - Ghi rõ trong rubric: "Độ dài không phải tiêu chí; thông tin thừa không được
>   cộng điểm". Claim không có evidence hoặc lan man ngoài câu hỏi bị **trừ điểm**.
> - Thêm dimension riêng cho conciseness/clarity để câu trả lời dài dòng không
>   được thưởng ở dimension correctness.
> - Few-shot anchor: một ví dụ ngắn đúng đủ = 5, một ví dụ dài nhưng có claim
>   thừa hoặc sai = 2–3.
> - Kiểm tra lại bằng thí nghiệm padding: thêm câu trung tính vào một answer. Nếu
>   score tăng thì rubric vẫn còn verbosity bias.

**Câu 3: Tại sao cần calibrate LLM judge với human labels?**

> *Câu trả lời:* Score của LLM judge chỉ là proxy cho đánh giá của con người.
> Trước khi dùng judge làm quality gate, cần biết nó đồng thuận với expert đến mức
> nào. Nếu không calibrate, judge có thể:
>
> - **Dễ dãi** (leniency: cho 4–5 gần như mọi answer).
> - Bỏ sót lỗi domain, ví dụ sai phí restocking hoặc sai thời hạn bảo hành.
> - Có position, verbosity hoặc self-preference bias.
>
> Cách làm:
>
> 1. Hai người gắn nhãn cùng 50–100 samples theo cùng rubric.
> 2. Đo inter-annotator agreement giữa hai người trước. Agreement thấp thì rubric
>    đang mơ hồ.
> 3. So judge với human bằng Cohen's kappa hoặc Spearman correlation.
> 4. Sửa rubric, few-shot examples hoặc threshold đến khi agreement đủ cao.
> 5. Re-calibrate định kỳ, vì judge model và phân phối câu hỏi thay đổi theo thời
>    gian.

### Exercise 1.3 — Evaluation trong CI/CD

**Câu 1: Chọn threshold để block deployment.**

| Metric | Threshold | Lý do |
|---|---:|---|
| Faithfulness | avg ≥ 0.70 và 0 case `hallucination` ở nhóm policy | Rủi ro cao nhất cho customer support: hứa sai refund/warranty/phí gây thiệt hại tài chính và khiếu nại. Ngưỡng cao hơn pass rule 0.5 của từng case vì đây là gate cho toàn bộ release. |
| Answer Relevance | avg ≥ 0.60 | Word-overlap đánh giá thấp các câu từ chối đúng và paraphrase, nên đặt quá cao sẽ block nhầm. 0.6 là ranh giới "Needs work" theo bài giảng. |
| Completeness | avg ≥ 0.60 | Thiếu điều kiện hoặc ngoại lệ là lỗi nghiêm trọng, nhưng expected answer do người viết có wording khác answer nên heuristic vốn thấp hơn thực tế. Kết hợp thêm rule: **block nếu bất kỳ metric nào giảm > 0.05 so với baseline** (`run_regression`). |

> Các ngưỡng trên là điểm khởi đầu. Sau lần benchmark thật đầu tiên, cần
> calibrate lại theo baseline và human review, tránh threshold làm block nhầm
> quá nhiều.

**Câu 2: Khi nào dùng offline evaluation, online evaluation và human review?**

> *Câu trả lời:*
>
> - **Offline evaluation:** trước khi deploy, chạy trên golden dataset cố định
>   (20 QA + regression set) mỗi khi đổi prompt, model, retriever, chunking,
>   `top-k` hoặc corpus policy. Kết quả tái lập được, so sánh được với baseline
>   và dùng làm **quality gate trong CI/CD**.
> - **Online evaluation:** sau khi deploy (canary/A-B test và production) trên
>   traffic thật. Sample hội thoại để chấm reference-free metrics (faithfulness
>   so với retrieved context, relevance, LLM judge) và theo dõi tín hiệu người
>   dùng: escalation rate, thumbs down, khách hỏi lại, CSAT. Mục tiêu là phát
>   hiện drift, loại câu hỏi mới chưa có trong golden set, hoặc tác động của
>   policy update.
> - **Human review:** dùng cho case rủi ro cao (refund dispute, fraud, account 
>   takeover, privacy); khi metrics mâu thuẫn hoặc judge có độ tin cậy thấp; 
>   khi audit sample định kỳ và trước các launch lớn. Các failure do người phát
>   hiện được đưa ngược vào golden/regression dataset.

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
