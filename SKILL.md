---
name: mederi-compose-skill
license: MIT
metadata:
  version: "0.1.0"
description: >
  Compose Multiplatform (CMP) architecture and UI engineering skill targeting all
  platforms (Android, iOS, Desktop/JVM, and Web/wasmJs). Enforces commonMain-first
  code sharing, data-layer SSOT with reactive ViewModel derivation, MVI (MVVM
  compatible), transparent container hierarchy, semantic theme inheritance, true
  constraint-based centering, modular file boundaries, and honest error display.
  Use whenever creating, editing, refactoring, or reviewing Compose Multiplatform
  screens, layouts, components, styling, state, or data bindings. Do not use for
  pure backend, server, or non-UI CLI tasks.
---

# Compose Multiplatform (Android, iOS, Desktop, WasmJs)

This skill governs full-lifecycle Compose Multiplatform (CMP) development across **Android, iOS, Desktop (JVM), and Web (`wasmJs`)**. Every design and implementation choice must work in `commonMain` across all four targets unless a platform capability genuinely cannot be expressed in common code. MVI is the primary presentation architecture, while adapting cleanly to existing MVVM codebases.

## Cross-Platform & WasmJs Non-Negotiables

1. **`commonMain` First**: All UI, ViewModels, domain logic, repositories, resources, and serialization live in `commonMain` by default. Isolate genuinely platform-specific capabilities (OS permissions, biometrics, native share, storage drivers) behind `commonMain` interfaces injected via Koin DI (preferred) or minimal `expect/actual` declarations.
2. **Strictly Forbidden in `commonMain`**:
   - JVM/Android APIs: `java.*` (`java.time`, `java.util.UUID`, `java.io.File`, `String.format`), `android.*`, Android `R.string`/`R.drawable`, `System.currentTimeMillis()`. Use `kotlin.time.Clock.System`, `Res.string`/`Res.drawable`, and multiplatform libraries instead.
   - Non-portable concurrency: `Dispatchers.IO` (does not exist on `wasmJs`), `runBlocking`, and JVM `synchronized`. Use `Dispatchers.Default`, injected platform dispatchers where blocking IO is needed on JVM/Native, `kotlinx.coroutines.sync.Mutex`, or `StateFlow.update`.
   - JVM reflection: Never rely on reflection-based serialization or navigation route parsing (fails on iOS and `wasmJs`); always register explicit `SerializersModule` for polymorphic types/routes.
   - Android-only UI/DI libraries: `Hilt`, `androidx.constraintlayout:constraintlayout-compose`, `androidx.paging:paging-runtime`, and `Macrobenchmark`.
3. **WasmJs-Specific Guardrails**:
   - **Persistence**: Verify exact target artifacts before adding storage libraries (e.g., DataStore 1.3.0+ supports `wasmJsMain` via `WebLocalStorage`; Room 3.0+ supports `wasmJs` via `androidx.room3` + `WebWorkerSQLiteDriver`, whereas Room 2.x does **not** support `wasmJs`).
   - **Dynamic Color**: Material You dynamic color is Android 12+ only — never require it in `commonMain`; provide a complete static cross-platform Theme and treat dynamic color only as an optional Android enhancement.
   - **Fonts**: Starting with Compose Multiplatform **1.12.0+**, web targets support automatic on-demand Noto font fallback (eliminating tofu glyphs for CJK/emoji without bundling). Bundle fonts via `Res.font` (or configure `FontFamily.Resolver.preload()`) only when targeting CMP `< 1.12.0`, using custom brand fonts, or deploying to offline/restricted networks where `fonts.gstatic.com` is blocked (whitelist `fonts.gstatic.com` in CSP when relying on automatic fallback).

## Existing Project Policy

- **Architecture layer (respect legacy structure)**: Do not force-migrate working MVVM screens to MVI or rewrite existing MVI base classes, and do not refactor untouched legacy files during small localized fixes.
- **Code layer (strict on touched code)**: All **newly written or directly modified** code must strictly follow the SSOT, UI layout, styling, serialization, and error rules below — never copy anti-patterns from surrounding legacy code.

## Workflow

1. **Read relevant existing code first** — inspect conventions, base classes, DI setup, and target configuration in `build.gradle.kts`. Limit reads to relevant files for small tasks.
2. **Identify the concern** — architecture, state/SSOT, UI layout, navigation, networking, persistence, performance, animation, or cross-platform bridge.
3. **Apply core rules in this file** — all UI, SSOT, MVI/MVVM, and `wasmJs` safety rules live here.
4. **Consult `references/` only for advanced topics** — load at most one relevant file from [Quick Routing](#quick-routing) when deep library guidance is needed.
5. **Verify dependencies before recommending** — confirm Maven coordinates, **all 4 targets including `wasmJs`**, and current API signatures before adding/upgrading any dependency.
6. **Flag production anti-patterns contextually** — call out real architectural or cross-platform violations without blocking quick answers.
7. **Write the minimal correct solution** — avoid speculative abstractions and generic frameworks.

## Dependency Verification Rule

Before recommending any new dependency or version upgrade, verify via a documentation MCP tool (e.g., Context7), official docs, or Maven Central:
1. **Coordinates** (`group:artifact:version`) exist and are current.
2. **Full target support** — explicitly confirm support for Android, iOS, Desktop, and `wasmJs` in `commonMain` (or state which targets require source-set isolation).
3. **API shape** — confirm signatures exist in that exact version.

If verification tools/network are unavailable, provide the best-known snippet with a brief `// Verify latest version & wasmJs target support` comment.

## Core Architecture: Data-Layer SSOT + MVI (MVVM Compatible)

Combine **Data-Layer SSOT** (eliminates cross-screen config/data drift) with **Presentation-Layer MVI** (eliminates intra-screen state scatter).

### 1. Data-Layer Single Source of Truth (SSOT)

- Every configuration, user setting, and shared domain data source exists as **exactly one authoritative `Repository`/`Store` instance** in the data layer (scoped as a singleton via Koin DI).
- The SSOT exposes read-only `Flow` streams and **serialized atomic write APIs** (`DataStore.edit`, `StateFlow.update`, DB transactions, or `Mutex`-guarded mutations).
- `@Composable` functions **never** instantiate, hold, or directly read/write a `Store`/`Repository`. All data interaction goes through the ViewModel.

### 2. ViewModel: Reactive Derivation, Zero Snapshots, Optimal Recomposition

- **Upstream Slice Before `combine`**: When observing a global config or broad SSOT `Flow`, project only the fields this screen needs and deduplicate first (`.map { ScreenConfig(it.a, it.b) }.distinctUntilChanged()`) before `combine`, so unrelated config changes never recompute this screen's state.
- **Pure Reactive Derivation**: Combine sliced SSOT flows and screen-local query/filter flows into a single immutable `UiState` exposed via `.stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), initialValue)`.
- **No Shadow Copies or One-Shot Reads**: Never copy SSOT values into mutable ViewModel fields via `.first()` or `.value` in `init`. Writes call the SSOT's atomic update API directly and let upstream emissions refresh `UiState`.
- **Preserve Unchanged References**: Reuse unchanged nested objects and `List` references when producing a new `UiState`. Compose Strong Skipping Mode skips recomposing child composables in $O(1)$ via reference equality (`===`); do not blindly annotate large lists with `@Immutable`.
- **Isolate High-Frequency Streams & Text Input**:
  - Keep high-frequency ticks (timers, playback progress, live sensor/chart values) in a dedicated `StateFlow` rather than combining them into the full-screen `UiState`.
  - Keep synchronous text editing out of async `combine` pipelines (prefer `TextFieldState` or dedicated synchronous state, writing to SSOT on submit/debounce) to avoid cursor jumps and IME glitches.
- **No Redundant Derived State**: Store only canonical inputs (e.g., `selectedId`); compute derived values reactively unless async/cost justifies caching.

### 3. MVI Primary, MVVM Compatible

Both share unidirectional data flow (`UI renders State -> User acts -> ViewModel updates State / SSOT -> UI re-renders`), immutable `StateFlow<FeatureState>`, and one-shot `Channel<Effect>(Channel.BUFFERED)` for navigation/snackbars/platform commands:
- **MVI (default for new features)**: `sealed interface Event` + single `onEvent(event: Event)` entry point.
- **MVVM (preserved where already used)**: named public functions (`onTitleChanged()`, `save()`).

### 4. UI Rendering Boundary

- **Route**: acquires ViewModel via Koin, collects state via `collectAsStateWithLifecycle()`, collects effects via `CollectEffect`, wires navigation/snackbar/platform bridges.
- **Screen**: stateless renderer receiving immutable `state` and callbacks (`onEvent` or named callbacks).
- **Leaf**: receives narrowest immutable sub-state and callbacks. Local `remember` state is permitted **only** for ephemeral visual concerns (focus, scroll position, animation). Never copy passed-in domain/config data into `remember { mutableStateOf(initial) }`.

## UI Layout, Styling, Organization & Safety Rules

These rules apply to **every** UI change (do not skip them even when not loading `references/`):

1. **Container Hierarchy & Theme Inheritance**:
   - **Root Canvas Only**: Exactly one root container per screen (`Scaffold`, root `Box`/`Surface`) sets the opaque page background. Inner scrolling lists and linear containers (`Column`, `LazyColumn`, `Row`) stay transparent.
   - **Explicit Surfaces Only for Elevation & Semantic Controls**: Apply semantic `MaterialTheme.colorScheme` surface colors (paired with matching `contentColor`) only to elevated/separated chrome (`TopBar`, `BottomBar`, `Card`, `Dialog`, `Sheet`, `Floating Panel`, `Banner`) and interactive/semantic controls (filled buttons, text fields, chips/tags, badges, active highlights). Custom container components exposing a background parameter must default to `Color.Unspecified`/`Color.Transparent`.
   - **Leaf Controls Inherit Colors**: `Text`, `Icon`, and dividers inherit `LocalContentColor` by default. Override only via Theme hierarchy tokens (`onSurfaceVariant` for secondary text/captions) or semantic states (`primary`, `error`, `disabled`). **Never** hardcode hex/RGB color literals (`Color(0xFF...)`) or ad-hoc per-widget backgrounds.
   - Never wrap a single primitive (`Text`, `Icon`, `Spacer`) in an empty wrapper container that adds no layout or visual structure.

2. **Positioning & True Centering (No `ConstraintLayout` in CMP)**:
   - **Docked Chrome**: Use `Scaffold(topBar, bottomBar)` or non-overlapping structural layout (`Column` with `weight(1f)` content + docked bar) so bars never occlude scrollable content.
   - **True Centering & Overlays**: Elements anchored to geometric center or edges that must not drift when asymmetric siblings change (centered titles, badges, FABs, centered loading/empty states) must use `CenterAlignedTopAppBar`, `Box` + `Modifier.align(...)`, or a custom Compose `Layout` for dynamic non-overlap measurement.
   - **Strictly Forbidden**: Fake centering via `Row`/`Column` + `Spacer`/`weight(1f)`/`Arrangement.SpaceBetween` between asymmetric sides; hardcoded coordinate offsets (`offset(dp)`); hardcoded symmetric padding/widths (even named as a token like `96.dp`) to avoid overlap.
   - `Row`/`Column` and `Spacer`/`weight` are strictly for sequential flow or separating opposing edge groups without an independently centered sibling between them.

3. **Modular File Organization & Concrete Reuse**:
   - **One File Per Public/Reusable Component**: Every `Screen`/`Route` and every `public`/`internal` reusable UI component (bar, card, dialog, list row, custom control) lives in its own dedicated file named after the component. Never create "God Files".
   - *Practical exception*: Short, single-screen `private` helper composables (roughly under 40 lines, not reused elsewhere) may stay in that screen's file.
   - **Search First & Rule of Two**: Search existing component directories before creating a new component. Extract a shared component when the same layout skeleton repeats **2+ times**. Do not over-abstract if merging requires **3+ unrelated boolean flags** (`isCompact`, `hideHeader`, etc.). Never create a file just to wrap a single `Text`, `Icon`, or `Spacer`.

4. **Design Tokens, Typed Models & Honest Error Display**:
   - **Tokens Over Magic Literals**: Shared spacing/radii scales, typography, colors, business thresholds, JSON keys, and user-facing copy must come from Theme tokens, named constants, `Res.string`/`Res.drawable`, or SSOT. *Exemptions*: token definitions themselves, structural language literals (`0`, `1`, booleans, `weight(1f)`, `maxLines = 1`), and trivial one-off micro-adjustments with no reuse value.
   - **Typed Models Only**: All structured payloads and settings use `@Serializable` DTOs/models with `kotlinx.serialization`. **Strictly forbidden**: hand-building or parsing JSON via string templates, concatenation, `joinToString`, or regex.
   - **Honest Error Display — `-` Only, Never `0`, Never Crash**: Never silently fall back to fabricated business defaults (e.g., returning `10` or `0` on failure) or swallow errors. Catch boundary/decoding/IO errors safely, display **`-`** (or `--`) for failed or missing **both numeric and text** fields (**never `0`**, which masquerades as real data), and surface explicit error feedback via MVI `Effect` (snackbar/banner) or state-driven inline error UI.
   - **Pure Composable Bodies**: Never trigger one-shot side effects (snackbar, toast, dialog trigger, navigation, network) directly inside a `@Composable` render body.

## State Modeling

For forms and calculators, separate state into four concerns:
1. **Editable input** — raw text/selection (`TextFieldState` or synchronous input state outside async `combine`)
2. **Derived display/business** — parsed, validated, calculated values
3. **Persisted domain snapshot** — live SSOT stream for dirty tracking/reset
4. **Transient UI-only** — local Compose state (`LazyListState`, focus, animation)

## Recommended Defaults

| Concern | Default (All CMP Targets: Android, iOS, Desktop, WasmJs) |
|---|---|
| ViewModel | One ViewModel per screen in `commonMain`. MVI: `onEvent(Event)` (default); MVVM: named functions |
| Data SSOT | Singleton `Repository`/`Store` via Koin DI; exposes read-only `Flow` + atomic writes |
| Screen State | `StateFlow<FeatureState>` derived via upstream `.map().distinctUntilChanged()` + `combine` + `stateIn(WhileSubscribed(5_000))` |
| One-shot Effects | `Channel<Effect>(Channel.BUFFERED)` collected in `Route` via `CollectEffect` |
| DI | Koin Multiplatform (`commonMain` modules + minimal platform modules) |
| Networking | Ktor Client (`commonMain`) with per-target engines + `kotlinx.serialization` |
| Persistence | Verify target versions: DataStore 1.3+ (`WebLocalStorage` on `wasmJsMain`, `FileStorage` on native/JVM) for settings; Room 3.0+ (`androidx.room3` + `WebWorkerSQLiteDriver`) or SQLDelight for relational data |
| Resources & Adaptive | Compose Resources (`Res.string`, `Res.drawable`, `Res.font`); `currentWindowAdaptiveInfo()` from `material3.adaptive` |
| Navigation | ViewModel emits semantic navigation `Effect`; explicit `SerializersModule` for non-JVM targets |
| Testing | `commonTest` ViewModel event→state→effect tests via Turbine; pure unit tests for mappers/validators; for non-trivial UI work, generate headless `jvmTest` screenshots via `org.jetbrains.compose.ui:ui-test` (`runDesktopComposeUiTest` + `captureToImage()`) — see [testing.md](references/testing.md) |

## Do / Don't Quick Reference

### Do
- Keep SSOT in the data layer and derive ViewModel `UiState` reactively with upstream `.map().distinctUntilChanged()`
- Preserve unchanged object/list references across `UiState` updates
- Import all types/functions at file top (use `import ... as ...` for clashes)
- Preserve previous content during refresh (flip loading flag instead of wiping the screen)
- Key lazy list items by stable domain ID
- Guard no-op state emissions and cancel superseded async jobs

### Don't
- Put `java.*`, `android.*`, `Dispatchers.IO`, `runBlocking`, `Hilt`, or `ConstraintLayout` in `commonMain`
- Copy SSOT values in ViewModel `init` (`.first()`/`.value`) or copy props into `remember { mutableStateOf(initial) }`
- Hardcode hex colors, fake-center with `Spacer`/`weight(1f)`, or use fixed symmetric padding (`96.dp`) to avoid overlap
- Hand-build JSON strings, silently fabricate fallback values, or display `0` on error
- Trigger snackbar/navigation side effects directly inside `@Composable` bodies or encode one-shot commands as state booleans
- Use fully qualified package names inline (`com.example.pkg.Foo()`)

## Detailed References

**Rule Precedence**: Reference files illustrate library APIs and may use brief inline literals (`16.dp`, `"Saved"`), simplified single-file snippets, or older Android-oriented examples. When writing production code, **all rules in this `SKILL.md` strictly override** shorthand literals, file bundling, Android-only APIs, or prop-drilling examples in `references/`.

**Do not load reference files for basic Compose usage.** Load **at most one** relevant file below only when the task requires deep library or domain guidance:

### Quick Routing

- **Recomposition, stability, Strong Skipping, or Compiler Metrics** → [performance.md](references/performance.md)
- **Channel vs SharedFlow, Flow operators, structured concurrency, exceptions** → [coroutines-flow.md](references/coroutines-flow.md)
- **Backpressure, callbackFlow, Mutex/Semaphore, or Turbine testing** → [coroutines-flow-advanced.md](references/coroutines-flow-advanced.md)
- **Nav 3 routes, scenes, deep links, or explicit `SerializersModule` setup** → [navigation-3.md](references/navigation-3.md)
- **Nav 2 NavHost, tabs, deep links, or nested graphs** → [navigation-2.md](references/navigation-2.md)
- **Wiring Koin with Navigation** → [navigation-3-di.md](references/navigation-3-di.md) or [navigation-2-di.md](references/navigation-2-di.md)
- **Migrating from Nav 2 to Nav 3** → [navigation-migration.md](references/navigation-migration.md)
- **Paging 3 KMP (`paging-common`/`paging-compose`, never `paging-runtime`)** → [paging.md](references/paging.md)
- **Offline-first paging & RemoteMediator** → [paging-offline.md](references/paging-offline.md)
- **Paging MVI integration & tests** → [paging-mvi-testing.md](references/paging-mvi-testing.md)
- **Ktor client setup, plugins, DTOs, or `@Resource` routes** → [networking-ktor.md](references/networking-ktor.md)
- **Auth (bearer), WebSockets, or SSE** → [networking-ktor-auth.md](references/networking-ktor-auth.md)
- **Network layer architecture & error handling strategy** → [networking-ktor-architecture.md](references/networking-ktor-architecture.md)
- **MockEngine, network testing, or Koin network DI** → [networking-ktor-testing.md](references/networking-ktor-testing.md)
- **Koin CMP setup, scoped modules, or platform module wiring** → [koin.md](references/koin.md)
- **Accessibility audit, semantics, touch targets, or WCAG contrast** → [accessibility.md](references/accessibility.md)
- **Animation API selection (`animate*AsState`, `Animatable`, `AnimatedVisibility`)** → [animations.md](references/animations.md)
- **Shared element transitions, gestures, Canvas, or `graphicsLayer`** → [animations-advanced.md](references/animations-advanced.md)
- **Code review or anti-pattern detection** → [anti-patterns.md](references/anti-patterns.md)
- **Exposing Kotlin to Swift, SKIE, or Flow→AsyncSequence** → [ios-swift-interop.md](references/ios-swift-interop.md)
- **ViewModel pipeline, SSOT derivation, domain layer, inter-feature communication** → [architecture.md](references/architecture.md)
- **MVI pipeline, Event/State/Effect, `onEvent` pattern, effect delivery** → [mvi.md](references/mvi.md)
- **MVVM pipeline, ViewModel named functions, direct-callback wiring** → [mvvm.md](references/mvvm.md)
- **File organization, naming conventions, component extraction** → [clean-code.md](references/clean-code.md)
- **Three phases, state primitives, side effects, modifiers** → [compose-essentials.md](references/compose-essentials.md)
- **M3 theme, adaptive layouts (`currentWindowAdaptiveInfo`), M3 components** → [material-design.md](references/material-design.md)
- **AsyncImage, image cache, SVG, or Coil 3** → [image-loading.md](references/image-loading.md)
- **LazyColumn, LazyRow, keys, grids, pager, or scroll state** → [lists-grids.md](references/lists-grids.md)
- **Nav 2 vs Nav 3 decision or MVI navigation rules** → [navigation.md](references/navigation.md)
- **Loading states, skeleton/shimmer, or inline validation UX** → [ui-ux.md](references/ui-ux.md)
- **Turbine, ViewModel tests, headless `jvmTest` visual screenshots (`ui-test`), or test matrix** → [testing.md](references/testing.md)
- **DataStore KMP (incl. 1.3+ `WebLocalStorage` for `wasmJs`)** → [datastore.md](references/datastore.md)
- **Room KMP (Room 3.0+ for `wasmJs` vs Room 2.x non-Web isolation)** → [room-database.md](references/room-database.md)
- **commonMain sharing, `expect/actual`, `wasmJs` & platform bridges** → [cross-platform.md](references/cross-platform.md)
- **CMP `Res` class, bundled fonts (`Res.font`), qualifiers, localization** → [resources.md](references/resources.md)
- **Gradle KMP targets, version catalog, convention plugins** → [gradle-build.md](references/gradle-build.md)
- **GitHub Actions CI/CD, desktop/web packaging, signing, notarization** → [ci-cd-distribution.md](references/ci-cd-distribution.md)

## Validation

Run `./scripts/validate.sh` to validate the skill package against the [agentskills.io spec](https://agentskills.io/specification) (name/directory match, broken links, token budgets, and structure). Fix all errors before committing.
