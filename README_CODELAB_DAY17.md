# Day 17 — Zep Memory for Agent

> Runbook thực hiện lab theo đúng thứ tự **1 → 15** của bộ tài liệu đã cung cấp.
>
> Mục tiêu chính: hoàn thiện **4 contract retrieval** trong `src/memory_student.py`, chứng minh retrieval đúng **session / user / knowledge scope / token budget**, chạy benchmark, privacy drill và chuẩn bị artefact nộp bài.

---

## 0. Nguyên tắc bắt buộc trước khi làm

- Làm đúng repo starter kit Day 17. Remote chuẩn được tài liệu nêu là:

```text
https://github.com/VinUni-AI20k/Day17-Track3-ZepMemory4Agent.git
```

- Không dùng `git add -A` ngay từ đầu.
- Không sửa test/evaluator/ground truth/reference để làm tăng điểm.
- Không copy `src/memory_reference.py` rồi đổi tên thành bài student.
- Không commit `.env`, API key hoặc `data/golden_eval.json`.
- `README_submission.md` là artefact nộp bài riêng, tối đa **400 từ**.
- `src/demo_short_term.py` chỉ được đổi tạm để quan sát compaction rồi phải khôi phục.
- `src/demo_ui.py` chỉ sửa nếu làm bonus mini-product.

### Các khu vực được phép / không được phép

| Khu vực | Quy định |
|---|---|
| `src/memory_student.py` | File code bắt buộc; hoàn thiện đúng 4 `LAB TODO`. |
| `src/demo_ui.py` | Chỉ sửa khi làm bonus; phần thiếu là `retrieve_for_case`. |
| `README_submission.md` | Tự tạo, tối đa 400 từ. |
| `reports/` | Evaluator sinh JSON/Markdown/comparison/HTML tùy chọn. |
| `submission/` | Tự tạo để lưu 4 ảnh minh chứng. |
| `src/demo_short_term.py` | Chỉ đổi tạm `max_recent_messages` để quan sát rồi restore. |
| `tests/`, `data/`, `src/evaluate.py`, `src/context_budget.py`, `src/zep_common.py` | Chỉ đọc để hiểu contract; không sửa để tăng điểm. |
| `src/memory_reference.py` | Reference/instructor demo; không copy thành bài student. |
| `control_plane/` | Đọc để hiểu identity/context/memory/task policy; không phải phần code bắt buộc. |

---

# Bước 1/15 — Mở đúng repo và xác định phần được phép sửa

Đứng tại root có tối thiểu:

```text
README.md
LAB.md
docker-compose.yml
src/
tests/
data/
```

Chạy:

```bash
git remote get-url origin
git status --short
rg --files
```

Xác nhận remote là starter kit đúng hoặc fork hợp lệ do giảng viên giao.

Luồng tổng quát của lab:

```text
Đọc contract + dataset
    -> chạy unit test + no-memory baseline
    -> seed user graph + shared graph
    -> kiểm tra short-term compaction
    -> hoàn thiện long-term
    -> hoàn thiện episodic
    -> hoàn thiện semantic
    -> assemble theo context budget
    -> full benchmark + comparison
    -> privacy drill
    -> reflection + evidence
    -> golden/UI tùy chọn
    -> final checks + commit + push
```

**Kết quả cần đạt:** biết chính xác 4 hàm bắt buộc, phần bonus, artefact cần tạo và các file scaffold phải giữ nguyên.

---

# Bước 2/15 — Hiểu 4 memory layer và luật chấm

## 2.1. Mapping layer → scope → case

| Layer | Scope/backend | Case practice | Evidence tiêu biểu |
|---|---|---|---|
| Short-term | Thread hiện tại, local `ShortTermMemory` | E01, E10 | `ORCHID-27`, deadline cũ sau compaction |
| Long-term | User graph + Zep Context Block | E02, E03, E08, E09 | preference, open loop, recency, user isolation |
| Episodic | Episode trong user graph | E04, E05 | debug trajectory, outcome, reflection |
| Semantic | Standalone shared graph | E06, E11 | payment retry rule, incident playbook |
| Mixed | Ghép nhiều layer | E07 | `Python` + `Idempotency-Key` |

## 2.2. Nguồn dữ liệu evaluator

- `data/sessions.json`: nguồn evaluator thật; gồm 2 synthetic user, 4 session theo 3 stage và 11 evaluation case.
- `data/ground_truth.json`: chỉ là bản trích để đọc nhanh; scorer **không** tải file này.
- `data/knowledge.jsonl`: 4 tài liệu domain knowledge dùng chung.

## 2.3. Luật scorer

Scorer chuẩn hóa hoa/thường và khoảng trắng, sau đó yêu cầu:

1. Mọi string trong `must_contain_all` phải xuất hiện trong retrieved text.
2. Không string nào trong `must_not_contain` được xuất hiện.
3. Exception hoặc evidence rỗng → case fail.
4. LLM không tham gia chấm; không thể “đoán câu trả lời đúng” để che retrieval sai.

Lưu ý:

- E01/E10 được `src.evaluate` chạy trực tiếp qua short-term local, không đi qua 4 hàm student.
- E07 mặc định lấy **long-term + semantic**.
- Router demo trong `src/router.py` / `src/graph_agent.py` không phải phần practice scorer chấm.
- Zep graph search phải dùng đúng `user_id` hoặc `graph_id`; nhầm scope có thể leak user hoặc lấy sai knowledge.

---

# Bước 3/15 — Tạo môi trường Docker và bảo vệ API key

Docker là đường chạy chuẩn; image khóa Python 3.12 và cài `requirements.txt`.
Core benchmark không cần OpenAI/Gemini nhưng cần **Zep Cloud**.

## Windows PowerShell

```powershell
Copy-Item .env.example .env
notepad .env
```

## macOS/Linux

```bash
cp .env.example .env
${EDITOR:-nano} .env
```

Trong `.env`:

```text
ZEP_API_KEY=<key tài khoản lab>
ZEP_SEMANTIC_GRAPH_ID=vinuni-lab17-domain-kb
REDIS_URL=redis://redis:6379/0
QDRANT_URL=http://qdrant:6333
GEMINI_API_KEY=
```

`GEMINI_API_KEY` có thể để trống; chỉ cần cho câu trả lời chat trong bonus UI.

**Không bao giờ** dán key vào Python, Markdown, commit, log hoặc screenshot.

Build/start stores + smoke test:

```bash
docker compose build
docker compose up -d redis qdrant
docker compose run --rm app python -m src.smoke
```

Smoke chỉ kiểm tra Redis/Qdrant/dataset và biến `ZEP_API_KEY` có giá trị; `src.seed` mới là integration check Zep thật.

macOS/Linux có quickstart:

```bash
sh scripts/quickstart.sh
```

Đường phụ nếu chỉ chạy unit test local:

### Windows

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install -r requirements.txt
python -m pytest -q
```

### macOS/Linux

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt
python -m pytest -q
```

**Quan trọng:** `src.seed` xóa rồi tạo lại 2 synthetic user `minh-lab17`, `lan-lab17` và standalone graph đã cấu hình. Chỉ dùng account/project dành cho lab.

---

# Bước 4/15 — Chạy baseline trước implementation

## 4.1. Unit test

```bash
docker compose run --rm app pytest -q
```

Starter kit kỳ vọng 11 test pass và 1 golden test skip nếu chưa có `data/golden_eval.json`.
`pytest` xanh **không** chứng minh 4 TODO retrieval đã xong.

## 4.2. No-memory baseline

```bash
docker compose run --rm app python -m src.evaluate --impl no_memory
```

Baseline chuẩn: **2/11**, chỉ E01 và E10 pass.

Artefact sinh ra:

```text
reports/benchmark_no_memory.json
reports/benchmark_no_memory.md
```

## 4.3. Seed cloud data

```bash
docker compose run --rm app python -m src.seed
```

Chờ đến khi có `Seed complete`.
Sau khi seed, các benchmark tiếp theo nên reuse dữ liệu đã seed để tránh ingest lặp.

> Bản PDF hiển thị một số command box bị cắt ngang sau dấu `--`. Tài liệu nêu rõ ý định dùng cờ `--reuse-seeded`; hãy xác nhận cú pháp chính xác bằng:
>
> ```bash
> docker compose run --rm app python -m src.evaluate -h
> ```

Có thể chạy student baseline trước khi sửa để thấy `NotImplementedError` được evaluator ghi thành fail; `reports/benchmark.json` lúc này chỉ là baseline tạm và phải được ghi đè bởi full student run sau cùng.

---

# Bước 5/15 — Quan sát short-term memory và compaction

Mở:

```text
src/short_term.py
src/demo_short_term.py
tests/test_short_term.py
```

Ba strategy:

| Strategy | Hành vi |
|---|---|
| `buffer` | Giữ toàn bộ message; token tăng theo độ dài hội thoại. |
| `summary` | Nén phần cũ, giữ 2 message gần nhất + durable notes. |
| `sliding` | Giữ summary + durable notes + recent window; default của lab. |

`ShortTermMemory.detect_pressure` compact khi số message vượt `max_recent_messages` hoặc token estimate vượt ngưỡng.
`extract_durable_notes` ưu tiên TODO, deadline, decision, constraint, preference và marker viết hoa.

Chạy:

```bash
docker compose run --rm app python -m src.demo_short_term
```

Quan sát:

```text
messages_kept
durable_notes
compactions
estimated_tokens
```

Thử nghiệm bắt buộc:

1. Đổi tạm `max_recent_messages=6` → `4` trong constructor demo.
2. Chạy lại.
3. Xác nhận vẫn thấy:

```text
REVIEW-DEADLINE-1600
Friday
16:00
```

4. Khôi phục file và kiểm tra:

```bash
git diff -- src/demo_short_term.py
```

Checkpoint:

```bash
docker compose run --rm app pytest -q tests/test_short_term.py
docker compose run --rm app python -m src.evaluate --impl no_memory
```

Trong `README_submission.md` sau này phải có 2–3 câu giải thích constraint nào được giữ và vì sao buffer không bền vững.

**Kết quả:** E01/E10 pass, deadline sống sau compaction, `src/demo_short_term.py` không còn diff thử nghiệm.

---

# Bước 6/15 — Hoàn thiện long-term memory bằng Context Block

Mở `src/memory_student.py`, tập trung vào `retrieve_long_term` và **giữ nguyên signature**.

Contract:

1. Giữ lời gọi `prime_eval_thread` có sẵn.
2. Lấy Context Block cho đúng `thread_id` bằng `client.thread.get_user_context`.
3. Trích thuộc tính `.context`; return `str`, không return SDK object.
4. Không copy toàn transcript cũ sang evaluation thread.
5. Nếu harden bằng edge search: chỉ search user graph hiện tại, cap query, limit đủ lớn, render provenance/validity; tuyệt đối không leak user khác.

Pseudocode:

```text
prime evaluation thread với user + query hiện tại
get user context cho đúng evaluation thread
extract context string
nếu bổ sung fact search:
    chỉ ghép evidence không rỗng của đúng user
return string
```

Checkpoint long-term cần các marker:

| Case | Phải có | Ý nghĩa |
|---|---|---|
| E02 | `Python` | Preference đi qua thread mới. |
| E03 | `benchmark report`, `16:00` | Open loop còn tồn tại. |
| E08 | `BLUEBIRD-42`, `TypeScript`, `NestJS` | Constraint project mới thắng preference chung đúng scope. |
| E09 | `LOTUS-88`, `Java`, `Spring Boot`; không có `ORCHID-27` | User isolation. |

Lỗi cần tránh:

- return toàn response object thay vì `.context`;
- dùng thread cũ / bỏ `prime_eval_thread`;
- ingest query chấm thành durable memory;
- search semantic `graph_id` thay cho `user_id`;
- để preference `Python` của Minh ghi đè project constraint `TypeScript` của E08.

> Chạy layer checkpoint bằng evaluator với `--only-layer` tương ứng. Bản PDF bị cắt phần cờ ở cuối command; xác nhận chính xác tên layer bằng `src.evaluate -h` trước khi chạy.

---

# Bước 7/15 — Hoàn thiện episodic memory từ user graph

Mở:

```text
src/memory_student.py       # retrieve_episodic
src/zep_common.py           # render_graph_search
data/sessions.json          # stage 2 của minh-lab17
```

Yêu cầu:

1. Import `cap_query` từ `src.utils`; mọi graph query ≤ 400 ký tự.
2. Search bằng `user_id`, không dùng shared `graph_id`.
3. Chọn `scope="episodes"`.
4. Limit đủ để episode chứa marker không bị rớt.
5. Dùng `render_graph_search` chuyển SDK result → text.
6. Nếu episode quá dài làm reflection bị đẩy khỏi budget, dùng `episode_char_cap` để giữ nhiều episode riêng biệt hơn; không trim mất marker cần chấm.

Pseudocode:

```text
q = cap_query(query)
results = graph.search(user_id=<đúng user>, scope="episodes", ...)
evidence = render_graph_search(results, ...)
return evidence
```

Marker cần đạt:

- E04: `ClientSession`, `concurrency=20`, `ASYNC-FIX-20`.
- E05: `connection churn`, `timeout threshold`.

Nếu E04 có outcome nhưng E05 thiếu reflection: in evidence và chỉnh render/limit, **không sửa scorer**.

---

# Bước 8/15 — Hoàn thiện semantic memory trên shared graph

Mở:

```text
src/memory_student.py       # retrieve_semantic
data/knowledge.jsonl
src/seed.py                 # tạo standalone semantic graph
```

Semantic memory là domain knowledge dùng chung, không thuộc riêng Minh/Lan.

Yêu cầu:

1. Cap query trước khi gọi Zep.
2. Search standalone graph bằng `graph_id` được truyền vào hàm.
3. Dùng `scope="episodes"` để lấy raw document content và giữ literal marker.
4. Nếu SDK/account không trả episode scope, fallback `scope="nodes"`.
5. Render result thành `str` bằng helper có sẵn.

Không dùng `scope="auto"` cho lab này vì scorer cần marker nguyên văn.

Marker:

| Case | Phải có |
|---|---|
| E06 | `Idempotency-Key`, `max-3-retries`, `exponential-backoff` |
| E11 | `connection pooling`, `CONN-POOL-FIRST` |

Lỗi cần tránh:

- search bằng `user_id` khiến policy domain thành ký ức cá nhân;
- auto search làm mất literal marker;
- không fallback khi episode scope khác giữa SDK/account;
- nuốt exception rồi return rỗng mà không đọc `error` trong report.

---

# Bước 9/15 — Ghép context theo budget 10/4/3/3

Mở:

```text
src/context_budget.py
tests/test_context_budget.py
src/memory_student.py       # assemble_context
```

Budget mặc định với tổng 8000 token:

| Layer | Tỷ lệ | Limit |
|---|---:|---:|
| Short-term | 10% | 800 token |
| Long-term | 4% | 320 token |
| Episodic | 3% | 240 token |
| Semantic | 3% | 240 token |

Manager giữ thứ tự:

```text
short_term -> long_term -> episodic -> semantic
```

Estimator dùng khoảng 4 ký tự/token.

`assemble_context` phải:

1. Không tự nối raw layer vô hạn.
2. Truyền dict `layers` vào budget manager được khởi tạo trong constructor.
3. Return nguyên tuple:

```python
tuple[str, dict[str, dict[str, int]]]
```

4. Không đổi tên 4 layer key.
5. Không đổi signature.

Checkpoint:

```bash
docker compose run --rm app pytest -q tests/test_context_budget.py
```

E07 chỉ pass nếu merged context còn cả:

```text
Python                # long-term
Idempotency-Key       # semantic
```

Nếu marker có ở raw layer nhưng mất ở merged text: đọc `budget_breakdown`, tối ưu retrieval/render tại nguồn; **không** tăng budget tùy ý.

---

# Bước 10/15 — Chạy full benchmark và đọc report

Sau khi 4 TODO không còn `NotImplementedError`, chạy theo thứ tự:

```bash
docker compose run --rm app pytest -q
docker compose run --rm app python -m src.evaluate --impl no_memory
# Student full run: dùng cờ reuse-seeded theo CLI của starter kit.
docker compose run --rm app python -m src.evaluate --impl student <REUSE_SEEDED_FLAG>
docker compose run --rm app python -m src.compare_reports
```

> Không hard-code tên flag nếu chưa kiểm tra: `docker compose run --rm app python -m src.evaluate -h`.

Artefact bắt buộc:

| File | Nội dung |
|---|---|
| `reports/benchmark.json` | Student result + evidence từng case. |
| `reports/benchmark.md` | Hit rate, latency, token reduction, evidence excerpt. |
| `reports/benchmark_no_memory.json` | No-memory baseline. |
| `reports/benchmark_no_memory.md` | Baseline Markdown. |
| `reports/comparison.md` | So sánh memory-enabled vs no-memory. |

Đọc report theo thứ tự:

1. `passed`, `memory_hit_rate`.
2. `error`, `missing`, `forbidden_found` của case fail.
3. `retrieved_tokens`, `token_reduction`.
4. `budget_breakdown` của E07.
5. Evidence excerpt để biết lỗi nằm ở retrieval hay trimming.

Mục tiêu practice:

```text
>= 9/11 case
memory hit rate >= 80%
```

Cấu trúc điểm theo tài liệu:

| Khối | Điểm tối đa |
|---|---:|
| E01–E11 | 56 |
| Privacy drill | 6 |
| Phân tích benchmark + comparison | 6 |
| 3 câu reflection trong README submission | 6 |
| Artefact + quy trình | 6 |
| Golden 20/20 | +10 hoặc 0 |
| UI/report nâng cao | tối đa +10 |

No-memory có thể token reduction gần 100% vì retrieve gần như rỗng; **không được** coi đó là thắng nếu hit rate thấp.

HTML report tùy chọn:

```bash
docker compose run --rm app python -m src.report_html --all
```

**Cảnh báo:** mọi run `--only-layer` có thể ghi đè `reports/benchmark.json/.md`; full 11-case student benchmark phải là practice run cuối cùng trước khi nộp.

---

# Bước 11/15 — Đọc control plane và chạy demo bổ trợ

Đọc:

| File | Ý chính |
|---|---|
| `control_plane/AGENTS.md` | Route trước retrieval, đúng scope, không tự cấp thêm quyền. |
| `control_plane/SOUL.md` | Nói layer nào cung cấp evidence; không giả vờ memory hit. |
| `control_plane/CONTEXT_LAYERS.md` | Policy context không được hy sinh chỉ để tiết kiệm token. |
| `control_plane/MEMORY.md` | Schema, recall priority, conflict theo recency + scope. |
| `control_plane/MEMORY_SCHEMA.md` | Durable record cần source, timestamp, confidence, TTL, validity khi có. |
| `control_plane/TASKS.md` | Open loop phải còn rõ sau compaction. |

Demo an toàn:

```bash
docker compose run --rm app python -m src.episodic_maintenance
docker compose run --rm app python -m src.heartbeat --dry-run
docker compose run --rm app python -m src.local_baseline
```

- `episodic_maintenance`: minh họa importance decay/LRU/consolidation; không xóa Zep episode.
- `heartbeat --dry-run`: chỉ đọc task + in action; không ghi Zep/Redis.
- `local_baseline`: ghi synthetic profile vào Redis + Qdrant local để so sánh managed memory.

Hai demo instructor/reference có thể thay đổi cloud graph; chỉ chạy trên account lab và không dùng report reference làm bài nộp.

Reflection sau này phải nêu được trade-off Zep Context Block vs Redis+Qdrant và guardrail trước heartbeat/durable write.

---

# Bước 12/15 — Privacy drill đúng thứ tự

Mở:

```text
data/consent.json
src/privacy_guard.py
src/forget.py
```

Starter kit thực hiện:

- từ chối durable ingestion nếu synthetic user chưa opt-in;
- redact email + số điện thoại trước khi gửi message vào Zep;
- xóa Zep user + Redis key theo user ID;
- giữ shared semantic graph vì chỉ chứa domain knowledge.

Trước khi xóa:

1. Xác nhận `reports/benchmark.md/.json` là **full 11-case student run**.
2. Lưu `reports/comparison.md`.
3. Nên commit snapshot practice trước deletion.

Chỉ xóa synthetic user được chỉ định (`minh-lab17`).

Tài liệu yêu cầu **delete rồi verify-only**, và evidence phải cho thấy:

```text
Zep user absent: True
Redis user keys remaining: 0
```

Bản PDF bị cắt phần cuối command box; user ID đầy đủ được các trang khác xác định là `minh-lab17`. Xác nhận option của `src.forget` bằng:

```bash
docker compose run --rm app python -m src.forget -h
```

Sau khi xác định CLI, chạy theo logic:

```bash
# 1) delete minh-lab17
# 2) verify-only minh-lab17
```

**Không** dùng user thật. **Không** xóa shared semantic graph. **Không** seed lại trước khi chụp privacy verification.

Sau khi đã chụp evidence, seed lại để chuẩn bị golden:

```bash
docker compose run --rm app python -m src.seed
```

---

# Bước 13/15 — Viết reflection và chuẩn bị artefact

Tạo `README_submission.md`, tối đa **400 từ**, dùng số thật trong `reports/benchmark.json`.

Phải trả lời:

## Phân tích benchmark

1. Layer nào có hit rate thấp nhất? Dựa trên case nào?
2. Query nào retrieve nhiều token nhất?
3. E07 cần kết hợp layer nào? Hai evidence bắt buộc là gì?
4. Token reduction so với full source context là bao nhiêu? Vì sao no-memory có thể reduction cao nhưng hit rate thấp?

## Reflection bắt buộc

1. Layer quan trọng nhất trong bộ test là layer nào? Chỉ rõ case.
2. Trade-off Zep Context Block vs Redis + Qdrant?
3. Guardrail nào chống memory poisoning hoặc background write tự cấp quyền?

Thêm 2–4 câu về:

- **E08 recency:** project-scoped constraint mới phải thắng preference chung theo đúng scope/recency.
- **E10 compaction:** durable constraint/deadline vẫn phải tồn tại khi raw message cũ bị evict.

Tạo thư mục evidence:

### Windows PowerShell

```powershell
New-Item -ItemType Directory -Force submission
```

### macOS/Linux

```bash
mkdir -p submission
```

Bốn ảnh bắt buộc:

```text
submission/long_term.png     # E02, E03, E08 hoặc E09 pass
submission/episodic.png      # E04 và E05 pass
submission/semantic.png      # E06 và E11 pass
submission/privacy.png       # delete + verify-only
```

Ảnh phải đọc được command + output, không chỉ badge PASS. Không chụp `.env`, key, account token hoặc golden input.

## Template `README_submission.md`

> Chỉ là template. **Không nộp khi còn placeholder**.

```markdown
# Day 17 Submission Reflection

## Benchmark analysis
Practice đạt **[PASS]/11** case, memory hit rate **[HIT_RATE]%**. Layer có hit rate thấp nhất là **[LAYER]**, thể hiện ở **[CASE_ID]** vì **[NGUYÊN NHÂN NGẮN]**. Query retrieve nhiều token nhất là **[CASE/QUERY]** với **[TOKENS]** token. E07 là mixed retrieval cần **long-term + semantic**, đồng thời giữ được `Python` và `Idempotency-Key`. Token reduction của student so với full source context là **[REDUCTION]%**. No-memory có thể reduction rất cao vì gần như không retrieve durable context, nhưng điều đó đi kèm hit rate thấp nên không phản ánh chất lượng retrieval.

## Reflection
Layer quan trọng nhất trong bộ test của tôi là **[LAYER]** vì **[CASE + LÝ DO]**. Zep Context Block giúp lấy cross-session user context đã được managed/indexed, trong khi Redis + Qdrant cho quyền kiểm soát storage/index/retrieval lớn hơn nhưng cần tự vận hành schema, isolation, lifecycle và consistency. Guardrail quan trọng là route/scope trước retrieval, chỉ durable-write khi đã có consent/quyền phù hợp, PII minimization, provenance/validity và không cho background task tự mở rộng quyền ghi.

E08 cho thấy conflict phải giải theo scope + recency: constraint mới của project `BLUEBIRD-42` (`TypeScript`, `NestJS`) phải thắng preference `Python` chung. E10 cho thấy compaction đúng không chỉ “tóm tắt hay” mà phải giữ durable constraint/deadline như `REVIEW-DEADLINE-1600`, `Friday`, `16:00` dù raw turn cũ đã bị evict.
```

---

# Bước 14/15 — Golden set và mini-product tùy chọn

**Không bắt buộc để pass. Chỉ làm sau practice + privacy + artefact nền.**

## 14.1. Golden set

Giảng viên cung cấp `data/golden_eval.json` gồm đúng 20 case G01–G20.

Quy định:

- không sửa file;
- file đã nằm trong `.gitignore`;
- không hard-code query, marker, case ID;
- chỉ 20/20 và `summary.perfect == true` mới được +10; 19/20 trở xuống = 0 điểm golden.

Sau khi đã seed lại, chạy golden bằng option mà evaluator của starter kit cung cấp.

> Bản PDF export cắt phần option ở cuối command. Xác nhận bằng:
>
> ```bash
> docker compose run --rm app python -m src.evaluate -h
> ```

Kết quả phải nằm ở:

```text
reports/golden_benchmark.json
reports/golden_benchmark.md
```

## 14.2. Mini-product UI

Mở `src/demo_ui.py`, chỉ hoàn thiện `retrieve_for_case`.

Contract dict:

```text
merged_context   # context sau budget
layers           # evidence riêng 4 layer
budget           # breakdown từ assemble_context
```

Luồng:

1. Load recent/fixture messages cho short-term + nối chat history mới.
2. Chọn durable layer theo `expected_layer`; mixed dùng `retrieve_layers` nếu dataset có, nếu không long-term + semantic như evaluator.
3. Giữ nguyên `user_id`, `thread_id` của case.
4. Gọi các method student + `assemble_context`.
5. Continued chat phải giữ cùng identity/thread và history ngắn trong session state.

Mở UI tại:

```text
http://localhost:8501
```

Retrieval cần Zep key; Gemini key chỉ cần để sinh reply. Nếu Gemini key trống, UI vẫn phải fallback hiển thị retrieved context.

> Command start UI trong bản PDF bị cắt phía phải sau phần `docker compose run --rm --service-ports -e PYTHONPATH=/workspace app ...`. Không tự đoán tail command nếu starter kit khác; kiểm tra `README.md`, `docker-compose.yml` hoặc help của app trong repo.

---

# Bước 15/15 — Final checks, commit và nộp

Chạy final checks:

```bash
docker compose run --rm app pytest -q
docker compose run --rm app python -m src.evaluate --impl no_memory
# Full student run, dùng reuse-seeded theo CLI thực tế.
docker compose run --rm app python -m src.evaluate --impl student <REUSE_SEEDED_FLAG>
docker compose run --rm app python -m src.compare_reports
rg -n "NotImplementedError" src/memory_student.py
git status --short
git diff --check
git ls-files .env data/golden_eval.json
```

Hai lệnh sau phải **không in gì**:

```bash
rg -n "NotImplementedError" src/memory_student.py
git ls-files .env data/golden_eval.json
```

Kiểm tra `reports/benchmark.json`:

```text
implementation = student
practice case count = 11
passed >= 9
```

Bản PDF có một Python one-liner đọc JSON nhưng phần sau bị cắt; không nên tự chế dựa trên schema chưa kiểm tra. Có thể đọc trực tiếp:

```bash
python -m json.tool reports/benchmark.json
```

hoặc xem `reports/benchmark.md` để xác nhận đủ 11 case.

Stage có mục tiêu; **không dùng `git add -A`**. Stage các artefact bắt buộc thực tế có trong repo, ví dụ:

```bash
git add src/memory_student.py README_submission.md reports/ submission/
```

Nếu làm bonus, stage thêm `src/demo_ui.py`, UI evidence/video và golden reports; tuyệt đối không stage `data/golden_eval.json`.

Sau đó:

```bash
git diff --cached --check
git status --short
git commit -m "Complete Day 17 multi-memory agent lab"
git push origin HEAD
```

---

# Checklist nộp bài cuối cùng

- [ ] 4 hàm trong `src/memory_student.py` hoàn tất, không còn `NotImplementedError`.
- [ ] Unit test pass; golden test chỉ skip khi instructor chưa phát file.
- [ ] Full practice report là implementation `student`, đủ 11 case, pass ít nhất 9.
- [ ] Có no-memory report.
- [ ] Có `reports/comparison.md`.
- [ ] Có `README_submission.md` ≤ 400 từ.
- [ ] Có `submission/long_term.png`.
- [ ] Có `submission/episodic.png`.
- [ ] Có `submission/semantic.png`.
- [ ] Có `submission/privacy.png`.
- [ ] Privacy đã verify **trước** khi seed lại.
- [ ] Không sửa test/evaluator/ground truth/reference để tăng điểm.
- [ ] `.env`, API key, `data/golden_eval.json` không bị Git track.
- [ ] `git diff --check` sạch.
- [ ] Commit đã push đúng remote trước deadline.

---

# Trình tự chạy ngắn gọn đề xuất

```bash
# 1) repo + env
# kiểm tra remote/status/files
# tạo .env, điền ZEP_API_KEY

docker compose build
docker compose up -d redis qdrant
docker compose run --rm app python -m src.smoke

# 2) baseline
docker compose run --rm app pytest -q
docker compose run --rm app python -m src.evaluate --impl no_memory
docker compose run --rm app python -m src.seed

# 3) short-term observation
docker compose run --rm app python -m src.demo_short_term
docker compose run --rm app pytest -q tests/test_short_term.py

# 4) implement 4 TODO theo thứ tự
# retrieve_long_term
# retrieve_episodic
# retrieve_semantic
# assemble_context

# 5) verify context budget
docker compose run --rm app pytest -q tests/test_context_budget.py

# 6) full benchmark
# xác nhận cờ evaluator bằng: python -m src.evaluate -h
# rồi chạy full student với reuse-seeded
docker compose run --rm app python -m src.compare_reports

# 7) tạo README_submission.md + 4 evidence images
# 8) privacy delete + verify-only, chụp privacy.png
# 9) seed lại nếu làm golden
# 10) golden/UI tùy chọn
# 11) final checks + targeted git add + commit + push
```

---

## Ghi chú về các command bị cắt trong PDF

Một số code box của bản PDF được xuất khi vùng code có horizontal scroll, nên phần bên phải không xuất hiện trong file. README này **không bịa** phần CLI bị mất. Với các lệnh liên quan `src.evaluate`, `src.forget`, golden hoặc UI, hãy lấy cú pháp thật từ chính starter kit bằng `-h`, `README.md`, `LAB.md` hoặc `docker-compose.yml`. Ý nghĩa/contract/checkpoint trong runbook vẫn bám theo đúng 15 bước của tài liệu.
