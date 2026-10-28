# touch-code Ghostty Fork

This fork is based on the upstream `v1.3.1` tag and maintained on the
`v1.3.1-tc` branch.

- **Fork URL:** https://github.com/wanggang316/ghostty
- **Upstream:** https://github.com/ghostty-org/ghostty (`v1.3.1` tag)

## Purpose

touch-code needs process identity from each embedded terminal surface so active
agent detection can classify the actual foreground job instead of relying on
window title or startup command text.

The fork keeps this surface-area intentionally small. Two read-only C
accessors are exported:

- `ghostty_surface_child_process_id`
- `ghostty_surface_foreground_process_group`

Their plumbing adds a `getProcessInfo` method to the internal
`Subprocess` / `Exec` / `Backend` chain in `src/termio/` so the exported C
symbols can delegate down to the existing pty layer.

## Carried Patches

- `feat: expose surface process identifiers` — the accessor patch
  (Surface / apprt / pty / Termio / Subprocess wiring + the two C symbols)
- `fix(termio): expose getProcessInfo on Exec backend` — adds the
  `Exec.getProcessInfo` delegation so `Backend.getProcessInfo`'s dispatch
  compiles. Required follow-up after the accessor patch lands.
- `feat(apprt): add a per-surface command wrapper and integration opt-out` —
  `ghostty_surface_config_s.command_wrapper` / `command_wrapper_count` /
  `disable_shell_integration`. The wrapper argv is prepended after
  `execCommand` (login(1) wrapping and shell-integration rewrite included),
  so touch-code runs each pane as `zmx attach <id> <resolved command>` and
  Ghostty still detects and integrates the user's real shell. Carried on the
  private `_command_wrapper` config field; not settable from config files.
- `feat(config): default shell-integration-features for the embedding app` —
  `sudo` and `ssh-env` on, `title` and `path` off. Kept on the struct because
  every `shell-integration-features` assignment starts from the struct
  defaults, so a layered config file cannot set them.
- `feat(apprt): seed the embedded surface size from the config` —
  `ghostty_surface_config_s.initial_width` / `initial_height` (device
  pixels). The child is spawned during `ghostty_surface_new` with a PTY
  sized from the surface size, before the embedder can push one, so without
  this every child starts at 800x600 px. touch-code passes the pane's last
  known size so a `zmx attach` to a live session does not resize the
  session's TUI through a placeholder width. Zero keeps the default.
- `build: normalize input archives before Darwin libtool merge` — backport of
  upstream ghostty-org/ghostty#11999 (`a83a82b3f8`, not in any 1.3.x tag).
  Recent Apple libtool (seen with Xcode 26.6) silently drops zig-written
  archive members that are not 8-byte aligned, which leaves
  `libghostty-fat.a` incomplete.
  Drop this patch once the fork base includes the upstream commit.
- `feat(terminal): backport the OSC 7501 program status parser` — backport
  of upstream ghostty-org/ghostty#14560 (parser and kitty metadata iterator
  only, not the libghostty-vt record API). Drop this patch once the fork base
  includes the upstream commit.
- `feat(apprt): expose program status reports (OSC 7501) as an action` —
  answers the `OSC 7501 ; ?` support query and delivers each report as
  `GHOSTTY_ACTION_PROGRAM_STATUS` with a pointer to
  `ghostty_action_program_status_s`; a full reset delivers `clear` with an
  empty id. The struct mirrors the field set of the upstream libghostty-vt
  `GhosttyTerminalProgramStatus`. touch-code keeps the records per pane.
  Upstream has no app-path action, so this patch stays after a rebase.
- `fix(renderer): clear the Metal display callback on deinit` — the
  embedder's view keeps the `IOSurfaceLayer` after `ghostty_surface_free`,
  and a later bounds change makes Core Animation call `display` on it. The
  layer's `display_ctx` still pointed at the freed renderer, so the call read
  freed memory (`EXC_BAD_ACCESS` in `object_getClass` under
  `CA::Context::commit_transaction`). Drop this patch once upstream clears
  the callback itself.

## Upstream PR Status

No upstream PR has been filed for these accessors. The fork is maintained
privately for touch-code's needs only. Before shipping a major release that
depends on these symbols long-term, consider proposing them upstream so the
fork can retire.

## Upgrade Workflow

1. Choose a stable upstream release tag.
2. Create a fork branch named `<upstream-tag>-tc` from that tag.
3. Cherry-pick the patches above in order, skipping any that the new tag
   already contains.
4. Build `GhosttyKit.xcframework` through the existing Tuist flow.
5. Run the macOS app build and active-agent detection tests.
6. Update `.gitmodules` to point at the new branch.
7. Update the submodule pointer in the root repository.

## Validation

Run these checks from the root repository after updating the submodule pointer:

```bash
make mac-build
xcodebuild test -workspace apps/mac/touch-code.xcworkspace -scheme TouchCodeCore -destination 'platform=macOS' -only-testing:TouchCodeCoreTests/AgentKindPatternsTests -only-testing:TouchCodeCoreTests/PaneAttentionInterpreterTests
```

If the process identifier patch grows beyond read-only accessors, document the
new host-app contract here before updating the root submodule pointer.
