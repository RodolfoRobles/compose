# Android Composable Architecture Design Document

## 1. Introduction

This document outlines a proposed architecture pattern for Android applications built with Kotlin
and Jetpack Compose. The goal is to establish a scalable, maintainable, and testable codebase by
promoting a clear separation of concerns and unidirectional data flow.

## 2. Core Principles

The architecture is guided by the following core principles:

* **Unidirectional Data Flow (UDF):** State flows down from higher-level components (ViewModels) to
  UI Composable, and events flow up from Composable to ViewModels. This simplifies state
  management and makes it easier to reason about data changes.
* **State Hoisting:** UI state should be hoisted to the lowest common ancestor of all Composable
  that need to read or write it. This often means state resides in ViewModels.
* **Single Source of Truth (SSoT):** For any piece of application data, there should be one
  definitive source that owns and manages it. This is typically managed within the `data` layer and
  exposed through ViewModels.
* **Dependency Injection:** Leverage a dependency injection framework (e.g., Hilt or Koin) to manage
  dependencies between components, promoting loose coupling and testability.

## 3. Proposed Folder Structure

The project will be organized into the following main package structure within each feature module:

your_project_root/
└── app/
└── src/
└── main/
└── java/
└── com/yourdomain/yourapp/
├── **data**             // Handles data sources and repositories
│   ├── **local**        // Local data sources (Room DB, SharedPreferences)
│   │   ├── dao
│   │   └── model        // Database entities
│   ├── **remote**       // Remote data sources (Retrofit APIs)
│   │   ├── api
│   │   └── dto          // Data Transfer Objects for network responses
│   └── **repository**   // Abstracts data sources
│
├── **domain**           // Business logic, use cases, and domain models
│   ├── **model**        // Core business models (often plain Kotlin classes)
│   └── **usecase**      // Specific business operations/interacts
│
├── **presentation**     // UI layer (Compose screens, ViewModels)
│   ├── **navigation**   // Navigation graph and route definitions
│   ├── **screen**       // Individual screen packages
│   │   └── **feature_x** // Example: product_details_screen
│   │       ├── **components** // Reusable UI components specific to this screen
│   │       ├── FeatureXScreen.kt // The main Composable for the screen
│   │       └── FeatureXViewModel.kt // ViewModel for the screen
│   ├── **components**   // Globally reusable UI components
│   └── **theme**        // App theme, colors, typography
│
├── **di**               // Dependency Injection modules (Hilt or Koin)
│
└── **utils**            // Utility classes, extensions, constants

## 4. Scope and Characteristics of Each Layer/Folder

### 4.1. `data` Layer

* **Scope:** Responsible for all data handling operations. This includes fetching data from remote
  APIs, managing local data persistence (databases, SharedPreferences), and caching strategies.
* **Characteristics:**
    * Contains implementations for data sources (e.g., Retrofit services for network calls, Room
      DAOs for database access).
    * Handles mapping between network Data Transfer Objects (DTOs) or database entities and the
      `domain` layer's models. In simpler cases, DTOs/entities might be directly exposed if they
      align well with domain needs.
    * Not directly aware of the UI layer.
* **Sub-folders:**
    * **`data/local`**:
        * `dao`: Room Data Access Object interfaces.
        * `model`: Kotlin data classes annotated with `@Entity` for Room database tables.
    * **`data/remote`**:
        * `api`: Retrofit `@Service` interfaces defining API endpoints.
        * `dto`: Kotlin data classes representing JSON request and response bodies.
    * **`data/repository`**:
        * **Repository Interfaces & Implementations:** These are the crucial components that the
          `domain` layer interacts with.
        * They abstract the origin of data (network, cache, database), providing a clean API for
          data retrieval and storage.
        * Responsible for logic like deciding when to fetch fresh data versus serving cached data.
        * Typically expose data using reactive streams, primarily Kotlin Flows (`Flow<T>`).

### 4.2. `domain` Layer

* **Scope:** Contains the core business logic and rules of the application. This layer should be
  pure Kotlin and independent of the Android framework and any specific UI or data source
  implementation.
* **Characteristics:**
    * Platform-agnostic.
    * Highly testable with simple unit tests.
* **Sub-folders:**
    * **`domain/model`**:
        * Plain Kotlin data classes, sealed classes, or enums representing the core entities and
          concepts of the application (e.g., `User`, `Product`, `Order`). These are the models that
          the rest of the application (UI and business logic) operates on.
    * **`domain/usecase`** (or `interactor`):
        * Classes that encapsulate a single, specific business operation or user story (e.g.,
          `GetUserProfileUseCase`, `SubmitOrderUseCase`, `LoadAvailableProductsUseCase`).
        * Orchestrate calls to one or more Repositories from the `data` layer.
        * May perform business rule validation, data aggregation, or transformation.
        * Typically expose a single public method (often `operator fun invoke()` for concise calling
          syntax).

### 4.3. `presentation` Layer (UI Layer)

* **Scope:** Responsible for displaying data to the user and handling user interactions. This is
  where all Jetpack Compose UI code resides.
* **Characteristics:**
    * Android framework dependent.
    * Observes state changes from ViewModels and updates the UI accordingly.
    * Delegates user actions to ViewModels.
* **Sub-folders:**
    * **`presentation/navigation`**:
        * Defines navigation routes, arguments, and the application's navigation graph using Jetpack
          Compose Navigation library.
    * **`presentation/screen`**:
        * Organized by individual features or screens of the application. Each screen package (e.g.,
          `product_details`) typically contains:
            * **`FeatureScreen.kt`**: The main Composable function that defines the UI layout and
              elements for that specific screen. It observes state from its corresponding ViewModel.
            * **`FeatureViewModel.kt`**: An Android `ViewModel` (`androidx.lifecycle.ViewModel`).
                * Holds and manages UI-related state (e.g., using
                  `kotlinx.coroutines.flow.StateFlow` or Compose `androidx.compose.runtime.State`).
                * Exposes this state to the Composable screen for observation.
                * Handles user events delegated from the screen and interacts with `domain` layer
                  use cases.
                * Should not have direct dependencies on Android framework classes like `Context`
                  unless absolutely necessary and carefully managed (e.g., via Hilt's
                  `@ApplicationContext`).
            * **`components`** (within a screen package): Smaller, reusable Composable functions
              that are specific to this particular screen.
    * **`presentation/components`** (top-level within `presentation`):
        * Globally reusable UI components that can be shared across multiple screens (e.g.,
          `CustomButton`, `LoadingIndicator`, `ErrorDisplay`).
    * **`presentation/theme`**:
        * Defines the application's `MaterialTheme`, including color palettes, typography, and
          shapes.

### 4.4. `di` Layer (Dependency Injection)

* **Scope:** Contains modules and configurations for the chosen dependency injection framework (
  e.g., Hilt modules using `@Module` and `@Provides`, or Koin modules using `module { }` blocks).
* **Characteristics:**
    * Defines how to construct and provide instances of repositories, use cases, ViewModels,
      database instances, API services, and other dependencies throughout the application.

### 4.5. `utils` Layer

* **Scope:** A place for utility classes, Kotlin extension functions, constants, and other helper
  code that doesn't fit neatly into the other layers but provides general-purpose functionality used
  across the application.
* **Characteristics:**
    * Code here should be generic and widely applicable. Examples: date formatters, string
      manipulation helpers, logging utilities.

## 5. Data Flow Example

A typical data flow in this architecture would be:

1. **User Interaction (Screen):** User performs an action (e.g., taps a button) in a Composable
   within `FeatureScreen.kt`.
2. **Event to ViewModel (Screen -> ViewModel):** The Composable calls a function on its
   `FeatureViewModel.kt` (e.g., `viewModel.onLoadDataClicked()`).
3. **ViewModel to Use Case (ViewModel -> Domain):** The `FeatureViewModel.kt` invokes the
   appropriate `UseCase` from the `domain` layer (e.g., `loadDataUseCase()`).
4. **Use Case to Repository (Domain -> Data):** The `UseCase` calls a method on a `Repository`
   interface (defined in `domain`, implemented in `data`) to request data.
5. **Repository to Data Source (Data):** The `Repository` implementation fetches data from the
   relevant data source (e.g., remote API via Retrofit, local database via Room, or cached data).
6. **Data Flows Back:**
    * Data source returns data (often as a `Flow<DomainModel>`) to the Repository.
    * Repository may perform caching or transformations and returns data to the Use Case.
    * Use Case may perform further business logic and returns data to the ViewModel.
7. **ViewModel Updates State (Presentation):** The `FeatureViewModel.kt` receives the data (or
   error) and updates its exposed UI state (e.g., a `StateFlow`).
8. **Screen Recomposes (Presentation):** The `FeatureScreen.kt`, which is observing the ViewModel's
   state, automatically recomposes to reflect the new data, updating the UI.

## 6. Benefits

* **Improved Separation of Concerns:** Each layer has a well-defined responsibility, leading to a
  more organized and understandable codebase.
* **Enhanced Testability:**
    * The `domain` layer (use cases, domain models) can be unit tested easily as it's pure Kotlin.
    * The `data` layer (repositories) can be tested by mocking data sources.
    * ViewModels in the `presentation` layer can be unit tested by mocking their use case
      dependencies.
    * Compose UI can be tested using Compose testing APIs.
* **Increased Scalability:** Adding new features or modifying existing ones becomes more manageable
  as changes are often localized to specific layers.
* **Better Maintainability:** A clear and consistent structure makes it easier for developers (
  current and future) to navigate, understand, and modify the code.
* **Potential for Reusability:** The `domain` and parts of the `data` layer can be designed to be
  potentially reusable in other projects or even across platforms (if the `domain` layer is strictly
  Kotlin Multiplatform compatible).

## 7. Conclusion

This architecture provides a solid foundation for building robust and maintainable Android
applications with Jetpack Compose. It encourages best practices and aims to simplify development by
clearly defining responsibilities and data flow patterns. It is adaptable and can be tailored to the
specific needs of the project as it evolves.