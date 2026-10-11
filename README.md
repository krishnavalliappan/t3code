# T3 Code, customized

This branch (`customize`) holds my changes to [T3 Code](https://github.com/pingdotgg/t3code) as patch files.
They are re-applied on top of every upstream **Nightly**, so the app stays current and keeps my tweaks.

- `patches/`: one patch per customization, applied in order with `git am`.
- `.github/workflows/custom-nightly.yml`: every 2 hours, it applies the patches to the newest nightly tag and builds an unsigned macOS arm64 DMG. The DMG is published as a prerelease here under the same tag. If a patch stops applying, the workflow opens an issue and the previous build stays current.
- `bin/t3code-customize`: the local side of the ritual.

```sh
bin/t3code-customize install   # install the latest CI build into /Applications
bin/t3code-customize status    # upstream nightly vs. my latest custom build

bin/t3code-customize edit      # ~/.t3code-custom/src = newest nightly + patches, as commits
#   ...hack, `pnpm dev:desktop`, commit each tweak as its own commit...
bin/t3code-customize save      # commits -> patches/; then commit + push this branch
bin/t3code-customize build     # optional: build + install locally instead of waiting for CI
```

## Current patches

- **Composer status bar**: a row under the composer shows 5h and weekly usage, context % and tok/s for the last turn. Usage and context come from data T3 Code already has. tok/s is the newest turn's output tokens over its wall-clock time, so tool calls lower it.

## Notes

- Builds are unsigned. `install` strips the quarantine flag; if you download a DMG by hand, run `xattr -dr com.apple.quarantine "/Applications/T3 Code (Nightly).app"`.
- The app's updater points at this fork, so it never "updates" back to stock. In-app updates of unsigned macOS builds can fail, so use `install` instead.
- The custom app replaces the stock `T3 Code (Nightly).app` and shares its data (`~/.t3`).
