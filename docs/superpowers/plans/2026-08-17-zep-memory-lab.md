# Day 17 Zep Memory Lab Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Hoàn thiện core retrieval của Day 17, chạy kiểm chứng benchmark/privacy, và tạo artefact nộp bài theo đúng giới hạn file được phép sửa.

**Architecture:** `StudentMemory` gọi Zep với đúng identity/scope cho từng memory layer và ủy quyền trim/merge cho `ContextBudgetManager`. Các task được tách theo contract, sau đó là benchmark/report/privacy/artefact; mỗi task có vòng kiểm tra riêng và commit riêng.

**Tech Stack:** Python 3.12, pytest, Zep Cloud SDK, Docker Compose, Redis/Qdrant local services.

## Global Constraints

- Chỉ sửa `src/memory_student.py` cho bốn LAB TODO; không sửa tests/evaluator/ground truth/reference.
- Không commit `.env`, API key, `data/golden_eval.json`, hoặc dữ liệu bí mật.
- Không dùng `git add -A`; stage file tường minh.
- `README_submission.md` tối đa 400 từ.
- Mỗi task phải được kiểm thử trước khi commit.

---

### Task 1: Baseline and allowed-scope checkpoint

**Files:**
- Modify: none
- Test: existing `tests/`

**Interfaces:**
- Consumes: repository and lab instructions.
- Produces: verified baseline evidence and a clean task boundary; no production changes.

- [ ] **Step 1: Verify repository identity and allowed files**

Run:

```powershell
git remote get-url origin
git status --short
rg --files
```

Expected: the remote is the student fork, only pre-existing `README_CODELAB_DAY17.md` is uncommitted, and the required `src/`, `tests/`, and `data/` files exist.

- [ ] **Step 2: Run local unit tests before implementation**

Run:

```powershell
python -m pytest -q
```

Expected: existing starter tests pass; if dependencies are absent, install only from `requirements.txt` and rerun.

No repository-owned setup change is required at this checkpoint. Keep the existing untracked `README_CODELAB_DAY17.md` untouched and do not create an empty commit.

### Task 2: Implement long-term Context Block retrieval

**Files:**
- Modify: `src/memory_student.py:15-25`
- Test: an in-memory fake-client harness executed from PowerShell; do not modify evaluator tests.

**Interfaces:**
- Consumes: `prime_eval_thread(client, user_id, thread_id, query)`.
- Produces: `StudentMemory.retrieve_long_term(...) -> str` returning `context.context`.

- [ ] **Step 1: Write and run a failing contract check**

Create a temporary file `.codex_tmp_contract_long_term.py` with a fake `thread` exposing `delete`, `create`, `add_messages`, and `get_user_context`, then run `python .codex_tmp_contract_long_term.py`. The check must assert that `get_user_context` receives `thread_id="thread-new"` and that the method returns `"Python preference"`; before implementation it must fail with `NotImplementedError`. Delete the temporary file after the red check.

- [ ] **Step 2: Implement the minimal Context Block call**

Implement:

```python
prime_eval_thread(self.client, user_id, thread_id, query)
context = self.client.thread.get_user_context(thread_id=thread_id)
return str(context.context)
```

- [ ] **Step 3: Verify the contract check passes**

Run the focused fake-client check, then run:

```powershell
python -m pytest -q
```

Expected: the focused check confirms thread id and returned string; all starter tests pass.

- [ ] **Step 4: Commit**

```powershell
git add src/memory_student.py
git diff --cached --check
git commit -m "feat: retrieve long-term context block"
```

### Task 3: Implement episodic user-graph retrieval

**Files:**
- Modify: `src/memory_student.py:27-35`
- Test: fake graph-search contract check, then evaluator layer check if Zep is available.

**Interfaces:**
- Consumes: `cap_query` behavior documented in `src/utils.py` and `render_graph_search`.
- Produces: `retrieve_episodic(user_id, query) -> str` using only the user graph.

- [ ] **Step 1: Write and run the failing contract check**

Use a fake client whose `graph.search` records kwargs and returns an episode object. Assert the call must contain `user_id`, `scope="episodes"`, and `limit=5`; call the current method and observe `NotImplementedError`.

- [ ] **Step 2: Implement minimal episodic retrieval**

Implement:

```python
results = self.client.graph.search(
    user_id=user_id,
    query=cap_query(query),
    scope="episodes",
    limit=5,
)
return render_graph_search(results, episode_char_cap=1200)
```

Import `cap_query` from `.utils`.

- [ ] **Step 3: Verify**

Run the focused fake-client check and:

```powershell
python -m pytest -q
```

If Zep is seeded, run `python -m src.evaluate --impl student --reuse-seeded --only-layer episodic` and verify E04/E05 evidence markers.

- [ ] **Step 4: Commit**

```powershell
git add src/memory_student.py
git diff --cached --check
git commit -m "feat: retrieve episodic user memory"
```

### Task 4: Implement semantic shared-graph retrieval

**Files:**
- Modify: `src/memory_student.py:37-47`
- Test: fake graph-search contract check, then semantic evaluator layer check if Zep is available.

**Interfaces:**
- Consumes: `graph_id` and `render_graph_search`.
- Produces: `retrieve_semantic(graph_id, query) -> str` using shared knowledge scope.

- [ ] **Step 1: Write and run the failing contract check**

Use a fake graph client and assert the current method fails with `NotImplementedError`; the test expectation records `graph_id`, `scope="episodes"`, and `limit=8`.

- [ ] **Step 2: Implement minimal shared graph retrieval**

Implement:

```python
results = self.client.graph.search(
    graph_id=graph_id,
    query=cap_query(query),
    scope="episodes",
    limit=8,
)
return render_graph_search(results)
```

Do not pass `user_id` and do not use `scope="auto"`.

- [ ] **Step 3: Verify**

Run the focused fake-client check and full pytest. If Zep is seeded, run semantic-only evaluation and verify E06/E11 markers including `PAYMENT-RULE-3` and `CONN-POOL-FIRST`.

- [ ] **Step 4: Commit**

```powershell
git add src/memory_student.py
git diff --cached --check
git commit -m "feat: retrieve shared semantic knowledge"
```

### Task 5: Assemble mixed context with teaching budget

**Files:**
- Modify: `src/memory_student.py:49-52`
- Test: fake layer input contract check plus `tests/test_context_budget.py`.

**Interfaces:**
- Consumes: `self.budget.assemble(layers)`.
- Produces: `assemble_context(layers) -> tuple[str, dict[str, dict[str, int]]]`.

- [ ] **Step 1: Write and run the failing contract check**

Call `StudentMemory.assemble_context` with four layer strings and observe `NotImplementedError`; the expected result is the exact tuple returned by a `ContextBudgetManager`.

- [ ] **Step 2: Implement minimal delegation**

Implement:

```python
return self.budget.assemble(layers)
```

- [ ] **Step 3: Verify**

Run:

```powershell
python -m pytest -q tests/test_context_budget.py
python -m pytest -q
```

Expected: priority ordering, 10/4/3/3 limits, and all starter tests pass.

- [ ] **Step 4: Commit**

```powershell
git add src/memory_student.py
git diff --cached --check
git commit -m "feat: assemble memory layers within budget"
```

### Task 6: Seed and produce benchmark reports

**Files:**
- Create/modify generated: `reports/benchmark.json`, `reports/benchmark.md`, `reports/benchmark_no_memory.json`, `reports/benchmark_no_memory.md`, `reports/comparison.md`
- Modify: none in source

**Interfaces:**
- Consumes: completed `StudentMemory`, Docker services, `.env` kept untracked.
- Produces: student/no-memory benchmark and comparison evidence.

- [ ] **Step 1: Validate environment without exposing secrets**

Run:

```powershell
docker compose build
docker compose up -d redis qdrant
docker compose run --rm app python -m src.smoke
```

Expected: smoke validates services, dataset, and presence of `ZEP_API_KEY` without printing its value.

- [ ] **Step 2: Seed once**

Run:

```powershell
docker compose run --rm app python -m src.seed
```

Expected: both synthetic users and shared semantic graph become searchable.

- [ ] **Step 3: Run baseline, student, and comparison**

Run:

```powershell
docker compose run --rm app python -m src.evaluate --impl no_memory
docker compose run --rm app python -m src.evaluate --impl student --reuse-seeded
docker compose run --rm app python -m src.compare_reports
```

Expected: required report files exist; student has at least 9/11 practice hits if Zep retrieval is healthy.

- [ ] **Step 4: Commit generated evidence**

```powershell
git add reports/benchmark.json reports/benchmark.md reports/benchmark_no_memory.json reports/benchmark_no_memory.md reports/comparison.md
git diff --cached --check
git commit -m "reports: add memory benchmark comparison"
```

### Task 7: Run control-plane demos and privacy drill

**Files:**
- Create: `submission/long_term.png`, `submission/episodic.png`, `submission/semantic.png`, `submission/privacy.png` when screenshot tooling is available
- Modify: none in source

**Interfaces:**
- Consumes: saved benchmark reports and seeded Zep/Redis state.
- Produces: control-plane evidence and privacy deletion/verification evidence.

- [ ] **Step 1: Run the read-only learning demos**

Run:

```powershell
docker compose run --rm app python -m src.episodic_maintenance
docker compose run --rm app python -m src.heartbeat --dry-run
docker compose run --rm app python -m src.compiled_kb --reset
```

Expected: demos complete without modifying student retrieval contracts.

- [ ] **Step 2: Delete only the instructed synthetic user**

Run after reports are saved:

```powershell
docker compose run --rm app python -m src.forget --user-id minh-lab17
```

Expected: user-scoped Zep memory and Redis keys are deleted; shared semantic graph is retained.

- [ ] **Step 3: Verify deletion**

```powershell
docker compose run --rm app python -m src.forget --user-id minh-lab17 --verify-only
```

Expected output contains `Zep user absent: True` and `Redis user keys remaining: 0`.

- [ ] **Step 4: Commit evidence**

Stage only generated screenshots/evidence files that exist, run `git diff --cached --check`, then commit:

```powershell
git commit -m "evidence: document privacy drill and memory layers"
```

### Task 8: Write submission reflection and final verification

**Files:**
- Create: `README_submission.md`
- Optional create: `reports/golden_benchmark.json`, `reports/golden_benchmark.md` only when `data/golden_eval.json` is provided

**Interfaces:**
- Consumes: benchmark/comparison outputs and privacy evidence.
- Produces: <=400-word submission reflection and final clean verification.

- [ ] **Step 1: Write the required reflection**

Include the tested most important layer and case, Context Block versus Redis/Qdrant trade-off, memory-poisoning guardrails, weakest layer, highest-token case, E07 long-term+semantic evidence, token reduction caveat, E08 recency, and E10 compaction. Keep the document at or below 400 words.

- [ ] **Step 2: Verify artefacts and secrets**

Run:

```powershell
python -c "from pathlib import Path; p=Path('README_submission.md'); print(len(p.read_text(encoding='utf-8').split()))"
rg -n -i "ZEP_API_KEY|GEMINI_API_KEY|sk-[A-Za-z0-9]|api[_-]?key\s*=" --glob '!README_CODELAB_DAY17.md' --glob '!reports/*.json' .
git status --short
```

Expected: word count <=400, no secret values, and only intended artefacts are changed. Do not add `data/golden_eval.json`.

- [ ] **Step 3: Run final checks**

```powershell
python -m pytest -q
docker compose run --rm app pytest -q
docker compose run --rm app python -m src.evaluate --impl student --reuse-seeded
```

Expected: pytest exits 0 and the final student report is regenerated from the implemented student code.

- [ ] **Step 4: Commit final submission artefacts**

```powershell
git add src/memory_student.py README_submission.md reports submission
git diff --cached --check
git commit -m "feat: complete Day 17 memory lab submission"
```
