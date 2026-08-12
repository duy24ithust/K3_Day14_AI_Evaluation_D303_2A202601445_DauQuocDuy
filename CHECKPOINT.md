# CHECKPOINT.md — Lab 14: AI Evaluation & Benchmarking Pipeline

## 📋 Tóm tắt Bài Lab 14

Lab 14 là bài học thực hành chuyên sâu về **Evaluation & Benchmarking Pipeline** áp dụng Scientific Method cho AI:
$$\text{Hypothesis} \longrightarrow \text{Experiment} \longrightarrow \text{Measure} \longrightarrow \text{Conclude} \longrightarrow \text{Iterate}$$

Trong bài lab này, học viên biến một evaluation core còn dở dang (`template.py`) thành một hệ thống đánh giá AI hoàn chỉnh với:
1. **Metrics RAGAS-style**: Answer-side (*Faithfulness, Relevance, Completeness*) và Retrieval-side (*Context Recall, Context Precision*).
2. **LLM-as-a-Judge**: Đánh giá theo rubric 1–5 domain-specific và phát hiện bias (*positional, leniency, severity*).
3. **Golden Dataset**: 20 Q&A pairs (5 Easy, 7 Medium, 5 Hard, 3 Adversarial) có trích dẫn verbatim evidence từ 10 tài liệu Northstar University Student Services.
4. **Benchmark Runner & Regression Detector**: Chạy tự động, đo pass rate, tính trung bình metrics và phát hiện giảm chỉ số $> 0.05$ (CI/CD quality gate).
5. **Failure Analyzer & 5 Whys**: Phân loại lỗi (*hallucination, irrelevant, incomplete, off_topic, refusal*) và phân tích nguyên nhân gốc rễ (Root Cause).
6. **Demo Presentation**: Xây dựng giao diện Web Dashboard tương tác hiện đại để trình bày kết quả đánh giá RAG trước giảng viên và lớp.

---

## 🎯 Bảng Phân Bổ Điểm & Tiêu Chí Đạt 100+ Điểm

| Thành phần | Tiêu chí chi tiết | Điểm | Checkpoint |
|---|---|:---:|:---:|
| **Core Coding (`template.py` / `solution.py`)** | Hoàn thành 5 Tasks, pass 41/42 public unit tests | **50** | CP2, CP5 |
| **Golden Dataset (`golden_dataset.json`)** | Đủ 20 QA, đúng 10 documents, verbatim evidence, pass `validate_golden_dataset.py` | **15** | CP3 |
| **LLM-as-a-Judge Rubric Design** | Rubric 1–5 domain-specific, xử lý edge cases & bias | **10** | CP1, CP3 |
| **Benchmark & Failure Analysis** | Báo cáo benchmark, 3 bài phân tích 5 Whys, Improvement Log | **15** | CP3, CP4 |
| **Code Quality & Regression Strategy** | Type hints, clean code, chiến lược regression testing CI/CD | **10** | CP2, CP4, CP5 |
| **TỔNG ĐIỂM CHUẨN** | | **100/100** | |
| **Bonus 3.4 (Framework Comparison)** | So sánh RAGAS, DeepEval, TruLens trong `exercises.md` | **+10** | CP3 |
| **Bonus 3.5 (Reranking Optimization)** | Implement `rerank_by_overlap`, pass test thứ 42 | **+5** | CP2, CP3 |
| **TỔNG ĐIỂM TỐI ĐA** | | **115/100** | |

---

## 🚀 Danh Sách Checkpoint Chi Tiết (Checklist)

### 📌 Checkpoint CP0: Setup & Baseline Tests (T+0p – T+15p)
- [ ] Kiểm tra môi trường Python $\ge 3.11$.
- [ ] Tạo & activate virtual environment `.venv`.
- [ ] Cài đặt dependencies từ `requirements.txt`.
- [ ] Chạy baseline test: `pytest tests/ -v` (xác nhận 42 tests collected, 42 failed là bình thường).
- [ ] Cấu hình file `.env` từ `.env.example`.
- **Nghiệm thu**: Python 3.11+, pytest thu thập đủ 42 tests không bị lỗi import.

### 📌 Checkpoint CP1: Part 1 — Warm-up (T+15p – T+30p)
- [ ] Hoàn thành **Exercise 1.1**: Phân tích RAG Evaluation Metrics (Recall, Precision, Faithfulness, Relevance, Completeness).
- [ ] Hoàn thành **Exercise 1.2**: Phân tích LLM-as-a-Judge Rubric & Bias (positional, verbosity, self-preference).
- [ ] Hoàn thành **Exercise 1.3**: Phân tích Evaluation trong CI/CD Quality Gate & Failure Taxonomy.
- **Nghiệm thu**: Điền đầy đủ câu trả lời vào `exercises.md` (Part 1).

### 📌 Checkpoint CP2: Part 2 — Core Coding trong `template.py` (T+30p – T+1h25p)
- [ ] **Task 1 (Data Models)**: Định nghĩa `QAPair`, `EvalResult`, và `overall_score()`.
  - *Targeted test*: `pytest tests/test_solution.py::TestEvalResultOverallScore -v` (3 passed).
- [ ] **Task 2 (RAGASEvaluator)**:
  - Implement 3 answer metrics: `evaluate_faithfulness`, `evaluate_relevance`, `evaluate_completeness`.
  - Implement 2 retrieval metrics: `evaluate_context_recall`, `evaluate_context_precision`.
  - Wire `run_full_eval` để xử lý cả optional `contexts`.
  - *Targeted test*: `pytest tests/test_solution.py::TestRAGASEvaluator tests/test_solution.py::TestContextMetrics tests/test_solution.py::TestRetrievalMetricWiring::test_run_full_eval_connects_optional_retrieval_metrics -v` (14 passed, 1 skipped).
- [ ] **Task 3 (LLMJudge)**: Implement `__init__`, `score_response()`, và `detect_bias()`.
  - *Targeted test*: `pytest tests/test_solution.py::TestLLMJudge -v` (4 passed).
- [ ] **Task 4 (BenchmarkRunner)**: Implement `run()`, `generate_report()`, `run_regression()`, `identify_failures()`.
  - *Targeted test*: `pytest tests/test_solution.py::TestBenchmarkRunner tests/test_solution.py::TestRunRegression tests/test_solution.py::TestRetrievalMetricWiring::test_runner_forwards_retrieved_contexts tests/test_solution.py::TestRetrievalMetricWiring::test_report_includes_retrieval_averages -v` (11 passed).
- [ ] **Task 5 (FailureAnalyzer)**: Implement `categorize_failures()`, `find_root_cause()`, `generate_improvement_suggestions()`, `generate_improvement_log()`.
  - *Targeted test*: `pytest tests/test_solution.py::TestFailureAnalyzer tests/test_solution.py::TestGenerateImprovementLog -v` (9 passed).
- [ ] **Bonus (Exercise 3.5)**: Implement `rerank_by_overlap()` trong `template.py`.
  - *Targeted test*: `pytest tests/test_solution.py::TestContextMetrics::test_reranking_improves_or_keeps_precision -v` (1 passed).
- **Nghiệm thu**: Full suite `pytest tests/ -v` đạt **42/42 passed** (hoặc 41 passed, 1 skipped nếu chưa làm bonus).

### 📌 Checkpoint CP3: Part 3 — Golden Dataset, RAG & Benchmark (T+1h25p – T+2h20p)
- [ ] **Đọc Corpus**: Đọc `data/student_services/manifest.json` và 10 file Markdown.
- [ ] **Xây dựng `golden_dataset.json`**:
  - 5 Easy (E01–E05), 7 Medium (M01–M07), 5 Hard (H01–H05), 3 Adversarial (A01–A03).
  - Đủ 10 documents, `text` là verbatim substring từ Markdown.
- [ ] **Validate Dataset**: `python validate_golden_dataset.py` $\rightarrow$ báo `PASS`.
- [ ] **Sinh Actual Answers**: Cấu hình `OPENAI_API_KEY`, chạy `python domain_assistant.py` $\rightarrow$ sinh `artifacts/actual_answers.json`.
- [ ] **Chạy Evaluation**: Chạy `python evaluate_answers.py` $\rightarrow$ sinh `artifacts/benchmark_results.json`.
- [ ] **Điền `exercises.md` (Part 3)**:
  - Exercise 3.1: Stratified distribution & dataset coverage.
  - Exercise 3.2: Bảng kết quả 20 QA & aggregate report.
  - Exercise 3.3: LLM-as-a-Judge Rubric 1–5 chi tiết.
  - Exercise 3.4 (Bonus): So sánh RAGAS vs DeepEval vs TruLens (+10đ).
  - Exercise 3.5 (Bonus): Đánh giá tác động Reranking lên Context Precision (+5đ).
- **Nghiệm thu**: Validator PASS, sinh đủ 20 actual answers & benchmark report, điền đầy đủ Part 3 trong `exercises.md`.

### 📌 Checkpoint CP4: Part 4 — Failure Analysis & Reflection (T+2h20p – T+2h35p)
- [ ] **Chọn 3 case kém nhất**: Lấy 3 QA pairs có Overall Score thấp nhất từ Exercise 3.2.
- [ ] **Phân tích 5 Whys**: Viết 3 chuỗi 5 Whys chi tiết từ Symptom $\rightarrow$ Root Cause hành động được trong `reflection.md`.
- [ ] **Improvement Log**: Tạo bảng Markdown tổng hợp Failure ID, Symptom, Root Cause, Actionable Fix, Verified Metric.
- [ ] **Regression Strategy**: Mô tả quy trình dùng `run_regression()` làm Quality Gate trong CI/CD pipeline.
- **Nghiệm thu**: `reflection.md` đầy đủ báo cáo evaluation, 3 phân tích 5 Whys, improvement log và regression strategy.

### 📌 Checkpoint CP5: Hoàn Thiện & Kiểm Tra Cuối (T+2h35p – T+2h45p)
- [ ] Copy `template.py` sang `solution/solution.py`.
- [ ] Chạy lại `pytest tests/ -v` (đạt 42/42 passed).
- [ ] Chạy lại `python validate_golden_dataset.py` (đạt PASS).
- [ ] Rà soát 4 deliverables bắt buộc:
  1. `solution/solution.py`
  2. `golden_dataset.json`
  3. `exercises.md`
  4. `reflection.md`
- [ ] Commit & Push lên repo cá nhân: `git status`, `git commit`, `git push`.
- **Nghiệm thu**: 4 file hoàn chỉnh trên GitHub repo cá nhân, tests pass 100%.

---

## 🖥️ Checkpoint CP6: DEMO Presentation & Live UI Dashboard (T+2h45p – T+3h45p)

Để thuyết trình ấn tượng trước giảng viên và lớp, một **AI Evaluation & Benchmarking Dashboard** tương tác trực quan sẽ được thiết lập với các tính năng:

### 🌟 Các Tính Năng Cần Có Trên Giao Diện Demo

1. **📊 Benchmark Overview Header & Metric Cards**:
   - Hiển thị Pass Rate (%), Total QAs (20/20), Overall Average Score.
   - 5 Thẻ chỉ số chính với thanh tiến trình trực quan: *Faithfulness, Relevance, Completeness, Context Recall, Context Precision*.

2. **🎯 Golden Dataset Browser & Filter**:
   - Bộ lọc theo Độ khó (*Easy, Medium, Hard, Adversarial*) và Document Source (*10 docs*).
   - Xem chi tiết từng QA pair: Question, Expected Answer, Gold Contexts vs Retrieved Chunks.

3. **🔬 Live Evaluation & Playground**:
   - Nhập Question + Context + Generated Answer $\rightarrow$ Tự động tính 5 RAGAS metrics & Overall Score ngay lập tức.
   - Thử nghiệm Reranking (*Before vs After Context Precision*).

4. **🕵️ Failure Analyzer & Interactive 5 Whys Viewer**:
   - Phân nhóm lỗi trực quan theo biểu đồ tròn (*Hallucination, Irrelevant, Incomplete, Off-topic*).
   - Xem chi tiết 3 case tệ nhất với cây phân tích **5 Whys Interactive Stepper** từ Triệu chứng $\rightarrow$ Nguyên nhân gốc rễ $\rightarrow$ Giải pháp đề xuất.

5. **⚡ CI/CD Quality Gate & Regression Simulator**:
   - Simulator so sánh Baseline vs New Run với threshold regression $\Delta > 0.05$.
   - Nút bấm mô phỏng: **Pass & Deploy** hoặc **Block Deployment (Regression Detected)**.

---

*CHECKPOINT.md đã được khởi tạo để làm kim chỉ nam thực hiện bài Lab 14 đạt 115/115 điểm!*
