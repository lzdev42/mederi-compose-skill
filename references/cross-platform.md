# Cross-Platform (KMP) Specifics

## Sharing Strategy

Share these first: reducers, ViewModels, validators, calculators, formatting policies, screen state models, most screen UI.

Keep platform-specific until proven otherwise: permissions, share sheets, clipboard, haptics, file pickers, notifications, deep links, review prompts, platform input traits, OS navigation shell.

## Placement Guide

### What belongs in `commonMain`

Feature state, intents/messages, reducer/ViewModel logic, calculators, validators, eligibility, repository interfaces, use cases that earn their keep, shared composables, presentation mapping, semantic nav effects and error keys.

### What should remain platform-specific

Runtime permissions, share/open sheet, haptics, clipboard, URLs, billing, notifications, biometrics, manifest/delegate deep links, OS widgets/shortcuts.

### Placement Table

| Concern | Default placement | Why |
|---|---|---|
| reducer/ViewModel | `commonMain` | pure, testable, reusable |
| validator/calculator | `commonMain` | pure domain logic |
| repository contract | `commonMain` | shared dependency boundary |
| haptics/share/clipboard | interface + platform impl | app capability, easy to fake |
| locale/number/date formatter | interface or shared library | locale-sensitive behavior |
| resource identifiers | `commonMain` UI | shared UI uses shared resources |
| permission prompt flow | platform-specific | OS-specific behavior |
| safe-area / keyboard handling | route/UI boundary | platform behavior differs |
| navigation controller binding | platform/UI shell | ViewModel should not know controller type |
| analytics SDK integration | platform or shared facade | real implementation differs |

### Dependency Verification for commonMain

**Before claiming `commonMain`:** confirm multiplatform artifacts exist. Much of AndroidX is still Android-only; some libs publish KMP (e.g. `lifecycle-viewmodel`, `datastore-preferences`) with version-dependent surfaces. Check Maven for `-jvm`, `-iosarm64`, `-iosX64`, etc.; use context7 `resolve-library-id` + `query-docs` when available. **If unverifiable**, say so—use platform placement or wrapper interfaces.

## Interfaces vs expect/actual

### Default recommendation

Use **interfaces** for app capabilities: haptics, clipboard, share, URL opener, analytics, date/number formatting, file opener.

Use **`expect/actual`** for thin platform facts or one-off helpers when an interface buys little.

### Practical rule

- **Interface** when the capability has lifetime, DI, fakes, or multiple implementations
- **`expect/actual`** when it is a tiny platform hook with no domain meaning

### Dependency Injection

Heavy/async/hardware services (GPS, biometrics, keystore): `commonMain` interface + Koin (or similar) for platform impls. Reserve `expect/actual` for tiny sync primitives (UUID, dates, clipboard).

## Platform Bridge Patterns

The rules above cover *when* to prefer interfaces vs `expect/actual`; below is *how* to wire each pattern.

### Choosing the Right Bridge

| Need | Pattern | Why |
|---|---|---|
| Service with lifecycle, state, or async (player, auth, payments, analytics) | Interface + DI | Testable, fakeable, swappable impls |
| Stateless platform fact (UUID, platform name, default locale) | `expect/actual` function | No DI overhead for a one-liner |
| Reuse existing platform type in common signature | `expect class` + `actual typealias` | Rare — prefer interface when possible |

### Pattern 1: Interface + DI (Primary)

Contract in `commonMain`; platform modules supply impls; DI binds them. ViewModel depends only on the interface. Koin setup: [koin.md](koin.md).

```kotlin
// commonMain
interface Player { fun play(uri: String); fun pause(); fun release() }

// androidMain
class AndroidPlayer(private val context: Context) : Player {
    private val mp = MediaPlayer()
    override fun play(uri: String) { mp.setDataSource(context, uri.toUri()); mp.start() }
    override fun pause() = mp.pause()
    override fun release() = mp.release()
}

// iosMain
class IosPlayer : Player {
    private var av: AVPlayer? = null
    override fun play(uri: String) { av = AVPlayer(uRL = NSURL(string = uri)); av?.play() }
    override fun pause() { av?.pause() }
    override fun release() { av = null }
}

// androidMain
val androidPlayerModule = module { single<Player> { AndroidPlayer(get()) } }
// iosMain
val iosPlayerModule = module { single<Player> { IosPlayer() } }

class PlayerViewModel(private val player: Player) : ViewModel() {
    fun onEvent(e: PlayerEvent) {
        when (e) { is PlayerEvent.Play -> player.play(e.uri); PlayerEvent.Pause -> player.pause() }
    }
}
```

### Pattern 2: expect/actual for Thin Platform Primitives

Stateless one-liners, no DI/interface/fakes. Note: do **not** use `expect/actual` for UUIDs or timestamps — use Kotlin stdlib `kotlin.uuid.Uuid` and `kotlin.time.Clock` / `kotlin.time.Instant` directly in `commonMain`:

```kotlin
// commonMain
expect fun platformName(): String

// androidMain
actual fun platformName(): String = "Android ${android.os.Build.VERSION.SDK_INT}"

// iosMain
actual fun platformName(): String =
    "${platform.UIKit.UIDevice.currentDevice.systemName()} ${platform.UIKit.UIDevice.currentDevice.systemVersion()}"

// jvmMain
actual fun platformName(): String = System.getProperty("os.name") ?: "Desktop"

// wasmJsMain
actual fun platformName(): String = "Web (wasmJs)"
```

### Pattern 3: expect/actual with Platform Wrapper (Genuine OS Bridges Only)

When wrapping a genuine OS capability without pulling platform types into shared APIs:

```kotlin
// commonMain
expect class ExternalUrlOpener {
    fun open(url: String)
}

// jvmMain (Desktop — unavoidable java.awt.Desktop encapsulated inside jvmMain impl)
actual class ExternalUrlOpener {
    actual fun open(url: String) {
        if (java.awt.Desktop.isDesktopSupported()) {
            java.awt.Desktop.getDesktop().browse(java.net.URI(url))
        }
    }
}
```

Prefer interface+DI for fakes or when lifecycle/async behavior is involved.

### Kotlin/KMP-First on JVM & Android Too

Even inside `jvmMain`, `desktopMain`, and `androidMain`, default to Kotlin stdlib and mature KMP libraries (`kotlin.time.Clock`/`Instant`, `kotlin.uuid.Uuid`, `kotlinx.serialization`, `kotlinx-io`, Ktor, Koin). Do **not** introduce Java/JDK libraries (e.g., Gson, Jackson, Guava, Apache Commons, direct OkHttp, `java.time`, `java.util.UUID`, `SimpleDateFormat`) unless **no mature Kotlin/KMP alternative exists** or when bridging genuine OS/JVM/Android platform APIs (e.g., desktop `java.awt.Desktop`, OS app-data paths, Android SDK APIs, or writing `jvmTest` PNG outputs). Keep any unavoidable `java.*` types strictly encapsulated inside platform implementations—never leak them into shared APIs.

### Bridge Anti-Patterns

- Platform leakage in state
- `Dispatchers.IO`, `runBlocking`, or JVM `synchronized` in `commonMain`

## `wasmJs` Target Guardrails

Web (`wasmJs`) enforces stricter runtime constraints than JVM or Native:
- **No `Dispatchers.IO` or blocking calls**: `Dispatchers.IO` and `runBlocking` do not exist on `wasmJs`; use `Dispatchers.Default` in `commonMain` (or inject platform dispatchers where JVM/Native requires blocking IO) and keep execution single-threaded and non-blocking.
- **No JVM reflection**: Never use reflection-based serialization, navigation key parsing, or DI scanning; register explicit `SerializersModule` and explicit Koin module bindings.
- **Prefer Interface + Koin DI over `expect/actual` sprawl**: Use `commonMain` interfaces bound in platform Koin modules (`androidMain`, `iosMain`, `jvmMain`, `wasmJsMain`) for capabilities with lifecycle, state, or async behavior; reserve `expect/actual` for tiny stateless primitives.

## Lifecycle

`lifecycle-viewmodel` / `lifecycle-runtime-compose` can expose `ViewModel`, `viewModelScope`, `collectAsStateWithLifecycle` in `commonMain`; not all Lifecycle APIs are MP—depends on androidx/KMP.

- Artifact must publish KMP targets (`-jvm`, `-iosarm64`, …) and expose the API on MP (many APIs stay Android-only); match project targets.
- **Confirm versions** via context7 or AndroidX notes; if not, say so—wrap platform lifecycle behind interfaces if needed.
- **Typical in `commonMain` (re-verify):** `ViewModel`, `viewModelScope`, `collectAsStateWithLifecycle`, `koinViewModel()`.

## State Restoration

- `rememberSaveable`: small local UI state only
- Cross-platform drafts: rehydrate from persistence, not assumed OS restoration parity
- Serialize ViewModel state only when product requires it

## Keyboard, Focus, and Input

- Test text input on real iOS hardware; isolate quirks at the UI/platform edge
- No keyboard workaround flags in reducer state; shared UI uses inset/safe-area layout
- Keep selection/composition local per field when needed

## Safe Area and Layout

Insets-aware shared layouts; verify safe areas, keyboard overlap, sheets, nav chrome. Never put “iOS safe-area hack” into feature state.

## Platform Capabilities

Model haptics, clipboard, share as semantic effects; shell executes them.

```kotlin
enum class HapticType { Confirm, Error, Selection }
interface Haptics { fun perform(type: HapticType) }
sealed interface ProductEffect {
    data class TriggerHaptic(val type: HapticType) : ProductEffect
    data class ShareQuote(val text: String) : ProductEffect
}
interface ShareText { suspend fun share(text: String) }
```

## Resources

CMP shared resources (strings, images, fonts, qualifiers, localization, Gradle setup). Full API surface: **[Multiplatform Resources](resources.md)**.

```kotlin
enum class ValidationMessageKey { Required, InvalidNumber, MustBePositive }

@Composable
fun ValidationMessage(messageKey: ValidationMessageKey?) {
    val text = when (messageKey) {
        ValidationMessageKey.Required -> stringResource(Res.string.error_required)
        ValidationMessageKey.InvalidNumber -> stringResource(Res.string.error_invalid_number)
        ValidationMessageKey.MustBePositive -> stringResource(Res.string.error_must_be_positive)
        null -> return
    }
    Text(text = text, color = MaterialTheme.colorScheme.error, style = MaterialTheme.typography.bodySmall)
}
```

## Code Examples

### GOOD: shared calculator

```kotlin
class PriceCalculator {
    fun calculate(d: PriceDraft): PriceDerived {
        val w = if (d.includeWaste) 1.10 else 1.0
        val mat = d.area * d.materialRate * w
        val lab = d.area * d.laborRate
        val sub = mat + lab
        val tax = sub * (d.taxPercent / 100.0)
        return PriceDerived(mat, lab, sub, tax, sub + tax)
    }
}
```

### BAD: platform leakage in state

```kotlin
@Immutable
data class ProductState(
    val input: ProductInput = ProductInput(),
    val iosKeyboardInsetHack: Int = 0,
    val androidHapticPattern: String = "",
    val shareSheetPresented: Boolean = false,
)
```

Platform leakage.
