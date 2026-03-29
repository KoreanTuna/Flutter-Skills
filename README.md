# Flutter-Skills

AI-friendly Flutter development guides for vibe-coding assistants.

**English** | **[한국어](README_KR.md)**

## Purpose

Structured, opinionated guides that help AI assistants (and developers) make correct implementation decisions during Flutter development.

When vibe-coding with AI, assistants often misuse Flutter APIs, choose wrong patterns, or generate anti-patterns that are hard to catch. This project provides **per-topic guides** covering required patterns, anti-patterns, decision criteria, and common AI mistakes — so the AI gets it right the first time.

> Built from 5 years of production Flutter experience.

## How to Use

### For AI-Assisted Development (Vibe Coding)

Point your AI assistant at the relevant guide **before** starting a task:

```
"Read `state-management/bloc-pattern.md` before generating any BLoC code for this project."
```

Or load multiple guides for cross-cutting concerns:

```
"Read `architecture/clean-architecture.md` and `state-management/bloc-pattern.md` to understand the project structure before implementing the new feature."
```

### For Developers

Browse topics by category below. Each guide follows a consistent template defined in [`TEMPLATE.md`](TEMPLATE.md).

## Topics

| Category | Guides |
|---|---|
| **State Management** | BLoC Pattern, Cubit Pattern, BLoC vs Cubit, Provider, Riverpod, State Restoration |
| **Architecture** | Clean Architecture, Feature-first vs Layer-first, Repository Pattern, Domain Modeling |
| **Navigation** | GoRouter, Navigator 2.0, Deep Linking, Navigation with BLoC |
| **Dependency Injection** | GetIt, Injectable, Service Locator vs DI |
| **Testing** | Unit Testing, Widget Testing, Integration Testing, Testable Code Patterns |
| **Networking** | Dio Setup, Retrofit, API Error Handling, Offline-First |
| **Persistence** | SharedPreferences, Drift, Hive/Isar, Secure Storage |
| **Performance** | Widget Rebuild Optimization, List Performance, Image Optimization, Isolates, DevTools Profiling |
| **UI** | Theme & Design System, Responsive Design, Animations, CustomPaint, Forms & Validation |
| **Error Handling** | Error Handling Patterns, Crashlytics Integration, Global Error Boundary |
| **Platform** | Platform Channels, Pigeon, FFI, Platform-Specific UI |
| **Internationalization** | ARB Localization, Intl Best Practices |
| **Firebase** | Setup, Authentication, Firestore Patterns, Cloud Functions |
| **CI/CD** | GitHub Actions, Fastlane, Code Quality |

## Guide Format

Every guide follows the template in [`TEMPLATE.md`](TEMPLATE.md) with these sections:

| Section | Purpose |
|---|---|
| **Overview** | What this topic covers and when it applies |
| **When To Use** | Scenarios where this is the right choice |
| **When NOT To Use** | Wrong scenarios, with pointers to alternatives |
| **Required Implementation** | Code patterns that MUST be followed |
| **Architecture / Structure** | Recommended file/folder layout |
| **Anti-Patterns** | What to avoid and why |
| **Decision Criteria** | Comparison tables when multiple options exist |
| **Testing Strategy** | How to test code using this pattern |
| **Common AI Mistakes** | Errors AI assistants frequently make in this area |
| **References** | Official docs and canonical resources |

## Project Structure

```
Flutter-Skills/
├── README.md
├── README_KR.md
├── TEMPLATE.md
├── CONTRIBUTING.md
│
├── state-management/
│   ├── bloc-pattern.md
│   ├── cubit-pattern.md
│   ├── bloc-vs-cubit.md
│   ├── provider.md
│   ├── riverpod.md
│   └── state-restoration.md
│
├── architecture/
│   ├── clean-architecture.md
│   ├── feature-first-vs-layer-first.md
│   ├── repository-pattern.md
│   └── domain-modeling.md
│
├── navigation/
│   ├── go-router.md
│   ├── navigator-2.md
│   ├── deep-linking.md
│   └── navigation-with-bloc.md
│
├── dependency-injection/
│   ├── get-it.md
│   ├── injectable.md
│   └── service-locator-vs-di.md
│
├── testing/
│   ├── unit-testing.md
│   ├── widget-testing.md
│   ├── integration-testing.md
│   └── testable-code-patterns.md
│
├── networking/
│   ├── dio-setup.md
│   ├── retrofit-code-gen.md
│   ├── api-error-handling.md
│   └── offline-first.md
│
├── persistence/
│   ├── shared-preferences.md
│   ├── drift-database.md
│   ├── hive-isar.md
│   └── secure-storage.md
│
├── performance/
│   ├── widget-rebuild-optimization.md
│   ├── list-performance.md
│   ├── image-optimization.md
│   ├── isolates-and-compute.md
│   └── devtools-profiling.md
│
├── ui/
│   ├── theme-design-system.md
│   ├── responsive-design.md
│   ├── animations.md
│   ├── custom-paint.md
│   └── forms-and-validation.md
│
├── error-handling/
│   ├── error-handling-patterns.md
│   ├── crashlytics-integration.md
│   └── global-error-boundary.md
│
├── platform/
│   ├── platform-channels.md
│   ├── pigeon-code-gen.md
│   ├── ffi.md
│   └── platform-specific-ui.md
│
├── internationalization/
│   ├── arb-localization.md
│   └── intl-best-practices.md
│
├── firebase/
│   ├── firebase-setup.md
│   ├── authentication.md
│   ├── firestore-patterns.md
│   └── cloud-functions.md
│
└── ci-cd/
    ├── github-actions.md
    ├── fastlane.md
    └── code-quality.md
```

## Contributing

See [`CONTRIBUTING.md`](CONTRIBUTING.md) for how to add or improve guides.

## License

This project is licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) — you are free to share and adapt the content with attribution.
