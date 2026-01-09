# Kitten
Kotlin DI fast and safe library

<img src="https://i.pinimg.com/236x/ae/a3/5a/aea35a7874af4c09d2ee73998d8f8b6d.jpg">

## Why?
This library fit to small and espesially **huge** projects. You can use it for multimodule Kotlin/Java project.

Advantages of this libraries:
- **Lightweight** - the entire library takes only 5 KB space
- **Fast** - this library **doesn't use Codegen or Reflection**, only your code
- **Api/Core Modules** - you can connect super-lightweight **api** module to feature libraries, and **core** module for main library
- **Safe** - unlike Dagger 2, kodin or koin you have to write all implmentation of objects, but API of this library really short
- **Simple** - it's probably takes less code than Dagger 2
- **Lifecycle Management** - there are a lot of helpers in library to mange lifecyle of **components/deps set/dep**

## How to make module system?
Commonly you don't have to create a lot of modules in your application, especially if you are using Gradle.
</br>
Try to create modules like a group of features. If some screen/parts are using in several modules, you can move it to common module.
</br>
Eventially your module system should looks like this.
![image](https://user-images.githubusercontent.com/15245196/155395076-9c6e679d-3444-4455-9c8c-2d9e1903e480.png)


## Guide
### 1. Add ":core" dependency to your Main Library (Application Entrypoint)
``` kotlin
implementation("io.github.andrewchupin:core:1.1.0")
```
### 2. Add ":api" dependency to your Secondary Modules (Feature Entrypoint)
``` kotlin
implementation("io.github.andrewchupin:api:1.1.0")
// or with android helpers
implementation("io.github.andrewchupin:android:1.1.0")
```
### 3. Create some dependecies
``` kotlin
// Dependencies
class Seed(val num: Int)
class NetworkObserver(app: Application, val seed: Seed)

// Dependency with interface
interface Service
class ServiceDefault(net: NetworkObserver) : Service

// Dependency with data
class Data
interface Repo
class RepoDefault(val id: Data, val service: Service) : Repo
```

### 4. Create some components in Main Module
``` kotlin
// Main Component
interface AppComponent : Component {
    val networkObserver: NetworkObserver
}

class AppComponentDefault(
    private val app: Application,
) : AppComponent {
    private val seed: Seed by depLazy {
        Seed(Random.nextInt()) // random each app session
    }

    override val networkObserver: NetworkObserver by depLazy {
        NetworkObserver(app, seed)
    }
}

// Feature Component (have to provide something)
interface DataComponent : Component {
    fun provideRepo(data: Data): Repo
}

class DataComponentDefault(
    private val appCmp: AppComponent
) : DataComponent {
    private val service: Service by depLazy {
        ServiceDefault(appCmp.networkObserver)
    }
    
    override fun provideRepo(data: Data): Repo {
       RepoDefault(data, dataCmp.service)
    }
}
```

### 5. Create Injector in each Secondary Module
``` kotlin
// Feature component
interface FooFeature
class FooFeatureViewModel(val repo: FooRepo) : FooFeature

interface FooComponent: Component {
    fun provideFooFeature(data: FooData): FooFeature
}

class FooComponentDefault(
    val dataCmp: DataComponent
) : FooComponent {
   override fun provideFooFeature(data: Data): FooFeature {
        return FooFeatureViewModel(dataCmp.serviceRepo, dataCmp.provideRepo(data))
   }
}

object ModInjector : Injector<FooComponent>()
```

### 6. Create Component Provider in Main Module
``` kotlin
class AppComponentProvider(
    private val app: Application
) : ComponentProvider() {

    // Live entire lifcycle of first owner (e.g. GlobalScope)
    val appCmp: AppComponent get() = singleton {
        AppComponentDefault(app)
    }

    val dataComp: DataComponent get() = singleton {
        DataComponentDefault(appCmp)
    }
    
    // Live when at least one owner/subowner is alive
    val fooCmp: FooComponent get() = scoped {
        FooComponentDefault(dataComp)
    }
}
```


### 7. Init Injector in Main Module

``` kotlin
class Application {

    fun onCreate() {
        Kitten.init(
            provider = AppComponentProvider(this)
        ) {  deps ->
            // create components immediately
            create { deps.appCmp }
            create { deps.dataComp }

            // Init delegate without deps and components
            register(ModInjector) { daps.fooCmp }
        }
    }
}
```


### 8. Get your dependencies in each Secondary Module
``` kotlin
class FooFragment : ComponentLifecycle {
    // View
    fun onAttach() {
        val feature = ModInjector.injectWith(this) { provideFoo(Data()) }
        // or short example
        val feature1 = ModInjector.inject { provideFoo(Data()) }
        // or viewModel short example
        val viewModel = ModInjector.viewModelLegacy { provideBar(Data()) }
    }
    
    // Compose
    @Composable
    fun Content() {
        val feature = ModInjector.injectWith(this) { provideBar(Data()) }
        // or short example
        val feature1 = ModInjector.inject { provideBar(Data()) }
        // or viewModel short example
        val viewModel = ModInjector.viewModel { provideBar(Data()) }
    }
}
```

## Scoped Component
```kotlin
interface SomeDependency
class SomeDependencyDefault(val data: Data) : SomeDependency
interface SomeComponent: Component {
    val someDependency: SomeDependency
}

class SomeComponentDefault(
    val data: Data // CHANGED: ADD DATA TO CONSTRUCTUR INSTEAD OF METHOD
) : FooComponent {
    override val someDependency: SomeDependency by depLazy {
        return SomeDependencyDefault(data)
    }
}

class FooComponentDefault( 
    val dataCmp: DataComponent,
    val someCmp: DynamicComponent<Data, SomeComponent>, // CHANGED: DYNAMIC COMPONENT PROVIDER
) {
    override fun provideFooFeature(data: Data): FooFeature {
        return FooFeatureViewModel(dataCmp.serviceRepo, dataCmp.provideRepo(data), someCmp.for(data)) // CHANGED: CREATE DYNAMIC COMPONENT FOR DATA
   }
}

class AppComponentProvider(
    private val app: Application
) : ComponentProvider() {
    ...
    fun someComponent(data: data): SomeComponent { // CHANGED: DYNAMIC COMPONENT CREATION
        // Live when at least one owner/subowner is alive with the same data
        return scoped(data) { SomeComponentDefault(data) } 
    }
    ...
}

Kitten.init(
    provider = AppComponentProvider(this)
) {  deps ->
    // create components immediately
    create { deps.appCmp }
    create { deps.dataComp }

    // Init delegate without deps and components
    register(ModInjector) {
        FooComponentDefault(
            dataComp = dataComp,
            someCmp = { data -> SomeComponent(data, deps.dataCmp) } // CHANGED: DYNAMIC COMPONENT PROVIDER
        )
    }
}
```
