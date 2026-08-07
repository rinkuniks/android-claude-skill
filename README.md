# Android Development & Audit Skill for Claude Code

A production-ready skill that enables Claude Code to **build** Android applications following Google's official architecture guidance and best practices from the [NowInAndroid](https://github.com/android/nowinandroid) reference app — and to **audit/refactor existing** Android, Jetpack Compose, and Kotlin Multiplatform (KMP) code down to zero technical debt.

## Overview

This skill provides Claude with comprehensive knowledge of modern Android development patterns, including:

- **Clean Architecture** with UI, Domain, and Data layers
- **Jetpack Compose** patterns and best practices
- **Multi-module project structure** with convention plugins
- **Offline-first architecture** with Room and reactive streams
- **Dependency injection** with Hilt
- **Comprehensive testing** strategies

...and a full **code audit & refactoring workflow** covering:

- Style & Kotlin idioms (4-space indentation, lambdas, dead code)
- Memory leak & ANR prevention
- Compose recomposition performance
- Security & data leakage (secrets, encrypted storage, exported components)
- UI integrity (edge-to-edge insets, layout bleeding, localization)
- Architectural compliance (MVVM/MVI/Clean, UDF)
- Test coverage gaps
- Accessibility (a11y)

See [SKILL.md](SKILL.md) for the full reference index and [references/audit-checklist.md](references/audit-checklist.md) for the complete audit workflow.

## Installation

Claude Code loads skills from a `SKILL.md` file in a recognized skills directory — personal, project-level, or anywhere you point it. Pick whichever fits:

### Option 1 — Personal skills directory (available in every project)
```bash
git clone https://github.com/rinkuniks/android-claude-skill.git ~/.claude/skills/android-claude-skill
```
Claude Code auto-detects skills under `~/.claude/skills/` on startup — no restart config needed beyond starting a new session.

### Option 2 — Project-level skill (shared with your team via your repo)
```bash
mkdir -p .claude/skills
git clone https://github.com/rinkuniks/android-claude-skill.git .claude/skills/android-claude-skill
```
Commit `.claude/skills/android-claude-skill` (or add it as a git submodule) so teammates get the skill automatically when they clone the project.

### Option 3 — Git submodule (stay pinned to a version, easy to update)
```bash
git submodule add https://github.com/rinkuniks/android-claude-skill.git .claude/skills/android-claude-skill
git submodule update --init --recursive
```

### Option 4 — Manual download (no git)
1. Download this repo as a ZIP (GitHub → Code → Download ZIP).
2. Unzip it into `~/.claude/skills/android-claude-skill` (personal) or `<your-project>/.claude/skills/android-claude-skill` (project-level).

### Updating later
```bash
cd ~/.claude/skills/android-claude-skill   # or the project-level path
git pull
```

After installing, start (or restart) Claude Code in that context — it discovers the skill automatically. No slash command needed; it's a description-triggered skill.

## Usage

The skill automatically activates when you request Android-related tasks. Simply ask Claude to:

**Build:**
- "Create a new Android feature module for user settings"
- "Build a Compose screen with MVVM pattern"
- "Set up a Repository with offline-first architecture"
- "Add navigation to my Android app"
- "Configure multi-module Gradle setup"

**Audit / refactor:**
- "Audit this Android module for memory leaks and ANR risk"
- "Review this Compose screen for unnecessary recompositions"
- "Refactor this repository class and fix security issues"
- "Check this codebase for accessibility gaps"

Claude will follow the patterns and best practices defined in this skill.

## Project Structure

```
claude-android-skill/
├── SKILL.md                    # Main skill definition and quick reference
├── references/                 # Detailed documentation
│   ├── architecture.md         # UI, Domain, Data layers patterns
│   ├── compose-patterns.md     # Jetpack Compose best practices
│   ├── gradle-setup.md         # Build configuration & convention plugins
│   ├── modularization.md       # Multi-module project structure
│   ├── testing.md              # Testing strategies and patterns
│   └── audit-checklist.md      # Code audit & refactoring workflow (10 dimensions)
├── assets/
│   └── templates/              # Project templates
│       ├── libs.versions.toml.template
│       └── settings.gradle.kts.template
└── scripts/
    └── generate_feature.py     # Feature module generator script
```

## Core Principles

This skill teaches Claude to follow these key Android development principles:

1. **Offline-first**: Local database as source of truth, synchronized with remote data
2. **Unidirectional data flow**: Events flow down, data flows up (UDF pattern)
3. **Reactive streams**: Use Kotlin Flow for all data exposure
4. **Modular by feature**: Each feature is self-contained with clear API boundaries
5. **Testable by design**: Use interfaces and test doubles, avoid mocking frameworks

## Reference Documentation

### Quick Navigation

| Topic | File | Description |
|-------|------|-------------|
| Architecture | [architecture.md](references/architecture.md) | MVVM pattern, layers, repositories, use cases |
| Compose UI | [compose-patterns.md](references/compose-patterns.md) | Screens, state hoisting, side effects, theming |
| Build Setup | [gradle-setup.md](references/gradle-setup.md) | Convention plugins, version catalogs, configuration |
| Modularization | [modularization.md](references/modularization.md) | Module types, dependencies, feature structure |
| Testing | [testing.md](references/testing.md) | Unit tests, UI tests, test doubles, strategies |

## Architecture Overview

```
┌─────────────────────────────────────────┐
│              UI Layer                    │
│  (Compose Screens + ViewModels)          │
├─────────────────────────────────────────┤
│           Domain Layer                   │
│  (Use Cases - optional, for reuse)       │
├─────────────────────────────────────────┤
│            Data Layer                    │
│  (Repositories + DataSources)            │
└─────────────────────────────────────────┘
```

### Module Types

```
app/                    # Application module
feature/
  ├── featurename/
  │   ├── api/          # Public navigation contracts
  │   └── impl/         # Internal implementation
core/
  ├── data/             # Repositories
  ├── database/         # Room DAOs & entities
  ├── network/          # Retrofit & API models
  ├── model/            # Domain models
  ├── ui/               # Reusable components
  ├── designsystem/     # Theme & design tokens
  └── testing/          # Test utilities
```

## Features

### Code Generation

The skill includes a Python script to generate feature modules:

```bash
python scripts/generate_feature.py settings \
  --package com.example.app \
  --path /path/to/project
```

This creates a complete feature module with:
- API module with navigation definitions
- Implementation module with Screen, ViewModel, UiState
- Gradle build files with proper dependencies
- Hilt dependency injection setup

### Templates

Pre-configured templates for common Android project files:
- `libs.versions.toml.template` - Gradle version catalog
- `settings.gradle.kts.template` - Project settings

## Standard Patterns

### ViewModel Pattern
```kotlin
@HiltViewModel
class MyFeatureViewModel @Inject constructor(
    private val repository: MyRepository,
) : ViewModel() {
    val uiState: StateFlow<MyFeatureUiState> = repository
        .getData()
        .map { MyFeatureUiState.Success(it) }
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5_000),
            initialValue = MyFeatureUiState.Loading,
        )
}
```

### Screen Pattern
```kotlin
@Composable
internal fun MyFeatureRoute(
    viewModel: MyFeatureViewModel = hiltViewModel(),
) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    MyFeatureScreen(uiState = uiState)
}
```

### Repository Pattern
```kotlin
interface MyRepository {
    fun getData(): Flow<List<MyModel>>
}

internal class OfflineFirstMyRepository @Inject constructor(
    private val dao: MyDao,
    private val api: MyNetworkApi,
) : MyRepository {
    override fun getData(): Flow<List<MyModel>> =
        dao.getAll().map { it.toModel() }
}
```

## Technology Stack

This skill configures projects with:

- **Language**: Kotlin
- **UI**: Jetpack Compose
- **Architecture**: MVVM with UDF
- **DI**: Hilt
- **Database**: Room
- **Network**: Retrofit + Kotlinx Serialization
- **Async**: Kotlin Coroutines + Flow
- **Testing**: JUnit, Turbine, Compose Testing
- **Build**: Gradle with Convention Plugins

## License

This project is licensed under the MIT License - see the LICENSE file for details.

## Acknowledgments

This repository is a fork/extension of [dpconde/claude-android-skill](https://github.com/dpconde/claude-android-skill) by David Perez, with an added code-audit & refactoring workflow. Original MIT license and copyright retained — see [LICENSE](LICENSE).

Based on patterns and practices from:
- [NowInAndroid](https://github.com/android/nowinandroid) by Google
- [Android Architecture Guidelines](https://developer.android.com/topic/architecture)
- [Jetpack Compose Best Practices](https://developer.android.com/jetpack/compose)

## Resources

- [Android Developer Documentation](https://developer.android.com)
- [NowInAndroid Repository](https://github.com/android/nowinandroid)
- [Kotlin Documentation](https://kotlinlang.org/docs/home.html)
- [Jetpack Compose Pathway](https://developer.android.com/courses/pathways/compose)
