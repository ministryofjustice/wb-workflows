# Website Builder workflows

A shared GitHub Actions workflow that builds a WordPress plugin or theme into a
zip and publishes it as a GitHub release, so sites can install compiled assets
without committing them.

It has two modes:

| Mode | Trigger in the calling repo | What it does |
|---|---|---|
| `release` | push to `main` | Releases `X.Y.Z` when the `Version:` header has been bumped to an untagged version. Skips if the version is already tagged; fails if it's lower than the latest release. |
| `preview` | manual (`workflow_dispatch`) on any branch | Builds the branch into a pre-release tagged `preview-<branch>-<short sha>`, with `Version:` in the zip stamped `X.Y.Z-preview.<short sha>`. Never marked as latest. |

## Using it in a plugin or theme

Each repo needs three files.

**`.github/workflows/release.yml`**

```yaml
name: Release

on:
  push:
    branches: [main]

permissions:
  contents: write

jobs:
  release:
    uses: ministryofjustice/wb-workflows/.github/workflows/wb-package.yml@main
    with:
      mode: release
      version-file: style.css   # or the plugin's main PHP file
      package-name: my-theme    # folder name WordPress installs it as
```

**`.github/workflows/preview.yml`**: the same, with `on: workflow_dispatch:`
and `mode: preview`.

**`.release-filter`**: an [rsync filter](https://download.samba.org/pub/rsync/rsync.1#FILTER_RULES)
listing what goes in the zip. Rules are checked top to bottom and the first
match wins; `***` means a folder and everything in it, and end with `- *` so
nothing else is included:

```
+ /style.css
+ /functions.php
+ /theme.json
+ /composer.json
+ /templates/***
+ /dist/***
- *
```

The repo also needs a `composer.json` with `name` and `type`; preview notes use
them to print the Composer snippet for installing the preview.

## Inputs

| Input | Required | Default | |
|---|---|---|---|
| `mode` | yes | | `release` or `preview` |
| `version-file` | yes | | File with the WordPress `Version:` header |
| `package-name` | yes | | Folder name inside the zip and the zip filename prefix |
| `filter-file` | no | `.release-filter` | rsync filter file |
| `build-command` | no | `npm run build` | Run after `npm ci` |
| `node-version` | no | `22` | npm is upgraded to 11 to match lock files written by npm 11 |

Outputs: `published` (`'true'` if anything was published) and `tag`.

## Notes

- `permissions: contents: write` must be set in the calling workflow; a reusable
  workflow can't grant itself more than its caller has.
- Pin callers to a commit SHA (or a tag of this repo) rather than `@main` once
  things are stable.
- The "Run workflow" button for previews only appears once `preview.yml` is on
  the calling repo's default branch.
- Delete previews when testing is done, but not while an environment still
  installs them: `gh release delete <tag> --cleanup-tag --yes`.
