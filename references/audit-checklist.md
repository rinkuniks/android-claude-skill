# Android Code Audit & Refactoring Checklist

Use this when auditing or refactoring **existing** Android/Compose/KMP code (as opposed to scaffolding new features — see main SKILL.md for that workflow). Acts as Principal Android Engineer + Security Auditor: exhaustive static audit, prioritized TODO matrix, sequential refactor to zero technical debt without regressions.

---

## 0. Scope Sizing (run before anything else)

- **Single file / small diff (<500 lines):** full inline read, audit directly.
- **Module or feature folder:** read module boundary first (`build.gradle.kts`, package root), then scan files within it. Do not pull in unrelated modules.
- **Whole repo / large monorepo:** do NOT read every file inline. Prioritize by risk — entry points (Activities, ViewModels, DI modules, network/storage layers) before leaf UI files. Use grep/search for known-risk patterns (`!!`, `SharedPreferences`, `GlobalScope`, `Log.d`, `static.*Context`) to shortlist files before deep reading. State explicitly which files/modules were skipped and why.
- If asked to audit "the codebase" with no path given, ask for a path or diff scope before proceeding — do not guess.

---

## 1. Audit Dimensions (all 10 required)

### A. Google Android Coding Standards & Style
- Language coverage: Java, XML, Kotlin, Compose, KMP.
- Exact 4-space indentation across all source files.
- Remove unused imports, dead variables, redundant methods, obsolete XML attributes.
- Replace verbose anonymous inner classes / manual loops with lambdas, `let`/`apply`/`also`/`run`/`with`, sequences, stdlib extensions.
- **Tooling cross-check:** if ktlint/detekt/Android Lint are configured, prefer their output (or config/baseline files) over manual style scanning — faster and more reliable. Note discrepancies.
- **Deprecated APIs & dependencies:** flag `AsyncTask`, legacy `Handler()` constructor, `onActivityResult`, and outdated AGP/Gradle/library versions in `build.gradle.kts` / `libs.versions.toml`.

### B. Memory Management & Leak Prevention
- Static references to `Context`, `View`, or `Activity`.
- Handlers, callbacks, RxJava subscriptions, Coroutine jobs tied to `LifecycleOwner` scope (`viewLifecycleOwner.lifecycleScope`, `repeatOnLifecycle`).
- Bitmap allocation/recycling, custom view lifecycle cleanup.

### C. Performance, Optimization & ANR Prevention
- Zero blocking operations (I/O, Room queries, heavy calc, JSON parsing) on Main Thread. Enforce `Dispatchers.IO` / `Dispatchers.Default` offload.
- Compose: `@Stable`/`@Immutable` models, `derivedStateOf` / lambda keys for unstable params, `remember` for heavy in-composable computation.
- KMP: correct platform dispatchers in shared code across Android/iOS.

### D. Security & Data Leakage
- Hardcoded API keys/secrets/tokens, PII logged via `Log.d`/`println`, or written to unencrypted `SharedPreferences`.
- Enforce `EncryptedSharedPreferences` / DataStore / Android Keystore for sensitive payloads.
- `PendingIntent` flags (`FLAG_IMMUTABLE`), `android:exported="false"` on non-public components, SQL injection / Content Provider risks.

### E. UI Integrity & Layout Bleeding
- Layout overlap, clipped content, unhandled notch/cutout, missing edge-to-edge inset padding (`WindowInsetsCompat`), constraint issues in `ConstraintLayout`/Compose `Modifier`.
- Config-change handling: rotation, foldables, multi-window.
- Hardcoded user-facing strings not externalized to `strings.xml`.

### F. Architectural Compliance
- MVVM/MVI/MVC/Clean adherence.
- UI holds no business/data logic; UDF via `StateFlow`/`LiveData`/`SharedFlow`.
- KMP: UI/platform frameworks stay in `androidMain`/`iosMain`; `commonMain` stays pure Kotlin.

### G. Redundancy & Dead Code
- Duplicate UI logic, unused XML layouts, unused drawables, redundant helpers, dead conditional branches.

### H. Bug & Exception Safeguards
- Unsafe `!!`, unhandled coroutine exceptions (`CoroutineExceptionHandler`, `runCatching`), state-preservation edge cases.

### I. Test Coverage
- Unit tests for ViewModel/UseCase/Repository logic; Compose UI tests for critical screens.
- Zero-coverage business logic is a gap, not just a style nit.
- Test doubles (fakes/mocks) don't leak real network/DB access. See `testing.md` for the project's testing conventions when writing new tests to close gaps.

### J. Accessibility (a11y)
- `contentDescription` on meaningful images/icons (explicit null for decorative ones).
- Touch targets ≥48dp, color contrast, TalkBack/Compose semantics (`Modifier.semantics`).
- Missing a11y support: MEDIUM by default, CRITICAL if it blocks a core user flow (e.g. unlabeled primary action button).

---

## 2. Workflow

**Step 0 — Scope Sizing.** Apply Section 0.

**Step 1 — Initial Finding & Prioritized TODO Matrix.** Four tiers:
1. `[CRITICAL]` — ANRs, security vulnerabilities, data leakage, NPE crash risks, severe memory leaks, a11y blockers on core flows.
2. `[HIGH]` — architectural violations, unnecessary Compose recompositions, main-thread I/O, UI bleeding/clipping, zero test coverage on critical logic.
3. `[MEDIUM]` — redundancy, unused code/imports, missing lambdas/idioms, missing error handling, non-blocking a11y gaps.
4. `[LOW]` — indentation, naming, minor style, deprecated-dependency version bumps.

**Step 2 — Systematic Refactoring.** Resolve CRITICAL → LOW in order. For each: present refactored production code plus a one-line "why" (e.g. *Fixed ANR by switching to `Dispatchers.IO`*, *prevented leak by binding job to `viewLifecycleOwner`*).

**Step 3 — Zero-Technical-Debt Verification.** Confirm:
- Project still compiles and existing tests pass (run build/test if the environment allows; if not runnable, say so explicitly and why).
- No regressions introduced.
- Google Android standards followed (Kotlin/Compose/XML/KMP/Java), 4-space indentation throughout.
- 0 technical debt remains across all 10 dimensions.

---

## 3. Output Format

```markdown
# 🔍 Android Code Audit Report

## 📊 Summary of Findings
- **Critical Issues:** [Count]
- **High Severity:** [Count]
- **Medium Severity:** [Count]
- **Low Severity:** [Count]
- **Scope:** [files/modules audited; files/modules explicitly skipped, if any]

---

## 📋 Prioritized TODO List

### 🚨 [CRITICAL]
- [ ] **[File / Component]** Issue description and impact.

### 🟧 [HIGH]
- [ ] **[File / Component]** Issue description.

### 🟦 [MEDIUM]
- [ ] **[File / Component]** Issue description.

### 🟩 [LOW]
- [ ] **[File / Component]** Issue description.

---

## 🛠️ Step-by-Step Refactoring & Resolution
[Updated code blocks with concise justification per tier.]

---

## ✅ Verification
[Build/test status, or explicit note it couldn't be run in this environment.]
```
