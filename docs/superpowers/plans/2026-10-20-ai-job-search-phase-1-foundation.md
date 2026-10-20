# AI Job Search Phase 1 Foundation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create a private, Codex-native local overlay for the upstream project and populate a source-aware candidate profile without exposing personal data to Git.

**Architecture:** Keep `MadsLorentzen/ai-job-search` as the upstream baseline. Add repo-scoped Codex Skills plus a gitignored `candidate/` runtime area; tracked templates, validators, and tests define the contract while the user's real profile remains local.

**Tech Stack:** Git, Markdown, YAML, Python 3.10+, `unittest`, Codex repo-scoped Skills

**Spec:** `docs/superpowers/specs/2026-10-20-ai-job-search-localization-design.md`

## Global Constraints

- Never create or push a public fork containing personal data.
- Real files under `candidate/` are local-only; only templates and documentation are tracked.
- Latest direct user confirmation overrides Notion, old resumes, and learning plans.
- Evidence levels are exactly A, B, C, and D; the main resume may later consume only A and B.
- Current search preferences are Beijing and Shanghai, expected salary 15k-18k, minimum 12k.
- No login, cookie storage, application prefill, or automatic submission.
- Preserve the upstream thin-pointer pattern; local Codex Skills may reference canonical upstream files but must not duplicate large workflows.

## Review Focus

- A real candidate file is accidentally trackable: `tests/test_local_privacy.py` must fail unless every private path is ignored.
- The checkout still points to a public fork as `origin`: Task 1 must rename the public baseline to `upstream` and leave no push target.
- Two sources disagree on a fact: `tests/test_candidate_contract.py` must pin the declared source precedence.
- A claim has no source or evidence level: the validator must reject it with the claim identifier.
- A generic coding request triggers the candidate Skill: `tests/test_codex_candidate_skill.py` must include negative trigger examples.

---

### Task 1: Local Checkout and Privacy Boundary

**Files:**
- Create: `docs/superpowers/specs/2026-10-20-ai-job-search-localization-design.md`
- Modify: `.gitignore`
- Modify: `AGENTS.md`
- Modify: `tools/security_guards.py`
- Create: `tests/test_local_privacy.py`

**Interfaces:**
- Consumes: upstream `master` and the approved design document.
- Produces: a local Git checkout with remote `upstream`, no writable public `origin`, and machine-checked private-path rules.

- [ ] **Step 1: Prepare the target checkout without creating a public fork**

Run from the chosen local workspace parent directory:

```powershell
git clone https://github.com/MadsLorentzen/ai-job-search.git ai-job-search-local
Set-Location ai-job-search-local
git remote rename origin upstream
git remote -v
```

Expected: only `upstream` exists and both URLs point to `MadsLorentzen/ai-job-search`.

- [ ] **Step 2: Copy the approved design into the checkout**

Copy the approved spec to `docs/superpowers/specs/2026-10-20-ai-job-search-localization-design.md` without modifying its content.

- [ ] **Step 3: Write failing privacy tests**

Create `tests/test_local_privacy.py` with assertions equivalent to:

```python
def test_private_candidate_paths_are_ignored():
    assert is_ignored("candidate/profile.md")
    assert is_ignored("candidate/work-experience.md")
    assert is_ignored("candidate/skills-evidence.md")
    assert is_ignored("candidate/preferences.yaml")
    assert is_ignored("candidate/history/old-resume.pdf")

def test_agents_declares_local_profile_precedence():
    text = Path("AGENTS.md").read_text(encoding="utf-8")
    assert "candidate/" in text
    assert "latest direct user confirmation" in text.lower()
```

- [ ] **Step 4: Run the tests and verify failure**

Run: `python -m unittest tests.test_local_privacy -v`

Expected: FAIL because the candidate paths and local overlay rules do not exist.

- [ ] **Step 5: Implement the privacy boundary**

Add exact ignore rules for real candidate files and history contents while re-including `candidate/README.md`, `candidate/templates/**`, and `.gitkeep`. Add the same required paths to `tools/security_guards.py`. Update `AGENTS.md` so the local `candidate/` snapshot overrides upstream profile placeholders and direct user confirmation is highest priority.

- [ ] **Step 6: Verify tests and repository state**

Run:

```powershell
python -m unittest tests.test_local_privacy tests.test_security_guards -v
git status --short
git remote -v
```

Expected: PASS; no private candidate file appears; no writable public `origin` exists.

- [ ] **Step 7: Commit**

```powershell
git add .gitignore AGENTS.md tools/security_guards.py tests/test_local_privacy.py docs/superpowers/specs/2026-10-20-ai-job-search-localization-design.md
git commit -m "chore: establish private local job search overlay"
```

### Task 2: Candidate File Contract and Validator

**Files:**
- Create: `requirements-local.txt`
- Create: `candidate/README.md`
- Create: `candidate/templates/profile.example.md`
- Create: `candidate/templates/work-experience.example.md`
- Create: `candidate/templates/skills-evidence.example.md`
- Create: `candidate/templates/preferences.example.yaml`
- Create: `tools/validate_candidate.py`
- Create: `tests/fixtures/candidate-valid/`
- Create: `tests/test_candidate_contract.py`

**Interfaces:**
- Consumes: privacy rules from Task 1.
- Produces: `read_preferences(path: Path) -> dict[str, Any]` and `validate_candidate(root: Path) -> list[str]`, where an empty list means valid; claim markers follow `[claim:<id>] [level:<A|B|C|D>] [source:<kind>] [status:<value>]`.

- [ ] **Step 1: Write failing contract tests**

```python
def test_valid_candidate_fixture_has_no_errors():
    assert validate_candidate(FIXTURE) == []

def test_claim_without_source_is_rejected(tmp_path):
    errors = validate_candidate(make_fixture(tmp_path, claim="[claim:x] [level:A]"))
    assert any("x" in error and "source" in error for error in errors)

def test_preferences_pin_current_targets():
    prefs = read_preferences(FIXTURE / "preferences.yaml")
    assert prefs["cities"] == ["北京", "上海"]
    assert prefs["salary"]["minimum_monthly_k"] == 12
    assert prefs["salary"]["expected_monthly_k"] == [15, 18]
```

- [ ] **Step 2: Run the tests and verify failure**

Run: `python -m unittest tests.test_candidate_contract -v`

Expected: FAIL because the validator and templates do not exist.

- [ ] **Step 3: Add the local YAML dependency**

Add `PyYAML>=6.0,<7` to `requirements-local.txt`; do not change upstream runtime dependencies.

- [ ] **Step 4: Implement templates, `read_preferences(path: Path) -> dict[str, Any]`, and `validate_candidate(root: Path) -> list[str]`**

Use `yaml.safe_load`. The validator must check required files, required preference keys, allowed evidence levels, unique claim IDs, non-empty sources, and allowed project statuses. It must not read outside the supplied root.

- [ ] **Step 5: Run contract and UTF-8 tests**

Run: `python -m unittest tests.test_candidate_contract tests.test_tools_utf8_output -v`

Expected: PASS, including Chinese fixture content.

- [ ] **Step 6: Commit**

```powershell
git add requirements-local.txt candidate/README.md candidate/templates tools/validate_candidate.py tests/fixtures/candidate-valid tests/test_candidate_contract.py
git commit -m "feat: define evidence-aware candidate profile contract"
```

### Task 3: Codex Candidate Profile Skill

**Files:**
- Create: `.codex/skills/candidate-profile/SKILL.md`
- Create: `.codex/skills/candidate-profile/references/source-precedence.md`
- Create: `.codex/skills/candidate-profile/references/profile-workflow.md`
- Create: `tests/test_codex_candidate_skill.py`
- Modify: `tools/lint_skills.py`

**Interfaces:**
- Consumes: candidate templates and `python tools/validate_candidate.py candidate`.
- Produces: a repo-scoped `candidate-profile` Skill that can import Notion/old-resume facts, resolve conflicts conservatively, and update only gitignored runtime files.

- [ ] **Step 1: Write failing Skill contract tests**

```python
def test_skill_has_precise_frontmatter():
    assert frontmatter["name"] == "candidate-profile"
    assert "Notion" in frontmatter["description"]

def test_skill_never_promotes_learning_goals_to_experience():
    assert "learning goals" in skill_text.lower()
    assert "evidence level D" in skill_text

def test_negative_triggers_are_documented():
    assert "ordinary coding" in workflow_text.lower()
```

- [ ] **Step 2: Run the tests and verify failure**

Run: `python -m unittest tests.test_codex_candidate_skill -v`

Expected: FAIL because the Skill does not exist.

- [ ] **Step 3: Implement the Skill and references**

`SKILL.md` must stay procedural and link to the two references. The workflow must use this precedence: direct user confirmation, current Notion work records, verified work materials, old resume, learning plans. Conflicts are recorded; missing metrics use conservative wording; the Skill runs the validator before reporting success.

- [ ] **Step 4: Extend Skill linting to scan `.codex/skills`**

Update `tools/lint_skills.py` without weakening existing `.agents/skills` or `.claude/skills` checks.

- [ ] **Step 5: Run targeted and full Skill tests**

Run:

```powershell
python -m unittest tests.test_codex_candidate_skill tests.test_lint_skills -v
python tools/lint_skills.py
```

Expected: PASS.

- [ ] **Step 6: Commit**

```powershell
git add .codex/skills/candidate-profile tools/lint_skills.py tests/test_codex_candidate_skill.py
git commit -m "feat: add Codex candidate profile workflow"
```

### Task 4: Populate and Verify the Private Candidate Snapshot

**Files:**
- Create locally, ignored: `candidate/profile.md`
- Create locally, ignored: `candidate/work-experience.md`
- Create locally, ignored: `candidate/skills-evidence.md`
- Create locally, ignored: `candidate/preferences.yaml`
- Create locally, ignored: `candidate/history/old-resume.pdf`

**Interfaces:**
- Consumes: the approved Notion facts, user corrections, and old resume.
- Produces: a valid private candidate snapshot for Phases 2 and 3; no tracked file changes.

- [ ] **Step 1: Run the `candidate-profile` Skill against the approved sources**

Populate the three selected work projects, current job, education, cities, salary, routes, and three-to-six-month timeline. Store old internship and DarkDuck only under history/C-or-D evidence.

- [ ] **Step 2: Validate the private snapshot**

Run: `python tools/validate_candidate.py candidate`

Expected: exit code 0 and `candidate profile valid`.

- [ ] **Step 3: Verify privacy**

Run:

```powershell
git status --short
git check-ignore -v candidate/profile.md candidate/work-experience.md candidate/skills-evidence.md candidate/preferences.yaml candidate/history/old-resume.pdf
```

Expected: every private path is ignored and none appears in `git status`.

- [ ] **Step 4: Run the Phase 1 suite**

Run: `python -m unittest tests.test_local_privacy tests.test_candidate_contract tests.test_codex_candidate_skill tests.test_security_guards -v`

Expected: PASS.

- [ ] **Step 5: Record the operational checkpoint**

No commit is created for private files. Record the passing commands and current local commit SHA in the implementation handoff only.
