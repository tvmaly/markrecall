# Plan

App name (working title): **MarkRecall**

## 1. Purpose

MarkRecall is an iOS app for students ages 8–18 to study math, science, history, music, and other subjects using open, community-authorable Markdown question files with embedded LaTeX. It applies spaced repetition (Leitner box system) and per-profile mastery tracking to make practice efficient, and gamifies consistency by showing days practiced in the current month. It solves the problem of study content being locked inside proprietary flashcard formats: any teacher or parent can author a plain Markdown file, host it on GitHub, and students can load, cache, and practice it offline.

## 2. Scope

**In scope**
- Local student profiles (nickname only), profile selection at launch
- Markdown+YAML deck format supporting single-answer, multiple-choice, and true/false questions with inline/block LaTeX
- Deck import from local device (Files picker) and from a GitHub-hosted raw URL; local caching
- Library with filters: subject, topic, not-yet-mastered
- Leitner spaced-repetition scheduling per profile; topic subscribe/unsubscribe
- Targeted (non-SR) practice of a chosen subject+topic for test prep
- Session results: elapsed time, correct count, list of wrong questions
- Report screen per profile: minutes practiced, mastery level, last practiced date, by subject and topic
- Days-practiced-this-month indicator on profile selection
- Offline-first operation for cached content

**Out of scope**
- Accounts, cloud sync, iCloud, multi-device
- Deck authoring/editing inside the app
- Free-text grading beyond exact/normalized match; no LLM grading
- Push notifications, leaderboards, social features
- Profile PINs/authentication (deferred; see §22)
- iPad-specific layouts beyond default SwiftUI adaptivity
- Analytics/telemetry leaving the device

## 3. Assumptions

- **A-01** Minimum iOS version: **iOS 17.0** (chosen; enables `@Observable`, modern `NavigationStack`, Swift 5.9+). Placeholder in the brief said AI may choose.
- **A-02** Persistence uses **plain JSON files + an append-only JSONL event log** rather than SwiftData/Core Data. [Inference] This is the simplest testable option for the data volume involved (dozens of decks, thousands of events) and matches the "no over-engineering" constraint.
- **A-03** Spaced repetition uses a **5-box Leitner system**, not SM-2/FSRS. [Inference] Leitner is deterministic, trivially explainable to children, and sufficient for the stated requirements.
- **A-04** "GitHub-hosted" means a **raw file URL** (`https://raw.githubusercontent.com/...`) or any HTTPS URL returning `text/markdown`/`text/plain`. No GitHub API auth; public files only.
- **A-05** Single-answer questions are graded by **normalized exact match** (trim, case-fold, Unicode-normalize; numeric answers compared numerically with optional tolerance declared in the file).
- **A-06** Profiles are unauthenticated (no PIN); the device is assumed shared within a trusted household/classroom.
- **A-07** LaTeX support is limited to **LaTeX math mode** (SwiftMath's documented constraint); no text-mode macros or full documents.
- **A-08** Verified library facts (sources at end of Plan): swift-markdown is a GFM/cmark-gfm-backed parser from swiftlang; SwiftMath (mgriebling) renders LaTeX math via `MTMathUILabel` (UIKit view, wrapped with `UIViewRepresentable`), iOS 11+; Yams provides `YAMLDecoder` Codable decoding of YAML. Front matter is split by a small (~20-line) in-app `FrontMatterSplitter` and decoded with Yams directly — the SwiftToolkit/frontmatter dependency is dropped to remove its macOS-only `platforms:` declaration risk and one avoidable dependency.
- **A-09** A "question set" for a targeted session = all questions in the chosen topic within one deck, presented in shuffled order.
- **A-10** Mastery is computed **deterministically by Swift code** from Leitner box state — never by heuristic or model judgment.
- **A-11** Deck files are **untrusted community content**: rendering is restricted to a safe Markdown subset and deck content can never trigger network activity (§10.1, §16).

## 4. Requirements

### Functional
- `REQ-001`: The app must support multiple local student profiles identified by nickname only.
- `REQ-002`: The app must allow creating, selecting, renaming, and deleting profiles; deleting a profile deletes its practice data after confirmation.
- `REQ-003`: The app must import a deck from a local Markdown file via the system file picker.
- `REQ-004`: The app must import a deck from a user-supplied HTTPS URL (GitHub raw file).
- `REQ-005`: The app must cache imported decks locally and use the cache when offline.
- `REQ-006`: The library must support filtering decks/topics by subject, topic, and "not yet mastered" (per active profile).
- `REQ-007`: The app must parse YAML front matter fields: `id`, `title`, `subject`, `topics`, `creator`, `version`, `age_range`, `audience`.
- `REQ-008`: The app must support three question types: single answer (`SA`), multiple choice (`MC`), true/false (`TF`).
- `REQ-009`: The app must render LaTeX math embedded in question text, choices, and answer reveals.
- `REQ-010`: The app must schedule review questions per profile using a 5-box Leitner system over the profile's subscribed topics.
- `REQ-011`: Students must be able to subscribe and unsubscribe topics for spaced repetition at any time.
- `REQ-012`: Students must be able to run a targeted practice session on a chosen subject+topic without spaced repetition.
- `REQ-013`: On completing a question set, the app must show elapsed time, number correct, and the specific questions answered wrong.
- `REQ-014`: The app must track mastery per profile per topic, derived deterministically from Leitner box state (`REQ-022`).
- `REQ-015`: The report screen must show, per subject and topic: minutes practiced, mastery level, last practiced date.
- `REQ-016`: After profile selection, the app must display the count of distinct days in the current calendar month on which that profile practiced.
- `REQ-017`: The app must never display an answer key before the student submits an answer; questions and answers are separated at parse time.
- `REQ-018`: A correct answer promotes a question one Leitner box (max 5); an incorrect answer demotes it to box 1. Only review-mode outcomes drive box transitions.
- `REQ-019`: Targeted practice sessions must not read or modify Leitner box state. Their outcomes are recorded to the event log with mode `targeted` and count toward minutes, streak, and last-practiced — but never toward mastery.

### Data
- `REQ-020`: Deck identity is front matter `id`. Re-import with `version` greater than the cached version replaces the cached copy; re-import with an equal or lower `version` is rejected with `E-VER-01`. Per-question progress is preserved only where both the question ID **and** its content hash (normalized prompt + choices + answer key) are unchanged; changed questions re-enter box 1 as new.
- `REQ-021`: All practice outcomes are recorded to an append-only event log; existing records are never mutated.
- `REQ-022`: The append-only event log (practice **and** subscription events) is the sole source of truth. `scheduler.json` is a derived, rebuildable cache; box state, mastery, and stats must all be reproducible by deterministic replay of the log through pure functions (same inputs → same outputs).
- `REQ-024`: If `scheduler.json` is missing, corrupt, or schema-mismatched, the app must silently rebuild it by replaying the event log, logging `E-PERSIST-02`.
- `REQ-023`: Deck validation must produce machine-readable errors: stable error code, message, and source line number.

### UI
- `REQ-040`: Every list screen must have an explicit empty state with a call to action.
- `REQ-041`: Async operations must show loading state; failures must show an error state with a retry affordance where retry is meaningful.

### Offline
- `REQ-030`: All practice features must work offline against cached decks.
- `REQ-031`: URL import failures (offline, 404, timeout, non-Markdown content) must show a specific error and allow retry without re-entering the URL.

### Accessibility
- `REQ-050`: Interactive elements must have VoiceOver labels; LaTeX views must expose the source LaTeX (or a provided `alt` text) as their accessibility label.
- `REQ-051`: All text (except rendered math images) must support Dynamic Type.
- `REQ-052`: Tap targets must be at least 44×44 pt.

### Privacy / Security
- `REQ-060`: The app must store no PII beyond a freeform nickname; no accounts, no third-party analytics, no data leaves the device except the outbound deck-URL fetch.
- `REQ-061`: All network requests must use HTTPS (default ATS).
- `REQ-062`: Deck content is untrusted: only a safe Markdown subset is rendered (no HTML, no images; links appear as non-tappable plain text), and deck content must never cause a network fetch at parse or render time.
- `REQ-063`: Decks exceeding resource limits (file > 1 MB, > 1,000 questions, prompt > 10,000 chars, LaTeX segment > 2,000 chars) are rejected with `E-LIM-*` codes and line numbers.

### Observability
- `REQ-070`: The app must emit structured logs with a per-session correlation ID for import, validation, scheduling, and session lifecycle events.
- `REQ-071`: Each requirement ID must be traceable to at least one acceptance scenario (see `AcceptanceTests.md`) and to unit/UI tests via test naming convention `test_REQ_xxx_*`.

## 5. User Experience Overview

**Primary screens**
1. **Profile Select** — grid of profiles + "New Profile". After selection, shows "🔥 N days practiced this month" (REQ-016) and continues to Home.
2. **Home** — two primary actions: "Review Due (n)" (SR queue size) and "Practice a Topic" (test prep); links to Library and Reports.
3. **Library** — cached decks grouped by subject; filter bar (subject / topic / not-yet-mastered); "+ Import" (local file or URL); per-topic subscribe toggles (REQ-011).
4. **Import** — segmented: Local File (fileImporter) | URL (text field + Fetch). Shows validation results; on failure, lists structured errors with line numbers.
5. **Practice Session** — one question at a time: rendered Markdown+LaTeX prompt; input control depends on type (choice list, true/false buttons, text field). Submit → immediate correct/incorrect feedback + correct answer reveal → Next.
6. **Session Results** — elapsed time, X/Y correct, list of wrong questions (tap to re-view question + correct answer) (REQ-013).
7. **Reports** — per active profile, table by subject → topic: minutes, mastery badge, last practiced (REQ-015).
8. **Settings (minimal)** — manage profiles, clear cache.

**Navigation flow**: `ProfileSelect → Home → {Library, Import, Session, Results, Reports}` via `NavigationStack`; profile switch returns to ProfileSelect.

**First-run**: no profiles → Profile creation sheet; empty library → Library empty state with "Import your first deck" and a "Load sample deck" button (bundled sample exercises the whole pipeline and gives previews/UI tests a fixture).

**Empty states**: no profiles; no decks; no subscribed topics ("Review Due" explains how to subscribe); no due questions ("All caught up — come back tomorrow"); no report data.

**Loading states**: URL fetch spinner with cancel; parse/validate progress for large files.

**Error states**: import validation failure (structured list); network failure (retry keeps URL); persistence failure (non-destructive alert).

**Success states**: import success shows deck title + question count + detected topics; session results screen doubles as the success state for practice.

## 6. High-Level Architecture

Pattern: **MVVM-lite with @Observable view models and protocol-injected services** (no third-party architecture framework).

```
SwiftUI Views ──▶ @Observable ViewModels ──▶ Domain Services (protocols)
                                              ├─ DeckImportService (fetch/read → parse → validate → cache)
                                              ├─ DeckStore            (cached decks + manifests)
                                              ├─ ProfileStore         (profiles.json)
                                              ├─ SchedulerService     (Leitner state per profile)
                                              ├─ SessionEngine        (runs a question set, grades, times)
                                              ├─ StatsService         (pure reducers over event log)
                                              └─ EventLog             (append-only JSONL)
Parsing layer: FrontMatterSplitter + Yams (YAMLDecoder) + QuestionParser (swift-markdown) + ContentSanitizer
Rendering layer: MarkdownBlockView + MathView (SwiftMath MTMathUILabel via UIViewRepresentable)
Networking: URLSession behind DeckFetching protocol
Persistence: FileStorage protocol (Application Support) — real + in-memory implementations
Validation: DeckValidator → [ValidationIssue] (code, message, line)
```

**Dependency injection**: a single `AppDependencies` struct holding protocol-typed services, created in `App.init`, injected via `.environment(\.deps)`. Every service has an in-memory/mock implementation for previews and tests.

## 7. Module Design

### 7.1 AppShell (entry/navigation)
- **Purpose**: app entry, root navigation, active-profile state.
- **Responsibilities**: build `AppDependencies`; own `NavigationStack` path; gate on profile selection.
- **Public interface**: `MarkRecallApp: App`, `AppRouter` (`@Observable`, `var path: [Route]`).
- **Inputs**: none. **Outputs**: routed screens. **Dependencies**: all stores (via env).
- **Failure modes**: corrupted profiles file → recovery flow (rename corrupt file, start fresh, log `E-PERSIST-01`).
- **Test strategy**: router unit tests (route enum transitions); launch UI test.

### 7.2 DeckFormat (parsing + validation) — pure, no I/O
- **Purpose**: turn raw Markdown text into a validated `Deck`.
- **Responsibilities**: split front matter (`FrontMatterSplitter`: file must begin with `---`, matter ends at the next lone `---` line, else `E-FM-01`); decode via Yams `YAMLDecoder`; walk swift-markdown tree; sanitize to the safe subset (§10.1); enforce resource limits (REQ-063); extract questions; validate; assign stable question IDs and per-question content hashes.
- **Public interface**:
  ```swift
  protocol DeckParsing { func parse(_ raw: String) -> Result<Deck, DeckParseFailure> }
  struct DeckParseFailure: Error { let issues: [ValidationIssue] }
  struct ValidationIssue: Codable, Equatable { let code: String; let message: String; let line: Int? }
  ```
- **Inputs**: `String`. **Outputs**: `Deck` or issues (errors `E-*`, warnings `W-*`). **Dependencies**: swift-markdown, Yams.
- **Failure modes**: missing front matter (`E-FM-01`), missing required key (`E-FM-02`), unknown question tag (`E-Q-01`), MC without exactly one `(x)` for single-correct (`E-Q-02`), TF answer not true/false (`E-Q-03`), SA missing `Answer:` (`E-Q-04`), duplicate question id (`E-Q-05`), zero questions (`E-Q-06`), question `topic` not declared in front matter (`E-Q-07`), question heading at wrong level (`E-Q-08`), unterminated math delimiter (`E-TEX-02`), limit breaches (`E-LIM-01` file size, `E-LIM-02` question count, `E-LIM-03` prompt/LaTeX length). Warnings (non-blocking): `W-SEC-01` HTML stripped, `W-SEC-02` image removed, `W-Q-01` unknown attribute ignored.
- **Test strategy**: golden-file tests — `Fixtures/valid/*.md` must parse to snapshot JSON; `Fixtures/invalid/*.md` must produce exact issue codes/lines. This module is the AI agent's primary deterministic check surface.

### 7.3 DeckImportService
- **Purpose**: orchestrate local/URL import.
- **Interface**: `func importLocal(url: URL) async -> ImportResult`, `func importRemote(url: URL) async -> ImportResult` where `ImportResult = Result<DeckSummary, ImportError>` and `ImportError = .network(URLError) | .invalidContentType | .parse([ValidationIssue]) | .versionConflict(cached: Int, incoming: Int) | .storage(Error)`.
- **Dependencies**: `DeckFetching`, `DeckParsing`, `DeckStore`. Security-scoped resource handling for picker URLs.
- **Failure modes**: each `ImportError` case maps to a distinct user-facing message and log code.
- **Test strategy**: service tests with stub fetcher (success/404/timeout/wrong MIME) and in-memory store.

### 7.4 DeckStore
- **Purpose**: cache decks + manifests; list/filter.
- **Interface**: `save(deck:sourceText:) throws`, `allDecks() -> [DeckSummary]`, `deck(id:) -> Deck?`, `delete(id:) throws`, `filter(subject:topic:notMasteredFor:) -> [DeckSummary]`.
- **Persistence**: `AppSupport/decks/<deckID>/source.md` + `manifest.json` (REQ-020 replace-on-reimport).
- **Test strategy**: repository tests against temp directory + in-memory `FileStorage`.

### 7.5 ProfileStore
- **Interface**: `profiles() -> [Profile]`, `create(nickname:) throws -> Profile`, `rename/delete`, `activeProfileID` (persisted in UserDefaults).
- **Failure modes**: duplicate nickname (`E-PROF-01`, allowed? no — rejected for clarity), empty nickname (`E-PROF-02`).

### 7.6 SchedulerService (Leitner)
- **Purpose**: due-question computation and box transitions per profile.
- **Interface**:
  ```swift
  protocol Scheduling {
    func dueQuestions(profile: Profile.ID, on date: Date) -> [QuestionRef]
    func record(profile: Profile.ID, question: QuestionRef, outcome: Outcome, at date: Date)
    func subscribe(profile: Profile.ID, topic: TopicKey) / unsubscribe(...)
  }
  ```
- **Rules**: boxes 1–5; review intervals `[0, 1, 3, 7, 14]` days; correct → box+1 (cap 5); wrong → box 1; new questions enter box 1 on subscription. Only `mode == .review` outcomes reach this service (REQ-019). Pure `LeitnerRules` type computes transitions; the service only applies them (REQ-010/018, A-10). Subscribe/unsubscribe append `SubscriptionEvent`s to the log; `scheduler.json` is a derived cache with `rebuild(profile:)` that deterministically replays the full event log (REQ-022/024).
- **Test strategy**: exhaustive unit tests on `LeitnerRules` (transition table, date math across month/DST boundaries with fixed `Calendar`/`TimeZone` injected).

### 7.7 SessionEngine
- **Purpose**: run one session (SR queue or targeted set): ordering, grading, timing, result assembly.
- **Interface**: `start(kind: .review | .targeted(deck:topic:), profile:) -> Session`, `submit(answer:) -> Grading`, `finish() -> SessionResult`.
- **Grading**: MC index equality; TF bool equality; SA normalized compare (A-05). Pure `Grader` type.
- **Outputs**: `SessionResult { total, correct, wrongQuestions: [QuestionRef], elapsed: Duration }` (REQ-013); always appends events to `EventLog`; forwards outcomes to `SchedulerService` **only for `.review` sessions** — `.targeted` sessions never touch box state (REQ-019).
- **Test strategy**: engine tests with injected `Clock`; grader table-driven tests.

### 7.8 EventLog + StatsService
- **EventLog interface**: `append(_ event: LogEvent) throws`, `events(profile:) -> [LogEvent]` where `LogEvent = .practice(PracticeEvent) | .subscription(SubscriptionEvent)` (JSONL, append-only, REQ-021). The log is the sole source of truth; `scheduler.json` is derived from it (REQ-022).
- **StatsService**: pure reducers: `minutes(by: subject/topic)`, `lastPracticed(by:)`, `daysPracticed(inMonthOf: Date)` (REQ-015/016/022).
- **Test strategy**: reducer tests over fixture event streams; property: appending events never changes prior aggregates for prior dates.

### 7.9 Rendering (MarkdownMathUI)
- **Purpose**: display Markdown with interleaved LaTeX.
- **Design**: `QuestionParser` pre-splits text into segments `[.text(AttributedString), .inlineMath(String), .blockMath(String)]` (delimiters `$...$`, `$$...$$`); `RichTextView` composes SwiftUI `Text` + `MathView`. `MathView` wraps `MTMathUILabel` in `UIViewRepresentable` (per SwiftMath's documented SwiftUI usage).
- **Failure modes**: invalid LaTeX → SwiftMath inline error display is suppressed; app shows raw source in monospace + logs `E-TEX-01`.
- **Test strategy**: segmenter unit tests (delimiters, escaped `\$`); snapshot tests for representative equations.

### 7.10 Logging
- `AppLogger` over `os.Logger`, categories: `import`, `parse`, `schedule`, `session`, `persist`. Every log line carries `correlationID` (UUID per import or session). Log level configurable (§18).

## 8. SwiftUI Screen Design

For brevity, per screen: purpose / state / actions / hierarchy / navigation / validation / accessibility / preview needs.

### 8.1 ProfileSelectView
- **State**: `ProfileSelectVM: @Observable { profiles: [Profile], streakDays: Int?, phase: Idle|Loaded|Error }`.
- **Actions**: select, create (sheet), delete (two-step: alert summarizing the data that will be lost — e.g., "340 practice records" — then typing the nickname to enable Delete), rename.
- **Hierarchy**: `ScrollView → LazyVGrid(ProfileCard) → NewProfileCard`; post-select overlay "N days practiced this month" (REQ-016) auto-continues.
- **Navigation**: sets `activeProfileID`, pushes Home.
- **Validation**: nickname non-empty, unique; inline error text.
- **Accessibility**: cards labeled "Profile, <nickname>"; streak announced via `accessibilityLabel` on overlay.
- **Preview**: mock ProfileStore with 0, 1, 5 profiles.

### 8.2 HomeView
- **State**: `HomeVM { dueCount: Int, subscribedTopicCount: Int, monthDays: Int }`.
- **Actions**: Start Review, Practice a Topic, Library, Reports, Switch Profile.
- **Empty**: `subscribedTopicCount == 0` → hint card linking to Library.
- **Accessibility**: "Review due, N questions, button".
- **Preview**: due 0 / due 12.

### 8.3 LibraryView
- **State**: `LibraryVM { decks: [DeckSummary], filter: LibraryFilter, phase }`; `LibraryFilter { subject: String?, topic: String?, notMasteredOnly: Bool }` (REQ-006).
- **Actions**: change filters, toggle topic subscription (REQ-011), open Import, delete deck, start targeted practice from a topic row (REQ-012).
- **Hierarchy**: filter bar → `List` sections by subject → topic rows (subscribe `Toggle`, mastery badge, Practice button).
- **Validation**: none.
- **Accessibility**: toggle labeled "Spaced repetition for <topic>".
- **Preview**: empty; 3 decks × 2 subjects; filtered.

### 8.4 ImportView
- **State**: `ImportVM { mode: .local|.url, urlText: String, phase: Idle|Fetching|Validating|Failed([ValidationIssue])|Succeeded(DeckSummary) }`.
- **Actions**: pick file (`.fileImporter`), fetch URL, retry (URL preserved, REQ-031), open imported deck.
- **Validation**: URL syntactic check before fetch; parse issues rendered as list with code + line (REQ-023).
- **Accessibility**: issues list readable row-by-row.
- **Preview**: each phase with fixture issues.

### 8.5 SessionView
- **State**: `SessionVM { index, total, current: QuestionViewData, input: AnswerInput, phase: Answering|Feedback(Grading)|Finished(SessionResult) }`.
- **Actions**: select choice / toggle TF / type SA; Submit; Next; Abandon (confirm; partial results logged).
- **Hierarchy**: progress header → `RichTextView` prompt → input control → Submit/Next.
- **Answer secrecy**: `QuestionViewData` contains no answer key; grading happens in `SessionEngine` (REQ-017).
- **Accessibility**: choices as buttons with rendered-text labels + LaTeX alt; feedback announced with `AccessibilityNotification`.
- **Preview**: one fixture per question type; feedback states.

### 8.6 SessionResultsView
- **State**: `SessionResult` (immutable).
- **Actions**: review wrong question (detail sheet shows prompt + correct answer), Done.
- **Accessibility**: summary sentence first: "You got 7 of 10 correct in 4 minutes 12 seconds."

### 8.7 ReportsView
- **State**: `ReportsVM { rows: [ReportRow], phase }`; `ReportRow { subject, topic, minutes, mastery: MasteryLevel, lastPracticed: Date? }` (REQ-015).
- **Hierarchy**: grouped `List` by subject; rows show topic, minutes, badge, relative date.
- **Preview**: fixture event log producing mixed mastery levels.

## 9. State Management

- **Local view state**: `@State` for transient UI (sheet visibility, text fields).
- **Shared app state**: `AppRouter` and `ActiveProfile` as `@Observable` objects in the environment.
- **Async loading**: every VM exposes `phase: Phase<Value>` enum (`idle/loading/loaded(Value)/failed(AppError)`) — one pattern app-wide.
- **Error state**: `AppError` (user-facing message + underlying code) rendered by a shared `ErrorBanner` view.
- **Form validation**: pure `validate()` functions on VMs returning `[FieldError]`; Submit disabled until empty.
- **Persistence state**: stores are the source of truth; VMs reload via async `refresh()`; no duplicated caches in VMs.
- **Retry state**: `phase = .failed` retains last inputs (e.g., `urlText`) so retry re-invokes with identical parameters (REQ-031).

## 10. Data Model

### 10.1 Deck file format (the contract)

````markdown
---
id: algebra-linear-eq-basics        # required, stable, kebab-case
title: "Linear Equations Basics"    # required
subject: math                       # required (free string; math|science|history|music|...)
topics: [linear-equations, algebra] # required, >= 1
creator: "Ms. Rivera"               # required
version: 3                          # required, integer, monotonic
age_range: "10-14"                  # required, "min-max"
audience: "Grade 6-8 pre-algebra"   # required, free text
---

### [MC] q001 {topic: linear-equations}
Solve for $x$: $2x + 3 = 11$

- ( ) $x = 3$
- (x) $x = 4$
- ( ) $x = 7$

### [TF] q002 {topic: linear-equations}
The equation $x + 1 = x$ has no solution.

Answer: true

### [SA] q003 {topic: algebra, tolerance: 0.01}
Evaluate $\frac{3}{4} + \frac{1}{4}$ as a decimal.

Answer: 1.0
Alt: one
````

Parsing rules (deterministic, cheap): each question = one H3 heading matching `[TYPE] <id> {attrs}`; body until next H3; MC answers are the list items, `(x)` marks the correct one; TF/SA use a trailing `Answer:` line (`Alt:` lines add accepted SA variants). LaTeX delimiters: `$...$` inline, `$$...$$` block, `\$` escapes a literal dollar. This grammar is expressible as a small walk over swift-markdown's heading/list/paragraph nodes with no backtracking.

Tightened grammar for nontechnical authors — every rule enforced with a line-numbered diagnostic (REQ-023):
- **Headings**: question headings must be exactly H3 (`###`). An H2/H4 heading matching the `[TYPE]` pattern raises `E-Q-08` ("use ### for questions") at its line.
- **Markers**: `(x)` is case-insensitive (`(X)` accepted); internal whitespace inside `( )`/`(x)` is ignored; any other paren content raises `E-Q-02` with the offending line.
- **`Answer:` / `Alt:`**: must be the last non-empty lines of a TF/SA question body. TF accepts only `true`/`false`, case-insensitive; `yes`/`t`/`1` raise `E-Q-03` and the message names the accepted values.
- **`{attrs}`**: optional. `topic` defaults to the first front-matter topic; unknown keys are ignored with warning `W-Q-01`.
- **Escaping**: `\$` is a literal dollar; `$` inside inline code or fenced code blocks is never math; an unterminated `$` raises `E-TEX-02` at its line.
- **Sanitization (untrusted input, REQ-062)**: inline/block HTML is stripped (`W-SEC-01`), images removed (`W-SEC-02`), links render as their plain text and are never tappable; deck content can never trigger a fetch.
- **Limits (REQ-063)**: file ≤ 1 MB, ≤ 1,000 questions, prompt ≤ 10,000 chars, LaTeX segment ≤ 2,000 chars.

### 10.2 Swift models

```swift
struct Deck: Codable, Equatable, Identifiable {
    let id: String                 // front matter id
    var meta: DeckMeta
    var questions: [Question]
}

struct DeckMeta: Codable, Equatable {
    var title: String, subject: String, topics: [String]
    var creator: String, version: Int, ageRange: String, audience: String
}

enum QuestionType: String, Codable { case mc = "MC", tf = "TF", sa = "SA" }

struct Question: Codable, Equatable, Identifiable {
    let id: String                 // unique within deck (E-Q-05)
    var type: QuestionType
    var topic: String
    var prompt: String             // raw markdown+latex, answer-free
    var choices: [String]?         // MC only, answer-free order preserved
    var key: AnswerKey             // kept separate from view data (REQ-017)
    let contentHash: String        // SHA-256 of normalized prompt+choices+key (REQ-020)
}

enum AnswerKey: Codable, Equatable {
    case mc(correctIndex: Int)
    case tf(Bool)
    case sa(accepted: [String], tolerance: Double?)
}

struct Profile: Codable, Equatable, Identifiable {
    let id: UUID
    var nickname: String
    var createdAt: Date
}

struct QuestionRef: Codable, Hashable { let deckID: String; let questionID: String; let topic: TopicKey }
struct TopicKey: Codable, Hashable { let subject: String; let topic: String }

struct BoxState: Codable, Equatable {          // scheduler state per (profile, question)
    var box: Int                                // 1...5
    var nextDue: Date
}

struct PracticeEvent: Codable, Equatable {      // append-only (REQ-021)
    let id: UUID
    let profileID: UUID
    let sessionID: UUID                         // correlation ID (REQ-070)
    let question: QuestionRef
    let outcome: Outcome                        // .correct | .incorrect
    let answeredAt: Date
    let responseSeconds: Double
    let mode: SessionMode                       // .review | .targeted
}

struct SubscriptionEvent: Codable, Equatable { // append-only (REQ-021/022)
    let id: UUID
    let profileID: UUID
    let topic: TopicKey
    let action: Action                          // .subscribe | .unsubscribe
    let at: Date
}

enum MasteryLevel: String, Codable { case new, learning, practicing, mastered }
// Deterministic mapping per topic (REQ-014/022); boxes reflect review-mode outcomes only (REQ-019):
// mastered:   >= 90% of topic questions in box >= 4
// practicing: >= 50% in box >= 3
// learning:   any event recorded
// new:        otherwise
```

### 10.3 Validation rules
Front matter: all eight keys present, `version` int ≥ 1, `topics` non-empty, `age_range` matches `\d+-\d+`. Questions: ≥ 1 question; unique IDs; MC has ≥ 2 choices and exactly one `(x)`; TF answer ∈ {true,false}; SA has ≥ 1 accepted answer; every question `topic` (explicit or defaulted) ∈ front matter `topics` (else `E-Q-07`). Resource limits per REQ-063 (`E-LIM-*`). Sanitization warnings (`W-SEC-*`) never block import; they are listed in the success state so the importer can review them.

## 11. Networking and Persistence

- **Networking**: single operation — `GET url` via `URLSession` behind `protocol DeckFetching { func fetch(_ url: URL) async throws -> String }`. Accepts `text/markdown`, `text/plain`, `application/octet-stream` (raw.githubusercontent serves text/plain); rejects others (`.invalidContentType`). Timeout 15 s. No auth.
- **Error handling**: `URLError` mapped to `.network`; body decode as UTF-8 with BOM strip.
- **Caching / offline**: the cache IS the source of truth (REQ-005/030); a deck is fetched once at import and never auto-refetched. Manual re-import = update (REQ-020).
- **Persistence layout** (Application Support/MarkRecall/):
  - `profiles.json` — `[Profile]`
  - `decks/<deckID>/source.md` + `manifest.json` (DeckMeta + question index with per-question content hashes, REQ-020)
  - `state/<profileID>/events.jsonl` — append-only, one `LogEvent` (practice or subscription) per line — **sole source of truth**
  - `state/<profileID>/scheduler.json` — derived cache: `[QuestionRef: BoxState]` + current subscriptions + `rebuiltFromEventCount`; missing/corrupt → rebuilt by replay (REQ-022/024)
  - Writes are atomic (`.atomic` / write-temp-then-rename); JSONL appends are single-line `write(contentsOf:)` with `O_APPEND` semantics via `FileHandle`.
- **Sync strategy**: none (out of scope).
- **UserDefaults**: `activeProfileID`, log level only. No Keychain need (no secrets).

## 12. AI-Agent Interaction Model

- **Inputs the agent can provide**: deck Markdown text (via fixture files or a `MarkRecall://import?path=` launch argument in DEBUG); launch arguments `-uiTestMode 1 -fixtureDeck <name> -frozenDate <ISO8601>` to seed deterministic state; simulated answers via XCUITest.
- **Outputs the agent can inspect**: (1) `ImportResult` JSON — in DEBUG, every import writes `last-import-result.json` (deck summary or `[ValidationIssue]`) to Documents for retrieval; (2) `events.jsonl` and `scheduler.json` — plain text, stable schema; (3) structured `os_log` output filtered by subsystem `com.MarkRecall.app`.
- **Validation signals**: exit contract of `DeckParsing` — success (Deck JSON) or failure (issue codes + lines). Codes are stable API (§7.2).
- **Retry/adjustment loop**: agent imports file → reads `last-import-result.json` → if `E-Q-02` at line 14, fixes the `(x)` marker at that line → re-imports. Same loop applies to URL import (retry preserves URL, REQ-031).
- **Error feedback format**: `{"status":"failed","issues":[{"code":"E-Q-02","message":"...","line":14}]}` — fixed schema, versioned by `formatVersion` field.
- **Trace/correlation IDs**: `sessionID` on every event + log line; import operations carry `importID`.
- **Deterministic checks**: `LeitnerRules`, `Grader`, `StatsService` reducers, and `DeckParsing` are pure — the agent can verify them by golden files without booting UI. `-frozenDate` makes due-queue computation reproducible. The agent can also delete `scheduler.json`, relaunch, and diff the rebuilt file against the original to verify replay determinism (REQ-022/024).
- **UI-test-verifiable behavior**: all controls carry `accessibilityIdentifier`s (`profile.card.<nickname>`, `session.submit`, `results.correctCount`, `library.filter.subject`, ...), enumerated in a checked-in `AXIdentifiers.swift` shared with the UI test target.

## 13. Verification Strategy

- **Unit tests**: LeitnerRules transition table; Grader (per type incl. normalization/tolerance); Markdown/LaTeX segmenter; validation rules (one test per error code); Stats reducers; date math with injected calendar/timezone.
- **View model tests**: phase transitions per VM using in-memory services (idle→loading→loaded/failed; retry keeps inputs).
- **Service tests**: DeckImportService with stub fetcher matrix (200 / 404 / timeout / wrong MIME / invalid body).
- **Repository tests**: DeckStore/ProfileStore/EventLog against temp dirs; re-import version gate + hash-based progress carry-over (REQ-020); append-only invariant (REQ-021); property test: delete scheduler.json → rebuild → byte-identical box states (REQ-022/024).
- **UI tests (XCUITest)**: first launch → profile → sample import → targeted session → results; SR subscribe → answer → relaunch with `-frozenDate +1d` → question due again after wrong answer.
- **Snapshot tests**: RichTextView with representative LaTeX corpus (fractions, roots, matrices) — catches SwiftMath regressions.
- **Accessibility checks**: XCTest `performAccessibilityAudit()` on each screen; VoiceOver label assertions via identifiers.
- **Fixtures**: `Fixtures/valid/` (each question type, unicode, large 500-question deck), `Fixtures/invalid/` (one per error code, expected-issues sidecar JSON = golden files).
- **Mock services**: in-memory implementation per protocol, checked into main target under `#if DEBUG` for previews reuse.
- **Human review points**: LaTeX rendering quality on-device; age-appropriateness of copy; final privacy review.

## 14. Observability and Traceability

- **Logs**: `os.Logger`, subsystem `com.MarkRecall.app`, categories per §7.10; every line includes correlation ID.
- **Metrics (local only)**: import duration, parse duration, session length — logged, not uploaded (REQ-060).
- **Analytics events**: none (out of scope by privacy stance).
- **Audit**: the event log itself is the audit trail for all practice claims (REQ-021/022).
- **Crash reporting**: none in v1; boundary documented so MetricKit could be added without new permissions.
- **Requirement-to-test mapping**: `Traceability.md` generated by a script that greps `REQ-\d+` across `AcceptanceTests.md` and test names `test_REQ_xxx_*`; CI fails if any REQ has zero references (REQ-071).

## 15. Error Handling

| Situation | User-facing behavior | Recovery |
|---|---|---|
| URL unreachable/timeout | "Couldn't reach that address." | Retry button, URL preserved (REQ-031) |
| HTTP 404 / non-text | "That link doesn't point to a Markdown file." | Edit URL, retry |
| Validation failure | Issue list with line numbers | Fix file, re-import (agent loop §12) |
| Corrupt cached deck | Deck marked unreadable; offer delete/re-import | Non-destructive; other decks unaffected |
| Persistence write failure | "Couldn't save — check device storage." | Retry write; session results held in memory until saved |
| Corrupt profiles.json | Rename to `.corrupt-<ts>`, start empty, log `E-PERSIST-01` | Old file kept for manual recovery |
| Corrupt/missing scheduler.json | Nothing visible — rebuilt silently by log replay, `E-PERSIST-02` logged | Automatic (REQ-024) |
| Re-import with equal/lower version | "This deck is already at version N." (`E-VER-01`) | Bump `version` in the file, or delete the deck first |
| Invalid LaTeX in deck | Raw source shown monospace, `E-TEX-01` logged | Content author fixes; question still answerable |
| Permissions (file picker denied/expired scope) | "Couldn't read that file — pick it again." | Re-open picker |

No authentication → no auth failures.

## 16. Permissions, Privacy, and Security

- **iOS permissions**: none beyond default. File access uses `fileImporter` (user-mediated, no entitlement prompt); network needs no permission. No purpose strings required. [Inference] based on the feature set; if a future feature adds photos/camera, purpose strings would be added then.
- **Data minimization**: nickname + practice events only; no birthdate, no real names requested (UI copy says "nickname, not your real name") (REQ-001/060).
- **Sensitive data**: none classified sensitive; still device-local only.
- **Keychain**: not used (no secrets).
- **Networking**: ATS default = HTTPS only (REQ-061); no cookies; ephemeral `URLSession`.
- **Untrusted content**: decks are community-authored input. The parser strips HTML and images, renders links as plain text, enforces resource limits, and guarantees deck content can never initiate network activity (REQ-062/063); the only network call in the app is the user-initiated import fetch.
- **Local storage risks**: shared-device profiles are unauthenticated by design (A-06); deletion requires a typed-nickname confirmation to protect against accidental or sibling deletion. Documented in App Privacy as "Data Not Collected".
- **Children's privacy**: no tracking, no ads, no third-party SDKs — aligns with kids-app expectations without opting into the Kids Category.

## 17. Accessibility

- **VoiceOver**: labels/hints on all controls; math views expose LaTeX source or author-supplied alt text (REQ-050); grading feedback posted as announcement.
- **Dynamic Type**: all text styles semantic (`.body`, `.title2`...); MathView font size scales with `@Environment(\.dynamicTypeSize)` mapped to point sizes.
- **Contrast**: system semantic colors; mastery badges use icon + text, not color alone.
- **Reduce Motion**: streak overlay and feedback transitions honor `accessibilityReduceMotion` (crossfade instead of spring).
- **Hit targets**: min 44×44 pt (REQ-052); TF buttons oversized for younger users.
- **Keyboard/input**: SA field uses appropriate keyboard (`.numbersAndPunctuation` when tolerance present); hardware-keyboard Return submits.

## 18. Configuration

| Key | Default | Notes |
|---|---|---|
| `network.timeoutSeconds` | 15 | URL import |
| `network.retryLimit` | 3 (manual) | retries are user/agent-driven, not automatic |
| `leitner.intervalsDays` | [0,1,3,7,14] | compile-time constant, single source |
| `mastery.thresholds` | mastered ≥ 0.9 @ box4; practicing ≥ 0.5 @ box3 | with `LeitnerRules` |
| `session.maxReviewBatch` | 20 | cap per review session |
| `log.level` | `.info` (`.debug` in DEBUG) | Settings toggle in DEBUG only |
| `uiTestMode`, `fixtureDeck`, `frozenDate` | off | launch args, DEBUG only |
| `limits.*` | 1 MB / 1,000 q / 10,000 ch / 2,000 ch | REQ-063, compile-time constants |
| `sampleDeck.enabled` | true | bundled fixture |

No remote config, no feature-flag service (out of scope).

## 19. Extensibility

- **New question type**: add `QuestionType` case + parser rule + `Grader` case + one input view; `ValidationIssue` codes namespaced `E-Q-*` leave room. Everything else (scheduler, events, stats) is type-agnostic via `QuestionRef`.
- **New scheduler**: `Scheduling` is a protocol; an SM-2/FSRS implementation can replace Leitner per-profile without touching UI (state file carries `algorithm` tag).
- **New content source** (GitLab, WebDAV): new `DeckFetching` implementation + import UI entry.
- **New screens**: `Route` enum case + view + VM; DI already environment-wide.
- **New validators**: `DeckValidator` is a list of `(Deck) -> [ValidationIssue]` rules; append to array.
- **New export/report types**: StatsService reducers are pure over the event log — any new aggregate is a new reducer, no schema change.

## 20. Testing Plan

1. **Unit** (fast, pure): rules/grader/parser/reducers — run on every commit.
2. **Integration**: import pipeline end-to-end (fixture file → cache → library listing) with real file system in temp dir.
3. **UI (XCUITest)**: the two golden journeys (§13) + empty-state and error-state walkthroughs.
4. **Accessibility**: audit API per screen + manual VoiceOver pass on Session screen.
5. **Regression**: golden files for parser and issue codes; snapshot corpus for math rendering; Leitner transition table locked by tests.
6. **Manual smoke** (per release): device install, import from real GitHub raw URL, offline airplane-mode practice, profile delete.
7. **Acceptance**: execute `AcceptanceTests.md` scenarios; each maps to automated tests by REQ ID where feasible.

## 21. Implementation Notes

- **Folder structure**
  ```
  MarkRecall/
    App/            (MarkRecallApp, AppRouter, AppDependencies, AXIdentifiers)
    DeckFormat/     (pure: models, parser, validator, grader, LeitnerRules)  ← SPM local package
    Services/       (stores, import, scheduler, session, stats, logging)
    UI/             (per-screen folders: View + VM; Shared/ for RichTextView, MathView, ErrorBanner)
    Fixtures/       (valid/, invalid/, sample deck)
  Tests/ UnitTests, ServiceTests, UITests (mirrors folders)
  ```
  `DeckFormat` as a local SPM package keeps the pure core UIKit-free and mac-testable.
- **Naming**: views `*View`, view models `*VM`, protocols verb-ing (`DeckParsing`), errors `E-<AREA>-NN`.
- **DI**: `AppDependencies` struct of protocol existentials; `EnvironmentKey` `\.deps`; previews use `.mock` factory.
- **Mocking**: hand-written in-memory implementations (no mocking framework); stubs return canned `Result`s.
- **Async/await**: all service APIs `async throws`; no Combine.
- **MainActor**: VMs are `@MainActor @Observable`; services are actor-isolated where they own mutable state (`DeckStore`, `EventLog` as `actor`s).
- **Previews**: every view has previews for empty/loaded/error using fixture data; `#Preview` macros.
- **Test targets**: `DeckFormatTests` (SPM, no simulator), `AppTests` (simulator, services+VMs), `AppUITests`.
- **SwiftMath integration**: wrap `MTMathUILabel` once in `MathView: UIViewRepresentable` (pattern shown in SwiftMath's README); size with `sizeThatFits` to avoid layout jumps.

## 22. Open Questions

1. Should SA grading support simple algebraic equivalence (e.g., `1/2` vs `0.5` beyond declared `Alt:` lines)? Current design: author-declared alternates only.
2. Should the review queue interleave topics or group by topic? Current: shuffled interleave.
3. Streak definition: distinct days with ≥ 1 answered question — should a minimum (e.g., 5 questions) count instead?
4. Kids Category App Store enrollment — desired or explicitly avoided?
5. Profile PINs: deferred for household use; if classroom deployment is likely, a 4-digit PIN per profile is the smallest adequate upgrade. Decide before any classroom rollout.
6. Should `W-SEC-*` sanitization warnings ever be surfaced to students, or only to the importer at import time? Current: import time only.

Resolved since v1 of this plan: targeted sessions never touch box state (REQ-019); the event log is the sole source of truth and `scheduler.json` a rebuildable cache (REQ-022/024); untrusted-content sanitization and limits added (REQ-062/063); equal/lower-version re-import rejected and progress carried only on content-hash match (REQ-020); frontmatter package replaced with Yams + in-app splitter (A-08); per-question `topic` attr made optional with a default.

---

### Sources (library facts verified 2026-07-07)
- swift-markdown (swiftlang), cmark-gfm-based parser: https://github.com/apple/swift-markdown
- SwiftMath (mgriebling), LaTeX math-mode rendering, `MTMathUILabel`, SwiftUI wrapper pattern, iOS 11+: https://github.com/mgriebling/SwiftMath ; https://swiftpackageregistry.com/mgriebling/SwiftMath
- Yams (jpsim), YAML Codable support (`YAMLDecoder`): https://github.com/jpsim/Yams
- SwiftToolkit/frontmatter (evaluated and **dropped** — manifest declares platforms `.macOS(.v13)` only): https://github.com/SwiftToolkit/frontmatter
