# Plan - Day27 Track 3: HITL PR Review Agent

Ngày lập kế hoạch: 15/05/2026

## Tóm tắt deliverables hôm nay

Mục tiêu là hoàn thiện một agent review Pull Request có Human-in-the-Loop (HITL), dùng LangGraph để điều phối luồng xử lý, dùng LLM để phân tích diff, cho người review can thiệp ở các ca cần xác nhận, và ghi lại toàn bộ phiên làm việc vào audit trail.

Deliverables bắt buộc:

- [x] **HITL agent**: agent đọc PR, phân tích thay đổi, đề xuất review comment, tạm dừng bằng `interrupt()` khi cần người duyệt, và tiếp tục bằng `Command(resume=...)`.
- [x] **Approval UI bằng Streamlit**: giao diện nhập PR URL, hiển thị trạng thái review, xử lý approve/reject/edit hoặc trả lời escalation questions.
- [x] **Confidence-based routing**: route theo confidence với 3 nhánh `auto_approve`, `human_approval`, `escalate`.
- [x] **Audit trail**: ghi mọi bước quan trọng vào bảng `audit_events`; lab dùng SQLite (`hitl_audit.db`) thay cho PostgreSQL để zero-setup, nhưng schema vẫn là dạng cột rõ ràng để chuyển sang Postgres khi production.

## Trạng thái triển khai

- [x] Đã implement `exercises/exercise_1_confidence.py`.
- [x] Đã implement `exercises/exercise_2_hitl.py`.
- [x] Đã implement `exercises/exercise_3_escalation.py`.
- [x] Đã implement `exercises/exercise_4_audit.py`.
- [x] Đã implement `app.py`.
- [x] Đã cấu hình LLM dùng API thật: ưu tiên `OPENAI_API_KEY`, fallback `OPENROUTER_API_KEY`.
- [x] Đã kiểm thử real OpenAI call với response `REAL_API_OK`.
- [x] Đã kiểm thử real GitHub read API với PR-Demo #1.
- [x] Đã kiểm thử Exercise 1: PR #1 -> `human_approval`, PR #2 -> `escalate`.
- [x] Đã smoke test Exercise 4: tạo interrupt `escalation` và ghi audit rows.
- [x] Đã chạy Streamlit tại `http://localhost:8501`.

## To-do list triển khai

### 1. Exercise 1 - Confidence routing

- [ ] Hoàn thiện `node_analyze` trong `exercises/exercise_1_confidence.py`.
- [ ] Gọi LLM với `with_structured_output(PRAnalysis)`.
- [ ] Đảm bảo output có `summary`, `comments`, `confidence`, `confidence_reasoning`, `escalation_questions`.
- [ ] Hoàn thiện `node_route` để trả về đúng nhánh theo threshold:
  - `confidence > 0.72` -> `auto_approve`
  - `0.58 <= confidence <= 0.72` -> `human_approval`
  - `confidence < 0.58` -> `escalate`
- [ ] Wire LangGraph: `START -> fetch_pr -> analyze -> route`.
- [ ] Thêm conditional edges từ `route` sang `auto_approve`, `human_approval`, `escalate`.
- [ ] Thêm terminal edges từ các node cuối về `END`.
- [ ] Kiểm thử PR demo #1 và #2 in ra các branch khác nhau.

Lệnh kiểm thử:

```bash
uv run python exercises/exercise_1_confidence.py --pr https://github.com/VinUni-AI20k/PR-Demo/pull/1
uv run python exercises/exercise_1_confidence.py --pr https://github.com/VinUni-AI20k/PR-Demo/pull/2
```

### 2. Exercise 2 - HITL bằng interrupt

- [ ] Hoàn thiện `node_human_approval` trong `exercises/exercise_2_hitl.py`.
- [ ] Tạo payload cho `interrupt()` gồm tối thiểu:
  - `kind="approval_request"`
  - PR URL
  - diff
  - LLM summary/reasoning
  - proposed comments
  - confidence
- [ ] Hỗ trợ 3 quyết định của reviewer: `approve`, `reject`, `edit`.
- [ ] Compile graph với `MemorySaver()` checkpointer.
- [ ] Hoàn thiện vòng lặp resume trong `main()`:
  - Nếu có `__interrupt__`, lấy payload.
  - Hiển thị bằng `prompt_human(payload)`.
  - Resume bằng `Command(resume=answer)`.
- [ ] Đảm bảo side effect post comment chỉ xảy ra sau khi reviewer approve/edit, không xảy ra trước interrupt.

Lệnh kiểm thử:

```bash
uv run python exercises/exercise_2_hitl.py --pr https://github.com/VinUni-AI20k/PR-Demo/pull/1
```

### 3. Exercise 3 - Escalation reviewer Q&A

- [ ] Cập nhật prompt trong `exercises/exercise_3_escalation.py` để khi confidence thấp, LLM phải sinh `escalation_questions`.
- [ ] Hoàn thiện `node_escalate` để gọi `interrupt()` với payload:
  - `kind="escalation"`
  - risk/context
  - diff
  - confidence reasoning
  - danh sách câu hỏi cụ thể cho reviewer
- [ ] Thu câu trả lời của reviewer theo từng câu hỏi.
- [ ] Hoàn thiện `node_synthesize` để gọi lại LLM với context + câu trả lời của reviewer.
- [ ] Sinh review/refined comments mới sau escalation.
- [ ] Wire graph: `escalate -> synthesize -> commit`.
- [ ] Đảm bảo nhánh `commit` vẫn đi tới `END`.

Lệnh kiểm thử:

```bash
uv run python exercises/exercise_3_escalation.py --pr https://github.com/VinUni-AI20k/PR-Demo/pull/2
```

### 4. Exercise 4 - Structured SQLite audit trail

- [ ] Hoàn thiện helper `audit(...)` trong `exercises/exercise_4_audit.py`.
- [ ] Gọi `write_audit_event(thread_id=state["thread_id"], pr_url=state["pr_url"], entry=entry)`.
- [ ] Tạo `AuditEntry` cho từng event quan trọng:
  - `fetch_pr`
  - `analyze`
  - `route`
  - `auto_approve`
  - `human_approval` trước interrupt với decision `pending`
  - `human_approval` sau resume với decision thật (`approve`, `reject`, `edit`)
  - `escalate` trước interrupt với decision `pending`
  - `escalate` sau resume với decision `escalate`
  - `synthesize`
  - `commit` hoặc rejected outcome
- [ ] Điền đầy đủ các trường audit:
  - `agent_id`
  - `action`
  - `confidence`
  - `risk_level`
  - `reviewer_id`
  - `decision`
  - `reason`
  - `execution_time_ms`
- [ ] Dùng `risk_level_for(confidence)` để đồng bộ risk level.
- [ ] Đảm bảo `AsyncSqliteSaver` lưu checkpoint để resume sau crash.
- [ ] Kiểm tra file `hitl_audit.db` được tạo sau khi chạy.
- [ ] Replay được session theo `thread_id`.

Lệnh kiểm thử:

```bash
uv run python exercises/exercise_4_audit.py --pr https://github.com/VinUni-AI20k/PR-Demo/pull/1
uv run python -m audit.replay --list
uv run python -m audit.replay --thread <thread_id>
```

### 5. Exercise 5 - Streamlit approval UI

- [ ] Hoàn thiện import graph builder/helper từ solution của Exercise 4 trong `app.py`.
- [ ] Thêm `streamlit` vào dependencies nếu chưa có.
- [ ] Hoàn thiện form nhập PR URL và nút `Run review`.
- [ ] Lưu `thread_id`, config, graph result và interrupt payload vào `st.session_state`.
- [ ] Render UI theo từng loại interrupt:
  - `approval_request`: hiển thị diff, reasoning, comments, nút approve/reject/edit.
  - `escalation`: hiển thị context/risk, form trả lời từng câu hỏi.
- [ ] Khi user thao tác, resume graph bằng `Command(resume=user_choice_or_answers)`.
- [ ] Hiển thị kết quả cuối:
  - auto approve thành công
  - human approve/edit đã post comment
  - reject không post comment
  - escalation đã synthesize và commit/refine
- [ ] Làm sidebar recent sessions từ `audit_events` tương tự `audit.replay --list`.
- [ ] Hiển thị lệnh replay theo `thread_id` để kiểm tra audit.

Lệnh chạy UI:

```bash
uv run streamlit run app.py
```

## Acceptance criteria trước khi nộp

- [ ] `rg "TODO|NotImplementedError" exercises app.py` không còn TODO bắt buộc chưa làm.
- [ ] Exercise 1 chạy được cho PR #1 và PR #2, route đúng 3 bucket confidence.
- [ ] Exercise 2 pause bằng `interrupt()` và resume được sau approve/reject/edit.
- [ ] Exercise 3 tạo escalation questions và synthesize review sau khi reviewer trả lời.
- [ ] Exercise 4 ghi đủ audit events vào `hitl_audit.db` và replay được bằng `audit.replay`.
- [ ] Streamlit UI chạy bằng `uv run streamlit run app.py`.
- [ ] Reviewer có thể hoàn thành toàn bộ flow từ browser mà không cần terminal prompt.
- [ ] Audit log có thể query được các cột quan trọng như `action`, `confidence`, `risk_level`, `decision`, `reviewer_id`.
- [ ] Không commit `.env`, token, hoặc `hitl_audit.db`.

## Ghi chú nộp bài

- Assignment ghi "PostgreSQL audit trail", nhưng scaffolding chính thức của lab dùng SQLite để không cần setup database. Khi báo cáo/nộp bài, cần ghi rõ: SQLite đang đáp ứng audit trail trong lab; để production/PostgreSQL chỉ cần đổi checkpointer/connection string và giữ schema audit dạng cột.
- PR demo #1 kỳ vọng đi nhánh `human_approval`; PR demo #2 kỳ vọng đi nhánh `escalate`.
- Nếu LLM quá tự tin với PR #2, có thể tạm tăng `ESCALATE_THRESHOLD` trong `common/schemas.py` để ép nhánh escalation khi demo.
