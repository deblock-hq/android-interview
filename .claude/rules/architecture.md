# Architecture Components and Patterns Overview: deblock

This document outlines the key architectural components and design patterns preferred. Understanding these components and patterns is crucial for comprehending the codebase, facilitating development, and ensuring consistent pull request reviews.

The project primarily adheres to a Clean Architecture philosophy, leveraging a multi-module structure to enforce separation of concerns. This typically involves distinct layers for UI, Domain, and Data, with clear unidirectional dependencies.

## 1. UI Layer (Presentation Layer)

This layer is responsible for displaying information to the user and handling user interactions.

*   **Screens/Activities/Fragments/Composeables**: These are the visual components that users interact with. Specifically, `*Screen.kt` files typically contain the route definition for navigation and the root composable UI elements for a particular screen. They are passive observers of `ViewModel` states and dispatch user events back to the `ViewModel`.
*   **ViewModels**: Act as a state holder and event processor for the UI. They hold and manage UI-related data in a lifecycle-conscious way, exposing `StateFlow` or `LiveData` to the UI. `*ViewModel.kt` files contain the presentation logic, transforming data from the Domain Layer into UI-consumable states and handling UI-specific actions. They receive necessary dependencies via constructor injection.

### Common Patterns in UI Layer:
*   **MVVM (Model-View-ViewModel)**: The primary architectural pattern for UI.
    *   **View (Screens/Composeables)**: Observes `ViewModel` state, renders UI, and sends user events to the `ViewModel`.
    *   **ViewModel**: Exposes UI state (often via `StateFlow` for Compose or `LiveData` for Fragments/Activities) and handles UI-related logic, delegating business logic to Use Cases in the Domain Layer. It survives configuration changes.
    *   **FlowViewModel Pattern**: For multi-screen flows (e.g., transfers, top-up, onboarding), a combination of ViewModels is used:
        *   **Flow ViewModel** (e.g., `FiatTransferFlowViewModel`, `CardTopUpViewModel`): A shared ViewModel scoped to the entire navigation flow using `viewModelScopedTo()`. It orchestrates the flow, maintains shared state across screens (e.g., selected currency, amount, transaction details), and provides common logic and dependencies. This ViewModel survives navigation between screens within the flow.
        *   **Screen ViewModels** (e.g., `TransfersAssetSelectorViewModel`, `CreateStandingOrderViewModel`): Individual ViewModels for each sub-screen within the flow. They receive the Flow ViewModel as an `@Assisted` dependency through their constructor, allowing them to access shared state and delegate flow-level operations. Screen ViewModels handle screen-specific UI logic while the Flow ViewModel manages cross-screen concerns.
*   **Unidirectional Data Flow (UDF)**: UI events flow upwards to the ViewModel, and UI state flows downwards from the ViewModel to the UI, ensuring predictable state management.
*   **UI State Data Classes**: Dedicated immutable data classes are used to represent the entire UI state for a screen, promoting clarity and ease of testing.
    *   **`*State.kt`**: Files named `[FeatureName]ScreenState.kt` (e.g., `CardsScreenState.kt`) define the full presentation state for a screen, encompassing all dynamic data required to render the UI.
    *   **`*UiModel.kt`**: Files named `[FeatureName]UiModel.kt` (e.g., `CardItemUiModel.kt`) represent static UI elements or smaller, reusable data structures that are part of the overall screen state, often representing how domain models are presented in the UI.
*   **Package Structure**: UI components typically reside in `ui` packages within feature modules (e.g., `feature/cards/ui/screens`, `feature/cards/ui/sensitiveinfo`).
*   **Naming Conventions**: `ViewModel` classes follow a `[FeatureName]ViewModel.kt` naming convention. `*Screen.kt` for UI entry points, `*ScreenState.kt` for dynamic UI state, and `*UiModel.kt` for static UI element representation.

## 2. Domain Layer (Business Logic Layer)

This layer contains the core business rules and logic of the application. It is independent of any specific UI, database, or third-party library, making it highly reusable and testable.

*   **Use Cases / Interactors**: These classes encapsulate specific business operations or workflows. They orchestrate interactions between the UI Layer (via ViewModels) and the Data Layer (via Repositories), applying business rules and transforming data into domain models.
*   **Domain Models**: Plain Kotlin classes that represent the entities and value objects central to the business logic. They are free from any Android-specific dependencies, ensuring the domain layer's purity.

### Common Patterns in Domain Layer:
*   **Use Case Pattern**: Each Use Case (or Interactor) represents a single, specific business operation. This promotes the Single Responsibility Principle, making the business logic modular and testable in isolation. They are typically invoked by ViewModels.
*   **Repository Interface Definition**: Repository interfaces, defining contracts for data access, are declared in the Domain Layer. This enforces the dependency inversion principle, allowing the Domain Layer to be independent of data implementation details.
*   **Pure Kotlin Models**: Domain models are designed as pure Kotlin data classes, free of any Android framework dependencies.
*   **Package Structure**: Domain modules (e.g., `:shared:CardsDomain`, `:libraries:CoreDomain`) contain use cases and domain models (`domain/model` package).
*   **Naming Conventions**: Use Cases are typically named `[Action]UseCase.kt` or `[Action]Interactor.kt`.

## 3. Data Layer

This layer is responsible for retrieving, storing, and managing application data. It abstracts the specifics of data sources from the Domain Layer, providing a clean API for data access.

*   **Repositories**: Implement the repository interfaces defined in the Domain Layer. They are responsible for coordinating data operations from various data sources (network, database, preferences) and resolving data conflicts. They map data models from data sources to domain models.
*   **Data Sources**: Implement the actual mechanisms for fetching and storing data from specific sources. Examples include `RemoteDataSource` for network APIs, `LocalDataSource` for databases (e.g., Room) or shared preferences.
*   **Data Models**: Data classes often used for network responses (DTOs) or database entities, which are then mapped to Domain Models by the Repositories.

### Common Patterns in Data Layer:
*   **Repository Pattern**: Provides an abstraction layer over data sources. The Domain Layer interacts solely with repository interfaces, unaware of the underlying data retrieval mechanisms.
*   **Data Source Pattern**: Specific implementations for fetching data from distinct sources (e.g., `[FeatureName]RemoteDataSource`, `[FeatureName]LocalDataSource`).
*   **Data Mapping**: Repositories or dedicated mapper classes are responsible for converting `Data Models` (network responses, database entities) into `Domain Models` and vice-versa, ensuring the Domain Layer remains clean.
*   **Package Structure**: Data modules (e.g., `:shared:CardsData`, `:libraries:Network`, `:libraries:Database`) contain repositories and data sources. Data models are typically found in `data/model`, `data/remote/response`, or `data/local/entity` packages.
*   **Naming Conventions**: Repository implementations reside in the Data Layer (e.g., `[FeatureName]RepositoryImpl.kt`).

## 4. Dependency Injection

The project utilizes a dependency injection framework to manage and provide dependencies throughout the application, promoting testability, modularity, and maintainability.

*   **Hilt**: The primary dependency injection library, built on top of Dagger, providing Android-specific integrations and simplified setup.

### Dependency Injection Patterns:
*   **Hilt Modules (`@Module`, `@InstallIn`)**: Files named `*Module.kt` (e.g., `NetworkModule.kt`, `RepositoryModule.kt`) represent the main elements for defining how dependencies are provided. These modules contain `@Provides` or `@Binds` functions that instruct Hilt on how to create and provision instances of various components (e.g., network clients, database DAOs, repository implementations, use cases).
*   **Constructor Injection**: The preferred method for providing dependencies, making classes easy to test and clearly defining their requirements. Dependencies are injected directly into the class constructors using `@Inject`.
*   **Entry Points (`@EntryPoint`)**: Used for dependencies that cannot be constructor-injected (e.g., field injection in Activities/Fragments that Hilt doesn't directly support via `@AndroidEntryPoint`, or injecting into classes not owned by Hilt).
*   **Scope Management**: Hilt annotations like `@Singleton`, `@ActivityScoped`, `@ViewModelScoped` are used to manage the lifecycle and scope of dependencies.

## 5. Navigation

Navigation within the application is handled using a dedicated component for a consistent and robust user experience.

*   **Jetpack Navigation Component**: Used for managing in-app navigation, defining navigation graphs, and handling transitions between destinations (Composables, Fragments, Activities).
*   **MainNavigator**: The central navigation component that serves as the main router and orchestrates the entire navigation structure. It contains:
    *   **NavHost**: The main navigation host with a router screen as the entry point that determines the initial destination based on app state (Dashboard, Onboarding, etc.).
    *   **All Feature Navigation Graphs**: Aggregates and registers all feature-specific navigation graphs for the entire project, including `landingGraph`, `walletGraph`, `transfersGraph`, `cardsGraph`, `profileGraph`, `onboardingGraph`, and many others. Each graph is registered by calling its respective graph function (e.g., `transfersGraph(appState.navController)`, `topUpGraph(appState.navController)`).
    *   **Bottom Sheet Integration**: Wraps the NavHost with `ModalBottomSheetLayout` to support bottom sheet navigation across the app.

### Navigation Patterns:
*   **Feature Navigation Graphs (`*NavGraph.kt`)**: Each feature module typically contains one or more `*NavGraph.kt` files (e.g., `CardsNavGraph.kt`, `TransfersNavGraph.kt`, `RecoverWalletNavGraph.kt`). These files describe a logical flow of screens within a feature, defining the navigation routes, nested graphs, and associated composables. These individual feature graphs are then aggregated and registered in the MainNavigator.
*   **Navigation Extension Functions (`*Navigation.kt` or `*NavigationExt.kt`)**: Files such as `[FeatureName]Navigation.kt` or `[FeatureName]NavigationExt.kt` provide extension functions on `NavController` (e.g., `NavController.navigateToCardsList()`, `NavController.navigateToTransfersGraph()`) to perform navigation actions in a type-safe and consistent manner. These extension functions are then used within the `*NavGraph.kt` files or directly by UI components to trigger navigation.
*   **Composable-based Navigation**: For Jetpack Compose screens, navigation is handled using composable functions and a `NavController`, with routes defined as sealed classes or objects.
*   **Arguments Passing**: Safe Args for Fragments/Activities and type-safe arguments for Compose are used to pass data between navigation destinations.
*   **Package Structure**: Navigation-related logic and routes are often defined in `navigation` packages within feature modules (e.g., `:features:Cards/navigation`).
*   **Naming Conventions**: `*NavGraph.kt` for navigation graph definitions, and `*Navigation.kt` or `*NavigationExt.kt` for `NavController` extension functions.

## 6. Testing

The project emphasizes comprehensive testing across different layers, employing various testing patterns and frameworks to ensure quality and reliability.

*   **Unit Tests**: Focus on testing individual components (e.g., Use Cases, ViewModels, Repository implementations) in isolation from the Android framework or external dependencies.
*   **Integration Tests**: Verify the interaction between multiple components or layers (e.g., a ViewModel and its Use Cases, or a Repository and its Data Sources).
*   **UI Tests (Instrumentation Tests)**: Test the user interface and user flows on an Android device or emulator, simulating user interactions.

### Testing Patterns:
*   **Test Doubles (Mocks, Fakes, Stubs)**: Widely used to isolate components under test by replacing their dependencies with controlled versions.
    *   **Mockk/Mockito**: Used for creating mock objects and verifying interactions.
*   **Given-When-Then Structure**: A common pattern for organizing test cases, clearly separating setup (Given), action (When), and assertion (Then).
*   **Faking Data at Repository Layer**: For integration and UI tests, repository implementations are often replaced with "fake" implementations that return predefined test data instead of interacting with actual network or database sources. This is achieved through Hilt's testing capabilities, where test-specific `@Module`s are provided to override the production bindings for repository interfaces with their fake implementations. This ensures fast, deterministic, and isolated testing of higher layers without external dependencies.
*   **Coroutines Test Dispatchers**: For testing coroutine-based code, `TestCoroutineDispatcher` or `StandardTestDispatcher` are used to control the execution of coroutines and ensure determinism.
*   **AndroidX Test Libraries**:
    *   **Espresso**: For UI testing of individual activities or fragments.
    *   **UI Automator**: For cross-app functional UI testing.
*   **Robolectric**: Used for running Android unit tests on the JVM without needing an emulator or device, speeding up feedback cycles.
*   **Package Structure**: 
    *   **Unit Tests**: Located in `src/test/kotlin` within each module, mirroring the package structure of the code they are testing.
*   **Naming Conventions**: Test files often append `Test` to the class name (e.g., `ViewModelTest.kt`, `RepositoryImplTest.kt`).

---