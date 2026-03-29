# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

AI-friendly Flutter development guides for vibe-coding assistants. This is a **documentation-only** repository — no Dart/Flutter source code, no build system, no tests. All content is Markdown files organized by topic category.

## Repository Structure

- Each top-level directory is a topic category (e.g., `state-management/`, `architecture/`, `navigation/`)
- Each `.md` file within a category is a standalone guide on a specific Flutter topic
- `TEMPLATE.md` / `TEMPLATE_KR.md` — canonical guide structure that all guides must follow
- `CONTRIBUTING.md` — contribution rules and quality checklist

## Writing Guides

All guides **must** follow the template in `TEMPLATE.md`. Required sections in order:

1. Overview → When To Use → When NOT To Use → Required Implementation (with Key Rules) → Architecture/Structure → Anti-Patterns → Decision Criteria → Testing Strategy → Common AI Mistakes → References

Key rules when writing or editing guides:
- **Every anti-pattern must include both the bad code AND the correct alternative** with explanation
- **Common AI Mistakes section requires at least 2 concrete examples** with AI-generated code, explanation, and correct approach
- **When NOT To Use must point to specific alternatives**, not just say "don't use this"
- All code examples use `dart` code fences and must be compilable
- Write concisely — this is a reference, not a tutorial
- Bilingual support: guides can be in English or Korean (한국어)

## Naming Conventions

- Guide files: `kebab-case.md` (e.g., `bloc-pattern.md`, `local-state-management-by-flutter-hook.md`)
- One guide per file, one topic per guide

## Content Principles

- Opinionated and prescriptive — guides define what MUST be followed, not just suggestions
- Anti-patterns are as important as correct patterns
- "Common AI Mistakes" is the most valuable section — document concrete errors AI assistants make
- Decision criteria tables help choose between competing approaches (e.g., BLoC vs Cubit)
