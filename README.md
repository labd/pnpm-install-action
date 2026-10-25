# pnpm-install-action

Composite GitHub Action that sets up a Node.js + pnpm project in CI in one step:

1. Installs Node.js with `actions/setup-node` (version from input, `.nvmrc` / `.node-version`, or `24`)
2. Installs pnpm with `pnpm/action-setup` (version from input or the `packageManager` field in `package.json`)
3. Restores/saves the pnpm store with `actions/cache` (optional)
4. Runs `pnpm audit --audit-level=critical --ignore-registry-errors` (optional)
5. Runs `pnpm install --frozen-lockfile`

This is the Lab Digital standard for Node.js + pnpm setup. It replaces separate
`actions/setup-node` / `pnpm/action-setup` / cache / install steps and
`labd/gh-actions-typescript/pnpm-install`.

## Usage

```yaml
steps:
  - uses: actions/checkout@<sha> # vX.Y.Z
    with:
      persist-credentials: false

  - uses: labd/pnpm-install-action@<sha> # v1.0.0

  - run: pnpm test
```

Pin the action to a commit SHA (`pinact run` does this for you).

### Monorepo / sub directory

```yaml
- uses: labd/pnpm-install-action@<sha> # v1.0.0
  with:
    cwd: frontend
    node-version: "22"
```

### Release / publish workflows

Do not use the dependency cache in workflows that publish packages or create
releases: a poisoned cache entry written by another workflow could end up in
your release (see zizmor's
[cache-poisoning](https://docs.zizmor.sh/audits/#cache-poisoning) audit).
Set `registry-url` so `actions/setup-node` writes an `.npmrc` that authenticates
with `NODE_AUTH_TOKEN` (changesets/action v2 no longer writes one for you).

```yaml
name: Release

on:
  push:
    branches: [main]

permissions: {}

jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pull-requests: write
      id-token: write
    steps:
      - uses: actions/checkout@<sha> # vX.Y.Z
        with:
          persist-credentials: false

      - uses: labd/pnpm-install-action@<sha> # v1.0.0
        with:
          cache: "false"
          registry-url: https://registry.npmjs.org

      - uses: changesets/action@<sha> # vX.Y.Z
        with:
          publish: pnpm publish-packages
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
```

## Inputs

All inputs are optional.

| Input | Default | Description |
| --- | --- | --- |
| `node-version` | `""` | Node.js version spec (e.g. `24`, `22.x`). When empty, `node-version-file` is used, then `<cwd>/.nvmrc`, then `<cwd>/.node-version`, and finally `24`. |
| `node-version-file` | `""` | File with the Node.js version, relative to the repository root (`.nvmrc`, `.node-version`, `.tool-versions`, `package.json`, ...). Ignored when `node-version` is set. |
| `pnpm-version` | `""` | pnpm version to install. When empty, the `packageManager` field of `<cwd>/package.json` is used (one of the two is required). |
| `cwd` | `.` | Directory containing `package.json` and `pnpm-lock.yaml`, relative to the repository root. Used for the pnpm version lookup, audit and install. |
| `cache` | `true` | Cache the pnpm store with `actions/cache`. Set to `false` in release/publish workflows; no cache step runs at all then. |
| `frozen-lockfile` | `true` | `true` runs `pnpm install --frozen-lockfile`, `false` runs `pnpm install --no-frozen-lockfile`. |
| `audit` | `true` | Run `pnpm audit --audit-level=critical --ignore-registry-errors` before installing. |
| `registry-url` | `""` | Registry URL passed to `actions/setup-node`, e.g. `https://registry.npmjs.org`. Writes an `.npmrc` that reads the token from `NODE_AUTH_TOKEN`. |

## Outputs

| Output | Description |
| --- | --- |
| `node-version` | The installed Node.js version (e.g. `v24.11.0`). |
| `pnpm-store-path` | Path of the pnpm store. |
| `cache-hit` | `true` when the pnpm store cache had an exact key match; empty when `cache` is `false`. |

## Migrating

From `labd/gh-actions-typescript/pnpm-install`:

```diff
-      - uses: labd/gh-actions-typescript/pnpm-install@<sha> # vX
+      - uses: labd/pnpm-install-action@<sha> # v1.0.0
         with:
           node-version: "24"
```

Note that this action runs `pnpm install --frozen-lockfile` by default (the old
action ran a plain `pnpm install`); set `frozen-lockfile: "false"` if you relied on that.

From hand-written steps, replace `actions/setup-node`, `pnpm/action-setup`, the
`pnpm store path` / `actions/cache` steps and `pnpm install` with this action.
If your workflow used `actions/setup-node` with `cache: pnpm`, drop that too.

## Caching details

The cache key is `<os>-<arch>-pnpm-store-<hash of <cwd>/**/pnpm-lock.yaml>`,
with `<os>-<arch>-pnpm-store-` as restore key. `actions/setup-node`'s own
package manager cache is disabled.
