# bumpy + GitHub Packages — live reproduction

Reproduces a [`@varlock/bumpy`](https://github.com/dmno-dev/bumpy) **1.14.0** bug:
a package published to **GitHub Packages** (`https://npm.pkg.github.com/`) is
labelled `npm` in the GitHub release notes, with a badge linking to **npmjs.com**
where the package does not exist (404).

This repo runs the **real** two-phase bumpy CI flow on GitHub Actions, not a
local simulation.

## The bug, observed

`@shtian/my-pkg@1.3.0` is published to GitHub Packages
(`https://github.com/Shtian/bumpy-github-packages-repro/packages`), yet its
release notes say:

```
#### Published to
- ✅ [npm](https://www.npmjs.com/package/@shtian/my-pkg/v/1.3.0)
```

That npmjs.com URL 404s — the package is on GitHub Packages, not npm.
Release: <https://github.com/Shtian/bumpy-github-packages-repro/releases>

## Setup

- `packages/my-pkg/package.json` — `@shtian/my-pkg`, `publishConfig.registry` =
  `https://npm.pkg.github.com/`
- `.bumpy/_config.json` — same registry under `packages`
- `.github/workflows/bumpy-release.yml` — the release workflow
- `.github/workflows/bumpy-check.yml` — PR bump-file check

## The flow (how to reproduce)

1. Add a bump file and push to `main`:
   ```bash
   npx bumpy add --packages "@shtian/my-pkg:minor" --message "..."
   git commit -am "..." && git push
   ```
   → workflow runs `bumpy ci release` → **mode `version-pr`** → opens the
   **"🐸 Versioned release"** PR (applies the bump, deletes the bump file).
2. Merge that PR.
   → workflow runs `bumpy ci release` → **mode `publish`** → `npm publish` to
   GitHub Packages + creates the GitHub release.
3. Open the release notes → the `npm` badge links to npmjs.com (404).

### Repo settings required for the flow

- **Settings → Actions → General → Workflow permissions:** "Read and write" +
  "Allow GitHub Actions to create and approve pull requests" (needed for step 1
  to open the PR).
- The publish step authenticates to GitHub Packages with the built-in
  `GITHUB_TOKEN` (`packages: write`); no extra secret required for a manual
  merge. `BUMPY_GH_TOKEN` is only needed to auto-trigger PR checks.

## Root cause (in installed `dist/`)

The configured `registry` is ignored at three sites:

1. `getPublishTargets` — `dist/status-*.mjs:228` — `targets.push({ type: "npm" })`
2. publish target push — `dist/publish-*.mjs:444` — `targets.push("npm")`
3. `buildPublishUrl` — `dist/publish-*.mjs:145` — `_registry` arg unused;
   `case "npm"` hardcodes `https://www.npmjs.com/package/...`

The actual publish goes to the correct registry — only the reported label/URL
is wrong.

## Suggested fix

When `pkgConfig.registry` is `npm.pkg.github.com`, label the target GitHub
Packages and build a `https://github.com/orgs/<org>/packages/npm/package/<pkg>`
URL. More generally, `buildPublishUrl` should honour its `registry` argument.
