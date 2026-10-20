# AI Job Search Phase 2 Discovery and Ranking Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Discover public Beijing/Shanghai AI jobs, normalize and deduplicate them, enforce hard filters, and produce explainable route-specific rankings.

**Architecture:** A deterministic Python pipeline owns job schemas, salary parsing, deduplication, filtering, state, and score validation. A Codex Skill owns semantic search and evidence-grounded scoring, using public pages or pasted JD text and degrading safely when portals block access.

**Tech Stack:** Python 3.10+, JSONL, `unittest`, Codex Skills, public web search

**Spec:** `docs/superpowers/specs/2026-10-20-ai-job-search-localization-design.md`

## Global Constraints

- Search BOSS, Liepin, Lagou, Nowcoder, Zhaopin, and company career sites using public information only.
- Never log in, store cookies, bypass anti-bot controls, or auto-submit.
- Cities are Beijing and Shanghai; explicit monthly salary below 12k is excluded.
- Unknown salary remains eligible with `salary_pending=true`.
- Requirements of 3+ years lower rank but do not hard-delete the job.
- Ranking weights are exactly 25/25/20/10/10/10 for direction/evidence/tech/experience/salary/growth.
- Every recommendation cites matching evidence and lists gaps.

## Review Focus

- Chinese salary strings such as `15-25K·13薪`, `1.5-2万`, `面议`, and yearly packages: Task 1 tests exact normalization.
- The same job arrives from two platforms with tracking parameters: Task 2 tests URL and company-title-city dedupe.
- A Beijing role says “全国远程” or lists multiple cities: Task 2 tests accepted-location semantics.
- A portal is blocked or returns only a snippet: Task 4 tests degraded records and manual-JD fallback.
- A score exceeds a dimension cap or lacks evidence: Task 3 rejects the score card instead of silently clipping it.

---

### Task 1: Job Record and Salary Normalization

**Files:**
- Create: `tools/local_jobs/__init__.py`
- Create: `tools/local_jobs/models.py`
- Create: `tools/local_jobs/normalize.py`
- Create: `tests/test_local_job_normalize.py`

**Interfaces:**
- Consumes: raw portal/search mappings.
- Produces: `Preferences`, `normalize_job(raw: Mapping[str, Any], source: str, discovered_at: datetime) -> JobRecord`, `parse_salary(text: str | None) -> SalaryRange`, and `load_preferences(path: Path) -> Preferences`.

- [ ] **Step 1: Write failing model and salary tests**

```python
def test_parse_monthly_salary():
    assert parse_salary("15-25K·13薪").monthly_k == (15, 25)
    assert parse_salary("1.5-2万").monthly_k == (15, 20)

def test_unknown_salary_is_pending():
    assert parse_salary("面议").pending is True

def test_normalize_requires_provenance(self):
    with self.assertRaisesRegex(ValueError, "source"):
        normalize_job({"title": "AI应用开发"}, "", NOW)
```

- [ ] **Step 2: Run tests and verify failure**

Run: `python -m unittest tests.test_local_job_normalize -v`

Expected: FAIL because the package does not exist.

- [ ] **Step 3: Implement immutable data models and normalization**

Define `SalaryRange`, `JobRecord`, `SourceRef`, and `Preferences` dataclasses. `load_preferences` must use the safe YAML reader from Phase 1 and validate cities plus salary fields. Preserve the raw salary string and raw source payload path; normalize whitespace and canonical URLs without discarding the original URL.

- [ ] **Step 4: Run tests**

Run: `python -m unittest tests.test_local_job_normalize -v`

Expected: PASS.

- [ ] **Step 5: Commit**

```powershell
git add tools/local_jobs tests/test_local_job_normalize.py
git commit -m "feat: normalize Chinese AI job records"
```

### Task 2: Deduplication and Hard Filters

**Files:**
- Create: `tools/local_jobs/dedupe.py`
- Create: `tools/local_jobs/filters.py`
- Create: `tests/test_local_job_dedupe.py`
- Create: `tests/test_local_job_filters.py`

**Interfaces:**
- Consumes: `JobRecord` and candidate preferences.
- Produces: `dedupe_jobs(records: Iterable[JobRecord]) -> list[JobRecord]` and `filter_job(job: JobRecord, preferences: Preferences) -> FilterDecision`.

- [ ] **Step 1: Write failing dedupe tests**

Assert that canonical URL matches merge first; otherwise normalized company + title + city merges records while preserving every source reference.

- [ ] **Step 2: Write failing filter tests**

```python
def test_below_floor_is_excluded():
    assert filter_job(job(salary=(8, 10)), PREFS).reason == "salary_below_minimum"

def test_unknown_salary_is_kept():
    decision = filter_job(job(salary=None), PREFS)
    assert decision.keep and decision.flags == ("salary_pending",)

def test_three_year_requirement_is_flag_not_veto():
    decision = filter_job(job(required_years=3), PREFS)
    assert decision.keep and "experience_stretch" in decision.flags
```

- [ ] **Step 3: Run tests and verify failure**

Run: `python -m unittest tests.test_local_job_dedupe tests.test_local_job_filters -v`

Expected: FAIL.

- [ ] **Step 4: Implement dedupe and filters**

Do not infer salary or city when absent. Mark stale or inaccessible jobs separately from hard exclusion so a still-live old posting can remain reviewable.

- [ ] **Step 5: Run tests and commit**

```powershell
python -m unittest tests.test_local_job_dedupe tests.test_local_job_filters -v
git add tools/local_jobs tests/test_local_job_dedupe.py tests/test_local_job_filters.py
git commit -m "feat: deduplicate and filter local job results"
```

### Task 3: Explainable Score Cards and JSONL State

**Files:**
- Create: `tools/local_jobs/scoring.py`
- Create: `tools/local_jobs/state.py`
- Create: `tools/local_jobs/cli.py`
- Create: `tests/test_local_job_scoring.py`
- Create: `tests/test_local_job_state.py`

**Interfaces:**
- Consumes: filtered `JobRecord` objects and model-produced dimension assessments with claim IDs.
- Produces: `validate_score_card(card: Mapping[str, Any]) -> ScoreCard`, `append_jobs(path: Path, jobs: Iterable[JobRecord]) -> AppendResult`, and CLI commands `normalize`, `dedupe`, `filter`, `validate-score`, `list`.

- [ ] **Step 1: Write failing scoring tests**

```python
def test_exact_weight_caps():
    assert SCORE_CAPS == {
        "direction": 25, "evidence": 25, "technology": 20,
        "experience": 10, "salary": 10, "growth": 10,
    }

def test_score_requires_evidence_and_gaps(self):
    with self.assertRaises(ValueError):
        validate_score_card({"scores": VALID_SCORES, "evidence": [], "gaps": []})
```

- [ ] **Step 2: Write failing JSONL state tests**

Pin UTF-8 output, atomic replacement, deterministic keys, merge-without-source-loss, and recovery from one malformed trailing line.

- [ ] **Step 3: Run tests and verify failure**

Run: `python -m unittest tests.test_local_job_scoring tests.test_local_job_state -v`

Expected: FAIL.

- [ ] **Step 4: Implement score validation, state, and CLI**

Reject out-of-range values and unknown routes. Recommendation thresholds are: `>=80` priority, `65-79` stretch, `50-64` cautious, `<50` not recommended. Scores never replace the written evidence and gap explanation.

- [ ] **Step 5: Run tests and commit**

```powershell
python -m unittest tests.test_local_job_scoring tests.test_local_job_state -v
git add tools/local_jobs tests/test_local_job_scoring.py tests/test_local_job_state.py
git commit -m "feat: add explainable job ranking state"
```

### Task 4: China Job Search Codex Skill

**Files:**
- Create: `.codex/skills/job-search/SKILL.md`
- Create: `.codex/skills/job-search/references/china-portals.md`
- Create: `.codex/skills/job-search/references/scoring-rubric.md`
- Create: `.codex/skills/job-search/references/output-format.md`
- Create: `tests/fixtures/jobs/`
- Create: `tests/test_codex_job_search_skill.py`

**Interfaces:**
- Consumes: private candidate files and `python -m tools.local_jobs.cli`.
- Produces: natural-language flows for search, normalization, ranking, and manual-JD fallback; writes private state under `data/jobs/`.

- [ ] **Step 1: Write failing Skill tests**

Tests must assert the six sources are named, login and anti-bot bypass are forbidden, blocked-page fallback requests pasted JD text, the three routes are supported, and every output includes link plus discovered time.

- [ ] **Step 2: Run tests and verify failure**

Run: `python -m unittest tests.test_codex_job_search_skill -v`

Expected: FAIL.

- [ ] **Step 3: Implement the Skill and references**

Portal guidance must use public search queries and official career pages. It must store degraded records when only title/company/link are known, never manufacture missing JD fields, and run the posting integrity guard before semantic analysis.

- [ ] **Step 4: Update privacy guards for runtime job data**

Add `data/jobs/**` to `.gitignore` and `tools/security_guards.py`, while allowing tracked fixture data only under `tests/fixtures/jobs/`.

- [ ] **Step 5: Run tests and commit**

```powershell
python -m unittest tests.test_codex_job_search_skill tests.test_local_privacy tests.test_posting_guard -v
python tools/lint_skills.py
git add .codex/skills/job-search .gitignore tools/security_guards.py tests/test_codex_job_search_skill.py tests/fixtures/jobs
git commit -m "feat: add China AI job search workflow"
```

### Task 5: Discovery End-to-End Verification

**Files:**
- Create: `tests/test_local_job_pipeline.py`
- Create: `tests/fixtures/jobs/mixed-search-results.json`
- Modify if required by evidence: `README.md`

**Interfaces:**
- Consumes: Tasks 1-4.
- Produces: a reproducible fixture-based search-to-shortlist verification and documented natural-language examples.

- [ ] **Step 1: Write the failing pipeline test**

Fixture must include a duplicate, Shanghai role, Beijing role, wrong-city role, below-floor salary, unknown salary, 3-year stretch, malformed salary, and blocked-page record. Assert the final shortlist and flags exactly.

- [ ] **Step 2: Run the test and verify failure**

Run: `python -m unittest tests.test_local_job_pipeline -v`

Expected: FAIL until the CLI orchestration covers the complete flow.

- [ ] **Step 3: Add the minimal orchestration needed for the fixture flow**

Expose one CLI `pipeline --input <json> --output <jsonl> --preferences <yaml>` that composes existing functions; do not duplicate their logic.

- [ ] **Step 4: Run Phase 2 verification**

```powershell
python -m unittest tests.test_local_job_normalize tests.test_local_job_dedupe tests.test_local_job_filters tests.test_local_job_scoring tests.test_local_job_state tests.test_codex_job_search_skill tests.test_local_job_pipeline -v
python tools/lint_skills.py
```

Expected: PASS.

- [ ] **Step 5: Commit**

```powershell
git add tools/local_jobs tests/test_local_job_pipeline.py tests/fixtures/jobs README.md
git commit -m "test: verify local job discovery pipeline"
```
