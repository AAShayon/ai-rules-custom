# Flutter Development Rules & Architecture Guide

You are an expert Flutter developer. Your primary goal is to build beautiful, scalable, and maintainable applications by strictly adhering to the following architectural principles and coding standards.

## 1. Core Philosophy

-   **Clean Architecture:** The foundation of every project is a strict separation of concerns into three layers: **Presentation**, **Domain**, and **Data**.
-   **Feature-Driven:** The codebase is organized into self-contained feature modules.
-   **Dependency Rule:** Dependencies must only point inwards: `Presentation` → `Domain` → `Data`.
-   **Single Responsibility:** Every class, from a use case to a widget, must have a single, well-defined purpose.

## 2. Project Structure

All projects must follow this exact directory structure within the `lib/` folder.

```
lib/
├── app/
│   ├── components/             # Shared, reusable widgets (buttons, text fields, etc.)
│   ├── core/                   # Core application infrastructure
│   │   ├── config/
│   │   │   └── router/
│   │   │       ├── app_pages.dart      # GetX route definitions
│   │   │       └── app_routes.dart     # Route names and paths
│   │   ├── constants/
│   │   │   └── app_constants.dart    # Global constants (API keys, etc.)
│   │   ├── di/
│   │   │   └── service_locator.dart  # GetIt dependency injection setup
│   │   ├── error/
│   │   │   ├── exceptions.dart       # Custom exception classes
│   │   │   └── failures.dart         # Abstract Failure and implementations
│   │   ├── network/
│   │   │   ├── api_constants.dart    # API endpoints
│   │   │   ├── auth_interceptor.dart # Interceptor for auth tokens
│   │   │   ├── dio_client.dart       # Dio wrapper
│   │   │   └── logger_interceptor.dart # Interceptor for logging
│   │   ├── theme/
│   │   │   ├── app_theme.dart        # ThemeData definitions
│   │   │   └── design_system.dart    # Design tokens (colors, typography, etc.)
│   │   └── utils/
│   │       └── data_parser.dart      # Safe data parsing utilities
│   └── features/                 # All feature modules reside here
│       └── [feature_name]/
│           ├── data/
│           │   ├── datasources/      # Remote and local data source abstractions & impls
│           │   ├── models/           # Data Transfer Objects (DTOs) with serialization
│           │   └── repositories/     # Implementation of domain repositories
│           ├── domain/
│           │   ├── entities/         # Core business objects (pure Dart)
│           │   ├── repositories/     # Abstract repository contracts
│           │   └── usecases/         # Business logic operations
│           └── presentation/
│               ├── controllers/      # GetX controllers for state management
│               ├── screens/          # The main screen/view for the feature
│               └── widgets/          # Feature-specific, reusable widgets
└── main.dart                     # Application entry point
```

## 3. Architecture & Technology Stack

The architecture is designed to be decoupled, allowing you to choose the state management library that best fits the project's needs without altering the core business logic in the Domain layer.

### State Management (Choose One)

You can specify the desired state management solution. The `presentation` layer will be adapted accordingly.

#### Option 1: GetX (Default)
This is the default approach, balancing simplicity and power.

-   **State & Logic:** State and business logic are managed in `GetxController` classes located in `presentation/controllers/`.
-   **Reactivity:** Make variables reactive using `.obs` (e.g., `var isLoading = false.obs;`).
-   **UI Binding:** Use `Obx` or `GetBuilder` widgets in the `screens/` to react to state changes.
-   **Dependency Injection:** Use **`get_it`** as the primary service locator (see below). `GetxController` dependencies are injected via `Get.find()` within the controller's constructor after being registered in the `service_locator.dart`.
-   **Routing:** Use the **`get`** package for navigation, configured in `app/core/config/router/`.

#### Option 2: Riverpod
A powerful and flexible choice that handles both dependency injection and state management.

-   **State & Logic:** Logic is placed in `StateNotifier` classes. The `presentation/controllers` directory can be renamed to `presentation/providers` or `presentation/notifiers`.
-   **Dependency Injection:** **Riverpod replaces `GetIt`**. The `service_locator.dart` file is not needed. Instead, define global `Provider`s for your repositories and use cases.
    -   `Provider` for repositories.
    -   `Provider` for use cases, which will `ref.watch` the repository provider.
    -   `StateNotifierProvider` for your `StateNotifier` classes, which will `ref.watch` the use case providers.
-   **UI Binding:** Convert widgets in `screens/` to `ConsumerWidget` or `ConsumerStatefulWidget`. Use `ref.watch()` to listen to providers and `ref.read()` to call methods on notifiers.
-   **Routing:** A separate routing package like **`go_router`** is recommended when using Riverpod.

#### Option 3: Bloc
A robust and predictable pattern, ideal for complex state transitions.

-   **State & Logic:** Logic is encapsulated in `Bloc` or `Cubit` classes within the `presentation/controllers` (or `presentation/blocs`) directory. These classes take use cases as constructor dependencies.
-   **Events & States:** Blocs receive events from the UI and emit states in response. Define immutable `Event` and `State` classes for each Bloc.
-   **Dependency Injection:** Use **`get_it`** as the primary service locator to inject repositories and use cases into your Blocs.
-   **UI Binding:** Use `BlocProvider` to make a Bloc instance available to the widget tree. Use `BlocBuilder`, `BlocListener`, or `BlocConsumer` to react to state changes. UI events are dispatched to the bloc using `context.read<MyBloc>().add(MyEvent())`.
-   **Routing:** A separate routing package like **`go_router`** is recommended when using Bloc.

### Dependency Injection: GetIt
-   **Primary Tool:** Use **`get_it`** for service location, especially when using **GetX** or **Bloc**. This can be replaced by Riverpod's provider system.
-   **Setup:** If used, all dependencies must be registered in `lib/app/core/di/service_locator.dart`.
-   **Lifecycles:**
    -   Register services and repositories as lazy singletons (`registerLazySingleton`).
    -   Register controllers/blocs as factories (`registerFactory`).

### Routing
-   **With GetX:** Use the built-in **`get`** navigation.
-   **With Riverpod/Bloc:** It is recommended to use **`go_router`** for a declarative, URL-based routing strategy that integrates well with these libraries.

### Networking: Dio
-   **HTTP Client:** Use the **`dio`** package for all network requests.
-   **Wrapper:** Implement a custom `DioClient` in `lib/app/core/network/dio_client.dart`. This client must be configured with interceptors for logging, error handling, and authentication.

### Error Handling: Dartz
-   **Functional Approach:** All methods in the Data and Domain layers that can fail must return an `Either<Failure, SuccessType>`.
-   **Failures:** Define an abstract `Failure` class in `lib/app/core/error/failures.dart`. Create concrete implementations like `ServerFailure` and `NetworkFailure`.
-   **Exception Handling:** Data sources should throw custom exceptions (defined in `exceptions.dart`). The repository implementation is responsible for catching these exceptions and converting them into `Failure` objects.

### Data Handling & Serialization
-   **Entities vs. Models:**
    -   **Entities (`domain/entities`):** Pure Dart objects representing core business logic. They must extend `Equatable`.
    -   **Models (`data/models`):** Data Transfer Objects that extend their corresponding Entity. Models are responsible for serialization (`fromJson`, `toJson`).
-   **Serialization:** Use `json_serializable` and `build_runner` to generate serialization boilerplate. This is preferred over manual serialization.

## 4. Coding Style & Best Practices

-   **File Naming:** `snake_case` (e.g., `auth_controller.dart`).
-   **Class Naming:** `PascalCase` (e.g., `AuthController`).
-   **Variable/Function Naming:** `camelCase` (e.g., `loginUser`).
-   **Immutability:** Widgets, Entities, and States should be immutable.
-   **`const`:** Use `const` constructors for widgets whenever possible to optimize performance.
-   **Comments:** Add documentation comments (`///`) to all public APIs. Use `//` for complex or non-obvious implementation details.
-   **Linting:** Strictly adhere to the rules defined in `flutter_lints`.

## 5. UI & Visual Design

-   **Responsiveness:** Use **`flutter_screenutil`** to ensure layouts adapt to different screen sizes.
-   **Design System:** Define a centralized design system in `lib/app/core/theme/design_system.dart`. This file must contain design tokens for:
    -   Colors (`AppColors`)
    -   Typography (`AppTypography`)
    -   Spacing (`AppPadding`, `AppMargin`)
    -   Border Radius (`AppRadius`)
-   **Theming:** Define `ThemeData` for both light and dark modes in `lib/app/core/theme/app_theme.dart`.
-   **Aesthetics:**
    -   Use multi-layered drop shadows to create a sense of depth and lift for elements like cards.
    -   Emphasize font sizes to create a clear visual hierarchy (hero text, headlines, body).
    -   Use icons to enhance understanding and navigation.
    -   Interactive elements should have a subtle "glow" effect on interaction.

## 6. Core Dependencies

A new project should start with the following `pubspec.yaml` dependencies:

```yaml
dependencies:
  flutter:
    sdk: flutter
  
  # Core Architecture
  get: <version>
  get_it: <version>
  equatable: <version>
  dartz: <version>

  # Networking
  dio: <version>
  connectivity_plus: <version>

  # UI/UX
  flutter_screenutil: <version>
  cached_network_image: <version>
  shimmer: <version>
  font_awesome_flutter: <version>

  # Utilities
  get_storage: <version>
  intl: <version>

dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: <version>
  build_runner: <version>
  json_serializable: <version>
```
