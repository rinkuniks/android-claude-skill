# Android Code Audit & Refactoring Checklist

Use this when auditing or refactoring **existing** Android/Compose/KMP code (as opposed to scaffolding new features — see main SKILL.md for that workflow). Acts as Principal Android Engineer + Security Auditor: exhaustive static audit, prioritized TODO matrix, sequential refactor to zero technical debt without regressions.

---

## 0. Full-Project Context Graph (mandatory, run first)

Before any dimension-level audit, read the **whole project's structure** so later findings are made with full cross-module context, not a single-file view. This is a structural pass, not a full file-body read:

- Read `settings.gradle.kts` and every `build.gradle.kts`/`build.gradle` to learn every module and its declared dependencies.
- Read top-level package layout per module (not full file bodies yet) to learn what each module actually contains.
- From this, build a **module/dependency graph** — the graph type that best fits Android's multi-module structure (analogous to how a tool like Graphify visualizes node/edge relationships, here applied to Gradle modules instead of generic code entities): which modules depend on which, and where `feature:*:api` vs `feature:*:impl` and `core:*` modules sit relative to each other and to `app`.
- **Single-module app (no multi-module Gradle setup):** fall back to a package/class-level import graph instead — same node/edge idea, one level down.
- This graph is a **reporting artifact only** — attach it in the final report (Section 4 below) alongside the TODO matrix. It does not gate or replace the per-file deep-reads in Section 1; it gives the reader (and you, mid-audit) the full-project map those deep-reads sit inside.

---

## 1. Scope Sizing (per-file deep read, after the context graph)

- **Single file / small diff (<500 lines):** full inline read, audit directly.
- **Module or feature folder:** read module boundary first (`build.gradle.kts`, package root), then scan files within it. Do not pull in unrelated modules.
- **Whole repo / large monorepo:** do NOT read every file inline. Prioritize by risk — entry points (Activities, ViewModels, DI modules, network/storage layers) before leaf UI files. Use grep/search for known-risk patterns (`!!`, `SharedPreferences`, `GlobalScope`, `Log.d`, `static.*Context`) to shortlist files before deep reading. State explicitly which files/modules were skipped and why.
- If asked to audit "the codebase" with no path given, ask for a path or diff scope before proceeding — do not guess.

---

## 2. Audit Dimensions (all 10 required)

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
- **`lateinit` misuse:** property accessed before initialization (e.g. from a callback that can fire before `onCreate`/`onViewCreated` finishes), `lateinit` used where a nullable type or `by lazy` would be safer, no `::prop.isInitialized` guard where access order isn't guaranteed.
- **Java interop nullability:** missing `@Nullable`/`@NonNull` (or JSR-305 `@Nullable`/androidx annotations) on Java methods/fields consumed from Kotlin — Kotlin treats unannotated Java types as platform types (`T!`), silently skipping null checks the compiler would otherwise enforce. Flag Java APIs called from Kotlin without nullability annotations, and unguarded use of their return values.

### I. Test Coverage
- Unit tests for ViewModel/UseCase/Repository logic; Compose UI tests for critical screens.
- Zero-coverage business logic is a gap, not just a style nit.
- Test doubles (fakes/mocks) don't leak real network/DB access. See `testing.md` for the project's testing conventions when writing new tests to close gaps.

### J. Accessibility (a11y)
- `contentDescription` on meaningful images/icons (explicit null for decorative ones).
- Touch targets ≥48dp, color contrast, TalkBack/Compose semantics (`Modifier.semantics`).
- Missing a11y support: MEDIUM by default, CRITICAL if it blocks a core user flow (e.g. unlabeled primary action button).

---

## 3. Workflow

**Step 0 — Full-Project Context Graph.** Apply Section 0: read whole-project structure, build the module/dependency (or import) graph.

**Step 1 — Scope Sizing.** Apply Section 1 for per-file deep reads.

**Step 2 — Initial Finding & Prioritized TODO Matrix.** Four tiers:
1. `[CRITICAL]` — ANRs, security vulnerabilities, data leakage, NPE crash risks, severe memory leaks, a11y blockers on core flows.
2. `[HIGH]` — architectural violations, unnecessary Compose recompositions, main-thread I/O, UI bleeding/clipping, zero test coverage on critical logic.
3. `[MEDIUM]` — redundancy, unused code/imports, missing lambdas/idioms, missing error handling, non-blocking a11y gaps.
4. `[LOW]` — indentation, naming, minor style, deprecated-dependency version bumps.

**Step 2.5 — Permission Setup (ask once, before the refactor loop).** A real audit refactors many findings across many files — asking for edit confirmation on every single one is disruptive. Before starting Step 3, check whether the project's `.claude/settings.json` already allows `Edit`/`Write` without prompting. If not, ask the user **once**:

> "This will touch [N] files across [M] findings — auto-accept file edits for this project so you're not asked hundreds of times?"

If approved, merge (don't overwrite) this into `<project-root>/.claude/settings.json`:
```json
{
    "permissions": {
        "allow": ["Edit", "Write"]
    }
}
```
Then proceed through all of Step 3 without further per-file confirmation. If declined, proceed with normal per-edit confirmation as usual.

**Step 3 — Systematic Refactoring.** Resolve CRITICAL → LOW in order. For each: present refactored production code plus a one-line "why" (e.g. *Fixed ANR by switching to `Dispatchers.IO`*, *prevented leak by binding job to `viewLifecycleOwner`*).

**Step 4 — Zero-Technical-Debt Verification.** Confirm:
- Project still compiles and existing tests pass (run build/test if the environment allows; if not runnable, say so explicitly and why).
- No regressions introduced.
- Google Android standards followed (Kotlin/Compose/XML/KMP/Java), 4-space indentation throughout.
- 0 technical debt remains across all 10 dimensions.

---

## 4. Output Format

```markdown
# 🔍 Android Code Audit Report

## 🗺️ Project Context Graph
[Mermaid module/dependency graph (or package/import graph for single-module apps), e.g.:]

​```mermaid
graph TD
    app --> feature_settings_impl
    feature_settings_impl --> feature_settings_api
    feature_settings_impl --> core_data
    core_data --> core_network
    core_data --> core_database
    core_data --> core_model
​```

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
