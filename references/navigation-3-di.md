# Navigation 3 + Dependency Injection

DI wiring for Nav 3 entries in Compose Multiplatform: entry-scoped ViewModels, modularization, and multi-module entry providers with Koin.

For Nav 3 core reference (routes, NavDisplay, scenes, animations), see [navigation-3.md](navigation-3.md).
For shared navigation concepts and anti-patterns, see [navigation.md](navigation.md).

## Entry-Scoped ViewModels

Nav 3 scopes ViewModels to entries via `rememberViewModelStoreNavEntryDecorator()`. Each entry gets its own `ViewModelStoreOwner` — VMs are created when the entry is added to the back stack and cleared when popped.

### BAD: Globally-scoped ViewModel for per-screen data

```kotlin
val viewModel: DetailViewModel = viewModel() // scoped too broadly, not entry-scoped
```

### GOOD: Entry-scoped ViewModel

```kotlin
// Requires rememberViewModelStoreNavEntryDecorator() in entryDecorators
val viewModel: DetailViewModel = viewModel() // scoped to entry via decorator
```

For shared state across entries, lift state to a parent composable or use a shared ViewModel at the App scope.

## Koin Integration

For general Koin setup, modules, and scopes, see [koin.md](koin.md). Below covers Nav 3–specific patterns only.

### koinViewModel in entry blocks (Android + CMP)

```kotlin
entry<Details> { key ->
    val viewModel = koinViewModel<DetailViewModel> { parametersOf(key.id) }
    DetailScreen(viewModel = viewModel)
}
```

### Koin navigation DSL + koinEntryProvider

Declare navigation entries inside Koin modules. Koin aggregates them automatically — no manual entry provider needed:

```kotlin
val appModule = module {
    navigation<HomeRoute> { HomeScreen(viewModel = koinViewModel()) }
    navigation<DetailRoute> { route ->
        DetailScreen(viewModel = koinViewModel { parametersOf(route.id) })
    }
}

NavDisplay(
    backStack = rememberNavBackStack(HomeRoute),
    onBack = { backStack.removeLastOrNull() },
    entryProvider = koinEntryProvider(),
)
```

### Platform-specific extensions

| Function | Platform | Description |
|---|---|---|
| `koinEntryProvider<T>()` | All (CMP) | Composable entry provider — use in `commonMain` |
| `getEntryProvider<T>()` | Android | Eager entry provider via `AndroidScopeComponent` |

## Modularization

### api / impl module split

```text
feature-home/
  api/
    HomeNavKey.kt             -- @Serializable data object HomeNavKey : NavKey
  impl/
    HomeScreen.kt             -- composable UI
    HomeEntryBuilder.kt       -- extension function on EntryProviderScope
```

- **api** — contains only the `NavKey` route definitions. Other features depend on this.
- **impl** — contains UI, ViewModels, and entry builder. Depends on its own api + other features' api modules.

### Entry builder extension functions

Each feature exposes an extension function; the app module aggregates them:

```kotlin
// feature-home/impl
fun EntryProviderScope<NavKey>.homeEntry(navigator: Navigator) {
    entry<HomeNavKey> {
        HomeScreen(onItemClick = { navigator.navigate(DetailsNavKey(it)) })
    }
}

// app module
NavDisplay(
    entryProvider = entryProvider {
        homeEntry(navigator)
        searchEntry(navigator)
        profileEntry(navigator)
    },
    // ...
)
```

How you wire the ViewModel and state inside each entry depends on your project's architecture. Navigation is driven by ViewModel effects — the route layer translates semantic effects to back-stack operations.

### Koin module aggregation (CMP)

```kotlin
// Feature module
val featureModule = module {
    navigation<HomeNavKey> { HomeScreen(viewModel = koinViewModel()) }
    navigation<ProfileNavKey> { ProfileScreen(viewModel = koinViewModel()) }
}

// App module
NavDisplay(
    backStack = backStack,
    onBack = { backStack.removeLastOrNull() },
    entryProvider = koinEntryProvider(),
)
```
