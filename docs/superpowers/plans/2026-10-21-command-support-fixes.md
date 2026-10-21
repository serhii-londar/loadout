# Command Support Review Fixes — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix the findings from the review of commit `bce9075` (“Offer commands only to assistants that load them”) so markdown commands are only offered to assistants that actually load them, legacy data is handled honestly, and the behavior is covered by regression tests.

**Architecture:** Keep the support decision in `Assistant.supportsCommands`, but make it a three-state rule: known-markdown assistants are supported, known non-markdown assistants are not, and only assistants of unknown format fall back to “a commands folder exists.” Filter the scanner and the enable path through the same rule, add explicit folder mappings where the conventional path is ambiguous, and update the user-facing copy/docs to match.

**Tech Stack:** Swift 6, SwiftUI, XCTest, `swift test`.

**Spec:** `spec/spec-flow-commands.md` (especially §5–§6, AC10.11–AC10.13), `CHANGELOG.md` (Unreleased > Added), and the review findings below.

## Global Constraints

- Do not change behavior for assistants that already work: Claude Code, Codex, Cursor, opencode and Factory must still be offered commands.
- A refused share must not create a directory or a file.
- Legacy links must remain removable: unshare/remove-copy/delete stay unguarded.
- Every production behavior change gets a test that failed first.
- Run `swift test` before claiming a task complete; the whole suite must stay green.
- Do not touch unrelated user work in the checkout.
- Keep the existing naming/copy style of the codebase.

## Findings

| ID | Severity | Finding | Status |
|---|---|---|---|
| F1 | Critical | The folder-existence fallback makes Gemini (TOML) and legacy Copilot folders “supported,” defeating the commit’s central promise. | Fixed |
| F2 | Important | The scanner still reports legacy markdown in Gemini/Copilot folders as loaded. | Fixed |
| F3 | Important | Enabling a parked command bypasses the support gate and can recreate an unsupported folder. | Fixed |
| F4 | Important | Codex can be pointed at `~/.codex/commands` instead of `~/.codex/prompts`. | Fixed |
| F5 | Important | `droid` is in the known set but is undocumented and may use the wrong folder. | Fixed |
| F6 | Important | Tests do not cover the promise: Gemini TOML, legacy Copilot, `copy`, `gaps`/sync-all, Codex root. | Fixed (core; app-level test deferred) |
| F7 | Minor | `fileExists` accepts a regular file named `commands` as a folder. | Fixed |
| F8 | Minor | Filesystem I/O runs from view/menu bodies on every render. | Deferred |
| F9 | Minor | The Assistants card can show “0 loaded of 0” with no explanation and stale help text. | Fixed |
| F10 | Minor | “Sync all” skips commands silently. | Fixed |
| F11 | Minor | Docs/tooltips still describe commands as Claude-only; CHANGELOG overpromises Gemini and omits Droid. | Fixed |
| F12 | Minor | Demo/automation callers pick assistants without checking command support. | Fixed |
| F13 | Minor | An unknown assistant with only a commands folder is not discovered, so “any assistant whose commands folder exists” is not literally true. | Fixed |

## Execution log

- 2026-10-21: F1–F7 and F9–F13 implemented on `fix/command-support-findings`.
  Added regression coverage for Gemini TOML, legacy Copilot, unsupported copy, Codex `prompts`,
  Droid, command-only discovery, a regular file named `commands`, and enabling a legacy parked
  copy. `swift test`: 446 tests, 0 failures. F8 (caching) remains deferred as a performance-only
  refactor; app-level `AppModel` tests remain unavailable in the current test target.

## Outcome

- Commits on `fix/command-support-findings`: `c137fa2` (core support rule, scanner, enable path,
  Codex/Droid, discovery, tests), `d67e4ba` (UI, sync status, demo/script filters), `7c98ac0`
  (changelog, docs, this plan), plus a final docs/UI polish commit.
- `swift build` succeeds; `swift test` passes with **446 tests, 0 failures**.
- F1–F7 and F9–F13 are fixed. F8 remains deferred; Antigravity’s command format and app-level
  sync-all tests are documented follow-ups.
- Final self-review of the diff found no remaining critical or important issues. A fresh reviewer
  subagent was dispatched twice, but the harness interrupted it before it returned a verdict; the
  final verification is the full test suite plus this self-review.

## Decisions

- **F1:** `AssistantRegistry.noMarkdownCommands = ["gemini", "copilot"]`. Known format wins over folder existence. Antigravity is Gemini-based but its command format is unconfirmed; leave it out of both sets and note it for a follow-up rather than guessing.
- **F5:** Remove `"droid"` from `commandAssistants`. The commit/CHANGELOG name Factory; `~/.droid/commands` is unconfirmed. Unknown `droid` still benefits from the folder fallback if it really has a markdown folder.
- **F2:** Live scanning skips assistants that do not support commands, so a legacy `.md` there is not claimed as loaded. Parked commands stay visible so a legacy file can still be deleted; enabling one is refused by F3. A live legacy link is hidden from the card, but deleting the item still removes every link because `delete` scans all command roots.
- **F8:** Deferred. Caching folder state on `Assistant` is a worthwhile performance refactor, but it changes the discovery/initializer surface and is not a correctness bug. The doc keeps it visible for a follow-up.
- **F6:** App-level `AppModel` tests are not available in the current test target; core predicate/scanner/mutation tests cover the logic. The sync-all status line is verified by build and manual reasoning, not a unit test.

## Task 1: Make command support format-aware (F1, F7)

**Files:**
- Modify: `Sources/LoadoutCore/Assistants.swift`
- Test: `Tests/LoadoutCoreTests/CommandTests.swift`

**Interfaces:**
- Produces: `Assistant.supportsCommands: Bool` (public, existing)
- Produces: `AssistantRegistry.noMarkdownCommands: Set<String>` (internal, new)
- Produces: `Assistant.isDirectory(_:)` helper (private, new)

- [ ] **Step 1: Write failing tests**

Add to `CommandTests`:

```swift
func testGeminiWithItsTomlCommandsFolderIsNotOfferedMarkdownCommands() throws {
    let fixture = Fixture()
    command("ship", in: fixture)
    try fm.createDirectory(at: fixture.paths.home.appendingPathComponent(".gemini/skills"), withIntermediateDirectories: true)
    try fm.createDirectory(at: fixture.paths.home.appendingPathComponent(".gemini/commands"), withIntermediateDirectories: true)
    try "prompt = \"hi\"".write(
        to: fixture.paths.home.appendingPathComponent(".gemini/commands/hi.toml"),
        atomically: true, encoding: .utf8
    )
    let gemini = AssistantRegistry.discover(paths: fixture.paths) { _ in nil }.first { $0.id == "gemini" }!
    XCTAssertFalse(gemini.supportsCommands)
    XCTAssertThrowsError(try Mutations(paths: fixture.paths).shareCommand(item("ship", in: fixture), with: gemini))
    XCTAssertFalse(fm.fileExists(atPath: gemini.commandsRoot.appendingPathComponent("ship.md").path))
}

func testAnOldCopilotCommandsFolderDoesNotMakeItSupported() throws {
    let fixture = Fixture()
    command("ship", in: fixture)
    try fm.createDirectory(at: fixture.paths.home.appendingPathComponent(".copilot/skills"), withIntermediateDirectories: true)
    try fm.createDirectory(at: fixture.paths.home.appendingPathComponent(".copilot/commands"), withIntermediateDirectories: true)
    let copilot = AssistantRegistry.discover(paths: fixture.paths) { _ in nil }.first { $0.id == "copilot" }!
    XCTAssertFalse(copilot.supportsCommands)
    XCTAssertThrowsError(try Mutations(paths: fixture.paths).shareCommand(item("ship", in: fixture), with: copilot))
    XCTAssertFalse(fm.fileExists(atPath: copilot.commandsRoot.appendingPathComponent("ship.md").path))
}

func testARegularFileNamedCommandsIsNotAFolder() throws {
    let fixture = Fixture()
    try fm.createDirectory(at: fixture.paths.home.appendingPathComponent(".hermes/skills"), withIntermediateDirectories: true)
    try "not a folder".write(
        to: fixture.paths.home.appendingPathComponent(".hermes/commands"),
        atomically: true, encoding: .utf8
    )
    let hermes = AssistantRegistry.discover(paths: fixture.paths) { _ in nil }.first { $0.id == "hermes" }!
    XCTAssertFalse(hermes.supportsCommands)
}
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `swift test --filter CommandTests`
Expected: the Gemini and Copilot tests fail because `supportsCommands` is true; the regular-file test fails because `fileExists` is true.

- [ ] **Step 3: Implement the rule**

In `AssistantRegistry`, add:

```swift
/// Assistants known to have no markdown commands: Gemini's are TOML, Copilot has none.
/// A folder that exists for one of these is not evidence that a markdown file would load.
static let noMarkdownCommands: Set<String> = ["gemini", "copilot"]
```

Replace `supportsCommands`:

```swift
public var supportsCommands: Bool {
    if AssistantRegistry.commandAssistants.contains(id) { return true }
    if AssistantRegistry.noMarkdownCommands.contains(id) { return false }
    return Self.isDirectory(commandsRoot)
}
```

Add the helper and use it in `existingFolder`:

```swift
private static func isDirectory(_ url: URL) -> Bool {
    var isDirectory: ObjCBool = false
    return FileManager.default.fileExists(atPath: url.path, isDirectory: &isDirectory)
        && isDirectory.boolValue
}

private func existingFolder(among names: [String]) -> URL? {
    names.map { configHome.appendingPathComponent($0) }
        .first { Self.isDirectory($0) }
}
```

- [ ] **Step 4: Run the tests**

Run: `swift test --filter CommandTests`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add Sources/LoadoutCore/Assistants.swift Tests/LoadoutCoreTests/CommandTests.swift
git commit -m "fix: never offer markdown commands to Gemini or Copilot"
```

## Task 2: Stop the scanner claiming legacy markdown as loaded (F2)

**Files:**
- Modify: `Sources/LoadoutCore/InventoryScanner.swift`
- Test: `Tests/LoadoutCoreTests/CommandTests.swift`

**Interfaces:**
- Consumes: `Assistant.supportsCommands` from Task 1.

- [ ] **Step 1: Write the failing test**

```swift
func testLegacyMarkdownInGeminiIsNotReportedAsLoaded() throws {
    let fixture = Fixture()
    fixture.command("ship")
    try fm.createDirectory(at: fixture.paths.home.appendingPathComponent(".gemini/skills"), withIntermediateDirectories: true)
    try fm.createDirectory(at: fixture.paths.home.appendingPathComponent(".gemini/commands"), withIntermediateDirectories: true)
    try "legacy".write(
        to: fixture.paths.home.appendingPathComponent(".gemini/commands/ship.md"),
        atomically: true, encoding: .utf8
    )

    let ship = InventoryScanner(paths: fixture.paths).scanAll().items.first { $0.name == "ship" && $0.kind == .command }!
    XCTAssertFalse(ship.assistants.contains("gemini"))
    XCTAssertEqual(ship.realCopyIn, ["claude"])
}
```

- [ ] **Step 2: Run it to verify it fails**

Run: `swift test --filter CommandTests.testLegacyMarkdownInGeminiIsNotReportedAsLoaded`
Expected: FAIL because `ship.assistants` contains `gemini`.

- [ ] **Step 3: Filter live command scanning**

In `personalCommands()`, skip assistants that do not support commands for the **live** scan:

```swift
for assistant in assistants where assistant.supportsCommands {
    for file in markdownFiles(in: assistant.commandsRoot) {
        ...
    }
}
```

Leave the `commands-off` scan over all assistants so a parked legacy file still appears and can be deleted.

- [ ] **Step 4: Run the test and the command suite**

Run: `swift test --filter CommandTests`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add Sources/LoadoutCore/InventoryScanner.swift Tests/LoadoutCoreTests/CommandTests.swift
git commit -m "fix: do not count commands in unsupported assistants as loaded"
```

## Task 3: Refuse enabling a command parked in an unsupported assistant (F3)

**Files:**
- Modify: `Sources/LoadoutCore/Mutations.swift`
- Test: `Tests/LoadoutCoreTests/CommandTests.swift`

**Interfaces:**
- Consumes: `Assistant.supportsCommands` from Task 1.

- [ ] **Step 1: Write the failing test**

```swift
func testEnablingACommandParkedOnlyInCopilotIsRefused() throws {
    let fixture = Fixture()
    try fm.createDirectory(at: fixture.paths.home.appendingPathComponent(".copilot/skills"), withIntermediateDirectories: true)
    try fm.createDirectory(at: fixture.paths.home.appendingPathComponent(".copilot/commands-off"), withIntermediateDirectories: true)
    try "do the thing".write(
        to: fixture.paths.home.appendingPathComponent(".copilot/commands-off/ship.md"),
        atomically: true, encoding: .utf8
    )

    let parked = InventoryScanner(paths: fixture.paths).scanAll().items.first { $0.name == "ship" && !$0.enabled }!
    XCTAssertThrowsError(try Mutations(paths: fixture.paths).setCommand(parked, enabled: true))
    XCTAssertFalse(fm.fileExists(atPath: fixture.paths.home.appendingPathComponent(".copilot/commands").path))
    XCTAssertTrue(fm.fileExists(atPath: fixture.paths.home.appendingPathComponent(".copilot/commands-off/ship.md").path))
}
```

- [ ] **Step 2: Run it to verify it fails**

Run: `swift test --filter CommandTests.testEnablingACommandParkedOnlyInCopilotIsRefused`
Expected: FAIL because the file is moved to `~/.copilot/commands`.

- [ ] **Step 3: Filter enabling roots and report the refusal**

In `setPersonalFile`:

```swift
let discovered = AssistantRegistry.discover(paths: paths)
var liveRoots: [URL] = discovered.compactMap {
    switch item.kind {
    case .agent: return $0.agentsRoot
    case .command: return $0.supportsCommands ? $0.commandsRoot : nil
    default: return nil
    }
}
```

Only add the item’s own live root when it is not owned by an unsupported command assistant:

```swift
let own = file.deletingLastPathComponent()
let ownLive = own.lastPathComponent.hasSuffix("-off")
    ? own.deletingLastPathComponent().appendingPathComponent(String(own.lastPathComponent.dropLast(4)))
    : own

if item.kind != .command {
    liveRoots.append(ownLive)
} else if !discovered.contains(where: {
    !$0.supportsCommands && $0.commandsRoot.standardizedFileURL == ownLive.standardizedFileURL
}) {
    liveRoots.append(ownLive)
}
```

Before the final `notFound`, turn the “only an unsupported owner exists” case into the existing user-facing error:

```swift
if enabled, item.kind == .command, copies.isEmpty {
    let parkedOwn = own.deletingLastPathComponent()
        .appendingPathComponent(own.lastPathComponent) // own is already the parked folder
    if let owner = discovered.first(where: {
        !$0.supportsCommands
            && $0.commandsRoot.deletingLastPathComponent()
                .appendingPathComponent($0.commandsRoot.lastPathComponent + "-off")
                .standardizedFileURL == parkedOwn.standardizedFileURL
    }) {
        throw Self.noCommands(owner)
    }
}
```

- [ ] **Step 4: Run the test**

Run: `swift test --filter CommandTests.testEnablingACommandParkedOnlyInCopilotIsRefused`
Expected: PASS, with no `~/.copilot/commands` created.

- [ ] **Step 5: Run the whole suite**

Run: `swift test`
Expected: all tests pass.

- [ ] **Step 6: Commit**

```bash
git add Sources/LoadoutCore/Mutations.swift Tests/LoadoutCoreTests/CommandTests.swift
git commit -m "fix: do not enable commands parked for an assistant that cannot load them"
```

## Task 4: Point Codex at `~/.codex/prompts` (F4)

**Files:**
- Modify: `Sources/LoadoutCore/Assistants.swift`
- Test: `Tests/LoadoutCoreTests/CommandTests.swift`

- [ ] **Step 1: Write the failing test**

```swift
func testCodexUsesPromptsEvenWhenACommandsFolderExists() throws {
    let fixture = Fixture()
    fixture.command("ship")
    try fm.createDirectory(at: fixture.paths.home.appendingPathComponent(".codex/skills"), withIntermediateDirectories: true)
    try fm.createDirectory(at: fixture.paths.home.appendingPathComponent(".codex/commands"), withIntermediateDirectories: true)
    try fm.createDirectory(at: fixture.paths.home.appendingPathComponent(".codex/prompts"), withIntermediateDirectories: true)

    let codex = AssistantRegistry.discover(paths: fixture.paths) { _ in nil }.first { $0.id == "codex" }!
    XCTAssertEqual(codex.commandsRoot.lastPathComponent, "prompts")
}
```

- [ ] **Step 2: Run it to verify it fails**

Run: `swift test --filter CommandTests.testCodexUsesPromptsEvenWhenACommandsFolderExists`
Expected: FAIL because `commandsRoot` picks `commands`.

- [ ] **Step 3: Add the explicit mapping**

In `AssistantRegistry.commandFolders`:

```swift
static let commandFolders: [String: [String]] = [
    "codex": ["prompts"],
    "opencode": ["commands", "command"],
]
```

Simplify the fallback in `commandsRoot` to `names.first ?? "commands"`; the Codex special case is no longer needed.

- [ ] **Step 4: Run the tests**

Run: `swift test --filter CommandTests`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add Sources/LoadoutCore/Assistants.swift Tests/LoadoutCoreTests/CommandTests.swift
git commit -m "fix: write Codex commands to prompts, not a stray commands folder"
```

## Task 5: Remove the unconfirmed `droid` command support (F5)

**Files:**
- Modify: `Sources/LoadoutCore/Assistants.swift`
- Test: `Tests/LoadoutCoreTests/CommandTests.swift`

- [ ] **Step 1: Write the failing test**

```swift
func testDroidWithoutACommandsFolderIsNotOfferedCommands() throws {
    let fixture = Fixture()
    try fm.createDirectory(at: fixture.paths.home.appendingPathComponent(".droid/skills"), withIntermediateDirectories: true)
    let droid = AssistantRegistry.discover(paths: fixture.paths) { _ in nil }.first { $0.id == "droid" }!
    XCTAssertFalse(droid.supportsCommands)
}
```

- [ ] **Step 2: Run it to verify it fails**

Run: `swift test --filter CommandTests.testDroidWithoutACommandsFolderIsNotOfferedCommands`
Expected: FAIL because `droid` is in `commandAssistants`.

- [ ] **Step 3: Remove `droid` from the known set**

Change:

```swift
static let commandAssistants: Set<String> = ["claude", "codex", "cursor", "opencode", "factory"]
```

Add a comment that Droid’s command format/home is unconfirmed; it still gets the folder fallback if it has a real markdown folder.

- [ ] **Step 4: Run the tests**

Run: `swift test --filter CommandTests`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add Sources/LoadoutCore/Assistants.swift Tests/LoadoutCoreTests/CommandTests.swift
git commit -m "fix: only claim command support for documented assistants"
```

## Task 6: Discover an unknown assistant that only has commands (F13)

**Files:**
- Modify: `Sources/LoadoutCore/Assistants.swift`
- Test: `Tests/LoadoutCoreTests/CommandTests.swift`

- [ ] **Step 1: Write the failing test**

```swift
func testAnUnknownAssistantWithOnlyACommandsFolderIsDiscovered() throws {
    let fixture = Fixture()
    try fm.createDirectory(at: fixture.paths.home.appendingPathComponent(".zzz/commands"), withIntermediateDirectories: true)
    try "do the thing".write(
        to: fixture.paths.home.appendingPathComponent(".zzz/commands/ship.md"),
        atomically: true, encoding: .utf8
    )

    let zzz = AssistantRegistry.discover(paths: fixture.paths) { _ in nil }.first { $0.id == "zzz" }
    XCTAssertNotNil(zzz)
    XCTAssertTrue(zzz!.supportsCommands)
}
```

- [ ] **Step 2: Run it to verify it fails**

Run: `swift test --filter CommandTests.testAnUnknownAssistantWithOnlyACommandsFolderIsDiscovered`
Expected: FAIL because discovery requires a skills folder or a known descriptor.

- [ ] **Step 3: Include command folders in discovery**

In `discover`, compute the candidate command homes before the guard:

```swift
let commandNames = AssistantRegistry.commandFolders[id] ?? ["commands", "prompts"]
let hasCommandsFolder = commandNames.contains {
    Assistant.isDirectory(entry.appendingPathComponent($0))
}
guard hasFolder || hasCommandsFolder || descriptor != nil else { continue }
```

Expose the helper as `static func isDirectory(_:)` (internal, not private) or duplicate the check locally. Keep the rest of discovery unchanged.

- [ ] **Step 4: Run the tests**

Run: `swift test --filter CommandTests`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add Sources/LoadoutCore/Assistants.swift Tests/LoadoutCoreTests/CommandTests.swift
git commit -m "feat: discover assistants that only have a commands folder"
```

## Task 7: Cover the copy guard and tighten the existing tests (F6)

**Files:**
- Test: `Tests/LoadoutCoreTests/CommandTests.swift`
- Test: `Tests/LoadoutCoreTests/RemoveAndCopyTests.swift`

- [ ] **Step 1: Add the copy test**

```swift
func testCopyingACommandToAnUnsupportedAssistantIsRefused() throws {
    let fixture = Fixture()
    command("ship", in: fixture)
    try fm.createDirectory(at: fixture.paths.home.appendingPathComponent(".copilot/skills"), withIntermediateDirectories: true)
    let copilot = AssistantRegistry.discover(paths: fixture.paths) { _ in nil }.first { $0.id == "copilot" }!

    XCTAssertThrowsError(try Mutations(paths: fixture.paths).copy(item("ship", in: fixture), to: copilot))
    XCTAssertFalse(fm.fileExists(atPath: copilot.commandsRoot.path))
}
```

- [ ] **Step 2: Fix the “known” test so it isolates the known set**

In `testKnownAssistantsAndOnesAlreadyUsingACommandsFolderAreOffered`, remove the `.codex/prompts` setup before the Codex assertion; assert Codex is supported with no folder. Keep a separate Hermes folder-existence assertion.

- [ ] **Step 3: Update the stale fixture comment**

`RemoveAndCopyTests.swift:9` should say four assistants (`zzz`, `yyy`, `cursor`, `codex`), not two.

- [ ] **Step 4: Run the tests**

Run: `swift test`
Expected: all pass.

- [ ] **Step 5: Commit**

```bash
git add Tests/LoadoutCoreTests/CommandTests.swift Tests/LoadoutCoreTests/RemoveAndCopyTests.swift
git commit -m "test: cover command copy refusal and known-assistant support"
```

## Task 8: Refine the Assistants card and sync status (F9, F10)

**Files:**
- Modify: `Sources/LoadoutApp/DetailView.swift`
- Modify: `Sources/LoadoutApp/AppModel.swift`

- [ ] **Step 1: Add an empty state and accurate help text**

In `assistantsCard`, when `assistants.isEmpty`, show a small line such as:

```swift
if assistants.isEmpty {
    Text("No visible assistant loads \(item.kind.briefingNoun)s.")
        .font(.system(size: 11.5))
        .foregroundStyle(V2.textMid)
        .padding(.horizontal, 13)
        .padding(.vertical, 10)
}
```

Update the count’s `.help` to say “out of the assistants Loadout found on this Mac, Settings keeps in view, and that load this kind of item.”

- [ ] **Step 2: Mention skipped commands in `syncAll`**

When `!assistant.supportsCommands`, count command items that are not in the assistant and append a note to the success message, for example:

```swift
let skipped = items.filter {
    $0.kind == .command && $0.origin == .personal && $0.enabled
        && !$0.assistants.isEmpty && !$0.assistants.contains(assistant.id)
}.count
if skipped > 0 {
    statusMessage += " \(skipped) command\(skipped == 1 ? "" : "s") skipped: \(assistant.label) doesn't load markdown commands."
}
```

- [ ] **Step 3: Build**

Run: `swift build`
Expected: success.

- [ ] **Step 4: Commit**

```bash
git add Sources/LoadoutApp/DetailView.swift Sources/LoadoutApp/AppModel.swift
git commit -m "ux: explain when no assistant loads commands and what sync-all skipped"
```

## Task 9: Update user-facing copy, changelog and spec (F11)

**Files:**
- Modify: `CHANGELOG.md`
- Modify: `Sources/LoadoutApp/Selection+UI.swift`
- Modify: `docs/SPEC.md`
- Modify: `spec/spec-flow-commands.md`

- [ ] **Step 1: Update the changelog entry**

Say that Gemini and Copilot are never offered a markdown command, even when a commands folder exists, and that other assistants whose folders already exist still are. Keep Droid out of the list.

- [ ] **Step 2: Update the Commands tab tooltip**

Change `Selection+UI.swift:11` from “Slash commands available to Claude in every project” to a sentence that covers the assistants that actually load them.

- [ ] **Step 3: Update the data-source table and command spec**

Add Codex/Cursor/opencode/Factory paths to `docs/SPEC.md:34` and correct the “Commands are read from Claude only” passage in `spec/spec-flow-commands.md` (§5). Keep the new rule explicit: Gemini (TOML) and Copilot (none) are not offered markdown commands.

- [ ] **Step 4: Build**

Run: `swift build`
Expected: success.

- [ ] **Step 5: Commit**

```bash
git add CHANGELOG.md Sources/LoadoutApp/Selection+UI.swift docs/SPEC.md spec/spec-flow-commands.md
git commit -m "docs: describe the real command support matrix"
```

## Task 10: Filter demo and automation callers (F12)

**Files:**
- Modify: `Sources/LoadoutApp/Scenes.swift`
- Modify: `Sources/LoadoutApp/DriveScript.swift`

- [ ] **Step 1: Filter the demo target**

In `Scenes.lastSwitches`, choose the first assistant that is missing the item **and** supports it:

```swift
let candidates = model.assistants.filter {
    !item.assistants.contains($0.id)
        && (item.kind != .command || $0.supportsCommands)
}
guard let missing = candidates.first else { return }
```

- [ ] **Step 2: Reject unsupported explicit shares in the drive script**

Before `model.setAssistant` for `share`, return a clear string if the item is a command and the assistant does not support commands.

- [ ] **Step 3: Build**

Run: `swift build`
Expected: success.

- [ ] **Step 4: Commit**

```bash
git add Sources/LoadoutApp/Scenes.swift Sources/LoadoutApp/DriveScript.swift
git commit -m "chore: keep demos and scripts on command-capable assistants"
```

## Task 11: Final verification and review (all findings)

- [ ] **Step 1: Run the full suite**

Run: `swift test`
Expected: all tests pass.

- [ ] **Step 2: Check the diff for unintended changes**

Run: `git diff --stat` and `git diff`

- [ ] **Step 3: Request an independent review**

Dispatch a fresh reviewer subagent with the base commit and the new head. Ask it to verify the fixes against the findings table and to report any remaining issue.

- [ ] **Step 4: Update this document**

Mark every finding’s status and add a short “Outcome” section with the test counts and the reviewer’s verdict.

## Deferred

- **F8 (performance):** cache `hasCommandsFolder` on `Assistant` at discovery. This is a refactor with a larger constructor surface; it does not affect correctness. Revisit after the functional fixes land.
- **App-level sync-all test:** `AppModel` lives in the executable target and has no XCTest target. The skipped-command message is covered by build verification; extract a pure helper if the project later adds app-model tests.
- **Antigravity command format:** Gemini CLI’s successor may use TOML too. Confirm its documentation before adding `antigravity` to `noMarkdownCommands` or `commandAssistants`.
- **Translating commands to TOML/prompt files:** a separate feature, not a fix.
