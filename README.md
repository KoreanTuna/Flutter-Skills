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
| **State Management** | [Local State Management by Flutter Hook](state-management/local-state-management-by-flutter-hook.md) |
| **Architecture** | [BLoC Apply Layer Architecture](architecture/bloc-apply-layer-architecture.md) |
| **Navigation** | [GoRouter Navigation](navigation/go-router-navigation.md) |

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
│   └── local-state-management-by-flutter-hook.md
│
├── architecture/
│   └── bloc-apply-layer-architecture.md
│
└── navigation/
    └── go-router-navigation.md
```

## Contributing

See [`CONTRIBUTING.md`](CONTRIBUTING.md) for how to add or improve guides.

## License

This project is licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) — you are free to share and adapt the content with attribution.
