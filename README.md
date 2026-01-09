# Kitten

**Fast and Safe Kotlin Multiplatform Dependency Injection Library**

<img src="https://github.com/Open-Store-Foundation/brandbook/blob/main/Kitten/KittenDi.png?raw=true" width="100%" alt="hello">

## Overview

Kitten is a dependency injection library suitable for projects of all sizes, from small prototypes
to massive multi-module Kotlin Multiplatform applications. It focuses on simplicity, speed, and safety
without the overhead of complex code generation or reflection.

## Key Features

* 🪶 **Lightweight**: The entire library footprint is approximately **5 KB**.
* ⚡ **Fast**: Uses **no reflection, code generation, or compiler plugins**. It relies entirely on
  standard Kotlin code.
* 🌍 **Multiplatform**: Designed for **Kotlin Multiplatform** (Android, iOS, Desktop, Web).
* 🧩 **Modular Architecture**: Promotes clean separation between **API** (interfaces) and **Core** (
  implementation) modules.
* 🛡️ **Type-Safe**: Unlike Dagger 2, Koin, or Kodein, you explicitly define implementations. The API
  remains minimal while ensuring compile-time safety.
* ✨ **Simple**: Significantly less boilerplate compared to Dagger 2.
* 🔄 **Lifecycle Management**: Built-in support for managing the lifecycle of **components**, 
  **dependency sets**, and **individual dependencies**.

---

## Modular Architecture Guide

You don't need to create a separate module for every feature. Instead, group related features into
modules. If a screen or component is shared across multiple modules, move it to a common module.

Your module graph should eventually look like this:

![Module Architecture](https://user-images.githubusercontent.com/15245196/155395076-9c6e679d-3444-4455-9c8c-2d9e1903e480.png)

---

## Integration Guide

### 1. Add Core Dependency

Add the `core` dependency to your **Main Library** (Application Entrypoint).

```kotlin
implementation("foundation.openstore.kitten:core:1.1.0")
```

### 2. Add API Dependency

Add the `api` dependency to your **Secondary Modules** (Feature Modules).

```kotlin
implementation("foundation.openstore.kitten:api:1.1.0")
// OR with Android helpers
implementation("foundation.openstore.kitten:viewmodel:1.1.0")
```

### 3. Define Dependencies

Create your classes and interfaces as usual.

```kotlin
// Simple Dependencies
class Seed(val num: Int)
class NetworkObserver(app: Application, val seed: Seed)

// Dependency with Interface
interface Service
class ServiceDefault(net: NetworkObserver) : Service

// Dependency with Data
class Data
interface Repo
class RepoDefault(val id: Data, val service: Service) : Repo
```

### 4. Create Components in Main Module

Define how your components provide dependencies.

```kotlin
// Main Component Interface
interface AppComponent : Component {
    val networkObserver: NetworkObserver
}

// Main Component Implementation
class AppComponentDefault(
    private val app: Application,
) : AppComponent {
    private val seed: Seed by depLazy {
        Seed(Random.nextInt()) // Generated once per app session
    }

    override val networkObserver: NetworkObserver by depLazy {
        NetworkObserver(app, seed)
    }
}

// Data Component Interface
interface DataComponent : Component {
    fun provideRepo(data: Data): Repo
}

// Data Component Implementation
class DataComponentDefault(
    private val appCmp: AppComponent
) : DataComponent {
    private val service: Service by depLazy {
        ServiceDefault(appCmp.networkObserver)
    }

    override fun provideRepo(data: Data): Repo {
        RepoDefault(data, service)
    }
}
```

### 5. Create Injector in Secondary Modules

Each feature module should expose an Injector and Component interface.

```kotlin
// Feature Definitions
interface FooFeature
class FooFeatureViewModel(val repo: Repo) : FooFeature

// Feature Component Interface
interface FooComponent : Component {
    fun provideFooFeature(data: Data): FooFeature
}

// Feature Component Implementation
class FooComponentDefault(
    val dataCmp: DataComponent
) : FooComponent {
    override fun provideFooFeature(data: Data): FooFeature {
        return FooFeatureViewModel(dataCmp.provideRepo(data))
    }
}

// Module Injector Object
object ModInjector : Injector<FooComponent>()
```

### 6. Create Component Provider in Main Module

This class manages the scope and lifecycle of your components.

```kotlin
class AppComponentRegistry(
    private val app: Application
) : ComponentRegistry() {

    // Singleton: Lives for the entire lifecycle of the provider
    val appCmp: AppComponent by singleton {
        AppComponentDefault(app)
    }

    val dataComp: DataComponent by singleton {
        DataComponentDefault(appCmp)
    }

    // Shared: Lives as long as at least one owner/sub-owner is alive
    val fooCmp: FooComponent by shared {
        FooComponentDefault(dataComp)
    }
}
```

### 7. Initialize Kitten in Application Class

Wire everything together in your `Application.onCreate`.

```kotlin
class Application : Application() {

    override fun onCreate() {
        super.onCreate()

        Kitten.init(
            registry = AppComponentRegistry(this)
        ) { registry ->
            // Eagerly create components
            create { registry.appCmp }
            create { registry.dataComp }

            // Register injectors
            register(ModInjector) { registry.fooCmp }
        }
    }
}
```

### 8. Inject Dependencies in Feature Fragments

Retrieve dependencies in your UI components.

```kotlin
class FooFragment : Fragment() {

    // Example: Standard View Injection
    fun onAttach() {
        val feature = ModInjector.injectWith(this) { provideFooFeature(Data()) }

        // Short syntax
        val feature1 = ModInjector.inject { provideFooFeature(Data()) }

        // Android ViewModel syntax
        val viewModel = ModInjector.viewModelLegacy { provideFooFeature(Data()) }
    }

    // Example: Jetpack Compose
    @Composable
    fun Content() {
        val feature = ModInjector.injectWith(this) { provideFooFeature(Data()) }

        // Short syntax
        val feature1 = ModInjector.inject { provideFooFeature(Data()) }

        // ViewModel syntax
        val viewModel = ModInjector.viewModel { provideFooFeature(Data()) }
    }
}
```

---

## Advanced: Scoped Components (Dynamic Injection)

Refactor your components to support dynamic data injection using shared (scoped) components.

```kotlin
// 1. Refactor DataComponent Interface
interface DataComponent : Component {
    val provideRepo: Repo // Change: Method -> Property
}

// 2. Refactor DataComponent Implementation
class DataComponentDefault(
    private val appCmp: AppComponent,
    private val data: Data // Change: Pass Data via Constructor
) : DataComponent {
    private val service: Service by depLazy {
        ServiceDefault(appCmp.networkObserver)
    }

    override val provideRepo: Repo by depLazy { // Change: Method -> Property
        RepoDefault(data, service)
    }
}

// 3. Update Feature Component to use DynamicComponent
class FooComponentDefault(
    // Change: Inject DynamicComponent wrapper
    val dataCmp: ComponentProvider<Data, DataComponent>,
) : FooComponent {
    override fun provideFooFeature(data: Data): FooFeature {
        // Change: Create component for specific data
        return FooFeatureViewModel(dataCmp[data].provideRepo)
    }
}

// 4. Update Provider
class AppComponentRegistry(
    private val app: Application
) : ComponentRegistry() {

    val appCmp: AppComponent by singleton {
        AppComponentDefault(app)
    }

    // Helper method to create DataComponent
    fun dataComponent(data: Data): DataComponent {
        // Usage: shared(key, factory)
        // Use local delegated property to resolve the component instance
        val component by shared(data) { DataComponentDefault(appCmp, data) }
        return component
    }
}

// 5. Update Initialization
Kitten.init(
    registry = AppComponentRegistry(this)
) { registry ->
    create { registry.appCmp }

    register(ModInjector) {
        FooComponentDefault(
            // Pass the factory lambda
            dataCmp = { data -> registry.dataComponent(data) }
        )
    }
}
```

## Advanced: Strict Modularization (External Dependencies)

In strict modular architectures, a feature component might require dependencies from other components
(like `AppComponent` or `DataComponent`) without depending on those components directly.
This is achieved by defining a `Deps` interface within the feature component.

```kotlin
// 1. Feature Component with Deps Interface
interface FooComponent : Component {
    fun provideFooFeature(): FooFeature
    
    // Define requirements here
    interface Deps {
         fun provideRepo(data: Data): Repo
    }
}

// 2. Feature Implementation
class FooComponentDefault(
    // Depend on Deps interface, not concrete components
    private val deps: FooComponent.Deps 
) : FooComponent {
    override fun provideFooFeature(): FooFeature {
        // Use dependencies from Deps
        val repo = deps.provideRepo(Data())
        return FooFeatureViewModel(repo)
    }
}

// 3. Registry Wiring in Main Module
class AppComponentRegistry(...) : ComponentRegistry() {
    
    // ... appCmp and dataComponent definitions ...

    val fooCmp: FooComponent by shared {
         FooComponentDefault(
             // Implement Deps using available components
             deps = object : FooComponent.Deps {
                 override fun provideRepo(data: Data): Repo {
                     return dataComponent(data).provideRepo
                 }
             }
         )
    }
}
```
