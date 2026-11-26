# Navigation 2 + Dependency Injection

DI wiring for Nav 2 destinations in Compose Multiplatform: destination-scoped and graph-scoped ViewModels with Koin.

For Nav 2 core reference (NavHost, tabs, deep links, animations), see [navigation-2.md](navigation-2.md).
For shared navigation concepts and anti-patterns, see [navigation.md](navigation.md).

## Koin Integration

### koinViewModel in composable destinations

Standard ViewModel injection using `koinViewModel()`:

```kotlin
composable<Detail> {
    val detail: Detail = it.toRoute()
    DetailScreen(viewModel = koinViewModel { parametersOf(detail.itemId) })
}
```

### koinNavViewModel — auto-populated SavedStateHandle

`koinNavViewModel()` automatically populates the ViewModel's `SavedStateHandle` with navigation arguments. The ViewModel receives route params via its constructor without manual extraction:

```kotlin
class DetailViewModel(
    private val repository: ItemRepository,
    savedStateHandle: SavedStateHandle,
) : ViewModel() {
    private val itemId: String = checkNotNull(savedStateHandle["itemId"])
}

// Module declaration
val featureModule = module {
    viewModelOf(::DetailViewModel)
}

// Composable destination — SavedStateHandle auto-populated with nav args
composable("detail/{itemId}") {
    val viewModel = koinNavViewModel<DetailViewModel>()
    DetailScreen(viewModel = viewModel)
}
```

### sharedKoinViewModel — graph-scoped sharing

Share a ViewModel within a navigation graph. The shared instance lives as long as the graph's back stack entry:

```kotlin
navigation(startDestination = "checkout/cart", route = "checkout") {
    composable("checkout/cart") { entry ->
        val sharedVm = entry.sharedKoinViewModel<CheckoutViewModel>(navController)
        CartScreen(viewModel = sharedVm)
    }
    composable("checkout/shipping") { entry ->
        val sharedVm = entry.sharedKoinViewModel<CheckoutViewModel>(navController)
        ShippingScreen(viewModel = sharedVm)
    }
}
```

Use `sharedKoinViewModel` to share a ViewModel instance across all destinations inside a nested navigation graph.

### Quick reference — Koin Nav 2 injection functions

| Function | Purpose |
|---|---|
| `koinViewModel<T>()` | Standard injection — new instance per destination |
| `koinNavViewModel<T>()` | Like `koinViewModel` but auto-populates `SavedStateHandle` with nav arguments |
| `sharedKoinViewModel<T>(navController)` | Share ViewModel within a navigation graph (experimental) |
| `koinViewModel(parameters = { parametersOf(...) })` | Pass runtime values to the ViewModel constructor |
