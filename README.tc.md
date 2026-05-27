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

## Upstream PR Status

No upstream PR has been filed for these accessors. The fork is maintained
privately for touch-code's needs only. Before shipping a major release that
depends on these symbols long-term, consider proposing them upstream so the
fork can retire.

## Upgrade Workflow

1. Choose a stable upstream release tag.
2. Create a fork branch named `<upstream-tag>-tc` from that tag.
3. Cherry-pick the two patches above in order.
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
