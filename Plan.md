# Plan.md — Milestones and Validations

**Companion to:** `PROMPT.md` (the specification). Where the two conflict, `PROMPT.md` wins.
**Status:** Draft v1 · blocked on `PROMPT.md` §19.1–19.4

---

## 0. How to use this file

Work through milestones in order. One milestone = one focused loop:

```
1. Read the milestone. Build only what it lists.
2. Run its validation command.
3. If it fails → STOP-AND-FIX (§5). Do not proceed.
4. If it passes → commit, then start the next milestone.
```

**Never** start milestone *n+1* while *n* is failing.
**Never** begin any milestone until the four open decisions in `PROMPT.md` §19.1–19.4 are answered.

---

## 1. Stages

| Stage | Milestones | Outcome |
|---|---|---|
| **A — Foundation** | M0–M3 | Scaffolding, contracts, decision layer talking to Laya |
| **B — Safety core** | M4–M5 | Rule engine and deterministic lookup tools |
| **C — Knowledge & generation** | M6–M7 | RAG and the LLM, behind policy |
| **D — Assembly** | M8–M11 | Orchestrator, permissions, service, evaluation |
| **E — Report** | M12 | **Phase A complete — baseline published** |
| **F — Improve** | M13–M15 | **Phase B complete — fine-tuned and compared** |

Phase A ends at **M12**. Nothing in Stage F may begin before the baseline report exists.

---

## 2. Milestones

### M0 — Scaffold and gates

**Build**
- `pyproject.toml` with pinned dependency versions
- `Makefile` implementing the §3 command contract
- `src/harness/` package skeleton with `__init__.py` files
- `scripts/check_architecture.py` — AST import checker (§4.3)
- `scripts/validate.sh` — runs every gate in order
- `.env.example` — `LAYA_BASE_URL`, `LAYA_API_KEY`, `LAYA_MODEL_ID`, `LLM_PROVIDER`, `LLM_API_KEY`, `LLM_MODEL_ID`
- Empty test dirs: `tests/unit`, `tests/integration`, `tests/contract`
- `.gitignore` — must exclude `.env`, `__pycache__`, `out/`, `.venv`, checkpoints

**Acceptance**
- [ ] `make validate` runs green with zero tests collected
- [ ] `make arch` passes
- [ ] `make type` passes
- [ ] No secrets in the repository; `.env.example` contains placeholders only

**Validation**
```bash
make validate
```

---

### M1 — Contracts and question schemas

**Build**
- `src/harness/contracts/` — Pydantic models: `HarnessRequest`, `HarnessResponse`, `Decision`, `Escalation`, `DecisionTrace`, `Abstention`, `Degradation`
- `data/questions/` — the four schema files from `PROMPT.md` §8 (acuity, red_flags, extraction, guard)
- `src/harness/contracts/questions.py` — loader, **content hash** of each schema file, and a `SCHEMA_VERSION` constant

**Acceptance**
- [ ] All four schema files exist and match `PROMPT.md` §8 verbatim
- [ ] Schema hash is stable across runs and changes when any byte changes
- [ ] Invalid schema files fail loudly with a path, not a stack trace
- [ ] `HarnessResponse.escalated` is typed such that escalation data is required when `escalated is True`

**Validation**
```bash
pytest tests/unit/test_questions.py tests/unit/test_contracts.py -q
make type
```

---

### M2 — Record Laya fixtures ⚠️ requires `PROMPT.md` §19.1

**Build**
- `scripts/record_fixtures.py` — sends each case in `data/evals/scenarios.jsonl` to the live Laya API, stores raw answers keyed by case id
- `data/fixtures/` — recorded raw responses, committed to the repository
- `data/evals/scenarios.jsonl` — first 20 seed cases (rest authored in M11)

**Acceptance**
- [ ] ≥ 20 fixtures recorded
- [ ] Each fixture stores the full raw Laya response plus the model id and schema hash at record time
- [ ] Re-running the recorder against an already-recorded case **reproduces identical output**, or reports a mismatch loudly
- [ ] No patient-identifying content in any fixture

**Validation**
```bash
python scripts/record_fixtures.py --check --limit 20
pytest tests/contract/test_fixture_fidelity.py -q
```

**If this fails:** the Laya API contract in `PROMPT.md` §19.1 is wrong or unavailable. **Stop** and amend §19.1 per §6. Do not write a mock and pretend the API works.

---

### M3 — Decision client

**Build**
- `src/harness/decision/client.py` — `DecisionClient` Protocol: `decide(text, question_set) -> RawDecision`
- `src/harness/decision/fixture_adapter.py` — replays from `data/fixtures/`
- `src/harness/decision/http_adapter.py` — calls the live API, with timeout and typed error mapping
- Credentials read from environment only

**Acceptance**
- [ ] Both adapters satisfy the same contract test, run against the same inputs
- [ ] `fixture_adapter` raises a named error on an unknown case id — never invents an answer
- [ ] `http_adapter` has an explicit timeout and maps transport failures to a typed error
- [ ] **All tests use the fixture adapter.** No test touches the network.

**Validation**
```bash
pytest tests/contract/test_decision_client.py -q
make arch && make type
```

---

### M4 — Rule engine ⚠️ safety-critical

**Build**
- `src/harness/rules/thresholds.py` — the five escalation conditions from `PROMPT.md` §9, versioned
- `src/harness/rules/engine.py` — pure function `(trace, thresholds) -> Escalation | None`

**Acceptance**
- [ ] Every escalation branch has a test: suicide flag, generic flag, acuity level, resuscitation probability, abstention-plus-flag
- [ ] A fired flag is never suppressed by a high-probability negative (downgrade test)
- [ ] Boundary values tested on both sides (`0.699` / `0.700`, `0.099` / `0.100`)
- [ ] `escalated=False` produces `escalation=None`
- [ ] **100% line and branch coverage** on `harness.rules`
- [ ] `rules` imports nothing but `contracts` — verified by `make arch`

**Validation**
```bash
pytest tests/unit/test_rules.py -q --cov=harness.rules --cov-branch --cov-fail-under=100
make arch
```

---

### M5 — Deterministic lookup tools

**Build**
- `src/harness/tools/labs.py`, `src/harness/tools/drugs.py`
- `data/tables/lab_ranges.json`, `data/tables/drug_interactions.json` — curated, versioned
- `tools/base.py` — shared `{ facts, source_ids, source_type }` contract

**Acceptance**
- [ ] Every lookup is a pure function of a data file — no LLM anywhere in the path
- [ ] Unknown test / unknown drug returns `not_found`, never a guess
- [ ] Every returned fact carries a source id
- [ ] Data files have a schema; a malformed file fails at load with a clear message
- [ ] Tables carry a `version` and a provenance note

**Validation**
```bash
pytest tests/unit/test_labs.py tests/unit/test_drugs.py -q --cov=harness.tools --cov-fail-under=90
make arch
```

---

### M6 — RAG (`kb_search`)

**Build**
- `src/harness/tools/kb.py` — chunking, embedding, cosine search, citation resolution
- `data/corpus/` — curated source documents
- `data/index/` — precomputed embeddings (`embeddings.npy` + metadata)
- A deterministic `hash` embedder for tests, so tests never download a model

**Acceptance**
- [ ] Every search result resolves to an id present in the corpus
- [ ] An unresolvable citation is a **hard failure**, never a warning
- [ ] Retrieved text is placed in the data channel, structurally separated from instructions
- [ ] Search results are reproducible: same query + same index → same ids in the same order
- [ ] Corpus has ≥ 10 documents with recorded provenance

**Validation**
```bash
pytest tests/unit/test_kb.py tests/security/test_injection_channel.py -q
make arch
```

---

### M7 — LLM adapter and policy

**Build**
- `src/harness/generation/llm.py` — provider-agnostic adapter, one concrete implementation
- `src/harness/generation/policy.py` — refusal rules from `PROMPT.md` §11.2
- `src/harness/generation/citations.py` — citation validation and banned-phrase check
- `src/harness/generation/schemas.py` — tool-call schemas

**Acceptance**
- [ ] Guard flags route to the correct refusal reason code
- [ ] Uncited factual claims are rejected before the response is returned
- [ ] Banned-phrase check runs on all generative output
- [ ] Suicide-risk path emits crisis text from **configuration**, not generated text
- [ ] Refusals name a reason code; no silent fallback anywhere
- [ ] Adapter has no dependency on `rules` or `decision` — verified by `make arch`

**Validation**
```bash
pytest tests/unit/test_policy.py tests/unit/test_citations.py -q
make arch && make type
```

---

### M8 — Orchestrator ⚠️ safety-critical

**Build**
- `src/harness/orchestrator.py` — wires Layer 1 → 2 → 3 per `PROMPT.md` §5
- Single entrypoint: `assess(text, actor_role) -> HarnessResponse`
- `decision_trace` emission per `PROMPT.md` §12

**Acceptance**
- [ ] **Escalation path never reaches the LLM.** Proven by a test that substitutes a raising LLM stub and asserts zero invocations across all escalation cases
- [ ] `escalated=True` responses carry the verbatim instruction text
- [ ] Every response carries a complete trace: model id, schema hash, rule version, thresholds, raw answers, tools called, citations, `llm_called`
- [ ] LLM unavailable → `degraded: true`, decision still returned, request does not fail
- [ ] Same input × 5 runs → byte-identical decision section
- [ ] Recorded trace replayed offline → identical escalation decision

**Validation**
```bash
pytest tests/integration/test_escalation_isolation.py -q
pytest tests/integration/test_orchestrator.py -q
make arch
```

---

### M9 — Role permissions

**Build**
- `src/harness/tools/registry.py` — permission matrix, enforced at the tool-call boundary

**Acceptance**
- [ ] Every capability in `PROMPT.md` §6 has both an allow and a deny test
- [ ] An out-of-role tool call is refused **inside the harness**, not by the caller
- [ ] Refusals return `permission_denied` and are logged
- [ ] Tests bypass the transport entirely and call the harness directly — proving enforcement is server-side

**Validation**
```bash
pytest tests/security/test_permissions.py -q
make arch
```

---

### M10 — Service edge

**Build**
- `src/harness/service/app.py`, `routes.py` — FastAPI
- `GET /health`, `GET /schemas/{name}`, `POST /assess`, `GET /traces/{request_id}`

**Acceptance**
- [ ] Every non-2xx response has a typed error body
- [ ] Role read from the trusted header only; no identity issuance
- [ ] OpenAPI schema generated and committed
- [ ] Health check reports Laya reachability separately from LLM reachability

**Validation**
```bash
pytest tests/integration/test_service.py -q
python -c "from harness.service.app import app; app.openapi()" > out/openapi.json
```

---

### M11 — Evaluation harness and sets ⚠️ requires clinical review

**Build**
- `evals/sets/scenarios.jsonl`, `red_flags.jsonl`, `refusals.jsonl`, `holdout.jsonl`
- `evals/metrics.py`, `run_eval.py`, `report.py`
- Exit-code gating per `PROMPT.md` §15.2

**Acceptance**
- [ ] Set sizes meet `PROMPT.md` §15.1 minimums
- [ ] `holdout` is used by nothing except final reporting — verify no threshold was tuned on it
- [ ] Blocking metrics cause a **nonzero exit code**
- [ ] Report renders all metrics in `PROMPT.md` §15.2, including those marked *report*
- [ ] Every set is synthetic and carries a header recording provenance and author

**Validation**
```bash
python -m evals.run_eval --set scenarios --report out/scenarios.md
python -m evals.run_eval --set red_flags --report out/red_flags.md; echo "exit=$?"
python -m evals.run_eval --set refusals --report out/refusals.md; echo "exit=$?"
```

**If recall or refusal correctness is below 100%, the command exits nonzero. That is the gate working, not a broken build.** Fix the system per §5 — never lower the target.

---

### M12 — Baseline report · **Phase A complete**

**Build**
- `out/baseline_report.md` — full results
- README updated with setup, architecture, and the `PROMPT.md` §4 non-goals

**Acceptance**
- [ ] Every metric in `PROMPT.md` §15.2 reported with an actual number
- [ ] Poor results reported plainly — `PROMPT.md` §14.3 no-hiding rule
- [ ] Measured p95 latencies for each `PROMPT.md` §7.1 budget
- [ ] Known limitations stated, including that off-the-shelf Laya has no medical training
- [ ] `PROMPT.md` §16.1–16.5 checks all pass, each demonstrable

**Validation**
```bash
make validate
make evidence          # prints a checklist of every PROMPT.md §16 item with pass/fail
```

---

### M13 — Dataset ⚠️ requires `PROMPT.md` §19.2

**Build**
- `finetune/dataset_build.py` — labelling, splits, dedup, leakage guard
- `data/ml/` — versioned dataset with a manifest

**Acceptance**
- [ ] Train / validation / holdout splits are disjoint, verified by an assertion not a comment
- [ ] **Provenance documented** for every source: where it came from, who labelled it, how
- [ ] Limitations stated explicitly, including if data is LLM-generated or synthetic
- [ ] Label distribution reported; imbalance acknowledged rather than hidden
- [ ] Dataset size is sufficient for the fine-tuning budget — verified against the trainer's own reported update count, not guessed

**Validation**
```bash
python -m finetune.dataset_build --validate
pytest tests/unit/test_dataset_splits.py -q
```

**If `PROMPT.md` §19.2 has no credible answer, stop here. Descope Phase B per §6 and say so in the report.** An honestly descoped project beats a fabricated dataset.

---

### M14 — Fine-tuning

**Build**
- `finetune/` notebook or script — RLCD loop, temperature calibration, evaluation, export
- Fine-tuned acuity and red-flag checkpoints

**Acceptance**
- [ ] Fine-tuning runs end to end and exports a checkpoint
- [ ] Offline inference from the export reproduces training-time outputs
- [ ] Temperature calibration is applied and recorded
- [ ] Training log reports optimizer-update count and it is not silently tiny
- [ ] Checkpoint id, base checkpoint, and data version are all recorded

**Validation**
```bash
python -m finetune.run --eval --report out/finetune_eval.md
```

---

### M15 — Before/after comparison · **Phase B complete**

**Build**
- `out/comparison_report.md` — baseline vs fine-tuned, same holdout set

**Acceptance**
- [ ] Both systems evaluated on the **same** holdout set, same harness, same metrics
- [ ] Improvement reported honestly, **in either direction**
- [ ] Red-flag recall still 100% after fine-tuning
- [ ] Thresholds recalibrated for the fine-tuned checkpoint and re-tested — not inherited
- [ ] `PROMPT.md` §16.6 checks all pass

**Validation**
```bash
python -m evals.run_eval --set holdout --checkpoint finetuned --report out/holdout_finetuned.md
make evidence
make validate
```

---

## 3. Validation command contract

Every gate is a `make` target. `make validate` runs them in order and stops at the first failure.

| Command | Runs | Gates |
|---|---|---|
| `make install` | dependency install from pinned versions | — |
| `make lint` | `ruff check` | style |
| `make type` | `mypy src` | types |
| `make arch` | `python scripts/check_architecture.py` | §4 dependency contracts |
| `make test` | `pytest -q` | unit + integration + contract |
| `make eval` | all four eval sets, nonzero exit on blocking failure | §15.2 metrics |
| `make evidence` | checklist of every `PROMPT.md` §16 item with pass/fail | §16 |
| `make validate` | `lint → type → arch → test → eval` | everything |
| `make run` | start the service locally | — |

**Blocking metrics** (nonzero exit): red-flag recall, refusal correctness, escalation isolation, citation resolvability, injection resistance, banned-phrase violations, decision determinism, latency budgets.

---

## 4. Codebase architecture

### 4.1 Layout

```
medical-harness/
├── pyproject.toml              # pinned deps
├── Makefile
├── .env.example                # placeholders only
├── src/harness/
│   ├── contracts/              # ← single source of truth for all types
│   │   ├── decisions.py        #   Decision, Escalation, DecisionTrace, Abstention
│   │   ├── questions.py        #   schema loader, content hash, SCHEMA_VERSION
│   │   ├── request.py          #   HarnessRequest, HarnessResponse
│   │   └── errors.py           #   typed error taxonomy
│   ├── decision/               # LAYER 1 — System 1, decides
│   │   ├── client.py           #   DecisionClient protocol
│   │   ├── http_adapter.py     #   live Laya API
│   │   ├── fixture_adapter.py  #   offline replay
│   │   └── recorder.py
│   ├── rules/                  # LAYER 2 — pure, deterministic
│   │   ├── thresholds.py       #   versioned threshold set
│   │   └── engine.py           #   ← ONLY module allowed to build an Escalation
│   ├── tools/                  # LAYER 3 — facts only
│   │   ├── base.py             #   { facts, source_ids, source_type }
│   │   ├── labs.py
│   │   ├── drugs.py
│   │   ├── kb.py               #   RAG
│   │   ├── reports.py
│   │   └── registry.py         #   ← ONLY module allowed to grant permission
│   ├── generation/             # LAYER 2 — explains, never decides
│   │   ├── llm.py
│   │   ├── policy.py
│   │   ├── citations.py
│   │   └── schemas.py
│   ├── orchestrator.py         # ← the single wiring point
│   └── service/                # thin FastAPI edge
│       ├── app.py
│       └── routes.py
├── data/
│   ├── questions/              # versioned schema JSON
│   ├── tables/                 # curated lab + drug tables
│   ├── corpus/                 # RAG documents
│   ├── index/                  # precomputed embeddings
│   ├── fixtures/               # recorded Laya traces  ← tests run on these
│   └── evals/                  # eval case seeds
├── evals/
│   ├── sets/                   # scenarios | red_flags | refusals | holdout
│   ├── metrics.py
│   ├── run_eval.py
│   └── report.py
├── tests/
│   ├── unit/  integration/  contract/  security/
├── scripts/
│   ├── check_architecture.py   # zero-dependency AST import checker
│   ├── record_fixtures.py
│   └── validate.sh
└── finetune/                   # Phase B
    ├── dataset_build.py
    ├── run.py
    └── README.md
```

### 4.2 Dependency direction

Dependencies point **inward and downward only**. This is what makes `PROMPT.md` §16.1 structurally true rather than aspirational.

| Module | May import |
|---|---|
| `contracts` | stdlib, pydantic |
| `rules` | **only** `contracts` |
| `tools` | `contracts`, `tools.*` |
| `generation` | `contracts`, `tools` |
| `decision` | `contracts`, `decision.*` |
| `orchestrator` | everything |
| `service` | `contracts`, `orchestrator` |

**Hard prohibitions** — each is a testable rule:

1. `rules` must not import `decision`, `tools`, `generation`, or `orchestrator`. *This is what structurally guarantees no LLM or model call can occur inside the escalation path.*
2. `tools` must not import `rules`, `generation`, or `orchestrator`.
3. `generation` must not import `rules`, `decision`, or `orchestrator`.
4. Only `rules/engine.py` may instantiate `Escalation`.
5. Only `tools/registry.py` may grant tool permission.
6. Only `orchestrator.py` may import more than two layers.

Rule 1 is the important one. Combined with the raising-LLM-stub test at M8, escalation isolation is proven twice — once statically, once dynamically.

### 4.3 Architecture checker

`scripts/check_architecture.py` — **zero external dependencies**, stdlib `ast` only. An external linter adds version drift to a project whose determinism is a hard constraint.

It must:
- Walk every module under `src/harness/`, resolve imports to first-party modules
- Assert all six prohibitions in §4.2
- Exit `0` clean, `1` with a file, line number, and the offending import for each violation
- Run as `make arch`, and inside `make validate`

### 4.4 Test strategy

| Layer | Network | Laya | LLM |
|---|---|---|---|
| unit | none | fixture adapter | stub |
| integration | none | fixture adapter | stub / recorded |
| contract | none | both adapters | — |
| security | none | fixture adapter | stub |
| eval | none | fixture adapter | stub by default, live via flag |
| **live** | yes | live API | live API — **manual only** |

**No automated test touches the network.** Live checks are a separate manual target (`make live`) that is never part of `make validate`.

### 4.5 Fixture-first testing

Because Laya is a remote service with a moving version, the entire test suite runs against **recorded fixtures** committed to the repo.

- `scripts/record_fixtures.py` captures real Laya output once
- `--check` mode re-records and diffs, so model drift is *detectable* rather than silent
- Tests are deterministic and offline, and a Laya outage cannot break the build
- Live behaviour is validated separately, on demand

This is the single most important decision in the plan. Without it, every test depends on someone else's uptime and a model version nobody pinned.

---

## 5. Stop-and-fix rule

> **If a validation command fails, the next action is to fix the code — not the test, not the threshold, not the spec.**

### 5.1 Ordered response

1. **Read the actual failure.** No speculative edits.
2. **Fix the cause in the implementation.**
3. **Re-run the failing command.** Narrowest first.
4. **Re-run `make validate`.** The narrow pass proves nothing on its own.
5. **Commit only when the full gate is green.**

### 5.2 Forbidden repairs

These are the specific shortcuts that produce a project that looks done and is not:

| Forbidden | Why |
|---|---|
| Weakening an assertion to make it pass | The assertion encoded a requirement |
| Deleting or skipping a failing test | Hides the defect |
| Loosening a threshold in `rules/thresholds.py` to hit 100% | Thresholds are safety parameters, tuned on `red_flags` — never on `holdout`, never to make a number look good |
| Editing an eval case so the system passes it | That is overfitting to the test |
| Adding an "expected" field the system can read | Leaks the answer into the input |
| Adding a dependency to make a test pass | Unreviewed supply chain |
| Deleting a failing test from the suite | Same as skipping |

### 5.3 If validation fails for a legitimate reason

Some gates legitimately cannot pass yet — a blocking metric is 83%, not 100%. That is a **finding**, not a build error:

- Record it in `out/baseline_report.md` (M12)
- Do **not** lower the target in `PROMPT.md` §15.2 to make it green
- Fix it in Phase B, where the before/after report shows the movement

The only exception: if the target itself was **wrong**, amend the spec per §6.

---

## 6. Spec amendment procedure

`PROMPT.md` is amended only when a milestone proves the spec wrong — never to make a check pass.

1. **Stop** the milestone.
2. State the finding: what was assumed, what the evidence shows.
3. Amend `PROMPT.md` with the smallest possible change.
4. Append an entry to §7 below: decision, rationale, date, what it invalidates.
5. Re-run `make validate` to find everything the amendment broke.
6. Only then resume.

**Guarding against oscillation:** an amendment to a safety constraint (`PROMPT.md` §7, §9, §16) requires a written justification in §7 and cannot be reverted by the same milestone that introduced it. If a decision flips twice, it is escalated to a human rather than amended a third time.

---

## 7. Decision notes

**Append-only.** Do not re-litigate these. Each was decided deliberately; reversing one is a change, not a fix.

| # | Decision | Rationale | Do not |
|---|---|---|---|
| D1 | **Python 3.10+**, not TypeScript | Laya is Python-native; the RAG and evaluation ecosystem is Python. The harness is consumed over HTTP, so callers can be anything. | Do not port to TS for "full stack" tidiness |
| D2 | **Laya over HTTP**, not in-process | Stated requirement. Also keeps a 421M checkpoint out of the test process. | Do not inline the library |
| D3 | **Fixture-first testing** | Determinism and offline CI. A Laya outage must not break the build. | Do not call the live API in a test |
| D4 | **Pydantic v2** for contracts | One validation point at the boundary; serialisable for the trace. | Do not mix raw dataclasses into contracts |
| D5 | **No async / no queue in Phase A** | Every operation is short and synchronous. Async adds failure modes with no measured benefit at this scale. | Do not add Celery, Redis, or a worker pool |
| D6 | **numpy cosine search**, not Chroma/FAISS | The corpus is small. Brute force is exact, deterministic, and dependency-free. A vector DB is a non-goal. | Do not add a vector database |
| D7 | **Precomputed embeddings** in `data/index/` | Search becomes reproducible and offline. | Do not embed at query time |
| D8 | **Hash embedder for tests** | Tests must never download a model. | Do not let tests touch Sentence-Transformers |
| D9 | **Zero-dependency AST checker** | A dependency in the architecture gate is version drift in the one gate that must not drift. | Do not swap it for `import-linter` |
| D10 | **Structured separation of data and instruction channels** | Prompt-injection defence must be structural, not a prompt instruction. | Do not merge them and rely on wording |
| D11 | **Suicide-risk crisis text from config** | Never generated. | Do not let the LLM produce it |
| D12 | **Separate acuity and red-flag checkpoints** | Different tasks, different failure modes. A single degraded checkpoint must not take down both. | Do not merge them into one |
| D13 | **Thresholds versioned in one file** | They are safety parameters and must change as a unit, on the record. | Do not scatter them through the code |
| D14 | **Blocking metrics exit nonzero** | A gate that cannot fail is not a gate. | Do not make them warnings |
| D15 | **`holdout` used only for final reporting** | Any tuning on it invalidates it. | Do not tune against it, ever |

---

## 8. Definition of done

### Phase A complete — end of M12
- [ ] All M0–M12 acceptance criteria met
- [ ] `make validate` green
- [ ] `make evidence` shows every `PROMPT.md` §16.1–16.5 item passing
- [ ] `out/baseline_report.md` published with real numbers, including bad ones
- [ ] Demo flow (`PROMPT.md` §17) runs end to end

### Phase B complete — end of M15
- [ ] All M13–M15 acceptance criteria met
- [ ] `out/comparison_report.md` published, honest in either direction
- [ ] Red-flag recall still 100%
- [ ] `PROMPT.md` §16.6 checks pass

### Explicitly not done
No frontend. No authentication system. No diagnosis. No treatment advice. No real patient data. No clinical validation. No cloud deployment.