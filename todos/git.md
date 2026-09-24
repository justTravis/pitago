# Git panel implementation plan

## Goal

Replace the current right-hand supporting-information sidebar with an interactive Git worktree panel while preserving the existing chat/agent UI on the left. The first version should be a safe, rudimentary Git client: show repository state, separate unstaged and staged changes, inspect diffs, and perform basic staging/unstaging and discard actions. It should remain responsive by running Git commands off the Bubble Tea update loop and should degrade cleanly when the working directory is not a repository.

## Current architecture and constraints

- `src/app/view.go` owns the two-column layout (`showSide`, `mainW`, `View`) and currently assembles the right side through `renderSidebar`/`buildSidebarContent`.
- `src/app/model.go` already owns sidebar state (`sideVp`, `ws`, refresh/cache fields) and initializes the two viewports. It also starts workspace polling from `Init`.
- `src/app/update.go` routes Bubble Tea messages, mouse events, keyboard input, viewport scrolling, and the existing workspace polling messages.
- `src/app/ws.go` already executes `git` with `exec.CommandContext`, has a five-second timeout, and reads the branch plus a small aggregate status/numstat summary. This is the natural starting point, but its `wsData` model is not rich enough for an interactive panel.
- `src/app/styles.go` defines the fixed `sideW`/`sideInnerW` geometry and shared theme-aware styles.
- Existing sidebar tests in `src/app/sidepanels_test.go` cover workspace-adjacent/sidebar behavior and should be expanded or split with Git-specific tests.
- The module already depends on Bubble Tea, Bubbles, and Lipgloss; prefer those existing dependencies rather than adding a Git library. Keep Git invocation centralized and use argument slices rather than shell strings.

## Proposed product scope

### MVP behavior

- Header showing repository state: current branch (or detached HEAD), repository root, and a compact count summary.
- Two independently visible sections:
  - **Changes**: unstaged tracked modifications/deletions/renames plus untracked files.
  - **Staged**: index entries waiting for commit.
- Selecting a file shows its relevant diff in a detail/preview area, with a scrollable diff viewport if the available height permits. At minimum, provide a way to open the full diff for the selected row without truncating silently.
- Basic actions:
  - `s` stage the selected file; for an already staged file, stage the current working-tree portion where Git permits.
  - `u` unstage the selected staged file.
  - `d` discard the selected tracked unstaged change, guarded by confirmation; never silently delete untracked files.
  - `a` stage all visible changes (or define this explicitly as `git add -A` after confirmation if untracked files are included).
  - `r` refresh.
- Read-only fallback for non-Git directories, Git command failures, unborn branches, detached HEAD, merge conflicts, renamed paths, binary files, and permission errors.
- Preserve `Ctrl+B` sidebar visibility and existing sidebar scrolling semantics, but retarget the content and hit-testing to the Git panel.

### Deliberately defer

- Commit creation, commit-message editing, push/pull/fetch, branch creation/switching, merge/rebase, stash, worktree management, partial/hunk staging, and remote management.
- A full-screen diff editor. These can be added after the state model and command runner are stable.

## Implementation todos

### 1. Define the Git domain model and command boundary

- [ ] Replace or generalize `wsData` in `src/app/ws.go` with explicit types for repository metadata, file status, staged/unstaged state, diff statistics, and command errors. Keep paths repository-relative and retain enough information to distinguish untracked, added, modified, deleted, renamed, copied, conflicted, and binary files.
- [ ] Decide whether one file row can have both staged and unstaged portions. Model those as separate flags/status fields rather than duplicating or losing the row.
- [ ] Add a `gitSnapshot` containing branch/HEAD, root, ahead/behind if cheap to obtain, staged rows, unstaged rows, and aggregate counts. Add a selected row identity that remains stable across refreshes (path plus section/status).
- [ ] Add a small command runner around `exec.CommandContext` with a per-operation timeout, captured stderr, and structured errors. Always use `git -C <cwd> ...`; do not invoke a shell or interpolate user-controlled paths into a command string.
- [ ] Use machine-readable Git output where possible: `status --porcelain=v1 -z` (or `--porcelain=v2 -z` after verifying parsing needs), `diff --numstat`, and `diff --cached --numstat`. Parse NUL-delimited output so spaces, tabs, quotes, and unusual filenames are safe.
- [ ] Add separate helpers for status refresh, unstaged diff, staged diff, stage, unstage, and discard. Return output/errors to Bubble Tea messages rather than mutating `Model` from goroutines.

### 2. Expand refresh and async state handling

- [ ] Replace the narrow `wsTickMsg`/`wsMsg` payload with Git-specific messages such as `gitTickMsg`, `gitSnapshotMsg`, `gitDiffMsg`, and `gitActionMsg` (names can follow the repository's existing conventions).
- [ ] Keep all Git work off the UI thread. Extend `Init`, the existing 10-second polling path, `agent_settled`, and relevant file-changing events so a refresh is requested without launching duplicate in-flight refreshes.
- [ ] Add `gitLoading`, `gitErr`, `gitActionInFlight`, and a monotonically increasing refresh/request generation so stale results cannot overwrite a newer action result.
- [ ] After every mutating action, refresh status and invalidate the selected diff. After an external change is detected, preserve the selected path where possible and clamp all cursors/offsets.
- [ ] Decide how often to refresh while the panel is open versus hidden. Keep polling bounded, and ensure a slow or unavailable Git command cannot block chat interaction.
- [ ] Reuse the existing notice/toast error language for transient action failures, but keep persistent repository errors visible in the Git panel.

### 3. Replace the right-side rendering with a Git panel

- [ ] Add `renderGitPanel` (likely in a new `src/app/git_view.go`) and route `View` from `renderSidebar` to it while retaining the same outer `sideStyle`, width, height, and theme behavior.
- [ ] Design a compact layout that fits `sideInnerW`: repository/branch header, summary counts, section headers, rows with status glyphs and paths, then a selected-file diff preview or a clear empty state.
- [ ] Add a dedicated diff viewport or an explicit “open diff” mode if the normal sidebar viewport cannot support both file-list scrolling and diff scrolling cleanly. Document the focus model in the footer (for example: `↑↓ select · Tab section · Enter diff · s stage · u unstage · d discard · r refresh`).
- [ ] Render Git status consistently: green additions, red deletions, yellow conflicts, dim untracked entries, and clear staged/unstaged markers. Ensure every row is width-clamped and cannot wrap the fixed-height layout.
- [ ] Use existing `codeStyle`, `toolStyle`, `okStyle`, `errStyle`, `warnStyle`, `rowHiStyle`, and `sideTitleStyle` where appropriate; add only Git-specific derived styles in `styles.go` if needed so `ApplyTheme` rebuilds them.
- [ ] Show binary diffs, empty diffs, rename metadata, and command errors as explicit text rather than attempting to render invalid patch content.
- [ ] Keep non-Git and clean-repository states useful: show “not a Git repository”, “clean”, “no staged changes”, or “no unstaged changes” instead of an empty panel.

### 4. Add keyboard, mouse, and confirmation interaction

- [ ] Extend `Model` with Git panel focus/section/row cursors and diff offset state. Keep those independent from the chat viewport cursor and textarea.
- [ ] Update `src/app/update.go` so Git panel navigation is handled before generic sidebar scrolling when focus is in the panel. Preserve existing `Alt`/`Ctrl` sidebar scroll behavior when the panel is not in an action/focus mode.
- [ ] Add key handling for refresh, section switching, selecting a row, opening/closing the diff, staging, unstaging, and discard confirmation. Avoid collisions with existing global bindings; if a binding is ambiguous, make it active only while Git focus is explicit.
- [ ] Extend mouse hit-testing (alongside `overSide`, `recentAt`, and plugin handling) so clicks select rows and the panel footer/actions, while wheel events scroll the active Git viewport.
- [ ] Use an existing dialog/confirm pattern for destructive discard operations. The confirmation must include the path and explicitly distinguish tracked discard from untracked-file deletion (the MVP should refuse untracked deletion).
- [ ] Make actions safe when the selected row disappears, the repository changes externally, or the user switches away while a command is running.

### 5. Decide how the old workspace/sidebar code is retired

- [ ] Remove `SideWorkspace` and workspace-only rendering from the visible right panel, or retain the underlying `readWS` code temporarily only if it is reused to seed the new Git snapshot.
- [ ] Remove obsolete `wsData` fields and polling if the new Git refresh fully replaces them; otherwise avoid running two independent Git pollers.
- [ ] Update `sideOrder`, labels, settings/preferences, and any `/settings` sidebar toggles so users do not see a configurable section that no longer exists. Decide whether Git is always present or has a `SideGit` visibility preference; the request suggests replacing the right-side content, so default it to always available when the side is shown.
- [ ] Update comments/documentation that currently describe the right side as a pi-style information sidebar or workspace summary.

### 6. Tests

- [ ] Add parser table tests for porcelain output covering modified, staged, unstaged, untracked, deleted, renamed, copied, conflict, spaces/tabs, and NUL-delimited filenames.
- [ ] Add tests for diff/numstat parsing, binary files, malformed output, command timeout, missing Git executable, non-repository directories, detached HEAD, and unborn repositories.
- [ ] Add command-construction tests using an injectable runner or temporary fake `git` executable; verify no shell expansion and correct `-C`/argument ordering.
- [ ] Add model/update tests for cursor movement, section changes, selection preservation after refresh, stale async messages, action success/failure, and discard confirmation.
- [ ] Add rendering tests for clean, staged-only, unstaged-only, mixed, conflict, non-Git, and long-path states. Assert that rows fit the panel width and that footer/help text communicates the active keys.
- [ ] Extend mouse tests for row selection and wheel behavior without regressing existing sidebar scrolling.
- [ ] Run `go test ./...`, `go vet ./...`, and the project CI script before considering the work complete.

### 7. Manual acceptance checklist

- [ ] Launch in a clean repository: branch and clean state are visible; chat remains fully usable.
- [ ] Modify, create, delete, rename, and stage files; verify the correct item moves between Changes and Staged and the diff preview matches the selected state.
- [ ] Verify stage/unstage refreshes immediately and does not freeze streaming chat.
- [ ] Verify discard requires confirmation and leaves untracked files untouched.
- [ ] Verify conflicts, binary files, detached HEAD, no commits, non-Git directories, missing Git, and command failures produce readable states.
- [ ] Resize below and above the sidebar breakpoint, toggle with `Ctrl+B`, scroll with keyboard and mouse, and confirm no wrapping/overflow or stale selection.
- [ ] Confirm existing dialogs, model switching, session switching, tool streaming, and quit behavior continue to work while Git refreshes/actions are in flight.

## Suggested delivery sequence

1. Build and test the Git command/parser package without changing the UI.
2. Wire snapshot polling and replace the old workspace data with the richer model while keeping a temporary text rendering.
3. Implement the Git panel rendering and independent list/diff navigation.
4. Add mutating actions and confirmation flows.
5. Remove obsolete workspace/sidebar settings and update documentation.
6. Run the full test/CI suite and perform the manual acceptance checklist.

## Open design decisions to settle before coding

- Whether Git is always the sole right-side panel or can be toggled as a preference.
- Whether `Enter` opens a full diff mode or merely switches focus to an embedded diff viewport.
- Whether “stage all” includes untracked files by default; safest MVP is an explicit confirmation for `git add -A`.
- Whether discard supports only tracked files (`git restore -- <path>`) or also untracked deletion; safest MVP is tracked files only.
- Whether ahead/behind counts are worth an additional Git call or should wait for a later iteration.
