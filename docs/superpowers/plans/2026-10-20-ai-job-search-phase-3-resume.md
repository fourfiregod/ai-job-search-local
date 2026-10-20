# AI Job Search Phase 3 Resume and Interview Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Generate evidence-grounded Chinese resumes in PDF and DOCX plus interview preparation based on the exact version used for a selected job.

**Architecture:** The Codex Skill creates a structured resume manifest whose bullets cite candidate claim IDs. A deterministic audit rejects unsupported or incorrectly promoted claims before rendering; separate PDF and DOCX renderers consume the accepted manifest.

**Tech Stack:** Python 3.10+, JSON, `python-docx`, XeLaTeX/ctex, existing PDF verification tools, `unittest`, Codex Skills

**Spec:** `docs/superpowers/specs/2026-10-20-ai-job-search-localization-design.md`

## Global Constraints

- The main resume uses evidence levels A and B only.
- POC/演示不得被写成生产上线，也不得添加未经验证的业务成效。
- No old internship or DarkDuck content appears unless the user explicitly requests a historical variant.
- Default structure is one-page Chinese: header, summary, verified skills, Hongji work experience with three selected projects, education.
- Produce R&D, R&D-delivery, and solution-delivery variants from the same facts.
- Every generated bullet cites at least one known claim ID in the private audit manifest.
- Generated files and application records remain gitignored.

## Review Focus

- A manifest cites an unknown or level C/D claim: Task 1 rejects it before rendering.
- A POC claim is rewritten with “上线” or unverified metrics: Task 1 rejects status promotion and metric drift.
- Chinese text or long company/role names overflow: Task 2 compiles and visually verifies fixtures.
- XeLaTeX or `python-docx` is missing: Task 2 returns a clear dependency error without deleting source files.
- Interview prep uses a different resume than the one submitted: Task 4 requires the archived manifest and rendered filenames.

---

### Task 1: Resume Manifest and Truth Audit

**Files:**
- Create: `tools/local_resume/__init__.py`
- Create: `tools/local_resume/models.py`
- Create: `tools/local_resume/audit.py`
- Create: `tests/fixtures/resume/`
- Create: `tests/test_resume_truth_audit.py`

**Interfaces:**
- Consumes: `candidate/skills-evidence.md` claim markers and a generated JSON manifest.
- Produces: `audit_manifest(manifest: ResumeManifest, evidence_text: str) -> AuditReport`; rendering is allowed only when `AuditReport.ok` is true.

- [ ] **Step 1: Write failing audit tests**

```python
def test_unknown_claim_is_rejected():
    report = audit_manifest(manifest(claim_ids=["missing.claim"]), EVIDENCE)
    assert not report.ok and "missing.claim" in report.errors[0]

def test_level_c_or_d_claim_is_rejected_for_main_resume():
    assert not audit_manifest(manifest(claim_ids=["darkduck.rag"]), EVIDENCE).ok

def test_poc_cannot_be_promoted_to_production():
    report = audit_manifest(manifest(text="项目已正式上线", claim_ids=["work.bid.poc"]), EVIDENCE)
    assert any("status promotion" in error for error in report.errors)
```

- [ ] **Step 2: Run tests and verify failure**

Run: `python -m unittest tests.test_resume_truth_audit -v`

Expected: FAIL.

- [ ] **Step 3: Implement manifest models and audit rules**

Manifest sections are `header`, `summary`, `skills`, `experience`, and `education`. Every summary, skill, and experience bullet carries `claim_ids`; education carries its source claim. Metric tokens not present in cited claims are rejected.

- [ ] **Step 4: Run tests and commit**

```powershell
python -m unittest tests.test_resume_truth_audit -v
git add tools/local_resume tests/fixtures/resume tests/test_resume_truth_audit.py
git commit -m "feat: audit resume claims before rendering"
```

### Task 2: Chinese PDF and DOCX Rendering

**Files:**
- Modify: `requirements-local.txt`
- Modify: `.gitignore`
- Create: `templates/local/resume_zh.tex`
- Create: `tools/local_resume/render_pdf.py`
- Create: `tools/local_resume/render_docx.py`
- Create: `tests/test_resume_renderers.py`

**Interfaces:**
- Consumes: an audited `ResumeManifest`.
- Produces: `render_pdf_source(manifest, template_path, output_tex) -> Path`, `compile_pdf(tex_path, output_dir) -> Path`, and `render_docx(manifest, output_path) -> Path`.

- [ ] **Step 1: Add the local rendering dependency and verification-output ignore rule**

Add `python-docx>=1.1,<2` to the Phase 1 `requirements-local.txt` and ignore `tests/.artifacts/**`; do not change upstream runtime dependencies.

- [ ] **Step 2: Write failing renderer tests**

Tests use `tests/.artifacts/resume_zh.pdf` as an ignored verification output and assert Chinese text remains extractable, output paths stay under the requested directory, long fixture text does not create more than one page, missing dependencies return named remediation, and source files survive compilation failure.

- [ ] **Step 3: Run tests and verify failure**

Run: `python -m unittest tests.test_resume_renderers -v`

Expected: FAIL.

- [ ] **Step 4: Implement the single-column Chinese template and renderers**

Use XeLaTeX with `ctex` for the PDF. Use `python-docx` for DOCX. Escape LaTeX metacharacters centrally; do not let the Skill write raw unescaped user strings into the template.

- [ ] **Step 5: Verify rendered fixtures**

Run:

```powershell
python -m unittest tests.test_resume_renderers -v
python tools/verify_pdf.py tests/.artifacts/resume_zh.pdf
```

Expected: PASS; extracted text contains all fixture headings and the PDF is one page. Render the PDF to PNG and visually inspect for clipping, overlap, font substitution, and broken CJK glyphs.

- [ ] **Step 6: Commit**

```powershell
git add requirements-local.txt templates/local/resume_zh.tex tools/local_resume tests/test_resume_renderers.py .gitignore
git commit -m "feat: render verified Chinese resumes"
```

### Task 3: Resume Tailoring Codex Skill

**Files:**
- Create: `.codex/skills/resume-tailor/SKILL.md`
- Create: `.codex/skills/resume-tailor/references/variant-rules.md`
- Create: `.codex/skills/resume-tailor/references/manifest-schema.md`
- Create: `.codex/skills/resume-tailor/references/review-checklist.md`
- Create: `tests/test_codex_resume_skill.py`
- Modify: `.gitignore`
- Modify: `tools/security_guards.py`

**Interfaces:**
- Consumes: selected `JobRecord`, candidate files, audit API, and render APIs.
- Produces: private `outputs/<company>_<role>_<date>/resume_manifest.json`, audit report, `.tex`, `.pdf`, and `.docx`.

- [ ] **Step 1: Write failing Skill tests**

Assert the Skill supports the three variants; defaults to work-only; forbids internship/DarkDuck; runs audit before rendering; stops on audit errors; and creates an interview follow-up list of claims likely to be challenged.

- [ ] **Step 2: Run tests and verify failure**

Run: `python -m unittest tests.test_codex_resume_skill -v`

Expected: FAIL.

- [ ] **Step 3: Implement Skill and references**

The Skill selects and reorders known facts, never mutates the private evidence file, and always presents the audit summary beside deliverables. Output filenames must include normalized company, role, and ISO date.

- [ ] **Step 4: Extend privacy rules**

Ignore `outputs/**` and generated resume artifacts while retaining tracked template sources and test fixtures. Add these patterns to `tools/security_guards.py`.

- [ ] **Step 5: Run tests and commit**

```powershell
python -m unittest tests.test_codex_resume_skill tests.test_local_privacy tests.test_security_guards -v
python tools/lint_skills.py
git add .codex/skills/resume-tailor .gitignore tools/security_guards.py tests/test_codex_resume_skill.py
git commit -m "feat: add evidence-grounded resume tailoring skill"
```

### Task 4: Interview Preparation and Manual Application Record

**Files:**
- Create: `.codex/skills/interview-prep/SKILL.md`
- Create: `.codex/skills/interview-prep/references/chinese-ai-interview.md`
- Create: `tools/local_resume/application_record.py`
- Create: `tests/test_interview_prep_skill.py`
- Create: `tests/test_application_record.py`

**Interfaces:**
- Consumes: selected job, archived resume manifest, audit report, and rendered filenames.
- Produces: `record_application(path: Path, record: ApplicationRecord) -> None` and a prep pack grounded in exactly the submitted resume version.

- [ ] **Step 1: Write failing application-record tests**

Pin UTF-8 CSV safety, formula-injection escaping, allowed manual statuses, immutable resume-manifest path, and dedupe by company-role-application date.

- [ ] **Step 2: Write failing interview Skill tests**

Assert the Skill reads the archived manifest, asks questions about every high-risk claim, distinguishes POC from production, and never introduces C/D evidence into suggested answers.

- [ ] **Step 3: Run tests and verify failure**

Run: `python -m unittest tests.test_application_record tests.test_interview_prep_skill -v`

Expected: FAIL.

- [ ] **Step 4: Implement the record helper and interview Skill**

The Skill produces project deep-dive questions, truthful gap-bridging answers, and a rehearsal checklist. It does not send messages or update external systems.

- [ ] **Step 5: Run tests and commit**

```powershell
python -m unittest tests.test_application_record tests.test_interview_prep_skill tests.test_tracker_notes_csv_safe -v
python tools/lint_skills.py
git add .codex/skills/interview-prep tools/local_resume/application_record.py tests/test_application_record.py tests/test_interview_prep_skill.py
git commit -m "feat: ground interview prep in submitted resume"
```

### Task 5: Resume-to-Interview End-to-End Verification

**Files:**
- Create: `tests/test_local_application_pipeline.py`
- Create: `tests/fixtures/application/ai-agent-poc-job.json`
- Modify if required by evidence: `README.md`

**Interfaces:**
- Consumes: all Phase 3 tasks and the Phase 2 job schema.
- Produces: one fixture-based, reproducible job-to-resume-to-interview workflow.

- [ ] **Step 1: Write the failing end-to-end test**

Use a Beijing AI Agent POC job fixture. Assert selection of the bidding-agent, medical-POC, and Xinjiang-RPA work claims; exclusion of internship/DarkDuck; audit pass; PDF/DOCX creation; and interview questions tied to the emitted bullets.

- [ ] **Step 2: Run the test and verify failure**

Run: `python -m unittest tests.test_local_application_pipeline -v`

Expected: FAIL until orchestration is connected.

- [ ] **Step 3: Add only the missing orchestration**

Compose existing job, audit, render, and record interfaces. Do not duplicate scoring or claim-validation logic.

- [ ] **Step 4: Run Phase 3 and full regression verification**

```powershell
python -m unittest tests.test_resume_truth_audit tests.test_resume_renderers tests.test_codex_resume_skill tests.test_application_record tests.test_interview_prep_skill tests.test_local_application_pipeline -v
python -m unittest discover -s tests -v
python tools/lint_skills.py
git status --short
```

Expected: all tests PASS; only intended tracked changes are present; generated private artifacts are ignored.

- [ ] **Step 5: Commit**

```powershell
git add tests/test_local_application_pipeline.py tests/fixtures/application README.md
git commit -m "test: verify local application workflow"
```
