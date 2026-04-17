# Fork Plan: ZS Code

Steps to fork this repository to your GitHub account, push your customizations, and produce a tagged build.

## 1. Pre-work ✅

- ~~Review `git status` and `git diff`.~~ Done.
- ~~`.github/copilot-instructions.md` modification — turns out `.claude/CLAUDE.md` is a symlink to it, so the build-notes section we wrote is supposed to be there. Include it.~~ Confirmed.
- ~~Create a branch so you don't commit directly on `main`:~~ Done — on branch `zs-customizations`.

```bash
git checkout -b zs-customizations
```

## 2. Fork on GitHub ✅

Forked to <https://github.com/zacharysnewman/zscode>.

## 3. Wire up remotes

Rename the Microsoft remote to `upstream` and add your fork as `origin`:

```bash
git remote rename origin upstream
git remote add origin https://github.com/zacharysnewman/zscode.git
git remote -v   # verify
```

## 4. Commit the work

One big commit or split semantically. Suggested split:

```bash
git add product.json resources/darwin/code.icns
git commit -m "Rebrand to ZS Code"

git add extensions/theme-defaults/ src/vs/workbench/services/themes/common/workbenchThemeService.ts
git commit -m "Add ZS Dark theme and make it the default"

git add src/vs/workbench/browser/parts/
git commit -m "Cursor-style horizontal activity bar with count cap and chevron overflow"

git add src/vs/workbench/contrib/welcomeGettingStarted/
git commit -m "Swap Welcome tab icon to sparkle"
```

## 5. Push branch to your fork

```bash
git push -u origin zs-customizations
```

## 6. Tag the commit

Annotated tag (recommended — carries metadata and message):

```bash
git tag -a v0.1.0 -m "ZS Code v0.1.0 - first custom build"
```

## 7. Push the tag

```bash
git push origin v0.1.0
```

After this, the tag appears on GitHub under **Tags**. That's a "build tag" in the git sense.

## 8. (Optional) Create a GitHub Release

A git tag alone is just a ref. A GitHub **Release** is a UI object attached to a tag, with a description and downloadable artifacts:

```bash
gh release create v0.1.0 \
  --title "ZS Code v0.1.0" \
  --notes "Custom VS Code fork. Cursor-style horizontal activity bar, green ZS Dark theme, rebrand to ZS Code."
```

## 9. Producing an actual built binary (optional, non-trivial)

A tag or release on GitHub does **not** automatically produce a `.app` / `.dmg` / `.exe`. For that you need:

- A GitHub Actions workflow that runs on `push: tags: ['v*']`
- The workflow runs `npm install`, `npm run compile-build` (or similar), then gulp targets like `vscode-darwin-arm64` to produce `.app`, and uses `gh release upload` to attach artifacts to the release.
- Realistic pain points:
  - Electron build output is large (>1 GB).
  - macOS code-signing + notarization needs an Apple Developer ID and secrets stored in GitHub Actions secrets. Not required for personal use — unsigned builds work locally, but macOS Gatekeeper will warn on first launch.
  - Build time on GitHub's default runners is substantial (30+ minutes).
