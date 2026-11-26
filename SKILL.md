---
name: mederi-compose-skill
license: MIT
metadata:
  version: "0.1.1"
description: >
  Auto-activate whenever creating, editing, refactoring, reviewing, or debugging
  any Compose Multiplatform (CMP) UI, layouts, components, themes, state flows,
  navigation, or visual/recomposition bugs across Android, iOS, Desktop (JVM),
  and Web (wasmJs). Enforces commonMain-first code, data-layer SSOT + reactive
  MVI/MVVM, transparent container hierarchy, true centering, honest error display,
  and official headless screenshot artifacts. Do not use for pure backend or
  non-UI CLI tasks.
---

# Compose Multiplatform (Android, iOS, Desktop, WasmJs)

Target **Android, iOS, Desktop (JVM), and Web (`wasmJs`)** from `commonMain` by default. MVI is primary; adapt cleanly to existing MVVM code.

## Scope & Workflow

- **Legacy vs. Touched Code**: Do not force-migrate working MVVM screens, rewrite existing MVI base classes, or refactor untouched files during localized fixes. All **new or directly modified** code must strictly obey every rule below—never copy surrounding anti-patterns.
- **Minimal Execution**: Read only relevant existing files first (`build.gradle.kts`, conventions, DI). Write the minimal correct solution without speculative abstractions.
- **Verify Dependencies First**: Before adding/upgrading any library, verify coordinates, **all 4 targets including `wasmJs`**, and current API shape via docs MCP (e.g., Context7), official docs, or Maven Central. If offline, provide the best-known snippet marked `// Verify latest version & wasmJs support`.

## Cross-Platform & WasmJs Rules

1. **`commonMain` First**: UI, ViewModels, domain logic, repositories, resources, and serialization live in `commonMain`. Isolate genuine OS/hardware capabilities behind `commonMain` interfaces injected via Koin (preferred) or minimal `expect/actual`.
2. **Forbidden in `commonMain`**:
   - JVM/Android APIs: `java.*` (`java.time`, `UUID`, `File`, `String.format`), `android.*`, Android `R.*`, `System.currentTimeMillis()`. Use `kotlin.time.Clock.System`, `Res.*`, and multiplatform libraries.
   - Non-portable concurrency: `Dispatchers.IO` (missing on `wasmJs`), `runBlocking`, JVM `synchronized`. Use `Dispatchers.Default`, injected platform dispatchers, `Mutex`, or `StateFlow.update`.
   - JVM reflection: Always register explicit `SerializersModule` for polymorphic models and routes (required on iOS/`wasmJs`).
   - Android-only libraries: `Hilt`, `androidx.constraintlayout:constraintlayout-compose`, `androidx.paging:paging-runtime`, `Macrobenchmark`.
3. **WasmJs Guardrails**:
   - **Persistence**: DataStore 1.3.0+ supports `wasmJsMain` (`WebLocalStorage`); Room 3.0+ (`androidx.room3` + `WebWorkerSQLiteDriver`) supports `wasmJs`, whereas Room 2.x does **not** (isolate Room 2.x outside `wasmJs`).
   - **Dynamic Color**: Android 12+ optional enhancement only; always ship a full static cross-platform Theme.
   - **Fonts**: CMP **1.12.0+** supports automatic on-demand Noto font fallback on web (CJK/emoji without bundling; allow `fonts.gstatic.com` in CSP). Bundle via `Res.font` (or `FontFamily.Resolver.preload()`) only for CMP `< 1.12.0`, custom brand fonts, or offline/restricted networks.

## Architecture: Data-Layer SSOT + MVI (MVVM Compatible)

- **Data-Layer SSOT**: Every config, setting, and shared data source exists as one Koin-managed singleton `Repository`/`Store` exposing read-only `Flow`s and atomic writes (`DataStore.edit`, `StateFlow.update`, transactions, `Mutex`). `@Composable` functions **never** hold or directly read/write stores.
- **ViewModel Pure Reactive Derivation**:
  - **Slice upstream first**: Project only needed fields and deduplicate (`.map { ScreenCfg(it.a, it.b) }.distinctUntilChanged()`) before `combine`, exposed via `.stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), initial)`.
  - **No snapshots**: Never read SSOT via `.first()`/`.value` into mutable fields in `init`. Writes call SSOT atomic APIs directly and let upstream emissions refresh `UiState`.
  - **Recomposition hygiene**: Reuse unchanged nested objects/`List` references so Strong Skipping skips in O(1) via `===` (do not blindly mark large lists `@Immutable`). Guard no-op updates, cancel superseded async jobs, preserve previous content during refresh (flip loading flag, never wipe the screen), and store only canonical inputs (compute derived values).
  - **Isolate hot paths**: Keep high-frequency ticks (timers, progress, live charts) in a separate `StateFlow`, and keep synchronous text editing out of async `combine` (`TextFieldState` or dedicated sync state; write to SSOT on submit/debounce).
- **MVI / MVVM & UI Boundaries**:
  - **MVI (default)**: `sealed interface Event` + `onEvent(Event)`. **MVVM (existing)**: named public functions. Both use immutable `StateFlow<FeatureState>` and `Channel<Effect>(Channel.BUFFERED)` for one-shot commands (never state booleans).
  - **Route** acquires ViewModel (Koin), collects state via `collectAsStateWithLifecycle()` and effects via `CollectEffect`. **Screen** is a stateless renderer. **Leaf** takes narrowest immutable sub-state and callbacks; local `remember` is strictly for ephemeral visual state (focus, scroll, animation)—never copy passed-in domain/config data into `remember { mutableStateOf(initial) }`.

## UI Layout, Styling, Files & Safety

1. **Containers & Theme**:
   - One root container (`Scaffold` / root `Box`/`Surface`) owns the opaque page background; inner lists/stacks stay transparent.
   - Apply `MaterialTheme.colorScheme` surface + matching `contentColor` only to elevated chrome (`TopBar`, `BottomBar`, `Card`, `Dialog`, `Sheet`, `Floating Panel`, `Banner`) and interactive/semantic controls (filled buttons, text fields, chips, badges, active highlights). Custom container background parameters default to `Color.Unspecified`/`Color.Transparent`.
   - Leaf controls (`Text`, `Icon`, dividers) inherit `LocalContentColor`; override only via Theme hierarchy (`onSurfaceVariant`) or semantic tokens (`primary`, `error`, `disabled`). **Never** hardcode hex/RGB colors or wrap a single primitive in an empty container.
2. **Positioning & True Centering (No `ConstraintLayout`)**:
   - Docked bars use `Scaffold` or non-overlapping `Column` (`weight(1f)` body + bar).
   - True-centered/overlay elements (titles, badges, FABs, centered empty/loading states) use `CenterAlignedTopAppBar`, `Box` + `Modifier.align(...)`, or a custom Compose `Layout` for dynamic non-overlap.
   - **Forbidden**: Fake centering via `Row`/`Column` + `Spacer`/`weight(1f)`/`Arrangement.SpaceBetween` between asymmetric sides; hardcoded `offset(dp)`; hardcoded symmetric widths/padding (even tokenized like `96.dp`) to avoid overlap.
3. **Files, Imports & Reuse**:
   - One file per `Screen`/`Route` and per `public`/`internal` reusable component (short single-screen `private` helpers under ~40 lines may stay in the screen file). Key lazy items by stable domain ID. Import all symbols at file top (no inline FQNs).
   - Search existing components before creating; extract on **2+ uses**; split instead of adding **3+ unrelated boolean flags**; never create a file wrapping a single primitive.
4. **Tokens, Serialization & Honest Errors**:
   - Shared spacing/radii, typography, colors, thresholds, JSON keys, and copy come from Theme tokens, named constants, `Res.*`, or SSOT (exempt: token definitions, structural literals `0`/`1`/booleans/`weight(1f)`/`maxLines = 1`, trivial one-off tweaks).
   - Structured data uses `@Serializable` + `kotlinx.serialization` only (**forbidden**: hand-built/regex JSON).
   - Never crash, swallow errors, or fabricate fallback defaults. Render **`-`** (or `--`) for failed/missing **both numeric and text** fields (**never `0`**), paired with explicit error UI or MVI `Effect` (never trigger side effects inside `@Composable` bodies).

## Visual Artifacts (Headless Screenshots for Human Review)

For non-trivial UI work (new/restructured screens/components, layout/alignment/theme changes, visual bug fixes; skip trivial copy or non-UI edits):
- **Official CMP `ui-test` Only**: Capture headlessly in `jvmTest` via `org.jetbrains.compose.ui:ui-test` (`runDesktopComposeUiTest` + `captureToImage()`). Do **not** add third-party screenshot libraries (Roborazzi, Dropshots) or Android-only plugins unless explicitly requested or already present.
- **Size-Driven Mobile & Desktop Capture**: Drive responsive layout by window size (`currentWindowAdaptiveInfo()` / `BoxWithConstraints`), never OS checks. Match pixels and `Density` to hit target logical `dp`: at minimum **mobile portrait** (`390dp x 844dp`) and **desktop/web expanded** (`1280dp x 800dp`), plus dark theme or error state when affected. Note `jvmTest` limits (`0.dp` system `WindowInsets`, no native `AndroidView`/`UIKitView` or browser-only `wasmJs` behavior) when relevant.
- **Realistic Test-Only Mock Data**: Render stateless `Screen`/components with domain-realistic deterministic fixtures (multiple items, fixed timestamps, stress cases like long titles or error `-` states; no empty defaults or `"Test"`/`"Lorem ipsum"`). Keep mocks strictly in test source sets, never `commonMain`. Pair with Compose assertions where useful.
- **Output & Report**: Write PNGs to a gitignored directory (e.g., `build/outputs/ui-verify/<screen>-<viewport>-<theme>-<state>.png`) and **keep them on disk**. Do **not** require AI vision inspection; **always list every generated PNG path with logical viewport, theme, and state in the final response** for human review.

## Stack Defaults & Quick Routing

**Defaults**: Koin Multiplatform (DI), Ktor Client + `kotlinx.serialization` (network), `Res.*` + `currentWindowAdaptiveInfo()` (resources/adaptive), Turbine in `commonTest` + `ui-test` in `jvmTest` (testing).

**Rule Precedence**: Rules in this `SKILL.md` strictly override shorthand inline literals (`16.dp`, `"Saved"`), file bundling, or older Android-oriented snippets in `references/`. **Skip `references/` for basic Compose work**; load **at most one** file only for deep topics:

- **Architecture & Review**: [architecture.md](references/architecture.md) (pipeline/domain), [mvi.md](references/mvi.md), [mvvm.md](references/mvvm.md), [clean-code.md](references/clean-code.md) (files/naming), [anti-patterns.md](references/anti-patterns.md)
- **Core UI & UX**: [compose-essentials.md](references/compose-essentials.md) (phases/effects/modifiers), [material-design.md](references/material-design.md) (M3/adaptive), [lists-grids.md](references/lists-grids.md), [ui-ux.md](references/ui-ux.md) (loading/validation), [accessibility.md](references/accessibility.md), [image-loading.md](references/image-loading.md) (Coil 3/SVG)
- **Animations & Perf**: [animations.md](references/animations.md), [animations-advanced.md](references/animations-advanced.md) (shared element/gestures/Canvas), [performance.md](references/performance.md) (recomposition/stability)
- **Coroutines & Paging**: [coroutines-flow.md](references/coroutines-flow.md), [coroutines-flow-advanced.md](references/coroutines-flow-advanced.md) (backpressure/Mutex/Turbine), [paging.md](references/paging.md) (KMP), [paging-offline.md](references/paging-offline.md), [paging-mvi-testing.md](references/paging-mvi-testing.md)
- **Navigation**: [navigation.md](references/navigation.md) (Nav 2 vs 3), [navigation-3.md](references/navigation-3.md) (`SerializersModule`), [navigation-2.md](references/navigation-2.md), [navigation-3-di.md](references/navigation-3-di.md) / [navigation-2-di.md](references/navigation-2-di.md) (Koin), [navigation-migration.md](references/navigation-migration.md)
- **Data, Network & DI**: [datastore.md](references/datastore.md) (1.3+ `WebLocalStorage`), [room-database.md](references/room-database.md) (Room 3.0+ vs 2.x), [networking-ktor.md](references/networking-ktor.md), [networking-ktor-auth.md](references/networking-ktor-auth.md), [networking-ktor-architecture.md](references/networking-ktor-architecture.md), [networking-ktor-testing.md](references/networking-ktor-testing.md), [koin.md](references/koin.md)
- **Platform, Test & Build**: [cross-platform.md](references/cross-platform.md) (`expect/actual`/`wasmJs`), [ios-swift-interop.md](references/ios-swift-interop.md) (SKIE), [resources.md](references/resources.md) (`Res`/fonts), [testing.md](references/testing.md) (Turbine/`jvmTest` screenshots), [gradle-build.md](references/gradle-build.md), [ci-cd-distribution.md](references/ci-cd-distribution.md)
