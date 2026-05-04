# Git Decorations in the File Tree Sidebar

**Date:** 2026-05-04
**Status:** Approved + revised after code audit (see "Revisions" at bottom)
**Owner:** fork of warpdotdev/warp

## Summary

Add VS Code-style git status decorations to the file tree sidebar — a letter badge (M/A/U/D/C/R) on the right of each file row plus a filename color tint, with a small dot rolled up onto parent directories that contain modified descendants. Decorations refresh live (~200ms after a save), driven by the existing filesystem watcher.

## Goals

- Surface per-file git status in the sidebar so the user can spot changes without running `git status`.
- Match VS Code's tree-explorer mental model: letter + tint + parent rollup.
- Reuse the existing watcher / repo-metadata / file-tree-update path. No new event types, no new subscription channels.

## Non-Goals

- Editor-gutter decorations (per-line +/- indicators inside a file).
- Conflict resolution UI (we show `C`; we don't help merge).
- Click-to-stage from the badge (that belongs in a future SCM panel, not in the tree).
- Decorating files that are already excluded from the tree by `IgnoredPathStrategy::Exclude`.

## Architecture

Three components, each with one purpose:

```
fs event (existing notify watcher, debounced ~150ms)
    │
    ▼
LocalRepoMetadataModel  ← extended:
    ├── recompute statuses for changed paths (git2)
    ├── update cache: git_statuses[repo][file] -> GitStatus
    ├── update rollup: dirty_dirs[repo] -> set of dir paths
    └── emit RepositoryMetadataEvent::FileTreeEntryUpdated  (existing event, no new types)
    │
    ▼
FileTreeView::render_item  ← extended:
    ├── look up status for the row's path
    ├── apply filename color tint
    ├── append letter badge on the right
    └── for directories: render dim dot if path is in dirty_dirs
```

### Component 1 — `crates/repo_metadata/src/git_status.rs` (new)

Pure git2 wrapper. No state. Independently testable with fixture repos.

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub enum GitStatus {
    Modified,
    Added,
    Untracked,
    Deleted,
    Conflict,
    Renamed,
    Ignored,
}

/// Compute statuses for a repo, optionally scoped to specific paths.
pub fn statuses_for_repo(
    repo_root: &Path,
    pathspec: Option<&[&Path]>,
) -> Result<HashMap<StandardizedPath, GitStatus>, GitStatusError>;
```

When `pathspec` is `Some`, uses `git2::StatusOptions::pathspec` to scope the walk — required for performance on large repos under high-frequency edits.

### Component 2 — `LocalRepoMetadataModel` extension

Two new fields:

```rust
git_statuses: HashMap<StandardizedPath /*repo*/, HashMap<StandardizedPath /*file*/, GitStatus>>,
dirty_dirs:   HashMap<StandardizedPath /*repo*/, HashSet<StandardizedPath /*dir*/>>,
```

Behavior changes:

- On every `handle_filesystem_event` batch, after the existing file-tree mutation pass, call `recompute_git_statuses(repo, changed_paths)`.
- `recompute_git_statuses` rebuilds the entries in `git_statuses[repo]` for the affected paths and recomputes `dirty_dirs[repo]` by walking ancestors of every non-clean file.
- The existing `FileTreeEntryUpdated` event already fires from this path — no new event needed. The view rebuild will read the updated status map.
- New public read API: `git_status_for(path: &StandardizedPath) -> Option<GitStatus>` and `is_dirty_dir(path: &StandardizedPath) -> bool`.

### Component 3 — `FileTreeView` row rendering

In `app/src/code/file_tree/view.rs`, the row builder reads status during render:

- File rows: look up status. If present, override `text_color` with the status color and append a 1-character `Text` widget on the right of `header_row` showing the badge letter.
- Directory rows: if `is_dirty_dir(path)`, render a small dim dot in the same right-edge slot. No tint.
- Theme-aware: colors come from `internal_colors::*` tokens, never hardcoded RGB. Tokens to use:
  - Modified → `internal_colors::yellow_*`
  - Untracked / Added / Renamed → `internal_colors::green_*`
  - Deleted / Conflict → `internal_colors::red_*`
  - Ignored → `internal_colors::neutral_*` (dimmed)

### Status → visual treatment table (v1, post-audit)

`Deleted` is intentionally omitted — see "Revisions" below.

| Status     | Letter | Filename color                                  | Extra            |
|------------|--------|-------------------------------------------------|------------------|
| Modified   | M      | `theme.ansi_fg_yellow()` / `ui_yellow_color()`  | —                |
| Untracked  | U      | `diff::add_color(appearance)` (green)           | —                |
| Added      | A      | `diff::add_color(appearance)` (green)           | —                |
| Conflict   | C      | `theme.ansi_fg_red()`                           | —                |
| Renamed    | R      | `diff::add_color(appearance)` (green)           | —                |
| Ignored    | (none) | `internal_colors::text_disabled(...)` (dimmed)  | —                |
| Dirty dir  | (dot)  | unchanged                                       | small `●` in `internal_colors::neutral_5/6` |

## Refresh path

Already wired by the existing watcher. No new pipeline:

```
notify event
  → BulkFilesystemWatcher (debounced 1s — see Revisions)
  → LocalRepoMetadataModel::handle_watcher_event
    → existing: file-tree mutations
    → NEW: recompute_git_statuses(changed_paths)
       — `.git/` internal events trigger full-repo refresh, not pathspec
    → existing: emit FileTreeEntryUpdated
  → FileTreeView::handle_repository_metadata_event (already supports exact + ancestor)
  → rebuild_flattened_items + ctx.notify
```

Initial full-repo status refresh runs once after `index_directory` succeeds (right after the existing initial `FileTreeEntryUpdated`).

## Performance

- Scoped recompute via `StatusOptions::pathspec` on changed paths only — avoids full-repo status walk per edit.
- Full-repo recompute happens only on initial repo load (one-shot) or on coarse events that lose path information.
- `dirty_dirs` recomputed in O(non-clean files × max depth). For a repo with 1000 dirty files at depth 10, that's 10k inserts — well under a millisecond.
- Status map memory: ~80 bytes per dirty file. 10k dirty files = ~800KB. Acceptable.

## Edge cases

- File outside any repo → no decoration.
- Repo not yet indexed → no decoration; appears once `LocalRepoMetadataModel` finishes the initial index.
- Submodule → treated as a separate repo; git2 reports submodule status on the parent repo, scoped on the submodule itself when traversed.
- `.gitignore`-matched files where `IgnoredPathStrategy::Include` keeps them in the tree → render with `Ignored` decoration (gray, no letter). Files with `Exclude` strategy are not in the tree, so this is a no-op for them.
- Renamed files (`R`) — git2 reports both old and new path. We decorate the new path; the old path no longer exists on disk and won't be in the tree.

## Files touched

| File                                                  | Change                                              |
|-------------------------------------------------------|-----------------------------------------------------|
| `crates/repo_metadata/src/git_status.rs`              | NEW — pure status computation module                |
| `crates/repo_metadata/src/lib.rs`                     | `pub mod git_status;`                               |
| `crates/repo_metadata/src/local_model.rs`             | Add fields + `recompute_git_statuses` + read APIs   |
| `crates/repo_metadata/Cargo.toml`                     | Add `git2` dep (vendored, mirroring `app/Cargo.toml`) |
| `app/src/code/file_tree/view.rs`                      | Status lookup + tint + badge rendering              |
| `app/src/code/file_tree/view/view_tests.rs`           | New tests covering status, rollup, deleted-strikethrough |

Estimated diff: ~400 lines across 5 files.

## Testing

- **Unit tests** (`git_status.rs`): with a temp git repo fixture, assert each enum variant maps from the expected `git2::Status` flags.
- **Model tests** (`local_model_test.rs`): construct a model, simulate fs events, assert `git_status_for` and `is_dirty_dir` return the right values; assert `FileTreeEntryUpdated` fires.
- **View tests** (`view_tests.rs`): mirror the existing `VirtualFS::test` pattern at line 478. Assert the rendered row has the expected color and badge text after status changes.

## Open questions (deferred for implementation)

1. Where to put the badge letter exactly — flush right of the row, or right-aligned with a fixed gutter? Decision: right-aligned, with right-padding matching existing row right-edge. There is no named "chevron right-padding token" — use `ITEM_PADDING` and the `4.` margin pattern from the chevron cell as reference.
2. Whether to render the dirty dot ALSO when the directory itself is the root (which would always be dirty if anything inside it is). Decision: yes — keeps the rule simple ("any descendant non-clean = dot"). Roots show a dot when the repo has any change.
3. Theme tokens for the badge letter background/foreground in case yellow-on-yellow gets unreadable. Decision: badge uses status color directly; if a theme has poor contrast, the theme is wrong (filed as a separate concern).

---

## Revisions (after code audit, 2026-05-04)

The original draft assumed a few API shapes that the codebase doesn't have. Corrections:

- **Hook name:** the watcher hook is `LocalRepoMetadataModel::handle_watcher_event` (not `handle_filesystem_event`).
- **Debounce:** the active file-tree watcher uses `FILESYSTEM_WATCHER_DEBOUNCE_SECS = 1` (`crates/repo_metadata/src/local_model.rs:39`). The original spec said ~200ms; v1 accepts 1s. Lowering the debounce is a separate cross-cutting perf decision — defer.
- **Deleted (`D`) status:** dropped from v1. The current model converts deletions to `FileTreeMutation::Remove` (`local_model.rs:576` → applied at `:652`), so a deleted file has no row to decorate. VS Code's explorer also omits deleted files (they live in the SCM panel). Re-introducing requires "virtual deleted entries" — 150-300 lines of red-risk work, deferred.
- **Color tokens:** `internal_colors::yellow_*` / `green_*` / `red_*` do not exist. Use `WarpTheme` methods (`ansi_fg_yellow`, `ui_yellow_color`, `ansi_fg_red`) and the editor diff helpers (`diff::add_color`, `diff::remove_color`). See updated treatment table.
- **`.git/` internal events:** `should_ignore_git_path` (`crates/repo_metadata/src/entry.rs:453`) lets some `.git/index`, `HEAD`, `refs/*` events through. Treat these as a full-repo refresh trigger, not literal pathspec args.
- **`render_item` signature:** currently `fn render_item(&self, item_id, appearance) -> ...` (`view.rs:1931`). Implementer must thread `&AppContext` (already available in the UniformList closure at `view.rs:2617`) so the row builder can read git status from the model.
- **`StandardizedPath::ancestors()` exists** (`crates/warp_util/src/standardized_path.rs:189`) — rollup approach is unchanged.
