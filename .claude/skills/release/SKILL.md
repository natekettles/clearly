---
name: release
description: Determine the next version, update the marketing site, and run the full release pipeline for Mac or iOS.
---

Cut a new release of Clearly. Mac and iOS ship independently — see `CLAUDE.md` "Versioning" and "Commit message rule". This skill picks a platform, derives the version from that platform's git tag history, and runs the matching release pipeline.

## Instructions

### Step 0: Pick the platform

Ask with `mcp__conductor__AskUserQuestion`:
- question: "Which platform is this release for?"
- header: "Release platform"
- multiSelect: false
- options:
  - "Mac (Sparkle + optional App Store)"
  - "iOS (TestFlight)"
  - "Both (Mac first, then iOS)"

If "Both", run the Mac flow end-to-end, then the iOS flow. If either platform fails, stop — do NOT auto-continue to the other.

### Step 1: Verify prerequisites

Mac flow:
1. `.env` exists at the project root. If not, stop and tell the user:
   "Missing `.env` file. Copy `.env.example` to `.env` and fill in APPLE_TEAM_ID, APPLE_ID, and SIGNING_IDENTITY_NAME."
2. `notarytool` keychain profile `AC_PASSWORD` works. If not, stop and tell the user to run:
   ```bash
   xcrun notarytool store-credentials "AC_PASSWORD" --apple-id "$APPLE_ID" --team-id "$APPLE_TEAM_ID" --password "<app-specific-password>"
   ```

iOS flow:
1. `.env` exists and contains `APPLE_TEAM_ID`.

Both flows:
3. Working tree clean (`git status --porcelain`). If dirty, stop and ask the user to commit or stash.
4. On the `main` branch. If not, stop.

### Step 2: Determine the next version

Pick the tag query and commit filter based on platform:

| Platform | Latest tag query | Commit scope filter |
|---|---|---|
| Mac | `git tag -l 'v*' \| grep -vE '^ios-' \| sort -V \| tail -1` | `^\[(mac\|shared)\]` |
| iOS | `git tag -l 'ios-v*' \| sort -V \| tail -1` | `^\[(ios\|shared)\]` |

Steps:
1. Get the latest tag for the platform (above).
2. Get commits since that tag:
   ```bash
   git log <latest_tag>..HEAD --oneline --format='%s'
   ```
3. **Un-scoped commit guard.** If ANY commit in that range does NOT start with `[mac]`, `[ios]`, `[shared]`, or `[chore]`, stop and list those commits to the user:
   > "Found commits without a scope prefix. Fix with `git commit --amend` or `git rebase -i` before releasing, or tell me to proceed anyway (commits will fall through the filter and may land in the wrong changelog)."
   Use `mcp__conductor__AskUserQuestion` to ask whether to halt or proceed anyway.
4. Filter commits with the platform's scope regex. If zero commits match after filtering, stop: "No commits scoped to <platform> since <tag>. Nothing to release." (Note: raw-range commits may still exist — they're for the *other* platform.)
5. Apply semver logic to the scoped commit list:
   - Any commit containing `feat:` or `feat(` after the scope → **minor** bump
   - All commits are `fix:` / `chore:` / `docs:` style → **patch** bump
   - Any commit contains `BREAKING CHANGE` or `!:` → ask the user
   - Ambiguous / no conventional-commit markers → ask via `mcp__conductor__AskUserQuestion`:
     - question: "Commits since the last release don't clearly indicate the version bump. What version should this release be?"
     - header: "Release version"
     - multiSelect: false
     - options: "Patch (X.Y.Z+1)", "Minor (X.Y+1.0)", "Major (X+1.0.0)", "Custom"

### Step 3: Confirm the version

Confirm before proceeding. Show the tag (`v<VERSION>` for Mac, `ios-v<VERSION>` for iOS) and the scoped commit list. Use `mcp__conductor__AskUserQuestion`:
- question: "Release as <TAG>? Commits included:\n<scoped commit list>"
- header: "Confirm release"
- multiSelect: false
- options:
  - "Yes, release <TAG>"
  - "Use a different version"
  - "Cancel"

If "Use a different version", ask for the version. If "Cancel", stop.

### Step 3.5: Update the changelog

Changelog file depends on platform:
- Mac → `CHANGELOG.md`
- iOS → `CHANGELOG-iOS.md`

1. Check if the changelog has an `## [Unreleased]` section with content.
2. If `## [Unreleased]` is empty or missing, draft entries from the **scoped** commit list (same regex used above):
   - **Rewrite each entry user-facing.** Don't echo commit messages. Describe what changed from the user's perspective.
   - Bad: "feat: synchronized scroll and fix editor font size"
   - Good: "Editor and preview scroll together so you always see what you're editing"
   - Strip the `[scope]` prefix and any `feat:`/`fix:` markers.
   - Drop entries with no user-visible impact (internal refactors, test harness updates).
   - Keep entries succinct — one line each, no technical jargon.
   - Confirm the drafted entries with `mcp__conductor__AskUserQuestion`.
3. Rename `## [Unreleased]` to `## [VERSION] - YYYY-MM-DD` (today's date).
4. Add a new empty `## [Unreleased]` above it.

### Step 4: Update version strings

Updates depend on platform:

**Mac:**
1. Edit `project.yml`. Update `MARKETING_VERSION` in all three Mac-side targets: `Clearly`, `ClearlyQuickLook`, `ClearlyCLI`. Do NOT touch `Clearly-iOS`.
2. Edit `website/index.html`. Update the `class="requires"` line:
   ```html
   <p class="requires">v<VERSION> &middot; Requires macOS Sonoma or later</p>
   ```
3. Commit:
   ```bash
   git add project.yml website/index.html CHANGELOG.md
   git commit -m "[mac] Update marketing site version to v<VERSION>"
   git push
   ```

**iOS:**
1. Edit `project.yml`. Update `MARKETING_VERSION` in the `Clearly-iOS` target only. Do NOT touch the Mac-side targets.
2. No website edit (iOS isn't on the marketing site yet).
3. Commit:
   ```bash
   git add project.yml CHANGELOG-iOS.md
   git commit -m "[ios] Update iOS version to v<VERSION>"
   git push
   ```

### Step 5: Run the release script

**Mac:**
```bash
./scripts/release.sh <VERSION>
```
Handles: xcodegen → archive → export → DMG → notarize → staple → git tag `v<VERSION>` → appcast → push → GitHub Release.

**iOS:**
```bash
./scripts/release-ios.sh <VERSION>
```
Handles: xcodegen → archive → upload to App Store Connect (→ TestFlight) → git tag `ios-v<VERSION>` → push tag.

Let each script run to completion. On failure, report the error and stop. Do NOT retry automatically.

### Step 6: App Store submission (Mac only, optional)

iOS stops at TestFlight for now — no App Store submission step.

For Mac, after the Sparkle release succeeds, ask:
- question: "Sparkle release complete. Also submit v<VERSION> to the App Store?"
- header: "App Store"
- multiSelect: false
- options: "Yes, submit to App Store", "No, skip App Store"

If yes:

#### 6a: Generate App Store copy

Output three blocks as **raw plain text** (no markdown, no code fences) so the user can paste into App Store Connect:

1. **What's New in This Version** — Consolidate all entries from `CHANGELOG.md` from v1.0.0 through the current release. Use `•` bullets. Each entry: feature name em-dashed with a short description. The release script sets the per-version "What's New" automatically; this cumulative version is for the listing body.

2. **Promotional Text** (170 characters max) — One sentence. Tone: confident, no fluff.

3. **Description** — Full App Store description. Structure:
   - Opening one-liner about Clearly
   - "No Electron. No bloat. No subscription." positioning line
   - 4-5 short paragraphs, each with a leading phrase, covering: editing, preview, media/diagrams/math, export, native macOS integration
   - Bullet list of current features
   - Close with "One-time purchase. No subscription."

Label each block so the user knows which ASC field it's for.

#### 6b: Run the App Store release script

```bash
./scripts/release-appstore.sh <VERSION>
```

Handles: strip Sparkle from `project.yml` → archive → export → upload → wait for processing → create version → set "What's New" from `CHANGELOG.md` → attach build → submit for App Review.

On failure after upload, the build is already in ASC — tell the user they can finish manually.

### Step 7: Push and report

Ensure all commits are on the remote:
```bash
git push
```

Tell the user:
- Platform and version released
- Link:
  - Mac: `https://github.com/Shpigford/clearly/releases/tag/v<VERSION>`
  - iOS: no public release page; direct the user to App Store Connect → TestFlight
- Whether App Store submission was included (Mac only)

## Important Rules

- ALWAYS confirm the version before proceeding
- NEVER run a release script if `.env` is missing or the working tree is dirty
- NEVER skip the changelog update
- NEVER update both `CHANGELOG.md` and `CHANGELOG-iOS.md` in the same release — one platform, one changelog
- NEVER bump Mac `MARKETING_VERSION` entries during an iOS release (and vice versa)
- If the release script fails, do NOT retry — report the error and stop
- The release scripts handle git tagging — do not duplicate those steps
- Un-scoped commits (no `[mac]`/`[ios]`/`[shared]`/`[chore]` prefix) halt the release until resolved
