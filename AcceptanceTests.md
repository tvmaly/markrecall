# Acceptance Tests

Conventions: `Given the app is launched in test mode with frozen date <D>` maps to launch args `-uiTestMode 1 -frozenDate <D>`. Fixture decks live in `Fixtures/`. Each scenario lists the requirement IDs it verifies.

## Feature: First Launch and Profiles

### Scenario: First launch shows profile creation
Given the app is installed with no existing data
When the app is launched
Then the Profile Select screen is shown with no profiles
And an empty state prompts "Create your first profile"
Requirements: REQ-001, REQ-040

### Scenario: Create a profile with a nickname
Given the Profile Select screen is shown
When the user creates a profile with nickname "Ada"
Then a profile card "Ada" appears
And no field other than nickname is requested
Requirements: REQ-001, REQ-002, REQ-060

### Scenario: Reject empty nickname
Given the profile creation sheet is open
When the user submits an empty nickname
Then an inline validation error is shown
And no profile is created
Requirements: REQ-002, REQ-023

### Scenario: Reject duplicate nickname
Given a profile "Ada" exists
When the user creates another profile named "Ada"
Then an inline error explains the nickname is taken
Requirements: REQ-002

### Scenario: Delete a profile removes its practice data after confirmation
Given profile "Ada" exists with recorded practice events
When the user deletes profile "Ada"
And the confirmation alert summarizes the practice data that will be lost
And the user types "Ada" to enable and confirm Delete
Then "Ada" no longer appears on Profile Select
And Ada's scheduler state and event log files are removed
Requirements: REQ-002, REQ-021

### Scenario: Mistyped delete confirmation does not delete
Given profile "Ada" exists with practice data
When the user attempts deletion and types "Adaa"
Then the Delete action remains disabled
And the profile and its data are unchanged
Requirements: REQ-002

### Scenario: Days-practiced-this-month shown on profile selection
Given the frozen date is 2026-07-15
And profile "Ada" has practice events on 3 distinct days in July 2026
And practice events on days in June 2026
When the user selects profile "Ada"
Then an overlay shows "3 days practiced this month"
Requirements: REQ-016, REQ-022

### Scenario: Month rollover resets the displayed count
Given profile "Ada" practiced on 10 distinct days in June 2026
And has not practiced in July 2026
And the frozen date is 2026-07-01
When the user selects profile "Ada"
Then the overlay shows "0 days practiced this month"
Requirements: REQ-016, REQ-022

## Feature: Deck Import — Local File

### Scenario: Import a valid deck from a local file
Given the user is on the Import screen in Local File mode
When the user picks fixture "valid/algebra-basics.md"
Then a success state shows the deck title, subject, topic list, and question count
And the deck appears in the Library under subject "math"
Requirements: REQ-003, REQ-005, REQ-007, REQ-040

### Scenario: Import an invalid deck shows structured validation errors
Given the user is on the Import screen in Local File mode
When the user picks fixture "invalid/mc-two-correct.md"
Then a failure state lists issue code "E-Q-02" with a message and line number
And the deck is not added to the Library
Requirements: REQ-023, REQ-041

### Scenario: Missing front matter key is reported with its code
When fixture "invalid/missing-subject.md" is imported
Then the issue list contains code "E-FM-02" naming the missing key "subject"
Requirements: REQ-007, REQ-023

### Scenario: Author-friendly marker tolerance
Given fixture "valid/marker-tolerance.md" using "(X)" and "( )" markers with extra internal spaces
When it is imported
Then the import succeeds and the correct choices are identified
Requirements: REQ-008, REQ-023

### Scenario: Wrong heading level gets a targeted diagnostic
Given fixture "invalid/h2-question.md" using "##" for a question heading
When it is imported
Then the issue list contains code "E-Q-08" with the heading's line number
And the message tells the author to use "###"
Requirements: REQ-023

### Scenario: Invalid true/false value names the accepted values
Given fixture "invalid/tf-yes.md" containing "Answer: yes"
When it is imported
Then the issue list contains code "E-Q-03" at the Answer line
And the message states only "true" or "false" are accepted
Requirements: REQ-023

### Scenario: Expired file access is recoverable
Given the system file picker grant has expired for a previously chosen file
When the import is attempted
Then a message asks the user to pick the file again
And re-picking the file completes the import
Requirements: REQ-003, REQ-041

## Feature: Deck Import — URL

### Scenario: Import a deck from a GitHub raw URL
Given the device is online
When the user enters a valid raw.githubusercontent.com URL for a valid deck and taps Fetch
Then a loading indicator is shown while fetching
And the deck is validated, cached, and shown as imported
Requirements: REQ-004, REQ-005, REQ-041

### Scenario: Network failure preserves the URL for retry
Given the device is offline
When the user fetches a deck URL
Then an error state explains the address couldn't be reached
And a Retry button is shown
And the previously entered URL is still populated
When connectivity is restored and the user taps Retry
Then the import completes successfully
Requirements: REQ-031, REQ-041

### Scenario: Plain HTTP URLs are rejected before any request
Given the user enters an "http://" URL on the Import screen
When the user taps Fetch
Then a validation message requires an HTTPS address
And no network request is made
Requirements: REQ-061, REQ-023

### Scenario: Non-Markdown content type is rejected
Given a URL that returns HTML
When the user fetches it
Then the error states the link does not point to a Markdown file
And nothing is cached
Requirements: REQ-004, REQ-023

### Scenario: Re-import with a higher version preserves progress only for unchanged content
Given deck id "algebra-basics" version 1 is cached
And profile "Ada" has box state on questions "q001" and "q002"
When version 2 is imported where "q001" is byte-identical and "q002"'s prompt changed
Then the Library shows version 2 exactly once
And Ada's box state for "q001" is preserved
And "q002" is reset to box 1 because its content hash changed
Requirements: REQ-020

### Scenario: Re-import with an equal or lower version is rejected
Given deck id "algebra-basics" version 2 is cached
When a file with the same id and version 2 (or lower) is imported
Then the import fails with code "E-VER-01" naming the cached version
And the cached deck and all progress are unchanged
Requirements: REQ-020, REQ-023

## Feature: Library and Filtering

### Scenario: Empty library shows import call to action
Given no decks are cached
When the Library is opened
Then an empty state offers "Import your first deck" and "Load sample deck"
Requirements: REQ-040

### Scenario: Filter by subject
Given decks exist with subjects "math" and "science"
When the user filters by subject "science"
Then only science decks are listed
Requirements: REQ-006

### Scenario: Filter by not-yet-mastered for the active profile
Given profile "Ada" has mastered topic "linear-equations" and not topic "fractions"
When the user enables the "Not yet mastered" filter
Then topic "fractions" is listed and "linear-equations" is not
Requirements: REQ-006, REQ-014

### Scenario: Cached decks are fully usable offline
Given deck "algebra-basics" was imported while online
And the device is now offline
When the user browses the Library and starts a targeted session on it
Then all questions render and the session completes normally
Requirements: REQ-005, REQ-030

## Feature: Topic Subscription (Spaced Repetition Enrollment)

### Scenario: Subscribe a topic adds its questions to the review queue
Given profile "Ada" has no subscriptions
When Ada subscribes to topic "linear-equations"
Then all questions of that topic enter box 1
And Home shows a non-zero "Review Due" count
And a subscription event is appended to the event log
Requirements: REQ-010, REQ-011, REQ-021, REQ-022

### Scenario: Unsubscribe removes a topic from review without deleting history
Given Ada is subscribed to "linear-equations" with recorded events
When Ada unsubscribes the topic
Then its questions no longer appear in the review queue
And the Reports screen still shows minutes and last-practiced for the topic
Requirements: REQ-011, REQ-015, REQ-021

## Feature: Review Session (Spaced Repetition)

### Scenario: Correct answer promotes a question one box
Given the frozen date is 2026-07-07
And question "q001" is in box 2 for Ada
When Ada answers "q001" correctly in a review session
Then "q001" moves to box 3
And its next due date is 3 days later per the interval table
Requirements: REQ-010, REQ-018

### Scenario: Incorrect answer demotes to box 1
Given question "q001" is in box 4 for Ada
When Ada answers it incorrectly
Then "q001" moves to box 1
And it is due again the same day
Requirements: REQ-018

### Scenario: Box 5 is a cap
Given question "q001" is in box 5
When Ada answers it correctly
Then it remains in box 5
Requirements: REQ-018

### Scenario: Due queue respects frozen date across relaunch
Given Ada answered "q001" correctly on 2026-07-07 moving it to box 2 (due +1 day)
When the app is relaunched with frozen date 2026-07-08
Then "q001" appears in the Review Due queue
Requirements: REQ-010, REQ-022

### Scenario: All caught up empty state
Given Ada has subscriptions but no questions due today
When Ada taps "Review Due"
Then an empty state says practice is complete for today
Requirements: REQ-040

## Feature: Targeted Practice (Test Prep, no SR)

### Scenario: Practice a specific subject and topic
Given deck "algebra-basics" contains 5 questions under topic "fractions"
When Ada starts targeted practice on math / fractions
Then exactly those 5 questions are presented in shuffled order
And Leitner boxes are not changed by this session (scheduler.json is byte-identical afterward)
Requirements: REQ-012, REQ-019

### Scenario: Session results after completing a set
Given Ada answers 4 of 5 questions correctly in a targeted session
When the last question is submitted
Then the results screen shows elapsed time for the full set
And shows "4 of 5 correct"
And lists the 1 question answered wrong, viewable with its correct answer
Requirements: REQ-013

### Scenario: Abandoning a session logs partial results
Given Ada has answered 2 questions mid-session
When Ada abandons the session and confirms
Then the 2 answered questions are recorded in the event log
And no results are shown for unanswered questions
Requirements: REQ-013, REQ-021

### Scenario: Targeted outcomes count toward minutes but never mastery
Given "q001" is in box 4 for Ada and its topic shows mastery "Practicing"
When Ada answers "q001" incorrectly in a targeted session
Then "q001" remains in box 4
And the topic's mastery badge is unchanged
And Reports minutes and last-practiced for the topic are updated
And the event is logged with mode "targeted"
Requirements: REQ-019, REQ-015, REQ-021

## Feature: Question Presentation and Answer Secrecy

### Scenario: Answers are never visible before submission
Given a session presents an MC question from a deck whose file marks choice B with "(x)"
Then no visual marker distinguishes the correct choice before submission
And the view's data contains no answer key (verified via view-model inspection in tests)
Requirements: REQ-017

### Scenario: Single-answer normalization
Given SA question "q003" accepts "1.0" with tolerance 0.01 and alt "one"
When Ada submits " 1 "
Then the answer is graded correct
When Ada submits "ONE"
Then the answer is graded correct
Requirements: REQ-008

### Scenario: True/false grading
Given TF question "q002" with answer true
When Ada taps "True"
Then the feedback state shows correct
Requirements: REQ-008

### Scenario: LaTeX renders in prompt and choices
Given a question whose prompt contains "$\frac{3}{4}$" and choices containing inline math
When the question is displayed
Then rendered math views are present (identifier "math.view") for prompt and each choice
And no raw dollar-delimited source is visible
Requirements: REQ-009

### Scenario: Invalid LaTeX degrades to visible source without blocking
Given a deck question containing malformed LaTeX
When the question is displayed
Then the malformed segment is shown as monospaced source text
And the question can still be answered and graded
And a log line with code "E-TEX-01" is emitted
Requirements: REQ-009, REQ-070

## Feature: Untrusted Deck Content Safety

### Scenario: HTML in a deck is stripped with a warning
Given fixture "valid/html-injection.md" containing a script tag and inline HTML in a prompt
When the deck is imported
Then the import succeeds with warning "W-SEC-01" listing the line
And rendering the question shows no HTML content
Requirements: REQ-062, REQ-023

### Scenario: Images are removed and never fetched
Given a deck prompt containing a Markdown image pointing to a remote URL
When the deck is imported and the question displayed
Then no image is rendered and warning "W-SEC-02" was reported at import
And no network request to the image host occurs (verified via stub fetcher in test mode)
Requirements: REQ-062

### Scenario: Links render as plain text and are not tappable
Given a deck prompt containing a Markdown link
When the question is displayed
Then the link's text appears as plain text
And tapping it performs no navigation
Requirements: REQ-062

### Scenario: Oversized deck file is rejected with a limit code
Given a deck file larger than 1 MB
When it is imported
Then the import fails with code "E-LIM-01"
Requirements: REQ-063, REQ-023

### Scenario: Excess question count is rejected
Given a deck containing 1,001 questions
When it is imported
Then the import fails with code "E-LIM-02"
Requirements: REQ-063

## Feature: Reports

### Scenario: Report rows by subject and topic
Given Ada has events across math/fractions (12 minutes, last practiced 2026-07-05) and science/plants (3 minutes, last practiced 2026-07-01)
When Ada opens Reports
Then a math section shows fractions with 12 minutes, a mastery badge, and "Jul 5"
And a science section shows plants with 3 minutes and "Jul 1"
Requirements: REQ-014, REQ-015

### Scenario: Mastery levels are deterministic from box state
Given topic "fractions" has 10 questions and 9 are in box 4 or higher for Ada
Then the topic's mastery badge is "Mastered"
Given only 5 of 10 are in box 3 or higher
Then the badge is "Practicing"
Requirements: REQ-014, REQ-022

### Scenario: Reports are per profile
Given profiles "Ada" and "Ben" practiced different topics
When Ben opens Reports
Then only Ben's topics and minutes are shown
Requirements: REQ-001, REQ-014, REQ-015

## Feature: Persistence and Data Integrity

### Scenario: Event log is append-only
Given Ada completes a session producing 5 events
When any later operation runs (new session, unsubscribe, re-import)
Then the original 5 event lines are byte-identical in events.jsonl
Requirements: REQ-021

### Scenario: Stats are recomputable from raw files
Given Ada's events.jsonl and scheduler.json fixtures
When StatsService reducers run twice on the same inputs
Then both runs produce identical report rows and month-day counts
Requirements: REQ-022

### Scenario: Scheduler cache is rebuilt deterministically from the event log
Given Ada has box states derived from 200 logged events
When scheduler.json is deleted or corrupted and the app relaunches
Then a log line with code "E-PERSIST-02" is emitted
And the rebuilt scheduler.json yields box states identical to before
Requirements: REQ-022, REQ-024, REQ-070

### Scenario: Corrupt profiles file recovers non-destructively
Given profiles.json contains invalid JSON
When the app launches
Then the corrupt file is renamed with a ".corrupt" suffix
And the app starts with an empty profile list
And a log line with code "E-PERSIST-01" is emitted
Requirements: REQ-041, REQ-070

### Scenario: Practice state survives relaunch
Given Ada answers questions and force-quits the app
When the app relaunches and Ada is selected
Then Review Due counts, mastery badges, and report minutes reflect the pre-quit session
Requirements: REQ-005, REQ-021, REQ-022

## Feature: Accessibility

### Scenario: VoiceOver labels on session controls
Given VoiceOver is enabled in a session
Then each choice reads its rendered text (or LaTeX alt text)
And the Submit button reads "Submit answer"
And grading feedback is announced after submission
Requirements: REQ-050

### Scenario: Math views expose LaTeX as accessibility label
Given a prompt containing "$x^2$"
Then the math view's accessibility label contains the LaTeX source or provided alt text
Requirements: REQ-050

### Scenario: Dynamic Type scales session text
Given the system text size is set to accessibility XL
When a question is displayed
Then prompt text uses the enlarged size without truncating the Submit button off-screen
Requirements: REQ-051

### Scenario: Hit targets meet minimum size
When any screen is audited
Then all tappable controls are at least 44x44 points
Requirements: REQ-052

## Feature: Observability and Traceability

### Scenario: Session events carry a correlation ID
Given Ada completes a review session
Then every event written for the session shares one sessionID
And session log lines include the same sessionID
Requirements: REQ-070

### Scenario: Import emits structured logs
When a URL import succeeds
Then log lines in category "import" include an importID, URL host, and duration
Requirements: REQ-070

### Scenario: Every requirement maps to at least one scenario
Given the traceability script runs over AcceptanceTests.md and test names
Then every REQ-xxx in Plan.md is referenced at least once
And the CI check fails if any requirement is unreferenced
Requirements: REQ-071

## Feature: AI-Agent Verification and Retry Loop

### Scenario: Agent verifies a successful import via machine-readable output
Given the app is launched with "-uiTestMode 1"
When the agent imports fixture "valid/algebra-basics.md"
Then Documents/last-import-result.json has status "succeeded"
And contains the deck id, version, topic list, and question count
Requirements: REQ-023, REQ-070

### Scenario: Agent adjusts input after validation failure and retries
Given the agent imports fixture "invalid/mc-two-correct.md"
And last-import-result.json reports code "E-Q-02" at line 14
When the agent edits line 14 to leave exactly one "(x)" marker
And re-imports the corrected file
Then last-import-result.json has status "succeeded"
Requirements: REQ-023

### Scenario: Agent verifies Leitner determinism via frozen dates
Given launch args set frozen date 2026-07-07 and fixture deck seeded to box 2 for "q001"
When the agent answers "q001" correctly via UI automation
And relaunches with frozen date 2026-07-10
Then "q001" is present in the due queue (box 3, 3-day interval elapsed)
And relaunching with frozen date 2026-07-09 shows it absent
Requirements: REQ-010, REQ-018, REQ-022

### Scenario: Agent inspects raw state files to verify grading claims
Given the agent completed a session answering 3 correct and 2 incorrect
When the agent reads events.jsonl for the profile
Then exactly 5 events with the session's sessionID exist
And 3 have outcome "correct" and 2 have outcome "incorrect"
Requirements: REQ-021, REQ-022, REQ-070

### Scenario: Agent verifies retry preserves inputs deterministically
Given URL import fails with a simulated timeout in test mode
When the agent triggers Retry without editing the URL field
Then the identical URL is requested a second time (verified via import log lines)
Requirements: REQ-031, REQ-070

## Acceptance Test Coverage Checklist

- First launch: "First launch shows profile creation"
- Main user flow: profile → import → subscribe → review → results (Features 1, 2, 5, 6, 7)
- Navigation: covered implicitly across features; golden journeys in §13 of Plan.md
- Form/input validation: empty/duplicate nickname; URL syntax; deck validation codes
- Empty states: library, review queue, profiles, reports
- Loading states: URL fetch scenario
- Error states: network, content type, validation, corrupt files, expired file access
- Success states: import success, session results
- Offline/network failure: offline library use; retry-preserving-URL
- Persistence: append-only log, relaunch survival, corrupt-file recovery, re-import replacement
- Permission handling: file-picker scope expiry (no OS permission prompts exist by design)
- Accessibility: VoiceOver, math alt text, Dynamic Type, hit targets
- AI-agent verification and retry loop: dedicated feature above
- Untrusted content safety: HTML/image/link stripping and resource limits (REQ-062/063)
- Targeted-vs-review isolation: box state untouched by targeted sessions (REQ-019)
- Derived-cache rebuild: scheduler.json replay determinism (REQ-022/024)
- Version gating on re-import: reject equal/lower, hash-based progress carry-over (REQ-020)
- Requirement-to-scenario traceability: automated check scenario (REQ-071)
