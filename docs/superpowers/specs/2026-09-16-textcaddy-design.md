# TextCaddy MVP Design

**Date:** 2026-09-16

**Status:** Approved in conversation; awaiting final document review

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
- **Unobtrusive:** the quick panel appears near the pointer and returns focus to the previous application.
- **Local:** the MVP has no account, cloud sync, telemetry, or network dependency.

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
5. The search box receives keyboard focus.
6. The user searches, navigates, and selects snippets.
7. Copy-and-close returns focus to the recorded window; cancellation changes neither the clipboard nor stored data.

The provisional default shortcut is `Ctrl+Alt+Space`. `Alt+Space` is not used because Windows reserves it for the window system menu. Users can change the shortcut in settings.

### 4.3 Closing and cancellation

- Pressing the global shortcut while the panel is open closes it.
- `Esc` closes the panel.
- Clicking outside the panel closes it.
- Cancellation clears the in-memory panel session and does not change the clipboard.

### 4.4 Search, navigation, and paging

- With no query, the panel shows the active group's ordered snippets.
- A query matches snippet titles and bodies without case sensitivity.
- Up and down arrows move the highlighted item.
- Left and right arrows change pages.
- Each page contains at most ten items.
- Digits `1` through `9` address items one through nine; `0` addresses item ten.
- Search state lives only for the current panel session.

### 4.5 Selection model

- A digit toggles the selection of its corresponding visible item.
- Multiple selected items are retained in the order in which the user selected them.
- Deselecting an item removes it from the selection sequence.
- Selecting it again appends it to the end of the sequence.
- Selected items remain selected if a later query or page change makes them temporarily invisible.
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

1. TextCaddy records a fingerprint of the exact clipboard value it wrote.
2. A configurable timer starts; the default is 30 seconds.
3. At expiry, TextCaddy reads the current clipboard value.
4. It clears the clipboard only if the value still matches the recorded fingerprint.

This prevents the cleanup task from deleting newer content copied by the user.

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

Owns WPF views, view models, UI resources, application startup, dependency assembly, tray behavior, quick-panel sessions, and management-window navigation.

It may depend on Core and Infrastructure. It must not contain SQL, cryptography, raw clipboard access, or Win32 interop beyond UI-specific window hooks delegated to a service.

#### TextCaddy.Core

Owns domain models, use cases, business rules, and service contracts. It has no dependency on WPF, SQLite, Windows Forms, Win32, or concrete infrastructure packages.

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
- `IAppLogger`

Interfaces remain small and use domain-oriented inputs and outputs rather than exposing SQLite, WPF, or Win32 types.

### 5.4 Primary use cases

- `SearchSnippets`
- `ManageSelection`
- `ComposeClipboardText`
- `ShowQuickPanel`
- `CopySelection`
- `ClearSensitiveClipboard`

Each use case has one entry point and one clear responsibility. UI event handlers translate user actions into use-case calls and render the returned state.

### 5.5 Application startup

```text
Process start
  → acquire the single-instance lock
  → notify the existing instance and exit when the lock is unavailable
  → initialize privacy-safe logging
  → create and validate application directories
  → back up and migrate the database
  → assemble services
  → register the configured global shortcut
  → create the tray icon
  → wait in the background
```

Database or hotkey failure must not produce an unhandled startup crash. The tray remains available whenever the application can safely offer recovery or exit actions.

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
│     ├─ Data/Migrations/
│     ├─ Data/Repositories/
│     ├─ Security/
│     ├─ Clipboard/
│     ├─ Hotkeys/
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

### 7.5 Migration and backup policy

- Migrations are monotonic and transactional where SQLite permits.
- Before a schema migration, TextCaddy creates a timestamped database backup.
- The application retains the five newest automatic migration backups.
- A failed migration leaves the original database and backup intact.
- Destructive downgrade migrations are not supported.

## 8. Security and privacy

### 8.1 Privacy guarantees

- No telemetry.
- No cloud service or account.
- No clipboard-history monitoring.
- No user snippet content in logs.
- No search query content in logs.
- No sensitive values in exception messages written to disk.

### 8.2 Sensitive storage

Sensitive snippet bodies are protected with Windows DPAPI in current-user scope. This protects a copied database from straightforward offline inspection but does not protect against malicious software already executing as the same Windows user.

TextCaddy must continue to state that it is not a password manager.

### 8.3 Memory handling

- Sensitive bodies are decrypted only when needed for display, search, editing, or copying.
- View models do not retain decrypted sensitive content longer than the active operation requires.
- Sensitive content is masked by default in list and detail views.
- The design does not claim guaranteed secure memory erasure under managed .NET runtime behavior.

### 8.4 Logging

Logs may include timestamps, operation categories, exception types, and sanitized technical messages. They must not include snippet titles, bodies, queries, composed clipboard text, encryption material, or raw database records.

## 9. Error handling and recovery

- **Hotkey conflict:** keep the tray application running, notify the user, and offer settings or exit.
- **Clipboard contention:** perform a small bounded retry; on failure keep the panel open and show a non-blocking error.
- **Database creation failure:** keep the original error context in a sanitized log and present the affected path.
- **Database corruption or write failure:** do not overwrite the original file; enter a safe read-only state when possible and expose the backup location.
- **Migration failure:** preserve the original database and its pre-migration backup; do not continue with a partially migrated schema.
- **Second process:** signal the existing process to show its panel, then exit.
- **Unhandled UI error:** show a safe recovery message and write a content-free diagnostic record.
- **Focus restoration failure:** close the panel safely and leave clipboard behavior intact; never simulate arbitrary input to regain focus.

## 10. Testing strategy

### 10.1 Core tests

- Title and body search.
- Case-insensitive matching.
- Stable ordering and page boundaries.
- `1` through `9` and `0` item mapping.
- Selection, deselection, reselection, and retained order.
- Selection persistence across search and page changes.
- Default and custom separator composition.
- Empty-selection fallback to the highlighted item.
- Validation of groups, snippets, and settings.

### 10.2 Infrastructure tests

- Database creation and schema migration using disposable directories.
- Ordinary and sensitive snippet round trips.
- DPAPI protect/unprotect behavior under the current Windows user.
- Settings persistence.
- Backup retention and failed-migration preservation.
- Clipboard fingerprint comparison logic without destructive interaction with unrelated clipboard content.

### 10.3 App tests

- Quick-panel view-model state transitions.
- Toggle, cancel, copy-and-close, and copy-and-stay-open orchestration.
- Empty-result and transient-message behavior.
- Hotkey-registration failure state.
- Startup decisions that can be isolated from the actual desktop.

### 10.4 Manual Windows smoke tests

- Global shortcut registration and conflict handling.
- Single-instance activation.
- Pointer-adjacent placement on every connected monitor.
- Mixed-DPI monitor transitions.
- Work-area clamping near every screen edge.
- Keyboard focus on open.
- Destination focus restoration on close.
- Tray lifecycle and clean exit.
- Clipboard contention behavior.

### 10.5 Continuous integration

GitHub Actions runs on a Windows runner for every pull request and performs package restore, build, and automated tests. Automatic installer publication is outside the first implementation slice.

## 11. First implementation slice

The first vertical slice establishes the product skeleton without premature snippet management features:

1. Create the solution, projects, centralized build properties, and tests.
2. Implement dependency assembly and application startup.
3. Enforce a single running instance.
4. Create the tray icon and exit command.
5. Register `Ctrl+Alt+Space` through an isolated hotkey service.
6. Display an empty quick panel near the pointer.
7. Clamp the panel to the current monitor work area.
8. Toggle the panel with the shortcut and close it with `Esc`.
9. Restore the previously active window when closing.
10. Add CI for restore, build, and tests.

### 11.1 Acceptance criteria

- `dotnet build` completes without errors.
- All automated tests pass.
- The application is single-instance and remains available through the system tray.
- `Ctrl+Alt+Space` opens and closes the quick panel.
- `Esc` closes the quick panel.
- The panel remains inside the current monitor's work area.
- Closing restores focus to the previous application where Windows permits it.
- A hotkey conflict does not crash the application and the tray exit command remains available.
- Logs contain no user snippet or clipboard content.
- README development status remains truthful.

## 12. Deferred decisions

The following decisions are intentionally deferred until their implementation slice:

- Installer technology and Microsoft Store packaging.
- Launch-at-login mechanism.
- ARM64 artifacts.
- Import and export format.
- Full application localization beyond the bilingual project documentation.
- Theme customization beyond accessible light and dark defaults.
