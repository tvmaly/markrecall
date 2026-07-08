# MarkRecall

MarkRecall is an iOS study app that turns simple Markdown files into question-and-answer practice decks.

It is designed for students, parents, teachers, and tutors who want a lightweight way to create and share study material without being locked into a proprietary flashcard format. A deck can be written as a plain Markdown file, shared through GitHub or the Files app, cached on the device, and practiced offline.

The app is built with SwiftUI and is intended to stay simple, testable, and easy to extend.

## Who it is for

MarkRecall is for:

- Students who want focused practice for school subjects
- Parents who want to create custom study decks for their children
- Teachers and tutors who want to share reusable question sets
- Homeschool families who prefer open, editable study material
- Developers who want a clean SwiftUI example of a small offline-first learning app

The first version is aimed at students ages 8 to 18, but the file format is general enough to support many subjects and age levels.

## Why this app exists

Many study apps store content inside closed systems. That makes it harder for teachers, parents, and students to own, edit, review, and reuse their material.

MarkRecall takes a different approach.

Study decks are plain text Markdown files with small pieces of YAML metadata. They can be edited in any text editor, reviewed in GitHub, versioned like code, and shared as normal files.

This keeps the content open, portable, and easy to inspect.

## Benefits

### Open study content

Decks are written in Markdown, not a private database format. This makes them easy to create, copy, edit, diff, and share.

### Works offline

Imported decks are cached locally. Once a deck is loaded, students can keep practicing without an internet connection.

### Supports spaced repetition

The app uses a simple Leitner box system to help students review questions at better intervals instead of repeating everything every time.

### Good for test prep

Students can run targeted practice by subject and topic without needing to wait for spaced repetition scheduling.

### Multiple local profiles

Each student can have a local profile with separate progress, mastery, and practice history.

### Simple authoring model

A teacher or parent can create a deck with headings, questions, answer keys, and optional LaTeX math.

Example:

```markdown
---
id: algebra-linear-equations
title: "Linear Equations"
subject: math
topics: [linear-equations]
creator: "Ms. Rivera"
version: 1
age_range: "10-14"
audience: "Grade 6-8"
---

### [MC] q001 {topic: linear-equations}
Solve for $x$: $2x + 3 = 11$

- ( ) $x = 3$
- (x) $x = 4$
- ( ) $x = 7$

### [TF] q002 {topic: linear-equations}
The equation $x + 1 = x$ has no solution.

Answer: true
```

### Math-friendly

Decks can include inline and block LaTeX math for subjects like algebra, geometry, physics, and chemistry.

### Privacy-first

The app does not require accounts, cloud sync, analytics, ads, or third-party tracking. Practice data stays on the device.

## Core features

Planned first-version features:

- Local student profiles
- Import decks from the Files app
- Import decks from a raw HTTPS Markdown URL
- Local deck caching
- Markdown and LaTeX rendering
- Multiple-choice, true/false, and single-answer questions
- Leitner spaced repetition
- Topic subscription and unsubscription
- Targeted practice by subject and topic
- Session results with wrong-question review
- Per-profile reports by subject and topic
- Days-practiced-this-month indicator
- Offline-first practice
- Structured validation errors for deck authors

## High-level design

MarkRecall uses a small SwiftUI architecture with protocol-based services.

```text
SwiftUI Views
    ↓
Observable View Models
    ↓
Domain Services
    ├── DeckImportService
    ├── DeckStore
    ├── ProfileStore
    ├── SchedulerService
    ├── SessionEngine
    ├── StatsService
    └── EventLog
```

The main design goals are:

- Keep parsing and validation pure
- Keep views thin
- Store data in simple local files
- Use dependency injection for testability
- Make the study rules deterministic
- Avoid cloud, accounts, and unnecessary infrastructure in version 1

## Data storage

The first version uses local JSON files and JSONL event logs.

Planned layout:

```text
Application Support/MarkRecall/
  profiles.json
  decks/
    <deck-id>/
      source.md
      manifest.json
  state/
    <profile-id>/
      scheduler.json
      events.jsonl
```

This keeps the data easy to inspect, back up, test, and rebuild.

## Testing approach

The app is designed so most logic can be tested without launching the UI.

Primary test areas:

- Markdown deck parsing
- Deck validation
- Question grading
- Leitner scheduling
- Stats reducers
- Import error handling
- Profile storage
- Event log behavior
- View model state transitions
- Core SwiftUI user flows

Deck parsing should use golden-file tests so valid and invalid Markdown examples have stable, repeatable results.

## Project status

This repository is currently in the planning and early implementation stage.

The intended first milestone is a working vertical slice:

1. Create a local profile
2. Import a bundled sample deck
3. Start a targeted practice session
4. Submit answers
5. Show session results
6. Write practice events locally

After that, the app can expand toward URL imports, reports, spaced repetition, and stronger deck-author tooling.

## Non-goals for version 1

The first version intentionally avoids:

- User accounts
- Cloud sync
- Deck editing inside the app
- Push notifications
- Leaderboards
- Social features
- LLM-based grading
- Third-party analytics
- Complex scheduling algorithms

These can be added later if needed, but they are not required for the core app to be useful.

## Repository structure

Proposed structure:

```text
MarkRecall/
  App/
  DeckFormat/
  Services/
  UI/
  Fixtures/
Tests/
  UnitTests/
  ServiceTests/
  UITests/
```

`DeckFormat` should stay as pure as possible so parsing, validation, grading, and scheduling can be tested quickly.

## License

License to be decided.
