# Reliability: keeping the custom build alive with less babysitting

Audit date: 2026-10-16. Upstream reference: `v0.1.1-nightly.20261016.3190`.
Line numbers refer to files in this repo unless prefixed with `upstream:`.

## Do these first

1. **Reinstall the custom app** — `/Applications/T3 Code (Nightly).app` is currently *stock* (0.0.46-nightly.20261015.3158, Apple-signed, updater -> pingdotgg). Run `bin/t3code-customize install`. (P0, 2 min)
2. **Make CI 3-way merges actually work** — `--depth=1` in the workflow means `git am --3way` silently degrades to plain apply. Fetch the patch base tag too. (P0, 15 min) -> [2.1](#21-3-way-merge-is-dead-in-ci)
3. **Typecheck after `git am`, before the DMG build** — a patch that applies but no longer compiles ships a broken app today. (P0, 30 min) -> [3.1](#31-no-verification-after-git-am)
4. **Stop in-app updates from failing forever** — ad-hoc signature changes every build, Squirrel.Mac rejects it. Pick: stable self-signed cert (P1, half a day) or a launchd `install` agent (P1, 30 min). -> [5](#5-updater-and-signing)
5. **Make `status` tell you which app is installed** so (1) can't happen silently again. (P1, 10 min) -> [6.1](#61-detect-the-stock-app-overwriting-the-custom-one)

---

## 1. State of things (verified 2026-10-16)

| Item | Status |
| --- | --- |
| CI | Healthy. Runs every ~2h; last publish `v0.1.1-nightly.20261016.3190` at 07:13Z. |
| GitHub "5 days ago / 5705 commits behind" | Expected. `customize` holds only patches; it never contains upstream source. **Never click "Sync fork"** — it would merge upstream `main` into this branch. |
| Updater config in CI build | Correct. `Contents/Resources/app-update.yml` has `owner: krishnavalliappan, repo: t3code, provider: github, releaseType: prerelease, channel: nightly`; releases carry `nightly-mac.yml` + zip + blockmaps. The custom app checks *this fork*. |
| Installed app | **Stock**, not custom. `0.0.46-nightly.20261015.3158`, `TeamIdentifier=ARK85ZXQ4Z`, app-update.yml owner `pingdotgg`. Either the custom build was never installed or stock replaced it. |
| CI build signature | `Signature=adhoc` (linker-signed). In-app updates **will fail to install**: Squirrel.Mac validates the new bundle against the running app's designated requirement; an ad-hoc DR is just a cdhash, different per build. |
| Patch vs. newest upstream | Applies cleanly (direct application) against `20261016.3190`; the mount hunk has already drifted from line 11892 to 11732 in `ChatView.tsx`. |

---

## 2. Patch application

### 2.1 3-way merge is dead in CI  (P0)

**Problem.** `.github/workflows/custom-nightly.yml:90` fetches `--depth=1`. `git am --3way` needs the *pre-image blobs* recorded in the patch's `index 87d1303e0..` line. With only the tag's tree present, those blobs exist only when the file is unchanged — in which case the patch applied cleanly anyway. Verified: `git apply --check --3way` on a shallow checkout prints `repository lacks the necessary blob to perform 3-way merge. Falling back to direct application`. So every ChatView drift that a 3-way would have absorbed becomes an issue + skipped build.

**Recommendation.** Record the base tag at `save` time and fetch it in CI too (two shallow fetches, both trees present).

```sh
# bin/t3code-customize, cmd_save: after format-patch
echo "$base" > "$PATCHES_DIR/BASE"

# custom-nightly.yml, "Checkout upstream" step
base="$(cat .customize/patches/BASE 2>/dev/null || true)"   # move checkout@v4 above this step
git fetch --no-tags --depth=1 upstream "refs/tags/${TAG}:refs/tags/${TAG}" \
  ${base:+"refs/tags/${base}:refs/tags/${base}"}
```

Alternative: `git clone --filter=blob:none` (as `bin/t3code-customize:39` already does locally) — lazy blob fetch lets 3-way find the pre-image. Either way, keep `--3way`; also add `git am --3way --reject`-style diagnostics to the issue body only on real conflicts.

### 2.2 Patch hygiene  (P1)

Rules, in priority order:

1. **Almost all code in new files.** Today: 117 lines in `chat/ComposerStatusBar.tsx`, 6 lines in `ChatView.tsx`. Good — keep that ratio.
2. **Touch upstream files only at seams: one import + one mount.** The mount currently passes three props sourced from `ChatView` locals (`activeProviderStatus`, `activeContextWindow`, `serverProjection`). Each prop name is a conflict surface when upstream renames. If those values are reachable through existing hooks/atoms, make the mount prop-less (`<ComposerStatusBar threadId={threadId} />`) and resolve data inside the new file.
3. **Prefer less-churny mount points.** `ChatView.tsx` is 12,000 lines and moves every nightly. Check whether `chat/ChatComposer.tsx` or `ComposerSurface` can host the bar; a 300-line file drifts far less than a 12k one.
4. **One concern per patch / per commit** on the `custom` branch. `save` maps commits 1:1 to files.
5. **Layer ordering + naming via commit subject.** `format-patch` numbers by commit order and slugs the subject, so order commits infra-first and prefix subjects:
   - `infra(build): allow custom mac signing identity` -> `0001-infra-build-...patch`
   - `feat(web): composer status bar` -> `0002-feat-web-...patch`
   Don't hand-rename to `0010-`; `save` regenerates names and would clobber.
6. **Seam header in every commit message**, so a conflict issue is self-explanatory:
   ```
   feat(web): composer status bar with 5h/weekly usage, context, tok/s

   Seams:
   - apps/web/src/components/ChatView.tsx: import block (top); mount directly after </ComposerSurface.Shell>
   New files: apps/web/src/components/chat/ComposerStatusBar.tsx
   ```

### 2.3 `save` can wipe `patches/`  (P1)

**Problem.** `bin/t3code-customize:88` runs `rm -f patches/*.patch` *before* `format-patch`. If `$base..custom` is empty (e.g. `edit` was run but nothing committed) or `format-patch` fails, you are left with zero patches; a push then publishes pristine upstream as a "custom" build.

**Fix.**

```sh
local n; n="$(git -C "$SRC" rev-list --count "$base..custom")"
[ "$n" -gt 0 ] || die "No commits on custom since $base; nothing to save."
local tmp; tmp="$(mktemp -d)"
git -C "$SRC" format-patch --quiet --zero-commit --no-signature -o "$tmp" "$base..custom"
rm -f "$PATCHES_DIR"/*.patch && mv "$tmp"/*.patch "$PATCHES_DIR"/
```

### 2.4 `edit` silently discards unsaved commits  (P1)

**Problem.** `bin/t3code-customize:53` does `git checkout -B custom "$tag"`. The dirty-tree check at line 48 only covers *uncommitted* changes; commits on `custom` that were never `save`d are dropped (recoverable only via reflog).

**Fix.** Before `-B`, compare commits against the saved patches:

```sh
if git -C "$SRC" rev-parse -q --verify custom >/dev/null && [ -f "$BASE_FILE" ]; then
  local unsaved; unsaved="$(git -C "$SRC" rev-list --count "$(cat "$BASE_FILE")..custom")"
  local saved; saved="$(ls "$PATCHES_DIR"/*.patch 2>/dev/null | wc -l | tr -d ' ')"
  [ "$unsaved" -le "$saved" ] || [ "${FORCE:-}" = 1 ] ||
    die "custom has $unsaved commits but patches/ has $saved. Run 'save' first, or FORCE=1 $0 edit."
fi
```

---

## 3. CI verification

### 3.1 No verification after `git am`  (P0)

**Problem.** `custom-nightly.yml:99-110` applies patches, then goes straight to a ~40-minute macOS DMG build (`:138`). A patch that applies but no longer typechecks (upstream renamed `ServerProviderUsageWindow`, moved `~/lib/contextWindow`, ...) ships a broken app. Nothing opens an issue.

**Recommendation.** Add a cheap `verify` job on `ubuntu-latest` between `resolve` and `build`: apply patches, install, `ensure:electron`, typecheck the touched packages (full typecheck as a fallback), run the touched packages' unit tests. Fail -> open an issue with the same dedupe pattern as the conflict step. Upstream's own release flow does exactly `ensure:electron` -> `vp run typecheck` (`upstream:.github/workflows/release.yml:257-263`).

```yaml
  verify:
    needs: resolve
    if: needs.resolve.outputs.should_build == 'true'
    runs-on: ubuntu-latest
    timeout-minutes: 25
    env: { TAG: ${{ needs.resolve.outputs.tag }}, GH_TOKEN: ${{ github.token }} }
    steps:
      # ...same "Checkout upstream" + checkout@v4 (.customize) + "Apply patches" steps as build;
      #    factor them into a composite action in .customize/.github/actions/apply-patches
      - uses: voidzero-dev/setup-vp@v1
        with: { node-version-file: package.json, cache: true, run-install: true }
      - run: vp run --filter @t3tools/desktop ensure:electron
      - name: Typecheck + test touched packages
        id: check
        run: |
          set -euo pipefail
          # apps/web -> @t3tools/web, apps/server -> t3, packages/x -> @t3tools/x
          mapfile -t pkgs < <(git diff --name-only "${TAG}..HEAD" | cut -d/ -f1-2 | sort -u \
            | while read -r d; do [ -f "$d/package.json" ] && jq -r .name "$d/package.json"; done)
          filters=(); for p in "${pkgs[@]}"; do filters+=(--filter "$p"); done
          if [ ${#filters[@]} -eq 0 ]; then vp run typecheck; exit 0; fi
          vp run "${filters[@]}" typecheck
          vp run "${filters[@]}" test
      - name: Report verification failure
        if: failure() && steps.check.outcome == 'failure'
        run: |  # same gh issue create/dedupe as "Report patch conflict", title "Patches break typecheck/tests on ${TAG}"
          ...
  build:
    needs: [resolve, verify]
```

Cost: ~5-8 Linux minutes per new nightly (Linux minutes are 10x cheaper than macOS). Also consider `vp fmt --check $(git diff --name-only "${TAG}..HEAD")` so a drive-by format change doesn't become a conflict later (P2).

### 3.2 Early warning against upstream `main`  (P1)

**Problem.** You learn about a conflict only when it's already in a nightly tag.

**Recommendation.** Daily job: apply patches to upstream `main` HEAD on ubuntu, no build. Open issue "Patches will conflict with upstream main" on failure; close it on success. Gives ~a day's head start, and the issue body shows the exact failing hunk.

```yaml
# .github/workflows/upstream-canary.yml
on: { schedule: [{ cron: "41 6 * * *" }], workflow_dispatch: {} }
jobs:
  canary:
    runs-on: ubuntu-latest
    steps:
      - run: |
          git init -q . && git remote add upstream https://github.com/pingdotgg/t3code.git
          git fetch --no-tags --depth=1 upstream main && git checkout -q --detach upstream/main
      - uses: actions/checkout@v4
        with: { path: .customize }
      - id: apply
        run: git -c user.name=x -c user.email=x@x am --3way .customize/patches/*.patch
      - if: failure()   # open/dedupe issue (reuse the conflict step body)
      - if: success()   # gh issue close any open "Patches will conflict with upstream main"
```

Optional: run `vp run --filter @t3tools/web typecheck` here too; it's the cheap place to find API drift.

### 3.3 Auto-close conflict issues  (P1)

**Problem.** `custom-nightly.yml:112-123` opens "Patches don't apply on TAG" but nothing closes it; stale issues accumulate and the dedupe search (`:119`) is title-scoped per tag, so each new nightly opens another.

**Fix.** Title without the tag (`Patches don't apply to upstream nightly`), tag in body; after a successful apply:

```yaml
      - name: Close stale conflict issues
        if: steps.apply.outcome == 'success'
        run: |
          gh issue list -R "$GITHUB_REPOSITORY" --state open --search '"Patches don'"'"'t apply" in:title' \
            --json number --jq '.[].number' |
            xargs -r -I{} gh issue close {} -R "$GITHUB_REPOSITORY" -c "Patches apply again on ${TAG}."
```

### 3.4 Publish step: delete-then-create gap  (P1)

**Problem.** `custom-nightly.yml:158-160` deletes the release, then creates it. For a forced rebuild of an already-published tag, there's a window (seconds to a minute of asset upload) where the fork's "latest" release is the *previous* tag, and if `gh release create` fails you've lost the build with no fallback. electron-updater reads the fork's `releases.atom` and picks the newest entry, so during that gap the running app may see an older version and — since the nightly channel sets `allowDowngrade: true` (`upstream:apps/desktop/src/updates/DesktopUpdates.ts:470`) — try to downgrade.

**Fix.** Update in place:

```sh
if gh release view "$TAG" -R "$GITHUB_REPOSITORY" >/dev/null 2>&1; then
  gh release upload "$TAG" -R "$GITHUB_REPOSITORY" --clobber "${files[@]}"
  gh release edit   "$TAG" -R "$GITHUB_REPOSITORY" --prerelease --notes "$notes"
else
  gh release create "$TAG" -R "$GITHUB_REPOSITORY" --prerelease --target "${{ github.sha }}" \
    --title "T3 Code Custom ${VERSION}" --notes "$notes" "${files[@]}"
fi
```

Also drop `builder-debug.yml` from the upload (`:155` glob `release/*.yml` catches it; it's internal electron-builder config, not needed by the updater): use `release/nightly-mac.yml` explicitly.

### 3.5 Two different "latest nightly" resolvers  (P2)

`custom-nightly.yml:54` uses `gh release list` on upstream (published releases, newest-created first). `bin/t3code-customize:30-33` uses `git ls-remote --tags | sort -V`. Tags appear before upstream's release finishes (right now the tag `v0.1.1-nightly.20261016.3203` exists with no release yet), so `status`/`edit`/`build` may target a nightly CI never builds, and `status` shows "upstream ahead" when it isn't. Pick one — `gh release list` — in both places. Bonus: `sort -V` would also misorder if upstream ever changed the tag scheme.

### 3.6 `install` picks by creation date, not version  (P2)

`bin/t3code-customize:120` uses `gh release list --limit 1`. After a forced rebuild of an older tag (workflow_dispatch with `tag:`), that release is newest-created and `install` downgrades you. Sort by version:

```sh
tag="${arg:-$(gh release list -R "$fork" --limit 50 --json tagName --jq '.[].tagName' | sort -V | tail -1)}"
```

---

## 4. Local workflow

### 4.1 `check` command  (P1)

Apply to latest nightly + typecheck, no DMG, no `/Applications` touch. Use a throwaway worktree so it never disturbs an in-progress `edit`.

```sh
cmd_check() {
  local tag="${1:-$(latest_nightly_tag)}"
  ensure_src
  git -C "$SRC" fetch --no-tags origin "refs/tags/$tag:refs/tags/$tag"
  local wt; wt="$(mktemp -d)"
  git -C "$SRC" worktree add -q --detach "$wt" "$tag"
  trap 'git -C "$SRC" worktree remove --force "$wt"' EXIT
  log "Applying patches to $tag"
  git -C "$wt" -c user.name=check -c user.email=check@local am --3way "$PATCHES_DIR"/*.patch
  ( cd "$wt" && pnpm install --frozen-lockfile && pnpm --filter @t3tools/desktop ensure:electron \
      && pnpm --filter @t3tools/web typecheck && pnpm --filter @t3tools/web test )
  log "OK: patches apply and typecheck on $tag"
}
```

Run it before `save`/push and whenever the canary (3.2) warns.

### 4.2 `install <tag>` for rollback  (P1)

Already works: `cmd_install` accepts a tag (`bin/t3code-customize:120`). Document it in the README and make `status` print the last 5 fork releases so you can see what to roll back to. Pair with retention (4.3) so the tag still exists.

### 4.3 Release retention + last-known-good  (P1)

**Problem.** A release every new upstream nightly (~5/day, 320 MB each). Hundreds of prereleases in weeks; `gh release list --limit 1` and the updater's atom feed get noisier; storage is free but the release page becomes useless.

**Recommendation.** Keep the newest 15 plus anything marked `lkg`. Mark via notes, since releases have no labels:

```sh
# bin/t3code-customize bless [TAG]   -> appends "lkg: true" to the installed/selected release notes
gh release edit "$tag" -R "$fork" --notes "$(gh release view "$tag" -R "$fork" --json body --jq .body)"$'\n'"lkg: true"
```

```yaml
      - name: Prune old prereleases
        run: |
          gh release list -R "$GITHUB_REPOSITORY" --limit 200 --json tagName,body \
            --jq '[.[] | select(.body | test("lkg: true") | not)] | .[15:] | .[].tagName' |
            xargs -r -I{} gh release delete {} -R "$GITHUB_REPOSITORY" --yes --cleanup-tag
```

`--cleanup-tag` matters: `gh release create --target` creates a tag in the fork each time (`:160`), so without it tags pile up too.

---

## 5. Updater and signing

### 5.1 Why in-app updates fail today  (context)

- electron-updater on macOS hands the downloaded zip to Squirrel.Mac (Electron's native `autoUpdater`). Squirrel.Mac verifies the new bundle satisfies the **running** app's designated requirement (DR).
- The CI build is ad-hoc signed; its DR is `cdhash H"..."`, unique per build. New build != old DR -> "code failed to satisfy specified code requirement(s)" -> install never completes.
- The app polls every 4 minutes (`upstream:DesktopUpdates.ts:53`) and will keep downloading/failing. The feed itself is correct (fork, prerelease, channel nightly).

### 5.2 Option A: stable self-signed certificate  (P1, recommended)

Same cert every build -> same DR -> Squirrel accepts custom -> custom updates. No notarization needed. One-time setup, then hands-off.

**Constraint found in the build script.** `scripts/build-desktop-artifact.ts` has only two modes:
- unsigned (default): `upstream:build-desktop-artifact.ts:3984-3990` forces `CSC_IDENTITY_AUTO_DISCOVERY=false` and *deletes* `CSC_LINK`/`CSC_KEY_PASSWORD`, so exporting those secrets alone does nothing;
- `--signed`: requires `T3CODE_APPLE_TEAM_ID`, a provisioning profile and a Clerk passkey config (`:1270-1300`, `:3801-3806`) — the official pipeline, not usable with a self-signed cert.

So this needs a small **infra patch** (that's what layer `0001-infra-*` is for) plus CI secrets.

**Steps.**

1. Create the cert once, locally. Name it so electron-builder's identity lookup accepts it (it filters `security find-identity` output by type prefix; "Developer ID Application" is the first it tries):
   ```sh
   openssl req -x509 -newkey rsa:2048 -nodes -days 3650 -keyout key.pem -out cert.pem \
     -subj "/CN=Developer ID Application: Krishna Custom" \
     -addext "keyUsage=critical,digitalSignature" -addext "extendedKeyUsage=critical,codeSigning" \
     -addext "basicConstraints=critical,CA:false"
   openssl pkcs12 -export -inkey key.pem -in cert.pem -out t3custom.p12 -passout pass:"$P12_PW"
   # trust it on your Mac so Gatekeeper/codesign treat it as valid
   sudo security add-trusted-cert -d -r trustRoot -p codeSign cert.pem
   base64 -i t3custom.p12 | pbcopy    # -> secret CSC_LINK (electron-builder accepts base64 p12)
   ```
   Secrets: `CSC_LINK` (base64 p12), `CSC_KEY_PASSWORD`, plus `T3CODE_CUSTOM_MAC_CERT_PEM` (the public cert, to trust on the runner).
2. Infra patch to `scripts/build-desktop-artifact.ts` (two seams, ~6 lines):
   ```ts
   // ~:2854 inside buildConfig.mac
   ...(process.env.T3CODE_CUSTOM_MAC_IDENTITY
     ? { identity: process.env.T3CODE_CUSTOM_MAC_IDENTITY, hardenedRuntime: true, gatekeeperAssess: false, notarize: false }
     : {}),
   // ~:3984
   if (!options.signed && !process.env.T3CODE_CUSTOM_MAC_IDENTITY) {
   ```
   Keep `--signed` untouched; the custom path is "unsigned mode + explicit identity".
3. Workflow, before the build step (runner has passwordless sudo):
   ```yaml
   - name: Import custom signing cert
     env: { CSC_LINK: ${{ secrets.CSC_LINK }}, CSC_KEY_PASSWORD: ${{ secrets.CSC_KEY_PASSWORD }}, CERT_PEM: ${{ secrets.T3CODE_CUSTOM_MAC_CERT_PEM }} }
     run: |
       set -euo pipefail
       echo "$CERT_PEM" > "$RUNNER_TEMP/custom.pem"
       sudo security add-trusted-cert -d -r trustRoot -p codeSign "$RUNNER_TEMP/custom.pem"
       # electron-builder imports CSC_LINK into its own temp keychain; the trust setting above is what
       # makes `security find-identity -v -p codesigning` list it as valid.
   - name: Build macOS arm64 DMG
     env:
       T3CODE_DESKTOP_UPDATE_REPOSITORY: ${{ github.repository }}
       T3CODE_CUSTOM_MAC_IDENTITY: "Developer ID Application: Krishna Custom"
       CSC_LINK: ${{ secrets.CSC_LINK }}
       CSC_KEY_PASSWORD: ${{ secrets.CSC_KEY_PASSWORD }}
   ```
4. Verify once: `codesign -dr - "/Applications/T3 Code (Nightly).app"` should print `designated => identifier "com.t3tools.t3code" and certificate leaf = H"<sha1>"` — the same on two consecutive builds. Then let the app update itself once and confirm.
5. First switch is manual: an installed ad-hoc (or stock) app will reject the first self-signed update. Run `install` once; after that updates flow.

**Trade-offs / gotchas.**
- Gatekeeper still flags *downloaded* self-signed apps; `install` already strips quarantine. Squirrel-written updates aren't quarantined (electron-updater writes via `fs`, not LaunchServices), but verify on the first real update.
- Cert expiry: set 10 years. Rotating the cert = DR change = one manual `install`.
- electron-builder may still attempt notarization heuristics; `notarize: false` and the deleted `APPLE_*` envs (`:3988-3990`) keep it off.
- Keep the p12 + password in a password manager; losing it = new cert = one manual reinstall, nothing worse.
- Until Option A lands, stop the 4-minute fail loop with `launchctl setenv T3CODE_DISABLE_AUTO_UPDATE 1` (read at `upstream:apps/desktop/src/app/DesktopConfig.ts:59`; applies to Dock/Finder launches after the next app start).

### 5.3 Option B: launchd agent that runs `install`  (P1, fallback or interim)

No signing work. Poll the fork; when a newer release exists and the app isn't running, install it.

```sh
# bin/t3code-customize autoupdate   (new subcommand; also installs the plist below)
installed="$(/usr/libexec/PlistBuddy -c 'Print CFBundleShortVersionString' "/Applications/$APP_NAME/Contents/Info.plist")"
latest="$(gh release list -R "$(fork_repo)" --limit 50 --json tagName --jq '.[].tagName' | sort -V | tail -1)"
[ "v$installed" = "$latest" ] && exit 0
pgrep -xq "T3 Code (Nightly)" && { echo "running; try later"; exit 0; }
cmd_install "$latest"
```

```xml
<!-- ~/Library/LaunchAgents/dev.krishna.t3code-autoupdate.plist -->
<plist version="1.0"><dict>
  <key>Label</key><string>dev.krishna.t3code-autoupdate</string>
  <key>ProgramArguments</key><array><string>/Users/krishnakumarvalliappan/IPAC-DEV/t3code-krishna/bin/t3code-customize</string><string>autoupdate</string></array>
  <key>StartInterval</key><integer>3600</integer>
  <key>EnvironmentVariables</key><dict><key>PATH</key><string>/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin</string></dict>
  <key>StandardOutPath</key><string>/tmp/t3code-autoupdate.log</string>
  <key>StandardErrorPath</key><string>/tmp/t3code-autoupdate.log</string>
</dict></plist>
```

Trade-offs: only updates when the app is closed (the app is usually open, so effectively "on next morning"); `install` already quits/relaunches, so you could allow it during idle hours instead. Needs `gh auth` in the launchd environment. Combine with `T3CODE_DISABLE_AUTO_UPDATE=1` so the in-app updater stops downloading 150 MB zips it can't install.

---

## 6. Guarding against the stock app

### 6.1 Detect the stock app overwriting the custom one  (P1)

Make `status` read the installed bundle; it's the cheapest guard and would have caught today's state immediately.

```sh
app="/Applications/$APP_NAME"
if [ -d "$app" ]; then
  v="$(/usr/libexec/PlistBuddy -c 'Print CFBundleShortVersionString' "$app/Contents/Info.plist")"
  owner="$(sed -n 's/^owner: //p' "$app/Contents/Resources/app-update.yml" 2>/dev/null)"
  sig="$(codesign -dv "$app" 2>&1 | sed -n 's/^TeamIdentifier=//p')"
  case "$owner" in
    "${fork%%/*}") echo "installed        : $v  (custom, updater -> $owner)";;
    *)             echo "installed        : $v  (STOCK: updater -> ${owner:-?}, team ${sig:-none}). Run: $0 install";;
  esac
fi
```

How stock ends up there: the stock app's updater points at pingdotgg and only ever updates *itself*; it cannot overwrite a custom install. So the current state means the custom build was never installed, or stock was installed over it by hand/DMG. The custom app's updater only sees the fork, so once custom is installed it stays custom (modulo 5.1).

### 6.2 Separate bundle ID / product name?  (P2, probably not)

Changing `DESKTOP_APP_ID` (`upstream:build-desktop-artifact.ts:59`) and `resolveDesktopProductName` (`:2746`) to e.g. `com.t3tools.t3code.custom` / `T3 Code (Custom)` is a 2-line infra patch and makes both apps coexist.

| For | Against |
| --- | --- |
| Stock DMG/updater can never touch the custom app | Electron `userData` (`~/Library/Application Support/<productName>`) forks: window state, renderer storage, Keychain items |
| Easy A/B against stock | Both register `t3code://` and `http(s)` handlers (`:2841-2852`); macOS picks one |
| | Both share `~/.t3` (server state, DB, worktrees); running both at once = two servers on the same data |
| | Passkey entitlements are tied to the appId (`:1366`) |

Verdict: keep the shared name; rely on 6.1 + 5.2/5.3. Revisit only if you want to run stock and custom side by side.

---

## 7. Smaller things noticed

| Where | Issue | Fix |
| --- | --- | --- |
| `custom-nightly.yml:49` | Every push to `patches/**` forces a rebuild even if the patch hash is unchanged (e.g. whitespace-only commit). | Compute `FORCE` only from `inputs.force`; the hash check already handles real changes. |
| `custom-nightly.yml:155` | `release/*.yml` uploads `builder-debug.yml`. | Upload `release/nightly-mac.yml` explicitly (see 3.4). |
| `bin/t3code-customize:105-107` | `osascript quit` + `sleep 2` + `rm -rf`. Slow quits leave a half-deleted bundle. | `for i in $(seq 20); do pgrep -xq "T3 Code (Nightly)" \|\| break; sleep 0.5; done`. |
| `bin/t3code-customize:124-125` | Falls back to any `*.dmg` — on an Intel Mac that installs an arm64 build. | `die` if `uname -m` is `x86_64` and no x64 asset; CI only builds arm64 anyway. |
| `bin/t3code-customize:144` | `git checkout -q -- .` after build also reverts any uncommitted edits in `$SRC` (apply_patches already requires a clean tree, so safe today; fragile if that check moves). | Restrict to the files the version script touches: `git checkout -q -- '**/package.json'`. |
| README "Notes" | Says in-app updates "can fail"; they *always* fail with ad-hoc signing. | Reword once 5.2/5.3 is chosen; add the "never Sync fork" note. |
| `patches/*.patch` | `Co-Authored-By` trailer and author email are published in every release body (`git log --format='- %s'` is subject-only, so only the subject leaks; fine). | None needed. |

---

## 8. Suggested order of work

| # | Item | Effort | Payoff |
| --- | --- | --- | --- |
| 1 | `bin/t3code-customize install` + 6.1 `status` check | 15 min | Actually running the custom app |
| 2 | 2.1 fetch base tag for 3-way; 2.3/2.4 `save`/`edit` guards | 45 min | Fewer false conflicts; no lost work |
| 3 | 3.1 verify job (typecheck + web tests) | 1 h | No broken DMGs |
| 4 | 3.3 auto-close, 3.4 in-place publish, 3.6 version-sorted install | 45 min | Less noise, no downgrade window |
| 5 | 5.3 launchd agent + `T3CODE_DISABLE_AUTO_UPDATE` | 30 min | Hands-off updates (coarse) |
| 6 | 5.2 self-signed cert + infra patch | half day | Real in-app updates |
| 7 | 3.2 canary, 4.1 `check`, 4.3 retention/`bless` | 1-2 h | Early warning, rollback safety |
