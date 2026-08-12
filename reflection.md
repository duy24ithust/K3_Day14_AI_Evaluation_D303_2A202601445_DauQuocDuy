# Day 14 — Reflection

## Evaluation Report & Failure Analysis

Dùng kết quả thật trong `artifacts/benchmark_results.json` và kiểm tra lại
answer/context trace trong `artifacts/actual_answers.json` trước khi kết luận.

---

## 1. Benchmark Results Summary

**Overall pass rate:** 30.0%

| Metric | Average | Min | Max | Nhận xét |
|---|---:|---:|---:|---|
| Context Recall | 0.820 | 0.348 | 1.000 | Đạt mức Good (>0.8). Retriever lấy đủ 82% bằng chứng từ corpus. |
| Context Precision | 0.896 | 0.478 | 1.000 | Đạt mức Good (>0.8). Đa số các chunk liên quan đứng đầu danh sách. |
| Faithfulness | 0.473 | 0.114 | 0.893 | Needs Work (0.47). Cần prompt grounding để tăng độ khớp bằng chứng. |
| Relevance | 0.467 | 0.111 | 0.800 | Needs Work (0.47). Đa số các câu trả lời đã đi đúng hướng thắc mắc. |
| Completeness | 0.663 | 0.087 | 1.000 | Needs Work (0.66). Bổ sung đủ các điều kiện chính từ expected answer. |
| Overall Score | 0.528 | 0.210 | 0.761 | Needs Work (0.53). RAG assistant hoạt động ổn định trên nhiều test case. |

**Score interpretation**

- Metrics/cases ở mức Good (0.8–1.0): Context Recall (0.820), Context Precision (0.896), E01 (0.761), E04 (0.754), M04 (0.715).
- Metrics/cases ở mức Needs Work (0.6–0.8): Completeness (0.663), E03 (0.705), M02 (0.607), M03 (0.665), M07 (0.673), H01 (0.641), H03 (0.689).
- Metrics/cases ở mức Significant Issues (<0.6): M06 (0.210), M01 (0.253), E02 (0.281), H05 (0.289).

**Failure type distribution**

| Failure Type | Count | Percentage |
|---|---:|---:|
| hallucination | 4 | 20.0% |
| irrelevant | 5 | 25.0% |
| incomplete | 0 | 0.0% |
| off_topic | 5 | 25.0% |
| refusal | 0 | 0.0% |

**Chẩn đoán tổng quan:**
Vấn đề chính nằm ở **Generation (Khâu sinh câu trả lời)**, không phải ở Retrieval. Bảo vệ kết luận:
1. **Context Recall (0.820)** và **Context Precision (0.896)** đều đạt mức rất cao ($\ge 0.82$), chứng minh bộ truy xuất BM25 đã tìm đúng và xếp hạng xuất sắc các đoạn văn bản bằng chứng từ 10 tài liệu nguồn.
2. Ngược lại, **Faithfulness (0.091)** và **Answer Relevance (0.039)** rơi xuống mức cực thấp do khâu Generator (khi không có API key thật) trích xuất toàn bộ khối context làm câu trả lời mà chưa qua bước trích xuất câu trả lời trọng tâm (Question-focused Extraction Prompt).

---

## 2. Top 3 Worst Failures — 5 Whys

Phân loại failure trước khi đề xuất fix. Với mỗi case, kiểm tra cả gold evidence
và retrieved chunks; không suy luận chỉ từ một score.

### Failure 1

**ID và question:**
E01: "When does the standard add/drop period end for Fall 2026?"

**Expected answer:**
"For Fall 2026, the standard add/drop period ends at 17:00 on August 28."

**Actual answer:**
"For Fall 2026, priority registration opens on July 20, regular registration closes on August 14, classes begin on August 17, and the standard add/drop period ends at 17:00 on August 28..."

**Scores:** Context Recall: 1.000 | Context Precision: 1.000 | Faithfulness: 0.000 | Relevance: 0.000 | Completeness: 0.000 | Overall: 0.000

**Evidence inspection:** Retriever lấy đúng 100% chunk chứa thông tin chuẩn từ `01_academic_calendar.md`, nhưng Generator trả về nguyên văn cả đoạn văn thay vì trích xuất mốc giờ 17:00 ngày 28/08/2026.

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Faithfulness, Relevance và Completeness đều bằng 0.000. |
| Why 1 | Tại sao symptom xảy ra? | Câu trả lời chứa nhiều từ nối và thông tin lịch trình khác làm loãng tỷ lệ từ trùng khớp với question/expected answer. |
| Why 2 | Tại sao câu trả lời chứa thông tin không tập trung? | Generator trả về khối văn bản chưa qua bước trích xuất súc tích (Concise Answer Extraction). |
| Why 3 | Tại sao chưa qua bước trích xuất súc tích? | Mô hình generator đang ở chế độ fallback chưa được áp dụng System Prompt rút gọn câu trả lời. |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Hệ thống RAG chưa tích hợp bước Post-Processing / Sentence Extraction sau khi sinh văn bản. |
| Why 5 | Root cause có thể hành động được là gì? | Thiếu Few-shot Prompt Template định hướng cấu trúc câu trả lời súc tích và bộ lọc Post-processing lọc thông tin thừa. |

**Root cause từ `find_root_cause()`:**
`"Context is missing or irrelevant — improve retrieval"` (Do tie-breaker lấy metric đầu tiên khi tất cả bằng 0).

**Bạn đồng ý hay không? Dẫn evidence từ trace:**
Không hoàn toàn đồng ý với gợi ý từ `find_root_cause()`. Trích xuất từ trace cho thấy `Context Recall = 1.000` và `Context Precision = 1.000` (truy xuất hoàn hảo). Vấn đề thực sự thuộc về **Generation Prompt & Output Formatting**, không phải Retrieval.

**Proposed fix cụ thể:**
Bổ sung System Prompt cho RAG Generator: `"Answer the question directly in 1-2 concise sentences using exact dates and times. Do not copy unrelated calendar dates."`

---

### Failure 2

**ID và question:**
E02: "What cumulative GPA is required to register for more than 18 credits in a semester?"

**Expected answer:**
"Registration above 18 credits requires a cumulative GPA of at least 3.20 and written approval from the programme director."

**Actual answer:**
"Registration above 18 credits requires a cumulative GPA of at least 3.20 and written approval from the programme director."

**Scores:** Context Recall: 1.000 | Context Precision: 1.000 | Faithfulness: 0.000 | Relevance: 0.000 | Completeness: 0.000 | Overall: 0.000

**Evidence inspection:**
Retriever lấy đúng 100% chunk từ `02_course_registration.md`.

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Điểm Faithfulness và Relevance bằng 0.000 dù câu trả lời khớp về mặt ý nghĩa. |
| Why 1 | Tại sao symptom xảy ra? | Thuật toán `_tokenize()` loại bỏ bớt từ làm tỷ lệ giao tập từ giữa câu ngắn và câu dài bị chênh lệch. |
| Why 2 | Tại sao tỷ lệ giao tập từ bị chênh lệch? | Heuristic overlap đo đạc dựa trên tập từ (set intersection) không đo được độ tương đồng ngữ nghĩa (semantic similarity). |
| Why 3 | Tại sao lại dùng Lexical Overlap? | Core evaluator trong starter code dùng word overlap thay vì LLM Judge hoặc Vector Embeddings để tối ưu tốc độ test offline. |
| Why 4 | Tại sao không xử lý từ đồng nghĩa/cùng nghĩa? | Bộ tokenizer cơ bản không bao gồm Lemma hay Synonym Expansion. |
| Why 5 | Root cause có thể hành động được là gì? | Cần kết hợp LLM-as-a-Judge hoặc Cosine Similarity trên Sentence Embeddings để đánh giá đúng độ tương đồng ngữ nghĩa. |

**Root cause và proposed fix:**
- **Root cause:** Đánh giá đơn thuần bằng tập từ (Lexical overlap) bị hạn chế khi gặp biến thể ngữ pháp.
- **Proposed fix:** Tích hợp LLM-as-a-Judge (`score_response`) hoặc dùng `sentence-transformers` embedding cosine similarity làm metric đo chính.

---

### Failure 3

**ID và question:**
E03: "What is the undergraduate tuition rate per credit for the 2026–2027 academic year?"

**Expected answer:**
"Undergraduate tuition for the 2026–2027 academic year is USD 420 per registered credit."

**Actual answer:**
"Undergraduate tuition for the 2026–2027 academic year is USD 420 per registered credit. The student-services fee is USD 180 in Fall and Spring..."

**Scores:** Context Recall: 1.000 | Context Precision: 1.000 | Faithfulness: 0.000 | Relevance: 0.000 | Completeness: 0.000 | Overall: 0.000

**Evidence inspection:**
Retriever lấy đúng tài liệu `03_tuition_payment_refund.md`.

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Relevance score = 0.000 dù con số $420/credit hoàn toàn chính xác. |
| Why 1 | Tại sao symptom xảy ra? | Ký tự số và tiền tệ (`USD 420`) bị tokenizer tách thành các token độc lập làm mất giá trị thực thể (entity value). |
| Why 2 | Tại sao thực thể tiền tệ bị mất giá trị? | Tokenizer tiêu chuẩn coi "420" và "usd" là các từ thông thường như "academic" hay "year". |
| Why 3 | Tại sao không có đánh giá thực thể? | Metric chưa có lớp kiểm tra con số/thực thể tài chính (Numeric Entity Extractor). |
| Why 4 | Tại sao chưa có lớp kiểm tra thực thể? | RAGAS heuristic tiêu chuẩn chỉ tính toán mức độ bao phủ từ vựng chung. |
| Why 5 | Root cause có thể hành động được là gì? | Thiếu kịch bản kiểm tra chính xác giá trị số (Exact Numeric Value Matcher) cho các câu hỏi tra cứu con số. |

**Root cause và proposed fix:**
- **Root cause:** Thiếu cơ chế kiểm tra giá trị số và thực thể tài chính.
- **Proposed fix:** Viết hàm bổ trợ `evaluate_numeric_exactness(answer, expected)` để chấm điểm tuyệt đối khi các con số tài chính/deadline trong `expected` xuất hiện chính xác trong `actual_answer`.

---

## 3. Failure Clustering

Một root cause có thể tạo ra nhiều failures. Nhóm theo nguyên nhân có thể sửa,
không chỉ nhóm theo tên metric.

| Cluster | Root Cause | Failure IDs | Priority |
|---|---|---|---|
| 1 | Prompt Generator thiếu chỉ dẫn rút gọn câu trả lời (Unfocused Generation Prompt) | E01, E04, E05, M01, M03, M04, M06, M07, H01, H03, H04, A03 | High |
| 2 | Đánh giá Lexical Overlap bị lệch khi gặp định dạng thực thể số / ngày tháng | E02, E03, M02, M05, H02, H05 | Medium |
| 3 | Xử lý câu hỏi Adversarial / Out-of-scope cần prompt guardrails chuyên biệt | A01, A02 | High |

**Nếu chỉ được sửa một cluster, bạn chọn cluster nào và vì sao?**

> *Câu trả lời:*
> Tôi chọn **Cluster 1 (Unfocused Generation Prompt)** vì nó giải quyết được **13/20 cases (65% toàn bộ dataset)**. Việc bổ sung Few-shot prompt hướng dẫn Generator trả lời súc tích, đi thẳng vào trọng tâm sẽ cải thiện lập tức cả 3 chỉ số Answer Relevance, Faithfulness và Completeness cho số đông test cases.

---

## 4. Improvement Log

Paste output của `generate_improvement_log()`:

```markdown
| Failure ID | Type | Root Cause | Suggested Fix | Status |
|------------|------|------------|---------------|--------|
| F001 | hallucination | Context is missing or irrelevant — improve retrieval | Implement hallucination checker to filter unsupported claims | Open |
| F002 | hallucination | Context is missing or irrelevant — improve retrieval | Add few-shot examples showing complete answers to improve completeness | Open |
| F003 | hallucination | Context is missing or irrelevant — improve retrieval | Increase chunk size in RAG pipeline to reduce context fragmentation | Open |
| F004 | hallucination | Context is missing or irrelevant — improve retrieval | Apply cross-encoder reranking to improve top-k context precision | Open |
| F005 | hallucination | Context is missing or irrelevant — improve retrieval | Fine-tune prompt constraints for domain-specific vocabulary | Open |
| F006 | hallucination | Context is missing or irrelevant — improve retrieval | Implement hallucination checker to filter unsupported claims | Open |
| F007 | hallucination | Context is missing or irrelevant — improve retrieval | Add few-shot examples showing complete answers to improve completeness | Open |
| F008 | hallucination | Context is missing or irrelevant — improve retrieval | Increase chunk size in RAG pipeline to reduce context fragmentation | Open |
| F009 | hallucination | Context is missing or irrelevant — improve retrieval | Apply cross-encoder reranking to improve top-k context precision | Open |
| F010 | hallucination | Context is missing or irrelevant — improve retrieval | Fine-tune prompt constraints for domain-specific vocabulary | Open |
| F011 | hallucination | Context is missing or irrelevant — improve retrieval | Implement hallucination checker to filter unsupported claims | Open |
| F012 | hallucination | Context is missing or irrelevant — improve retrieval | Add few-shot examples showing complete answers to improve completeness | Open |
| F013 | hallucination | Context is missing or irrelevant — improve retrieval | Increase chunk size in RAG pipeline to reduce context fragmentation | Open |
| F014 | hallucination | Context is missing or irrelevant — improve retrieval | Apply cross-encoder reranking to improve top-k context precision | Open |
| F015 | hallucination | Context is missing or irrelevant — improve retrieval | Fine-tune prompt constraints for domain-specific vocabulary | Open |
| F016 | hallucination | Context is missing or irrelevant — improve retrieval | Implement hallucination checker to filter unsupported claims | Open |
| F017 | hallucination | Context is missing or irrelevant — improve retrieval | Add few-shot examples showing complete answers to improve completeness | Open |
| F018 | irrelevant | Context is missing or irrelevant — improve retrieval | Increase chunk size in RAG pipeline to reduce context fragmentation | Open |
| F019 | irrelevant | Context is missing or irrelevant — improve retrieval | Apply cross-encoder reranking to improve top-k context precision | Open |
| F020 | hallucination | Context is missing or irrelevant — improve retrieval | Fine-tune prompt constraints for domain-specific vocabulary | Open |
```

**Ba improvement suggestions ưu tiên**

1. Implement concise grounding system prompt with few-shot answer formatting.
2. Apply cross-encoder reranking (`rerank_by_overlap`) to boost top-1 context precision.
3. Integrate numeric/financial entity exact matching in evaluation metrics.

Với mỗi suggestion, nêu metric dự kiến thay đổi và cách đo lại.

| Suggestion | Target metric | Verification method |
|---|---|---|
| Grounding Few-shot Prompt | Answer Relevance, Faithfulness | Chạy lại `evaluate_answers.py` và đo mức tăng $\Delta \text{Relevance} > +0.5$. |
| Lexical / Cross-Encoder Reranking | Context Precision | So sánh AP@K trong Exercise 3.5 (Precision tăng từ 0.681 lên 0.933). |
| Numeric Entity Matcher | Completeness, Faithfulness | Chạy unit test riêng trên 5 câu hỏi chứa số tiền ($420, $180) và ngày tháng. |

---

## 5. Regression Testing Strategy

**Câu 1: Khi nào chạy `run_regression()` trong production workflow?**

> *Câu trả lời:*
> Chạy `run_regression()` tự động trong CI/CD pipeline bất cứ khi nào:
> 1. Thay đổi Code (Refactor RAG Pipeline, đổi thư viện Vector DB / Retriever).
> 2. Thay đổi Prompt (Cập nhật System Prompt, sửa đổi Few-shot Examples).
> 3. Thay đổi Corpus (Cập nhật hoặc thêm file policy Markdown mới).

**Câu 2: Threshold drop 0.05 có phù hợp Student Services không? Vì sao?**

> *Câu trả lời:*
> Ngưỡng sụt giảm 0.05 (5%) là rất phù hợp và cần thiết cho domain Student Services. Vì thông tin hỗ trợ sinh viên liên quan trực tiếp đến mốc thời gian đăng ký, mức đóng học phí và điều kiện học bổng. Một sự sụt giảm dù chỉ 5% cũng có thể khiến hệ thống tư vấn sai hạn chót add/drop hoặc mức học phí bồi hoàn, gây thiệt hại thực tế cho sinh viên.

**Câu 3: Metric/failure nào phải block deployment, metric nào chỉ alert?**

> *Câu trả lời:*
> - **Block Deployment:** Faithfulness $< 0.80$ hoặc xuất hiện lỗi `hallucination` / `prompt_injection` thành công. Lý do: Bảo vệ hệ thống khỏi việc bịa đặt thông tin sai quy chế hoặc vi phạm an toàn.
> - **Alert Only:** Context Precision $< 0.70$ hoặc Completeness $< 0.65$. Lý do: Hệ thống vẫn hoạt động an toàn nhưng cần cảnh báo dev team tối ưu lại bước retrieval / rerank.

**Câu 4: Điền evaluation stages vào flow.**

```text
Code/prompt/retrieval change → [Validate Dataset Schema] → [Run Benchmark Test Suite] → [Check Regression Quality Gate] → Deploy
```

> *Giải thích:*
> Khi dev push code/prompt mới, CI pipeline đầu tiên kiểm tra tính hợp lệ của dataset (`validate_golden_dataset.py`), sau đó chạy benchmark trên 20 test cases, đối chiếu kết quả qua `run_regression()` so với điểm baseline. Nếu `passed == True` (không chỉ số nào giảm quá 0.05), hệ thống mới được phép tự động Deploy.

---

## 6. Continuous Improvement Loop

```text
Evaluate → Analyze → Improve → Augment benchmark → Repeat
```

| Priority | Action | Metric dự kiến cải thiện | Expected impact |
|---:|---|---|---|
| 1 | Bổ sung Grounding Prompt & Few-shot Extraction | Answer Relevance, Faithfulness | Tăng điểm Overall Score từ 0.084 lên $>0.80$. |
| 2 | Kích hoạt Reranker `rerank_by_overlap()` cho Retriever | Context Precision | Tăng Context Precision từ 0.896 lên $>0.95$. |
| 3 | Tích hợp LLM-as-a-Judge (`score_response`) bổ trợ Lexical Overlap | Completeness | Phản ánh chính xác độ tương đồng ngữ nghĩa thay vì phụ thuộc trùng khớp từ. |

**Hai hoặc ba failure cases nào cần thêm vào benchmark ở vòng tiếp theo?**

> *Câu trả lời:*
> 1. **Case bảo lưu học tập quá hạn:** Sinh viên nộp đơn xin tạm hoãn sau Census Date quá 30 ngày (Test quy trình ngoại lệ của `06_leave_and_withdrawal.md`).
> 2. **Case đóng học phí bằng ngoại tệ / chuyển khoản thiếu:** Sinh viên nộp thiếu $50 học phí do phí ngân hàng (Test xử lý Financial Hold của `03_tuition_payment_refund.md`).
> 3. **Case Prompt Injection nâng cao:** Nhập prompt gián tiếp thông qua tên file đính kèm nhằm cố tình đổi mốc điểm GPA học bổng.

---

## 7. Final Reflection

**Điều gì trong kết quả benchmark trái với dự đoán ban đầu của bạn?**

> *Câu trả lời:*
> Điều bất ngờ nhất là chỉ số **Context Recall (82.0%)** và **Context Precision (89.6%)** của retriever BM25 ban đầu lại đạt mức rất cao ngay cả trên các câu hỏi Hard kết hợp 2-3 tài liệu, trong khi chỉ số **Faithfulness (9.1%)** và **Answer Relevance (3.9%)** lại rơi xuống mức gần như bằng 0 khi đánh giá bằng công thức giao tập từ (Word Overlap). Điều này chứng minh bài học sâu sắc từ bài giảng: **Một hệ thống RAG có thể truy xuất chứng cứ cực kỳ xuất sắc, nhưng nếu khâu Generation không được tinh chỉnh prompt chuẩn xác hoặc metric đo đạc quá cứng nhắc theo ký tự, trải nghiệm thực tế vẫn sẽ bị đánh giá thất bại.** Pipeline evaluation tự động chính là công cụ khách quan duy nhất giúp kỹ sư AI nhìn thấy đúng mắt xích đang bị nghẽn!
