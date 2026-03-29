# Contributing to Flutter-Skills

Thank you for helping improve Flutter development guides for AI assistants.

## How to Contribute

### Adding a New Guide

1. Check the [README](README.md) topic list — pick an unwritten guide or propose a new one via Issue.
2. Copy [`TEMPLATE.md`](TEMPLATE.md) into the appropriate category folder.
3. Rename it to match the topic (e.g., `bloc-pattern.md`).
4. Fill in **every section** of the template. Incomplete guides are not accepted.
5. Submit a Pull Request.

### Improving an Existing Guide

- Fix inaccuracies, add missing anti-patterns, or update for new Flutter/package versions.
- **Common AI Mistakes** section is the most valuable — if you've seen an AI assistant generate bad Flutter code, document it here.

## Guide Quality Checklist

Before submitting, verify:

- [ ] Follows the [`TEMPLATE.md`](TEMPLATE.md) structure exactly
- [ ] All code examples compile and follow current Dart/Flutter conventions
- [ ] Anti-patterns include both the bad code AND the correct alternative
- [ ] **Common AI Mistakes** section has at least 2 concrete examples
- [ ] **When NOT To Use** section points to specific alternatives
- [ ] No placeholder text remains from the template

## Style Guidelines

- Write in English for maximum accessibility
- Use concrete code examples, not abstract descriptions
- Every "don't do this" must have a "do this instead"
- Keep explanations concise — this is a reference, not a tutorial
- Use `dart` code fences for all Flutter/Dart code blocks

## Reporting AI Mistakes

If you've encountered an AI assistant making a specific Flutter mistake that isn't covered:

1. Open an Issue with the label `ai-mistake`
2. Include: the AI-generated code, why it's wrong, and the correct approach
3. Specify which guide it belongs to (or suggest a new one)
