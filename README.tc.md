# touch-code Ghostty Fork

This fork is based on the upstream `v1.3.1` tag and maintained on the
`v1.3.1-tc` branch.

## Purpose

touch-code needs process identity from each embedded terminal surface so active
agent detection can classify the actual foreground job instead of relying on
window title or startup command text.

The fork keeps this surface-area intentionally small:

- `ghostty_surface_child_process_id`
- `ghostty_surface_foreground_process_group`

Both accessors are read-only and expose existing surface process identifiers to
the macOS host app.

## Upgrade Workflow

1. Choose a stable upstream release tag.
2. Create a fork branch named `<upstream-tag>-tc` from that tag.
3. Reapply or cherry-pick the minimal process identifier accessor patch.
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
