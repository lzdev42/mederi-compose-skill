# Testing Strategy

## What to Test in commonMain

### ViewModel Tests with Turbine (highest ROI)

Test the full event→state→effect cycle through the ViewModel. Use `kotlinx-coroutines-test` with the **Turbine** library:

```kotlin
@Test
fun `save with empty title shows validation error`() = runTest {
    val viewModel = CreateItemViewModel(FakeItemRepository())

    viewModel.state.test {
        val initial = awaitItem()
        assertTrue(initial.errors.isEmpty())

        viewModel.onEvent(CreateItemEvent.OnSaveClick)
        val afterSave = awaitItem()

        assertEquals("Title is required", afterSave.errors["title"])
        assertFalse(afterSave.isSaving)
    }
}

@Test
fun `save with valid input transitions through saving to success`() = runTest {
    val viewModel = CreateItemViewModel(FakeItemRepository())

    viewModel.state.test {
        awaitItem() // initial

        viewModel.onEvent(CreateItemEvent.OnTitleChanged("New item"))
        awaitItem()
        viewModel.onEvent(CreateItemEvent.OnAmountChanged("42.5"))
        awaitItem()

        viewModel.onEvent(CreateItemEvent.OnSaveClick)
        val saving = awaitItem()
        assertTrue(saving.isSaving)

        val done = awaitItem()
        assertFalse(done.isSaving)
    }
}

@Test
fun `title changed clears title validation error`() = runTest {
    val viewModel = CreateItemViewModel(FakeItemRepository())

    viewModel.state.test {
        awaitItem() // initial

        viewModel.onEvent(CreateItemEvent.OnSaveClick)
        val withError = awaitItem()
        assertTrue(withError.errors.containsKey("title"))

        viewModel.onEvent(CreateItemEvent.OnTitleChanged("A"))
        val cleared = awaitItem()
        assertFalse(cleared.errors.containsKey("title"))
    }
}

@Test
fun `save emits ShowMessage effect on success`() = runTest {
    val viewModel = CreateItemViewModel(FakeItemRepository())

    viewModel.onEvent(CreateItemEvent.OnTitleChanged("New item"))
    viewModel.onEvent(CreateItemEvent.OnAmountChanged("10"))

    viewModel.effect.test {
        viewModel.onEvent(CreateItemEvent.OnSaveClick)
        val effect = awaitItem()
        assertTrue(effect is CreateItemEffect.ShowMessage)
    }
}
```

**What to test:**

- Event→state transitions: field edits, validation triggers, loading states
- Event→effect emissions: navigation, snackbar, error messages
- Async flows: loading → success, loading → failure, retry
- Edge cases: empty input, duplicate detection, concurrent saves
- State preservation: old content kept during refresh, error doesn't wipe data

### Testing State and Effects Separately

When a single event produces both state changes and effects, test them independently for clarity:

```kotlin
@Test
fun `back click emits NavigateBack effect without changing state`() = runTest {
    val viewModel = CreateItemViewModel(FakeItemRepository())

    viewModel.effect.test {
        viewModel.onEvent(CreateItemEvent.OnBackClick)
        assertEquals(CreateItemEffect.NavigateBack, awaitItem())
    }

    viewModel.state.test {
        val state = awaitItem()
        assertEquals(CreateItemState(), state)
    }
}
```

### Validation Tests

Test validation logic as pure functions when extracted into a dedicated validator:

```kotlin
@Test
fun `validator rejects blank title`() {
    val errors = CreateItemValidator.validate(title = "", amount = "10")
    assertEquals("Title is required", errors["title"])
}

@Test
fun `validator accepts valid input`() {
    val errors = CreateItemValidator.validate(title = "Widget", amount = "25.0")
    assertTrue(errors.isEmpty())
}
```

If validation is inline in the ViewModel (acceptable for simple cases), test it through ViewModel events as shown above.

### Calculation Engine Tests

Test pure calculation/domain services directly — no ViewModel needed:

```kotlin
@Test
fun `calculator computes correct monthly payment`() {
    val result = LoanCalculator.monthlyPayment(amount = 100000.0, rate = 5.0, years = 30)
    assertEquals(536.82, result, 0.01)
}
```

Test: edge cases, rounding policy, domain invariants, regression fixtures.

### Fake Repositories for ViewModel Tests

Use fakes (not mocks) for repositories and services:

```kotlin
class FakeItemRepository : ItemRepository {
    private val items = mutableListOf<Item>()
    var shouldThrow: Exception? = null

    override suspend fun create(title: String, amount: Double) {
        shouldThrow?.let { throw it }
        items.add(Item(title = title, amount = amount))
    }

    override suspend fun getAll(): List<Item> = items.toList()
}
```

Fakes give you control over success/failure scenarios without mock framework complexity.

## Compose UI Tests

Compose Multiplatform common UI testing uses `runComposeUiTest` rather than Android's JUnit `TestRule` model.

Test:

- Critical field entry flows
- Submit enable/disable behavior
- Error visibility
- Loading placeholder/content swap
- Preserved content during refresh
- Accessibility labels on critical controls

## Platform Tests

### Android/iOS specific

Test:

- Platform shell wiring
- Deep-link entry
- Navigation host integration
- Share sheet / clipboard / haptic bindings
- Platform lifecycle edge cases
- Keyboard/safe-area regressions

## Visual Artifacts (CMP Official Headless Screenshots for Human Review)

For non-trivial UI work — creating a new screen/component, changing layout or alignment, modifying theming/colors, or fixing a visual bug — always generate headless screenshots with realistic mock data using CMP's official `ui-test` tooling (no third-party screenshot libraries by default) and include them in the final report for human inspection.

1. **Trigger Scope**: Required for new/restructured UI, styling changes, and visual bug fixes. Skip for trivial copy edits, non-UI logic, or refactors with no visual impact.
2. **Zero Third-Party Dependencies by Default**:
   - Use only CMP's official `org.jetbrains.compose.ui:ui-test` (`runDesktopComposeUiTest` + `captureToImage()` in `jvmTest`).
   - Do **not** add third-party screenshot libraries (e.g., Roborazzi, Dropshots) or Android-only plugins (`com.android.compose.screenshot`) unless the user explicitly asks for them or the project already uses them.
3. **Size-Driven Responsive Capture (Mobile & Desktop via `jvmTest`)**:
   - Drive responsive layouts strictly by window size (`currentWindowAdaptiveInfo()` / `WindowWidthSizeClass` / `BoxWithConstraints`), never by OS checks, so `jvmTest` accurately renders mobile layouts at compact logical dimensions.
   - Match pixel dimensions and `Density` explicitly so logical `dp` hits the intended size class: at minimum **mobile portrait** (`390dp x 844dp` logical, e.g., `390x844` at `1f` or `780x1688` at `2f`) and **desktop/web expanded** (`1280dp x 800dp` logical), plus **dark theme** or **error/empty state** when affected.
   - Note limitations honestly in the report when relevant: `jvmTest` has `0.dp` system `WindowInsets` (no notch/status bar/IME) and cannot render native `AndroidView`/`UIKitView` interop or browser-only `wasmJs` behavior.
4. **Realistic, Deterministic Mock Data (Test-Only)**:
   - Render the stateless `Screen` or component directly with **realistic, domain-relevant mock data** (multiple realistic items, representative values, fixed timestamps, no infinite animations, empty callbacks). Never showcase a screen with empty defaults or `"Test"` / `"Lorem ipsum"` placeholders unless specifically capturing an empty state.
   - Include layout stress cases where relevant (long titles to verify true centering and ellipsis without sibling overlap, multi-line text, error states showing `-` placeholders).
   - Keep all mock fixtures strictly in test source sets (`jvmTest` / `commonTest`). Never place mock data in `commonMain` production code.
   - Pair capture with programmatic Compose assertions where useful (key nodes displayed, `-` shown on error, non-overlapping bounds).
5. **Output Hygiene & Final Report**:
   - Export PNGs to a gitignored build/output directory (e.g., `build/outputs/ui-verify/<screen>-<viewport>-<theme>-<state>.png`) and **keep the PNG files on disk** for human review. Never commit ad-hoc screenshots to tracked source directories.
   - **Do not require AI vision inspection** (agents may be text-only, and visual sign-off belongs to the human).
   - **Mandatory in the final response**: List every generated screenshot's exact path alongside its logical viewport, theme, and state (using clickable local file links when supported) so the user can open and inspect them directly.

## Lean Default Test Matrix

1. ViewModel event→state→effect tests for every feature in `commonTest` (via Turbine)
2. Validation/calculation/mapper tests for every rule-heavy feature (pure unit tests in `commonTest`)
3. Multiplatform Compose UI tests (`runComposeUiTest`) + headless `jvmTest` visual artifact screenshots (`runDesktopComposeUiTest` + `captureToImage()`) for non-trivial UI work
4. Platform integration tests only for real platform behavior (Android `Macrobenchmark` is Android-only and optional — never include it in `commonMain`/`commonTest`)

## Anti-Patterns

| Anti-pattern | Why it hurts | Better replacement |
|---|---|---|
| No ViewModel tests, only UI tests | slow feedback, flaky, hard to isolate failures | ViewModel event→state→effect tests with Turbine first |
| Testing implementation details (private functions, internal state) | brittle tests that break on refactoring | test through public API: send event, assert state/effect |
| Mocking the DI framework | couples tests to DI internals | swap real implementations with fakes via constructor injection |
| Adding third-party screenshot libraries/plugins unrequested or putting mock data in `commonMain` | bloats build, breaks multiplatform, pollutes prod code | use official `org.jetbrains.compose.ui:ui-test` in `jvmTest` with test-only realistic mocks |
| Testing derived/computed properties in isolation from ViewModel | duplicates logic, drifts from real behavior | test derived values through ViewModel state assertions |
| Sharing mutable test fixtures across tests | hidden coupling, order-dependent failures | fresh state per test, explicit setup in each test function |

## Domain-Specific Testing

Some reference files contain their own testing sections with domain-specific patterns:

| Domain | Reference | What it covers |
|---|---|---|
| Paging 3 | [paging-mvi-testing.md](paging-mvi-testing.md) | PagingSource unit tests, `asSnapshot`, `TestPager` transformations |
| Room Database | [room-database.md](room-database.md) | In-memory DB tests, migration tests, fake DAOs |
| Networking | [networking-ktor-testing.md](networking-ktor-testing.md) | MockEngine, API response testing, DI integration |
