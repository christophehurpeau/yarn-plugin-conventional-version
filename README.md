<h3 align="center">
  yarn-plugin-conventional-version
</h3>

<p align="center">
  yarn berry versionning using conventional commits
</p>

# DEPRECATED

Due to the lack of support of ESM in yarn plugin, lack of yarn plugin update support, as well as pain to update patches for conventional-changelog packages using fs to read files, I've move this to a regular [npm package](https://www.npmjs.com/package/yarn-version) here: https://github.com/christophehurpeau/pob/tree/main/packages/yarn-version

This new version also fixes some bugs so I highly recommend the migration.

# OLD README

## Install

```bash
yarn plugin import https://raw.githubusercontent.com/christophehurpeau/yarn-plugin-conventional-version/main/bundles/@yarnpkg/plugin-conventional-version.cjs
yarn add --dev --exact conventional-changelog-conventionalcommits
```

## Usage

```bash
yarn version --help
```

## Options

`--bump-dependents-highest-as` is for updates in dependent packages. In a monorepo, when a package B has a peer dependency on package A, if A is updated with a breaking change (major), but B has no changes, B will be updated with this option. Sometimes, a major in A should update B with patch or minor. Sometimes, it should also be a major as it also breaks things for B. The default is `major`. If A is `minor` and the value is `major`, B will be updated as `minor`.

## Example with Github Action, for a simple repository

```yml
name: Release
on:
  workflow_dispatch:
    inputs:
      dry-run:
        description: "Dry run"
        required: true
        type: boolean
        default: false

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          token: ${{ secrets.GH_TOKEN }}
          fetch-depth: 0

      - uses: actions/setup-node@v3
        with:
          node-version: 18

      - name: Enable Corepack
        run: corepack enable

      - name: Install Dependencies
        run: yarn install --immutable --immutable-cache

      - name: New version (dry run)
        if: github.ref == 'refs/heads/main' && inputs.dry-run
        run: yarn version --dry-run

      - name: Configure Git user
        if: github.ref == 'refs/heads/main' && !inputs.dry-run
        run: |
          git config --global user.name 'github-actions[bot]'
          git config --global user.email 'github-actions[bot]@users.noreply.github.com'

      - name: New version
        if: github.ref == 'refs/heads/main' && !inputs.dry-run
        run: |
          yarn version --create-release=github  -m 'chore: release %v [skip ci]'
        env:
          HUSKY: 0
          GH_TOKEN: ${{ secrets.GH_TOKEN }}
          YARN_ENABLE_IMMUTABLE_INSTALLS: false

      - name: Publish to npm
        run: |
          if [ -z "$NODE_AUTH_TOKEN" ]; then
            echo "Missing env variable NODE_AUTH_TOKEN"
            exit 1
          fi
          echo >> ./.yarnrc.yml
          echo "npmAuthToken: $NODE_AUTH_TOKEN" >> ./.yarnrc.yml
          yarn npm publish
        if: github.ref == 'refs/heads/main' && !inputs.dry-run
        env:
          NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
```

## Example with Github Action, for a monorepo

```yml
name: Release
on:
  workflow_dispatch:
    inputs:
      dry-run:
        description: "Dry run"
        required: true
        type: boolean
        default: false
      bump-dependents-highest-as:
        description: "Bump dependents highest as"
        required: false
        type: choice
        options:
          - major
          - minor
          - patch
        default: "major"

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          token: ${{ secrets.GH_TOKEN }}
          fetch-depth: 0

      - uses: actions/setup-node@v3
        with:
          node-version: 18

      - name: Enable Corepack
        run: corepack enable

      - name: Install Dependencies
        run: yarn install --immutable --immutable-cache

      - name: New version (dry run)
        if: github.ref == 'refs/heads/main' && inputs.dry-run
        run: yarn version --dry-run --bump-dependents-highest-as=${{ inputs.bump-dependents-highest-as }}
      - name: Configure Git user
        if: github.ref == 'refs/heads/main' && !inputs.dry-run
        run: |
          git config --global user.name 'github-actions[bot]'
          git config --global user.email 'github-actions[bot]@users.noreply.github.com'

      - name: New version
        if: github.ref == 'refs/heads/main' && !inputs.dry-run
        run: |
          yarn version --create-release=github  --bump-dependents-highest-as=${{ inputs.bump-dependents-highest-as }} -m 'chore: release [skip ci]\n\n%t'
        env:
          HUSKY: 0
          GH_TOKEN: ${{ secrets.GH_TOKEN }}
          YARN_ENABLE_IMMUTABLE_INSTALLS: false

      - name: Publish to npm
        run: |
          if [ -z "$NODE_AUTH_TOKEN" ]; then
            echo "Missing env variable NODE_AUTH_TOKEN"
            exit 1
          fi
          echo >> ./.yarnrc.yml
          echo "npmAuthToken: $NODE_AUTH_TOKEN" >> ./.yarnrc.yml
          yarn workspaces foreach --parallel --no-private npm publish --tolerate-republish
        if: github.ref == 'refs/heads/main' && !inputs.dry-run
        env:
          NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
```
