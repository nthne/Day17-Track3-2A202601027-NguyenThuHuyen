# Lab 17 Memory Demo UI Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement `retrieve_for_case` so the Streamlit mini-product demonstrates all four bonus rubric requirements and supports grounded follow-up chat.

**Architecture:** Keep the existing Streamlit shell and implement one orchestration function that builds local short-term memory, selects durable layers from the case, and delegates retrieval/assembly to `StudentMemory`. Use temporary contract checks outside `tests/` because the lab forbids changing evaluator/tests; delete each temporary check after its RED/GREEN cycle.

**Tech Stack:** Python 3.12, Streamlit, Zep Cloud V3, pytest, Docker Compose, Gemini API optional for chat replies.

## Global Constraints

- Only modify `src/demo_ui.py` for the UI feature; do not modify evaluator, dataset, ground truth, or reference implementation.
- Use the existing `StudentMemory`, `ShortTermMemory`, `load_dataset`, and `load_json` interfaces.
- Keep `user_id` and `thread_id` from the selected case for every retrieval/chat turn.
- `GEMINI_API_KEY` is optional for retrieval demo; without it, show retrieved context as the fallback assistant response.
- Never commit `.env`, API keys, or `data/golden_eval.json`.

---

### Task 1: Contract test for case retrieval orchestration

**Files:**
- Modify: none
- Test: temporary `.codex_tmp_ui_contract.py` only; delete it after the test cycle.

**Interfaces:**
- Consumes: `StudentMemory.retrieve_*`, `StudentMemory.assemble_context`, `ShortTermMemory`, and a case dict.
- Produces: required result shape with `merged_context`, `layers`, and `budget`.

- [ ] **Step 1: Write the failing contract check**

Create a temporary script that imports `retrieve_for_case`, uses a fake memory object whose methods record calls and return marker strings, and passes this case:

```python
case = {
    "id": "UI-MIXED",
    "expected_layer": "mixed",
    "retrieve_layers": ["long_term", "semantic"],
    "user_id": "minh-lab17",
    "thread_id": "minh-s2",
    "query": "choose a payment retry approach",
    "fixture_messages": [],
}
```

Assert that the current function raises `NotImplementedError`; the fake methods must expect long-term with `user_id/thread_id/query`, semantic with `settings.semantic_graph_id/query`, and assembly with all four layer keys.

- [ ] **Step 2: Run the check and verify the expected failure**

```powershell
.\.venv312\Scripts\python.exe .codex_tmp_ui_contract.py
```

Expected: `NotImplementedError: BONUS TODO: run student retrieval for the loaded case`.

### Task 2: Implement short-term and durable layer orchestration

**Files:**
- Modify: `src/demo_ui.py:80-112`
- Test: temporary `.codex_tmp_ui_contract.py` from Task 1.

**Interfaces:**
- Consumes: `case`, `extra_messages`, `load_dataset()`, `ShortTermMemory(strategy="sliding", max_recent_messages=6, pressure_tokens=450)`.
- Produces: `retrieve_for_case(memory, case, extra_messages) -> dict[str, Any]`.

- [ ] **Step 1: Implement short-term source selection**

Build a four-key `layers` dict with empty strings. Select messages in this order:

```python
messages = case.get("fixture_messages")
if not messages:
    dataset = load_dataset()
    messages = []
    for user in dataset["users"]:
        if user["user_id"] != case["user_id"]:
            continue
        for session in user.get("sessions", []):
            if session["thread_id"] == case["thread_id"]:
                messages = list(session.get("messages", []))
                break
memory_short = ShortTermMemory(strategy="sliding", max_recent_messages=6, pressure_tokens=450)
for message in messages or []:
    memory_short.add(message["role"], message["content"])
for message in extra_messages:
    memory_short.add(message["role"], message["content"])
layers["short_term"] = memory_short.render()
```

- [ ] **Step 2: Implement durable layer selection**

Use the case layer or its explicit mixed selection:

```python
wanted = case.get("retrieve_layers") or [case["expected_layer"]]
if "long_term" in wanted:
    layers["long_term"] = memory.retrieve_long_term(
        user_id=case["user_id"], thread_id=case["thread_id"], query=case["query"]
    )
if "episodic" in wanted:
    layers["episodic"] = memory.retrieve_episodic(case["user_id"], case["query"])
if "semantic" in wanted:
    layers["semantic"] = memory.retrieve_semantic(settings.semantic_graph_id, case["query"])
```

- [ ] **Step 3: Finish the result contract**

Return:

```python
merged_context, budget = memory.assemble_context(layers)
return {"merged_context": merged_context, "layers": layers, "budget": budget}
```

- [ ] **Step 4: Run the contract check and full tests**

```powershell
.\.venv312\Scripts\python.exe .codex_tmp_ui_contract.py
.\.venv312\Scripts\python.exe -m pytest -q
```

Expected: the contract passes, the mixed case calls only long-term and semantic retrieval, and all pytest tests pass.

- [ ] **Step 5: Delete the temporary test**

Delete `.codex_tmp_ui_contract.py` after the GREEN run; it must not be committed.

- [ ] **Step 6: Commit the UI retrieval implementation**

```powershell
git add src/demo_ui.py
git diff --cached --check
git commit -m "feat: wire memory retrieval demo UI"
```

### Task 3: Run the live UI and capture bonus evidence

**Files:**
- Create: `submission/ui_demo.png` after successful manual interaction.
- Modify: none in source.

**Interfaces:**
- Consumes: completed `retrieve_for_case`, seeded Zep data, `data/sessions.json`, optional golden file, and `.env`.
- Produces: a runnable UI and one screenshot showing retrieval evidence/chat history.

- [ ] **Step 1: Start the Streamlit app**

```powershell
docker compose run --rm --service-ports app streamlit run src/demo_ui.py --server.address 0.0.0.0 --server.port 8501
```

Open `http://localhost:8501`.

- [ ] **Step 2: Verify case selection and metadata**

Select a case such as `E07` or `G20` and confirm the page shows case id, expected layer, user id, thread id, query, and description.

- [ ] **Step 3: Verify retrieval evidence**

Click `Run retrieval on this case`. Confirm the page shows active layer badges, merged context, per-layer evidence, and four budget metrics. For `E07`, confirm both long-term and semantic evidence are visible.

- [ ] **Step 4: Verify follow-up chat**

Send a follow-up prompt in the chat box. Confirm the user message and assistant reply remain in the conversation, retrieval runs again on the same case user/thread, and either Gemini produces a grounded answer or the fallback displays retrieved context when Gemini is unavailable.

- [ ] **Step 5: Capture evidence and commit**

Capture the UI after the four checks into `submission/ui_demo.png`, verify it contains case metadata, layer evidence, and chat history, then run:

```powershell
git add submission/ui_demo.png
git diff --cached --check
git commit -m "evidence: add memory demo UI screenshot"
```

### Task 4: Final verification

**Files:**
- Modify: none

**Interfaces:**
- Consumes: UI implementation and evidence.
- Produces: verified final state with no secrets staged.

- [ ] **Step 1: Run tests and syntax check**

```powershell
.\.venv312\Scripts\python.exe -m pytest -q
.\.venv312\Scripts\python.exe -m py_compile src/demo_ui.py
```

- [ ] **Step 2: Check repository safety**

```powershell
git diff --check
git status --short
git ls-files .env data/golden_eval.json
```

Expected: tests pass, syntax compilation exits 0, only intended artefacts are changed, and the final command prints no tracked secret/golden file.
