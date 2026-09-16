# TextCaddy MVP Foundation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Deliver the first usable TextCaddy loop on Windows 11 x64: save ordinary snippets, summon them near the pointer, select them with digits, copy their newline-composed text, and return to the destination.

**Architecture:** A WPF composition root depends on an application Core and Windows/SQLite Infrastructure. Core contains domain rules and outbound ports, Infrastructure serializes SQLite work off the UI thread and adapts Win32, while App owns WPF lifecycle and presentation.

**Tech Stack:** C# 14, .NET 10, WPF, `Microsoft.Data.Sqlite` 10.0.12, `Microsoft.Extensions.DependencyInjection` 10.0.12, xUnit v3, Win32 P/Invoke, `System.Windows.Forms.NotifyIcon`, GitHub Actions.

**Spec:** `docs/superpowers/specs/2026-09-16-textcaddy-design.md`

## Global Constraints

- Official MVP target: Windows 11 x64; Windows 10 is best effort only.
- Target framework: `net10.0-windows`; nullable reference types and implicit usings are enabled; warnings are treated as errors.
- The first slice supports ordinary snippets in one automatically created default group only.
- Core must not reference WPF, Windows Forms, Win32, or SQLite.
- SQLite work uses synchronous provider calls on a serialized background executor; the WPF Dispatcher never blocks on database I/O.
- Clipboard, hotkey, tray, HWND, focus, and WPF operations run on the UI STA thread.
- No clipboard-history monitoring, telemetry, network dependency, or automatic paste.
- Logs and exceptions written to disk must never contain snippet titles, bodies, search queries, or clipboard text.
- Use test-driven development for business and persistence behavior; real desktop integration is verified by the documented manual smoke test.
- The machine currently has no discoverable `dotnet` executable. Install the .NET 10 SDK before Task 1 and verify it with `dotnet --info`.
- This plan implements design section 12.1 only. Search, full organization, sensitive content, and recovery/polish are intentionally assigned to separate plans by design section 12.2.

## Planned file map

```text
TextCaddy.slnx                         Solution entry point
Directory.Build.props                  Shared compiler and analyzer policy
Directory.Packages.props               Central NuGet versions
.editorconfig                          Repository formatting policy
.gitignore                             Build and local-data exclusions
src/TextCaddy.Core/
  Models/Snippet.cs                    Snippet domain record
  Models/SnippetGroup.cs               Group domain record
  Interfaces/ISnippetRepository.cs     Snippet persistence port
  Interfaces/IGroupRepository.cs       Group persistence port
  Interfaces/IClipboardService.cs      Clipboard port
  Selection/SelectionSession.cs        Ordered toggle selection
  Composition/SnippetComposer.cs       Newline composition rule
src/TextCaddy.Infrastructure/
  Data/SqliteOptions.cs                Database location
  Data/ISqliteExecutor.cs              Serialized database executor contract
  Data/SqliteExecutor.cs               Background synchronous SQLite execution
  Data/SqliteDatabase.cs               Schema v1 and default-group initialization
  Data/Repositories/SqliteGroupRepository.cs
  Data/Repositories/SqliteSnippetRepository.cs
  SingleInstance/SingleInstanceNames.cs
  SingleInstance/SingleInstanceCoordinator.cs
  Hotkeys/GlobalHotkeyService.cs
  Windows/WindowPlacement.cs
  Windows/WindowPlacementService.cs
  Windows/ForegroundWindowService.cs
  Clipboard/WindowsClipboardService.cs
src/TextCaddy.App/
  Bootstrap/ServiceRegistration.cs      DI registrations
  Services/TrayIconService.cs           NotifyIcon lifecycle
  ViewModels/ObservableObject.cs
  ViewModels/RelayCommand.cs
  ViewModels/AsyncRelayCommand.cs
  ViewModels/ManagementViewModel.cs
  ViewModels/QuickPanelViewModel.cs
  Views/ManagementWindow.xaml(.cs)
  Views/QuickPanelWindow.xaml(.cs)
  App.xaml(.cs)                         STA lifecycle and composition root
  app.manifest                          Per-Monitor V2 declaration
tests/TextCaddy.Core.Tests/             Pure domain tests
tests/TextCaddy.Infrastructure.Tests/   Temp-database and pure adapter tests
tests/TextCaddy.App.Tests/              View-model tests with fakes
.github/workflows/ci.yml                Windows restore/build/test
docs/testing/manual-smoke-test.md       Interactive Windows verification
```

---

### Task 1: Toolchain and buildable solution foundation

**Files:**
- Create: `TextCaddy.slnx`
- Create: `Directory.Build.props`
- Create: `Directory.Packages.props`
- Create: `.editorconfig`
- Create: `.gitignore`
- Create: `src/TextCaddy.Core/TextCaddy.Core.csproj`
- Create: `src/TextCaddy.Infrastructure/TextCaddy.Infrastructure.csproj`
- Create: `src/TextCaddy.App/TextCaddy.App.csproj`
- Create: `tests/TextCaddy.Core.Tests/TextCaddy.Core.Tests.csproj`
- Create: `tests/TextCaddy.Infrastructure.Tests/TextCaddy.Infrastructure.Tests.csproj`
- Create: `tests/TextCaddy.App.Tests/TextCaddy.App.Tests.csproj`

**Interfaces:**
- Produces: a restorable `TextCaddy.slnx` with `App -> Core + Infrastructure`, `Infrastructure -> Core`, and test-project references.

- [ ] **Step 1: Install and verify the .NET 10 SDK**

Run in an elevated-capable user session:

```powershell
winget install Microsoft.DotNet.SDK.10 --accept-package-agreements --accept-source-agreements
dotnet --info
```

Expected: the SDK list contains a stable `10.0.x` entry and the command exits successfully.

- [ ] **Step 2: Scaffold the solution and projects**

```powershell
dotnet new sln --name TextCaddy --format slnx
dotnet new classlib --name TextCaddy.Core --output src/TextCaddy.Core --framework net10.0
dotnet new classlib --name TextCaddy.Infrastructure --output src/TextCaddy.Infrastructure --framework net10.0
dotnet new wpf --name TextCaddy.App --output src/TextCaddy.App --framework net10.0
dotnet new xunit --name TextCaddy.Core.Tests --output tests/TextCaddy.Core.Tests --framework net10.0
dotnet new xunit --name TextCaddy.Infrastructure.Tests --output tests/TextCaddy.Infrastructure.Tests --framework net10.0
dotnet new xunit --name TextCaddy.App.Tests --output tests/TextCaddy.App.Tests --framework net10.0
dotnet sln TextCaddy.slnx add src/TextCaddy.Core src/TextCaddy.Infrastructure src/TextCaddy.App tests/TextCaddy.Core.Tests tests/TextCaddy.Infrastructure.Tests tests/TextCaddy.App.Tests
```

- [ ] **Step 3: Add project references and central package versions**

```powershell
dotnet add src/TextCaddy.Infrastructure reference src/TextCaddy.Core
dotnet add src/TextCaddy.App reference src/TextCaddy.Core src/TextCaddy.Infrastructure
dotnet add tests/TextCaddy.Core.Tests reference src/TextCaddy.Core
dotnet add tests/TextCaddy.Infrastructure.Tests reference src/TextCaddy.Core src/TextCaddy.Infrastructure
dotnet add tests/TextCaddy.App.Tests reference src/TextCaddy.Core src/TextCaddy.App
```

Create `Directory.Packages.props` with central versions for:

```xml
<Project>
  <PropertyGroup><ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally></PropertyGroup>
  <ItemGroup>
    <PackageVersion Include="Microsoft.Data.Sqlite" Version="10.0.12" />
    <PackageVersion Include="Microsoft.Extensions.DependencyInjection" Version="10.0.12" />
    <PackageVersion Include="Microsoft.NET.Test.Sdk" Version="18.10.0" />
    <PackageVersion Include="xunit.v3" Version="4.0.1" />
    <PackageVersion Include="xunit.runner.visualstudio" Version="4.0.0" />
  </ItemGroup>
</Project>
```

Set shared build properties in `Directory.Build.props`:

```xml
<Project>
  <PropertyGroup>
    <LangVersion>14.0</LangVersion>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
    <Deterministic>true</Deterministic>
  </PropertyGroup>
</Project>
```

Add `Microsoft.Data.Sqlite` to Infrastructure and `Microsoft.Extensions.DependencyInjection` to App. Each test project references `Microsoft.NET.Test.Sdk`, `xunit.v3`, and `xunit.runner.visualstudio` without inline versions. Set Core to `net10.0`; set Infrastructure, App, Infrastructure.Tests, and App.Tests to `net10.0-windows`. Set `UseWPF`, `UseWindowsForms`, and `OutputType=WinExe` only in App.

- [ ] **Step 4: Remove template placeholder source and verify the baseline**

Run:

```powershell
dotnet restore TextCaddy.slnx
dotnet build TextCaddy.slnx --configuration Release --no-restore
dotnet test TextCaddy.slnx --configuration Release --no-build
```

Expected: restore, build, and all template tests succeed with zero warnings.

- [ ] **Step 5: Commit the foundation**

```powershell
git add TextCaddy.slnx Directory.Build.props Directory.Packages.props .editorconfig .gitignore src tests
git commit -m "build: scaffold TextCaddy solution"
```

---

### Task 2: Core snippet selection and composition

**Files:**
- Create: `src/TextCaddy.Core/Models/Snippet.cs`
- Create: `src/TextCaddy.Core/Models/SnippetGroup.cs`
- Create: `src/TextCaddy.Core/Selection/SelectionSession.cs`
- Create: `src/TextCaddy.Core/Composition/SnippetComposer.cs`
- Test: `tests/TextCaddy.Core.Tests/Selection/SelectionSessionTests.cs`
- Test: `tests/TextCaddy.Core.Tests/Composition/SnippetComposerTests.cs`

**Interfaces:**
- Produces: `Snippet.Create(string title, string body, int sortOrder, Guid groupId, bool isSensitive = false)`, `SelectionSession.Toggle(Guid)`, `SelectionSession.Clear()`, `IReadOnlyList<Guid> SelectionSession.OrderedIds`, and `SnippetComposer.Compose(IEnumerable<Snippet>, string)`.

- [ ] **Step 1: Write failing ordered-selection tests**

```csharp
[Fact]
public void Toggle_deselects_and_reselection_moves_item_to_end()
{
    var first = Guid.NewGuid();
    var second = Guid.NewGuid();
    var session = new SelectionSession();
    session.Toggle(first);
    session.Toggle(second);
    session.Toggle(first);
    session.Toggle(first);
    Assert.Equal([second, first], session.OrderedIds);
}
```

Run `dotnet test tests/TextCaddy.Core.Tests --filter SelectionSessionTests`; expected: FAIL because the type does not exist.

- [ ] **Step 2: Implement the minimal ordered selection**

Use a `List<Guid>` plus `HashSet<Guid>` so membership checks are constant-time while order remains explicit. `Toggle` removes an existing ID or appends a new ID; `Clear` empties both collections.

- [ ] **Step 3: Write failing composition tests**

```csharp
[Fact]
public void Compose_preserves_selection_order_and_separator()
{
    var groupId = Guid.NewGuid();
    var snippets = new[]
    {
        Snippet.Create("Second", "B", 1, groupId),
        Snippet.Create("First", "A", 0, groupId)
    };
    Assert.Equal("B\r\nA", SnippetComposer.Compose(snippets, "\r\n"));
}
```

Run `dotnet test tests/TextCaddy.Core.Tests --filter SnippetComposerTests`; expected: FAIL.

- [ ] **Step 4: Implement and verify Core behavior**

`Snippet.Create` generates an ID and UTC timestamps. `SnippetComposer.Compose` validates a non-null sequence and separator and joins bodies without trimming or rewriting user content.

Run:

```powershell
dotnet test tests/TextCaddy.Core.Tests --configuration Release
```

Expected: all Core tests pass.

- [ ] **Step 5: Commit Core behavior**

```powershell
git add src/TextCaddy.Core tests/TextCaddy.Core.Tests
git commit -m "feat(core): add ordered snippet composition"
```

---

### Task 3: SQLite schema, default group, and snippet CRUD

**Files:**
- Create: `src/TextCaddy.Core/Interfaces/IGroupRepository.cs`
- Create: `src/TextCaddy.Core/Interfaces/ISnippetRepository.cs`
- Create: `src/TextCaddy.Infrastructure/Data/SqliteOptions.cs`
- Create: `src/TextCaddy.Infrastructure/Data/ISqliteExecutor.cs`
- Create: `src/TextCaddy.Infrastructure/Data/SqliteExecutor.cs`
- Create: `src/TextCaddy.Infrastructure/Data/SqliteDatabase.cs`
- Create: `src/TextCaddy.Infrastructure/Data/Repositories/SqliteGroupRepository.cs`
- Create: `src/TextCaddy.Infrastructure/Data/Repositories/SqliteSnippetRepository.cs`
- Test: `tests/TextCaddy.Infrastructure.Tests/Data/SqliteDatabaseTests.cs`
- Test: `tests/TextCaddy.Infrastructure.Tests/Data/SqliteSnippetRepositoryTests.cs`

**Interfaces:**
- Produces: `Task<SnippetGroup> IGroupRepository.GetOrCreateDefaultAsync(CancellationToken)`, `Task<IReadOnlyList<Snippet>> ISnippetRepository.ListByGroupAsync(Guid, CancellationToken)`, `Task AddAsync(Snippet, CancellationToken)`, `Task<bool> UpdateAsync(Snippet, CancellationToken)`, and `Task<bool> DeleteAsync(Guid, CancellationToken)`.
- Consumes: `Snippet` and `SnippetGroup` from Task 2.

- [ ] **Step 1: Write a failing schema/default-group integration test**

Create a unique temporary directory per test, initialize `SqliteDatabase`, then query through `SqliteGroupRepository`:

```csharp
[Fact]
public async Task Initialize_creates_exactly_one_default_group()
{
    await using var fixture = await SqliteFixture.CreateAsync();
    var group = await fixture.Groups.GetOrCreateDefaultAsync(TestContext.Current.CancellationToken);
    Assert.Equal("Default", group.Name);
    Assert.Equal(group.Id, (await fixture.Groups.GetOrCreateDefaultAsync(default)).Id);
}
```

Run the test; expected: FAIL because database types do not exist.

- [ ] **Step 2: Implement schema v1 and serialized executor**

Schema v1 contains `schema_info`, `snippet_groups`, and `snippets`. Enforce `FOREIGN KEY(group_id)`, `NOT NULL` text fields, and an index on `(group_id, sort_order)`. Configure foreign keys, WAL, and a 2-second busy timeout. `SqliteExecutor.ExecuteAsync<T>` uses a `SemaphoreSlim`, `Task.Run`, and a new connection per operation; it never exposes a connection outside the delegate.

- [ ] **Step 3: Write failing CRUD round-trip tests**

Cover add/list/update/delete, sort order, UTC timestamps, and a missing-ID update that returns `false` instead of inserting.

```csharp
var added = Snippet.Create("Address", "1 Example Road", 0, group.Id);
await repository.AddAsync(added, cancellationToken);
Assert.Equal(added, Assert.Single(await repository.ListByGroupAsync(group.Id, cancellationToken)));
```

- [ ] **Step 4: Implement repositories and verify persistence**

Use parameterized SQL only. Store GUIDs as canonical strings and timestamps as round-trip UTC strings. Do not log values. Run all Infrastructure tests twice to expose cleanup or file-lock leaks.

```powershell
dotnet test tests/TextCaddy.Infrastructure.Tests --configuration Release
dotnet test tests/TextCaddy.Infrastructure.Tests --configuration Release
```

- [ ] **Step 5: Commit storage**

```powershell
git add src/TextCaddy.Core/Interfaces src/TextCaddy.Infrastructure/Data tests/TextCaddy.Infrastructure.Tests/Data
git commit -m "feat(storage): persist snippets in SQLite"
```

---

### Task 4: Minimal management window and dependency composition

**Files:**
- Create: `src/TextCaddy.App/Bootstrap/ServiceRegistration.cs`
- Create: `src/TextCaddy.App/ViewModels/ObservableObject.cs`
- Create: `src/TextCaddy.App/ViewModels/RelayCommand.cs`
- Create: `src/TextCaddy.App/ViewModels/AsyncRelayCommand.cs`
- Create: `src/TextCaddy.App/ViewModels/ManagementViewModel.cs`
- Create: `src/TextCaddy.App/Views/ManagementWindow.xaml`
- Create: `src/TextCaddy.App/Views/ManagementWindow.xaml.cs`
- Modify: `src/TextCaddy.App/App.xaml`
- Modify: `src/TextCaddy.App/App.xaml.cs`
- Test: `tests/TextCaddy.App.Tests/ViewModels/ManagementViewModelTests.cs`

**Interfaces:**
- Consumes: repositories from Task 3.
- Produces: `ManagementViewModel.LoadAsync`, `SaveAsync`, `DeleteAsync`, and an explicit editable draft with title/body validation.

- [ ] **Step 1: Write failing view-model CRUD tests with in-memory fakes**

```csharp
[Fact]
public async Task Save_new_snippet_refreshes_list_and_clears_draft()
{
    var repository = new FakeSnippetRepository();
    var sut = ManagementViewModelTestFactory.Create(repository);
    sut.DraftTitle = "Email";
    sut.DraftBody = "hello@example.com";
    await sut.SaveAsync();
    Assert.Equal("Email", Assert.Single(sut.Snippets).Title);
    Assert.Equal(string.Empty, sut.DraftTitle);
}
```

Also test trimmed-empty validation and confirmed deletion.

- [ ] **Step 2: Implement focused MVVM primitives and view model**

Commands prevent concurrent execution. The view model exposes an error message rather than throwing validation failures through WPF bindings. It never calls `.Result`, `.Wait()`, or synchronous Dispatcher waits.

- [ ] **Step 3: Build the minimal management UI**

Use a two-column window: snippet list on the left; title, body, Save, New, and Delete controls on the right. Bind `AutomationProperties.Name`, keyboard focus order, and validation text. Do not add groups, search, themes, or sensitive controls in this slice.

- [ ] **Step 4: Register services and verify tests/build**

`ServiceRegistration` computes `%LocalAppData%\TextCaddy\data\textcaddy.db`, registers singleton database/executor services, repositories, and transient view models/windows.

Run App tests and the full build; expected: PASS with zero warnings.

- [ ] **Step 5: Commit management flow**

```powershell
git add src/TextCaddy.App tests/TextCaddy.App.Tests
git commit -m "feat(app): add basic snippet management"
```

---

### Task 5: Single-instance process and tray lifecycle

**Files:**
- Create: `src/TextCaddy.Infrastructure/SingleInstance/SingleInstanceNames.cs`
- Create: `src/TextCaddy.Infrastructure/SingleInstance/SingleInstanceCoordinator.cs`
- Create: `src/TextCaddy.App/Services/TrayIconService.cs`
- Modify: `src/TextCaddy.App/App.xaml.cs`
- Test: `tests/TextCaddy.Infrastructure.Tests/SingleInstance/SingleInstanceNamesTests.cs`

**Interfaces:**
- Produces: `SingleInstanceCoordinator.TryAcquireAsync`, `SendShowCommandAsync`, and `ShowRequested` event; `TrayIconService.Show()` and `Dispose()`.

- [ ] **Step 1: Test deterministic per-user names**

```csharp
[Fact]
public void ForUser_is_stable_and_contains_no_raw_sid_punctuation()
{
    var names = SingleInstanceNames.ForUser("S-1-5-21-123");
    Assert.Equal(names, SingleInstanceNames.ForUser("S-1-5-21-123"));
    Assert.DoesNotContain('-', names.MutexName);
}
```

- [ ] **Step 2: Implement mutex and named-pipe coordination**

Hash the SID into names, use current-user-only pipe options, a versioned one-line `SHOW 1` message, a 1-second client timeout, cancellation, and no arbitrary serialization. Start the pipe server only for the acquired owner.

- [ ] **Step 3: Implement and manually verify tray lifecycle**

Use `NotifyIcon` with Open, Show Quick Panel, and Exit items. Set `Visible=false` before disposal. Closing the management window hides it; only Exit terminates the app.

- [ ] **Step 4: Run tests and commit**

```powershell
dotnet test tests/TextCaddy.Infrastructure.Tests --filter SingleInstance --configuration Release
git add src/TextCaddy.Infrastructure/SingleInstance src/TextCaddy.App/Services src/TextCaddy.App/App.xaml.cs tests/TextCaddy.Infrastructure.Tests/SingleInstance
git commit -m "feat(app): add single-instance tray lifecycle"
```

---

### Task 6: Global hotkey, placement, and focus adapters

**Files:**
- Create: `src/TextCaddy.Core/Interfaces/IGlobalHotkeyService.cs`
- Create: `src/TextCaddy.Core/Interfaces/IWindowPlacementService.cs`
- Create: `src/TextCaddy.Core/Interfaces/IForegroundWindowService.cs`
- Create: `src/TextCaddy.Infrastructure/Hotkeys/GlobalHotkeyService.cs`
- Create: `src/TextCaddy.Infrastructure/Windows/WindowPlacement.cs`
- Create: `src/TextCaddy.Infrastructure/Windows/WindowPlacementService.cs`
- Create: `src/TextCaddy.Infrastructure/Windows/ForegroundWindowService.cs`
- Create: `src/TextCaddy.App/app.manifest`
- Test: `tests/TextCaddy.Infrastructure.Tests/Windows/WindowPlacementTests.cs`

**Interfaces:**
- Produces: hotkey registration with `MOD_CONTROL | MOD_ALT | MOD_NOREPEAT`, pure rectangle clamping, pointer-aware placement, foreground capture, and best-effort restore.

- [ ] **Step 1: Write failing pure placement tests**

Cover right/bottom overflow, negative-coordinate monitors, and a panel larger than the work area.

```csharp
var result = WindowPlacement.Clamp(new(1910, 1070), new(420, 500), new(0, 0, 1920, 1080));
Assert.Equal(new PixelPoint(1500, 580), result);
```

- [ ] **Step 2: Implement pure placement and verify tests**

Keep Win32 retrieval separate from pure clamping. Define pixel-coordinate value types in Infrastructure so no WPF types leak into Core.

- [ ] **Step 3: Implement UI-thread Win32 adapters**

Use an `HwndSource` to receive `WM_HOTKEY`; unregister and remove hooks on disposal. Capture `GetForegroundWindow` before showing TextCaddy. Before restore, validate `IsWindow`, visibility, and ownership; check `SetForegroundWindow` result and never synthesize input.

- [ ] **Step 4: Declare Per-Monitor V2 and verify manually**

Add `<dpiAwareness>PerMonitorV2</dpiAwareness>` in the manifest. Verify panel placement on the primary monitor and a negative-coordinate secondary monitor at two scale factors.

- [ ] **Step 5: Commit platform adapters**

```powershell
git add src/TextCaddy.Core/Interfaces src/TextCaddy.Infrastructure/Hotkeys src/TextCaddy.Infrastructure/Windows src/TextCaddy.App/app.manifest tests/TextCaddy.Infrastructure.Tests/Windows
git commit -m "feat(windows): add hotkey and panel placement adapters"
```

---

### Task 7: Quick panel and copy workflow

**Files:**
- Create: `src/TextCaddy.Core/Interfaces/IClipboardService.cs`
- Create: `src/TextCaddy.Infrastructure/Clipboard/WindowsClipboardService.cs`
- Create: `src/TextCaddy.App/ViewModels/QuickPanelViewModel.cs`
- Create: `src/TextCaddy.App/Views/QuickPanelWindow.xaml`
- Create: `src/TextCaddy.App/Views/QuickPanelWindow.xaml.cs`
- Modify: `src/TextCaddy.App/Bootstrap/ServiceRegistration.cs`
- Modify: `src/TextCaddy.App/App.xaml.cs`
- Modify: `src/TextCaddy.App/Services/TrayIconService.cs`
- Test: `tests/TextCaddy.App.Tests/ViewModels/QuickPanelViewModelTests.cs`

**Interfaces:**
- Consumes: repositories, selection/composition, hotkey, placement, focus, and clipboard services.
- Produces: the first complete save-to-copy product loop.

- [ ] **Step 1: Write failing quick-panel view-model tests**

Test initial first-ten load, digit mapping (`1` to index 0 and `0` to index 9), ordered multi-selection, deselection, copy-and-close, copy-and-stay-open, and empty-list behavior using fakes.

```csharp
await sut.LoadAsync();
sut.ToggleDigit(2);
sut.ToggleDigit(1);
await sut.CopyAsync(closeAfterCopy: true);
Assert.Equal("second\r\nfirst", clipboard.LastText);
Assert.True(sut.CloseRequested);
```

- [ ] **Step 2: Implement the view model and pass tests**

The first item is highlighted when available. With no explicit selection, copy the highlighted item. Expose ordered selected titles so hidden state never exists even in this simplified panel.

- [ ] **Step 3: Implement the accessible quick-panel window**

Create a borderless, size-to-content WPF window with ten numbered rows and a selection-order summary. Handle digit keys, `Ctrl+C`, `Ctrl+Shift+C`, and `Esc` in `PreviewKeyDown`. Do not implement search in this slice.

- [ ] **Step 4: Wire the complete lifecycle**

On hotkey or named-pipe Show command: capture the destination HWND, load snippets asynchronously, calculate placement, and show the panel. On close: clear the session, hide the panel, and request best-effort focus restoration. Copy uses `System.Windows.Clipboard.SetText` on the UI STA thread with three bounded retries for contention.

- [ ] **Step 5: Run automated and manual product-loop verification**

```powershell
dotnet test TextCaddy.slnx --configuration Release
dotnet build TextCaddy.slnx --configuration Release --no-restore
dotnet run --project src/TextCaddy.App
```

Manually create two snippets, restart, invoke the panel, select them in reverse order, copy, and paste into Notepad. Expected: the exact reverse-order newline composition appears and focus returns where Windows permits.

- [ ] **Step 6: Commit the vertical slice**

```powershell
git add src tests
git commit -m "feat: complete the first snippet copy workflow"
```

---

### Task 8: CI, manual verification guide, and truthful project status

**Files:**
- Create: `.github/workflows/ci.yml`
- Create: `docs/testing/manual-smoke-test.md`
- Modify: `README.md`

**Interfaces:**
- Consumes: the complete first slice.
- Produces: reproducible hosted checks and explicit interactive verification.

- [ ] **Step 1: Add Windows CI**

Use `actions/checkout`, `actions/setup-dotnet` with `10.0.x`, then run restore, Release build with `--no-restore`, and Release tests with `--no-build`. Do not run interactive desktop tests in hosted CI.

- [ ] **Step 2: Write the manual smoke matrix**

Document single-instance activation, tray exit, snippet persistence, hotkey conflict, primary and secondary monitor placement, 100%/150% DPI, digit order, both copy commands, `Esc`, focus restoration, clipboard contention, and confirmation that logs contain no user text.

- [ ] **Step 3: Update README status without overstating features**

Mark only the completed first-slice items as available. Keep search, groups, sensitive storage, custom separators, installer, and release download under planned features.

- [ ] **Step 4: Run final verification**

```powershell
dotnet restore TextCaddy.slnx
dotnet build TextCaddy.slnx --configuration Release --no-restore
dotnet test TextCaddy.slnx --configuration Release --no-build
git diff --check
git status --short
```

Expected: restore/build/test succeed, whitespace check is clean, and status lists only intentional documentation changes.

- [ ] **Step 5: Commit CI and documentation**

```powershell
git add .github/workflows/ci.yml docs/testing/manual-smoke-test.md README.md
git commit -m "ci: verify the TextCaddy MVP foundation"
```
