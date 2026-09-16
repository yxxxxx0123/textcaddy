# TextCaddy MVP Design

**Date:** 2026-09-16

**Status:** Revised after adversarial review; awaiting final document approval

**Target:** Windows 11 x64

**Technology:** C#, .NET 10, WPF

## 1. Product summary

TextCaddy (文匣) is a lightweight, keyboard-first local text snippet library for Windows. It stores text that users deliberately add and makes that text available from a floating panel opened with a global shortcut.

The product solves a narrow problem: frequently reused text should not disappear when the clipboard is overwritten. TextCaddy is not a clipboard-history recorder and does not monitor every copy operation.

### 1.1 One-line positioning

> A keyboard-first local text snippet manager for Windows.

### 1.2 Target users

- People who repeatedly enter addresses, email addresses, account identifiers, commands, code snippets, prompts, or canned responses.
- Customer support, operations, software development, administration, and other keyboard-heavy workflows.
- Users who want persistent snippets without the collection behavior of a clipboard-history tool.

### 1.3 Core value

- **Persistent:** snippets are stored independently of the clipboard.
- **Fast:** opening, searching, selecting, and copying are keyboard-first.
- **Unobtrusive:** the quick panel appears near the pointer and requests focus restoration to the previous application without synthesizing input.
- **Local:** the MVP has no account, cloud sync, telemetry, or network dependency.

### 1.4 Product success criteria

The MVP succeeds only when a user can complete this loop reliably:

```text
Save a snippet → invoke TextCaddy → find and select it → copy it → return to the destination
```

Infrastructure that does not exercise this loop is scaffolding, not a product milestone.

## 2. Scope

### 2.1 MVP capabilities

- A single-instance Windows system-tray application.
- A configurable global shortcut that toggles a floating quick panel.
- Positioning near the pointer while remaining inside the active monitor's work area.
- Groups, snippets, ordering, search, and ten-item pages.
- Number-key selection, multi-selection, and deselection.
- Selection-order text composition with a configurable separator.
- Copy-and-close and copy-and-stay-open commands.
- A separate management window for snippet, group, and settings maintenance.
- Sensitive-item masking and Windows DPAPI protection for sensitive bodies.
- Clipboard-history and cross-device-roaming exclusion for sensitive content.
- Conditional timed clearing of clipboard content written from sensitive items.
- Local SQLite persistence, migration, and bounded backups.

### 2.2 Explicit non-goals

- Automatically collecting clipboard history.
- Browser autofill, credential login management, or password generation.
- Replacing a dedicated password manager.
- Cloud synchronization or team sharing.
- macOS or Linux support in the MVP.
- Rich text, images, or files in the MVP.
- Intercepting ordinary `Ctrl+V` behavior in other applications.
- Automatic text expansion while the user types.

## 3. Platform support

- Windows 11 x64 is the supported MVP platform.
- Windows 10 is best-effort only and is not part of the release acceptance matrix.
- ARM64 packaging is deferred until demand justifies its build and test cost.
- The application uses .NET 10 and WPF.

## 4. User experience

### 4.1 Application modes

TextCaddy has two UI modes with separate responsibilities:

1. **Quick panel:** transient, pointer-adjacent, keyboard-first retrieval and copying.
2. **Management window:** persistent editing of snippets, groups, ordering, and settings.

The system tray exposes commands to open the management window, open the quick panel, view application information, and exit.

### 4.2 Quick-panel lifecycle

1. The user presses the global shortcut.
2. TextCaddy records the foreground window that should receive focus later.
3. The panel is placed near the pointer on the pointer's current monitor.
4. Placement is clamped to the monitor work area and calculated using the monitor's DPI.
5. The panel opens in selection mode with the first result highlighted.
6. The user optionally enters search mode, filters the list, returns to selection mode, and selects snippets.
7. Copy-and-close returns focus to the recorded window; cancellation changes neither the clipboard nor stored data.

The provisional default shortcut is `Ctrl+Alt+Space`. `Alt+Space` is not used because Windows reserves it for the window system menu. Users can change the shortcut in settings.

Hotkey registration uses the Windows `MOD_NOREPEAT` flag. When a user changes the shortcut, TextCaddy registers the candidate shortcut before unregistering the working shortcut. A conflict therefore cannot leave the application without a valid hotkey unless no hotkey has ever been registered.

### 4.3 Closing and cancellation

- Pressing the global shortcut while the panel is open closes it.
- `Esc` closes the panel.
- Clicking outside the panel closes it.
- Cancellation clears the in-memory panel session and does not change the clipboard.

### 4.4 Search, navigation, and paging

- The panel has two visibly indicated input states: **selection mode** and **search mode**.
- It opens in selection mode so plain digits can select entries immediately.
- `/` or `Ctrl+F` enters search mode and focuses the search field.
- In search mode, all text input, including digits and IME composition, edits the query. Left and right arrows retain their normal text-caret behavior.
- `Enter` accepts the current query and returns to selection mode. `Esc` always closes the panel; it does not silently change modes.
- With no query, the panel shows the active group's ordered snippets.
- A query matches ordinary snippet titles and bodies without case sensitivity.
- Sensitive snippets are matched by title only; their protected bodies are not decrypted for incremental search.
- Up and down arrows move the highlighted result in either mode.
- `PageUp` and `PageDown` change pages without conflicting with text editing.
- Each page contains at most ten items.
- In selection mode, digits `1` through `9` address items one through nine and `0` addresses item ten.
- Search state lives only for the current panel session.

### 4.5 Selection model

- A digit toggles the selection of its corresponding visible item.
- Multiple selected items are retained in the order in which the user selected them.
- Deselecting an item removes it from the selection sequence.
- Selecting it again appends it to the end of the sequence.
- Selected items remain selected if a later query or page change makes them temporarily invisible.
- A persistent selection summary shows the count, ordered titles, and sensitive-item markers for all selected items, including invisible ones.
- `Ctrl+Backspace` clears the complete selection without closing the panel.
- Closing the panel clears the selection session.

### 4.6 Copy behavior

- `Ctrl+C` composes selected snippet bodies, writes the result to the clipboard, closes the panel, and restores focus.
- `Ctrl+Shift+C` performs the same copy but leaves the panel open.
- If nothing is selected, the highlighted item is copied.
- If nothing is selected or highlighted, the clipboard is not changed and the panel shows a brief non-blocking message.
- Multiple snippets are joined in selection order.
- The default separator is a Windows newline; the separator is configurable.
- The application does not synthesize a paste command. The user pastes normally in the destination application.

### 4.7 Sensitive clipboard clearing

When the composed result contains at least one sensitive snippet:

1. TextCaddy writes the value with Windows clipboard options that set `IsAllowedInHistory` and `IsRoamable` to `false`.
2. After a successful write, TextCaddy records the Windows clipboard sequence number rather than hashing or retaining the copied secret.
3. A configurable timer starts; the default is 30 seconds.
4. At expiry, TextCaddy clears the clipboard only if its sequence number is unchanged.

Sequence comparison prevents the cleanup task from deleting newer content, including a later copy whose text happens to be identical. Excluding history and roaming reduces Windows-managed persistence, but TextCaddy cannot prevent another process running as the same user from reading the clipboard before it is cleared.

### 4.8 Management window

The management window uses an explicit-save workflow:

- The group list selects the active group and supports create, rename, reorder, and delete.
- The snippet list supports create, edit, reorder, move to another group, and delete.
- Creating the first snippet also creates a default group when none exists.
- Titles, bodies, and group names must be non-empty after trimming. Duplicate snippet titles are allowed.
- Deleting a snippet requires confirmation.
- A non-empty group cannot be deleted until its snippets are moved or deleted; TextCaddy never cascades group deletion silently.
- Navigating away with unsaved edits prompts the user to save, discard, or cancel navigation.
- Sensitive bodies are masked by default, require an explicit reveal action, and are masked again when the editor loses focus or the management window is hidden.
- Reordering updates stable sort values in one transaction.

### 4.9 Accessibility

- Every interactive element is reachable by keyboard and has a visible focus indicator.
- Selection mode and search mode are conveyed by text and accessibility properties, not color alone.
- Buttons, list items, sensitive-state indicators, and transient messages expose accessible names.
- The UI follows Windows high-contrast settings and does not require animation to understand state changes.

## 5. Software architecture

### 5.1 Solution boundaries

```text
TextCaddy.slnx
├─ src/TextCaddy.App
├─ src/TextCaddy.Core
├─ src/TextCaddy.Infrastructure
├─ tests/TextCaddy.Core.Tests
├─ tests/TextCaddy.Infrastructure.Tests
└─ tests/TextCaddy.App.Tests
```

#### TextCaddy.App

Owns WPF views, view models, UI resources, application startup, dependency assembly, tray behavior, quick-panel sessions, management-window navigation, and presentation-specific window lifecycle.

It may depend on Core and Infrastructure. It must not contain SQL, cryptography, raw clipboard access, or Win32 interop beyond UI-specific window hooks delegated to a service.

#### TextCaddy.Core

Acts as the application core. It owns domain models, application use cases, business rules, and outbound service ports. Platform-oriented port names may appear here so use cases can request side effects, but Core has no dependency on WPF, SQLite, Windows Forms, Win32, or concrete adapter packages.

Core behavior includes:

- searching and ordering snippets;
- paging and digit-to-item mapping;
- selection sequencing and toggling;
- composing clipboard text;
- validation of snippets, groups, and settings.

#### TextCaddy.Infrastructure

Implements Core contracts for:

- SQLite persistence and migrations;
- DPAPI sensitive-data protection;
- Windows clipboard access;
- global hotkey registration;
- foreground-window capture and focus restoration;
- monitor, pointer, work-area, and DPI information;
- privacy-safe file logging.

Infrastructure depends on Core but never on App.

#### Test projects

- `TextCaddy.Core.Tests` tests deterministic domain behavior without Windows UI dependencies.
- `TextCaddy.Infrastructure.Tests` uses disposable data directories and temporary SQLite databases.
- `TextCaddy.App.Tests` tests view-model state and UI-independent orchestration.

### 5.2 Dependency direction

```text
TextCaddy.App ──────▶ TextCaddy.Core
      │
      └──▶ TextCaddy.Infrastructure ──▶ TextCaddy.Core

Tests ──▶ their corresponding production project
```

Dependencies are assembled in `TextCaddy.App` with `Microsoft.Extensions.DependencyInjection`. The MVP does not use the full generic host.

### 5.3 Primary interfaces

Core declares the contracts that isolate side effects:

- `ISnippetRepository`
- `IGroupRepository`
- `ISettingsRepository`
- `ISensitiveDataProtector`
- `IClipboardService`
- `IGlobalHotkeyService`
- `IWindowPlacementService`
- `IForegroundWindowService`

Interfaces remain small and use domain-oriented inputs and outputs rather than exposing SQLite, WPF, or Win32 types.

Logging uses `Microsoft.Extensions.Logging` abstractions directly in App and Infrastructure instead of introducing a project-specific logging interface.

### 5.4 Primary use cases

- `SearchSnippets`
- `ManageSelection`
- `ComposeClipboardText`
- `CopySelection`
- `ClearSensitiveClipboard`

Each use case has one entry point and one clear responsibility. UI event handlers translate user actions into use-case calls and render the returned state.

Opening, positioning, and closing the quick panel remain presentation workflows in `TextCaddy.App`; they are not Core use cases.

### 5.5 Application startup

```text
Process start
  → acquire the current-user single-instance lock
  → notify the existing instance through authenticated local IPC and exit when the lock is unavailable
  → initialize privacy-safe logging
  → create and validate application directories
  → back up and migrate the database
  → assemble services
  → register the configured global shortcut
  → create the tray icon
  → wait in the background
```

Database or hotkey failure must not produce an unhandled startup crash. The tray remains available whenever the application can safely offer recovery or exit actions.

### 5.6 Threading and Windows runtime model

- The executable entry point and WPF Dispatcher run in a single-threaded apartment.
- Clipboard, hotkey-message, tray, HWND, focus, and WPF operations execute on the UI Dispatcher thread.
- `WM_HOTKEY` is received through an `HwndSource` owned and disposed by the App lifecycle.
- SQLite operations use synchronous `Microsoft.Data.Sqlite` APIs on a serialized background data queue; the UI thread never waits synchronously for database I/O.
- Results that affect observable UI state are marshaled back to the WPF Dispatcher.
- Timers and IPC callbacks are cancellable. Shutdown unregisters the hotkey, stops timers and IPC, hides and disposes the tray icon, and then closes database resources.
- Event subscriptions with application lifetime are explicitly removed during disposal.

### 5.7 Single-instance and IPC model

- A named mutex scoped with the current Windows user SID prevents duplicate instances for that user.
- A named pipe using the same user scope carries a small versioned command such as `ShowQuickPanel` to the existing process.
- The pipe accepts only the current user and never accepts arbitrary serialized objects.
- The second process uses a bounded connection timeout. If the mutex exists but IPC repeatedly fails, it reports that the existing instance is unresponsive instead of starting a competing database writer.
- Mutex ownership is released automatically on process termination; no persistent lock file is used.

### 5.8 Tray, DPI, and focus adapters

- The MVP uses `System.Windows.Forms.NotifyIcon` in `TextCaddy.App`; Windows Forms is enabled only for that project.
- The application manifest declares Per-Monitor V2 DPI awareness.
- Pointer and monitor bounds from Win32 physical coordinates are converted explicitly to WPF device-independent units.
- The quick panel responds to DPI changes and recalculates placement before becoming visible.
- Focus restoration validates that the recorded HWND still exists, is visible, and is not owned by TextCaddy.
- TextCaddy checks the result of foreground activation. If Windows denies activation, it does not simulate input or loop aggressively; it leaves the copied content intact and uses a non-invasive attention signal when appropriate.

## 6. Domain model

### 6.1 Snippet

- `Id`: stable identifier.
- `GroupId`: owning group.
- `Title`: required display and search label.
- `Body`: ordinary plaintext or decrypted sensitive content at the domain boundary.
- `IsSensitive`: controls masking, protected persistence, and clipboard cleanup.
- `SortOrder`: stable order inside a group.
- `CreatedAtUtc` and `UpdatedAtUtc`: audit metadata.

Sensitive titles are not encrypted in the MVP. The UI warns users not to place secrets in titles.

Ordinary titles and bodies participate in incremental search. Sensitive entries participate by title only; repositories and search use cases must not decrypt every sensitive body to answer a query.

### 6.2 SnippetGroup

- `Id`: stable identifier.
- `Name`: required group label.
- `SortOrder`: stable group order.
- `CreatedAtUtc` and `UpdatedAtUtc`.

### 6.3 AppSettings

- Global shortcut modifiers and key.
- Composition separator.
- Sensitive clipboard-clear duration.
- Active group identifier.
- Optional launch-at-login preference when that feature is implemented.

### 6.4 QuickPanelSession

An in-memory object containing:

- current query;
- current input mode (`Selection` or `Search`);
- current page;
- highlighted snippet identifier;
- ordered selected snippet identifiers;
- captured destination window;
- transient message state.

It is never persisted.

## 7. Persistence and file management

### 7.1 Repository layout

```text
textcaddy/
├─ .github/
│  ├─ ISSUE_TEMPLATE/
│  ├─ workflows/ci.yml
│  └─ pull_request_template.md
├─ docs/
│  ├─ product/
│  │  ├─ product-requirements.md
│  │  └─ interaction-spec.md
│  ├─ architecture/
│  │  ├─ architecture.md
│  │  └─ data-storage.md
│  └─ superpowers/
│     ├─ specs/
│     └─ plans/
├─ src/
│  ├─ TextCaddy.App/
│  │  ├─ Bootstrap/
│  │  ├─ Views/
│  │  ├─ ViewModels/
│  │  ├─ Controls/
│  │  ├─ Converters/
│  │  ├─ Resources/
│  │  │  ├─ Styles/
│  │  │  ├─ Themes/
│  │  │  └─ Icons/
│  │  └─ Services/
│  ├─ TextCaddy.Core/
│  │  ├─ Models/
│  │  ├─ UseCases/
│  │  ├─ Interfaces/
│  │  ├─ Search/
│  │  ├─ Selection/
│  │  └─ Composition/
│  └─ TextCaddy.Infrastructure/
│     ├─ Data/Execution/
│     ├─ Data/Migrations/
│     ├─ Data/Repositories/
│     ├─ Security/
│     ├─ Clipboard/
│     ├─ Hotkeys/
│     ├─ SingleInstance/
│     ├─ Windows/
│     └─ Logging/
├─ tests/
│  ├─ TextCaddy.Core.Tests/
│  ├─ TextCaddy.Infrastructure.Tests/
│  └─ TextCaddy.App.Tests/
├─ Directory.Build.props
├─ Directory.Packages.props
├─ TextCaddy.slnx
├─ .editorconfig
├─ CONTRIBUTING.md
├─ SECURITY.md
├─ LICENSE
└─ README.md
```

Directories are added when their first real file is introduced; empty scaffolding directories are not committed.

### 7.2 Source organization rules

- A source file normally contains one public type.
- The file name matches its primary type.
- Views do not query repositories or call Windows APIs directly.
- Core does not reference WPF, SQLite, or Win32.
- Infrastructure does not reference App.
- UI strings, icons, themes, and control styles are centralized under `Resources`.
- Package versions are centralized in `Directory.Packages.props`.
- Shared compiler, nullable, analyzer, and warning policies live in `Directory.Build.props`.
- Test namespaces and folders mirror the production code they verify.
- Build output, published binaries, databases, logs, backups, and user settings are ignored by Git.

### 7.3 User-data layout

```text
%LocalAppData%\TextCaddy\
├─ data/
│  ├─ textcaddy.db
│  └─ backups/
├─ logs/
└─ cache/
```

- `data` contains durable application state.
- `backups` contains pre-migration database copies with bounded retention.
- `logs` contains rotating privacy-safe diagnostic files.
- `cache` is disposable and must never be required for recovery.
- MVP settings are stored in SQLite to avoid conflicting sources of truth.

### 7.4 SQLite schema responsibilities

The database owns:

- schema version;
- snippet groups;
- snippets;
- application settings.

Sensitive snippet bodies are stored as protected binary values. Ordinary bodies remain searchable plaintext. Repository implementations return domain objects and hide storage representation.

Database invariants include foreign keys between snippets and groups, unique stable identifiers, non-null trimmed titles and names, and explicit sort-order indexes. `PRAGMA foreign_keys = ON` is applied to every connection. WAL mode and a bounded busy timeout are configured during database initialization.

Database work is serialized through the Infrastructure data queue. The implementation uses synchronous SQLite calls on that queue because SQLite does not provide true asynchronous I/O; repository APIs may return tasks to callers without running database calls on the UI thread.

### 7.5 Migration and backup policy

- Migrations are monotonic and transactional where SQLite permits.
- Before a schema migration, TextCaddy opens the database in the startup migration phase and creates a consistent timestamped backup through `SqliteConnection.BackupDatabase`; no normal repository writer is active yet.
- The application retains the five newest automatic migration backups.
- A migration operates on the original only after the backup succeeds. A failed migration rolls back its transaction where possible, stops normal startup, and preserves the pre-migration backup.
- Destructive downgrade migrations are not supported.
- Automatic backups containing DPAPI-protected bodies are recoverable only under the same Windows user profile. They are disaster-recovery copies, not portable exports.

## 8. Security and privacy

### 8.1 Privacy guarantees

- No telemetry.
- No cloud service or account.
- No clipboard-history monitoring.
- Sensitive clipboard writes are marked as ineligible for Windows clipboard history and cross-device roaming.
- No user snippet content in logs.
- No search query content in logs.
- No sensitive values in exception messages written to disk.

### 8.2 Sensitive storage

Sensitive snippet bodies are protected with Windows DPAPI in current-user scope. This protects a copied database from straightforward offline inspection but does not protect against software already executing as the same Windows user. DPAPI ciphertext and its automatic backups are tied to the Windows user profile and are not a portable recovery format.

TextCaddy must continue to state that it is not a password manager.

### 8.3 Memory handling

- Sensitive bodies are decrypted only when needed for explicit display, editing, or copying. Incremental search never decrypts them.
- View models do not retain decrypted sensitive content longer than the active operation requires.
- Sensitive content is masked by default in list and detail views.
- The design does not claim guaranteed secure memory erasure under managed .NET runtime behavior.

### 8.4 Logging

Logs may include timestamps, operation categories, exception types, and sanitized technical messages. They must not include snippet titles, bodies, queries, composed clipboard text, encryption material, or raw database records.

## 9. Error handling and recovery

- **Hotkey conflict:** keep the tray application running, notify the user, and offer settings or exit.
- **Clipboard contention:** perform a small bounded retry; on failure keep the panel open and show a non-blocking error.
- **Database creation failure:** keep the original error context in a sanitized log and present the affected path.
- **Database corruption:** stop normal startup, preserve the original file, and present recovery guidance plus the backup directory. Do not pretend a corrupt database is safely readable.
- **Database write failure with readable data:** keep existing snippets available for copying, disable editing, label the session as read-only, and expose the data and backup paths.
- **Migration failure:** preserve the original database and its pre-migration backup; do not continue with a partially migrated schema.
- **Second process:** signal the existing process to show its panel, then exit.
- **Unhandled UI error:** show a safe recovery message and write a content-free diagnostic record.
- **Focus restoration failure:** close the panel safely and leave clipboard behavior intact; never simulate arbitrary input to regain focus. Report failure only through a non-invasive attention signal or later diagnostic surface.

## 10. Performance and quality targets

Performance is part of the product behavior, not a later optimization. Release measurements use a documented Windows 11 x64 reference machine and a library containing 10,000 ordinary snippets distributed across groups.

- Warm global-shortcut invocation to an interactive panel targets a 95th-percentile latency of 150 ms or less.
- Updating visible search results after an input change targets 50 ms or less.
- No UI-thread operation should block input for more than 100 ms during ordinary use.
- Database writes and backup work never run synchronously on the UI thread.
- Performance measurements are recorded in release notes or test artifacts so regressions can be compared on the same reference environment.

Accessibility acceptance includes complete keyboard operation, visible focus, text exposure of the current input mode, usable Windows high-contrast rendering, and accessible names for interactive controls.

## 11. Testing strategy

### 11.1 Core tests

- Title and body search.
- Case-insensitive matching.
- Sensitive-title search without sensitive-body decryption.
- Stable ordering and page boundaries.
- `1` through `9` and `0` item mapping.
- Selection-mode and search-mode transitions, including numeric search input.
- Selection, deselection, reselection, and retained order.
- Selection persistence across search and page changes.
- Default and custom separator composition.
- Empty-selection fallback to the highlighted item.
- Validation of groups, snippets, and settings.

### 11.2 Infrastructure tests

- Database creation and schema migration using disposable directories.
- Ordinary and sensitive snippet round trips.
- DPAPI protect/unprotect behavior under the current Windows user.
- Settings persistence.
- Backup retention and failed-migration preservation.
- Sensitive clipboard options disable history and roaming.
- Clipboard sequence comparison does not clear content after any later clipboard change, including an identical-text copy.
- Per-user single-instance naming and bounded IPC behavior.

### 11.3 App tests

- Quick-panel view-model state transitions.
- Toggle, cancel, copy-and-close, and copy-and-stay-open orchestration.
- Empty-result and transient-message behavior.
- Hotkey-registration failure state.
- Hotkey replacement preserves the previous registration when the candidate conflicts.
- Startup decisions that can be isolated from the actual desktop.
- Management-window unsaved-change, deletion, and non-empty-group rules.

### 11.4 Manual Windows smoke tests

- Global shortcut registration and conflict handling.
- Single-instance activation.
- Pointer-adjacent placement on every connected monitor.
- Mixed-DPI monitor transitions.
- Work-area clamping near every screen edge.
- Selection-mode keyboard focus on open, followed by explicit search-field focus after `/` or `Ctrl+F`.
- Destination focus restoration on close.
- Tray lifecycle and clean exit.
- Clipboard contention behavior.
- Sensitive values do not appear in Windows clipboard history and are not marked roamable.
- Search-mode IME input, numeric queries, and caret movement.
- Screen-reader names, keyboard focus visibility, and Windows high-contrast rendering.

### 11.5 Continuous integration

GitHub Actions runs on a Windows runner for every pull request and performs package restore, build, and non-interactive automated tests. Tests that require an interactive desktop, real global hotkeys, foreground activation, a visible tray, or the system clipboard are explicitly categorized and excluded from hosted CI; they run in the documented local smoke suite or a future interactive self-hosted runner. Automatic installer publication is outside the first implementation slice.

## 12. First implementation slice

The first vertical slice proves the smallest complete product loop. It deliberately supports only ordinary snippets in one automatically created default group; search, sensitive storage, custom separators, and full group management follow in later slices.

1. Create the solution, projects, centralized build properties, and core tests.
2. Create the initial SQLite schema and automatically create one default group.
3. Add a minimal management window that can create, edit, list, and delete ordinary snippets in that group.
4. Implement dependency assembly, STA application startup, a current-user single instance, a tray icon, and clean exit.
5. Register `Ctrl+Alt+Space` with `MOD_NOREPEAT` through the hotkey adapter.
6. Open a quick panel near the pointer and load the first ten persisted snippets from the default group.
7. Support digit-based multi-selection, visible selection order, and newline composition.
8. Implement `Ctrl+C` copy-and-close, `Ctrl+Shift+C` copy-and-stay-open, `Esc` cancellation, and destination-focus restoration.
9. Clamp the panel to the current Per-Monitor V2 work area.
10. Add Windows CI for restore, build, and automated tests.

### 12.1 Acceptance criteria

- `dotnet build` completes without errors.
- All automated tests pass.
- The application is single-instance and remains available through the system tray.
- A snippet created in the management window remains available after an application restart.
- `Ctrl+Alt+Space` opens and closes the quick panel.
- The quick panel displays persisted snippets from the default group rather than an empty shell.
- Digits select and deselect visible entries, and the UI exposes their composition order.
- `Ctrl+C` places the exact newline-composed result on the clipboard and closes the panel.
- `Ctrl+Shift+C` places the same result on the clipboard and keeps the panel open.
- `Esc` closes the quick panel.
- The panel remains inside the current monitor's work area.
- Closing restores focus to the previous application where Windows permits it.
- A hotkey conflict does not crash the application and the tray exit command remains available.
- Logs contain no user snippet or clipboard content.
- The end-to-end loop from saving a snippet to copying it from the quick panel passes a documented manual smoke test.
- README development status remains truthful.

### 12.2 Subsequent slices

1. **Search and navigation:** explicit selection/search modes, IME and numeric queries, paging, performance measurements, and retained selection summaries.
2. **Organization:** full group management, moving and reordering snippets, settings persistence, and configurable separators.
3. **Sensitive content:** DPAPI persistence, masking, title-only search, history/roaming exclusion, sequence-based timed clearing, and security tests.
4. **Recovery and polish:** migration backups, read-only failure mode, accessibility verification, diagnostics, packaging, and release documentation.

## 13. Deferred decisions

The following decisions are intentionally deferred until their implementation slice:

- Installer technology and Microsoft Store packaging.
- Launch-at-login mechanism.
- ARM64 artifacts.
- Import and export format.
- Full application localization beyond the bilingual project documentation.
- Theme customization beyond accessible light and dark defaults.
