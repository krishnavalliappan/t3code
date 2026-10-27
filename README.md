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

- **Composer status bar**: a row under the composer shows 5h and weekly usage **remaining** (e.g. `5h 62% ↻2h 14m`; hover for % used, how much of the window has elapsed, and the exact reset date/time), context % and tok/s for the last turn. Usage and context come from data T3 Code already has. tok/s is the newest turn's output tokens over its wall-clock time, so tool calls lower it.
- **Claude usage limits fix** (server): a throttled usage probe (`rate_limits: null`) no longer marks a Max account as "no subscription limits", and Claude Code 2.1.30x `unifiedWindows` events now fill 5h/weekly. Local threads only: remote (SSH) hosts download the stock server. Drop once upstream fixes it.
- **Floating chat panel** (web): `⇧⌘I`, or **✦ Chat** at the right of the composer status bar, opens a floating Claude chat for drafting replies. It minimizes to a pill, resizes from the left/top edges (double-click resets), lists past chats, shows Claude's activity above each reply (web searches, connector calls, thinking; open while running, folded after, click a row for its output), has Copy / Shorter / Warmer / More formal under replies, a Tone picker (`Mine`, `Client`, `Team`, `Formal`), and a speed picker (Fast / Balanced / Deep = Claude reasoning effort low / medium / high; Fast by default, since the provider default thinks for seconds per step). Chats run with full access, so connectors (Teams, Smartsheet, …) and WebFetch work without prompts. `~/Chat/.claude/settings.json` still denies Bash and edits to `CLAUDE.md`, `voice*.md` and `.claude/` (deny rules hold even in full access); Bash stays off because it could edit those files around the Edit rules. Chats are normal threads in a `~/Chat` project (hidden from the sidebar), so `~/Chat/CLAUDE.md`, `voice.md`, `voice-samples.md`, `people.md` and Claude's memory shape every reply. Picking a tone prefixes `Tone: <Name>`, which `CLAUDE.md` maps to that `## <Name>` section of `voice.md`.

## Notes

- Builds are unsigned. `install` strips the quarantine flag; if you download a DMG by hand, run `xattr -dr com.apple.quarantine "/Applications/T3 Code (Nightly).app"`.
- The app's updater points at this fork, so it never "updates" back to stock. In-app updates of unsigned macOS builds can fail, so use `install` instead.
- The custom app replaces the stock `T3 Code (Nightly).app` and shares its data (`~/.t3`).
- Reliability audit and hardening plan (CI verification, 3-way merges, signing/updater, rollback): [docs/RELIABILITY.md](docs/RELIABILITY.md).
