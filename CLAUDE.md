# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

`@solana/spl-token-registry` — the legacy Solana token list. The README states the repo is EOL/archived: new token metadata now goes through Metaplex Fungible Token Metadata, and the npm package stays only for reading the legacy list. It has three coupled parts:

- `src/tokens/solana.tokenlist.json` — the data (~185k lines, the source of truth; PRs add entries here plus a logo under `assets/mainnet/<mint address>/`).
- `src/lib/tokenlist.ts` — the TypeScript package (`TokenListProvider`, `TokenListContainer`, `ENV` chain IDs 101/102/103, `TokenInfo` types). It imports the JSON directly and can also fetch it from a CDN; `src/index.ts` re-exports it.
- `automerge/` — a separate Go module (`github.com/solana-labs/token-list/automerge`) that bot-reviews open PRs and merges those that only *add* valid tokens.

## Commands

TypeScript (use `yarn`; CI uses Node 12):

```bash
yarn build                 # tsc to dist/main and dist/module (both tsconfigs)
yarn test                  # build + eslint + prettier check + ava unit tests
yarn test:lint             # eslint src --ext .ts
yarn fix                   # prettier --write + eslint --fix
yarn test:unit             # ava via nyc; runs compiled JS in dist/main, so build first
yarn build:main && npx ava dist/main/lib/tokenlist.spec.js   # single test file
```

ava runs against `dist/main` (see `ava.typescript.rewritePaths` in `package.json`), not `src/`, so changes to specs/sources need a rebuild before they take effect.

Schema validation of the token list (needs the `cue` CLI v0.4.0; CI installs it with `go install cuelang.org/go/cmd/cue@v0.4.0`):

```bash
./validate.sh              # cue vet solana.tokenlist.json against automerge/schema.cue (#Tokenlist)
```

Go automerger (run from `automerge/`):

```bash
go build -o ~/automerge github.com/solana-labs/token-list/automerge
go test ./parser/...
~/automerge -v=1 -dryRun -max=N   # needs GITHUB_APP_PEM; -setRemoteForCI is CI-only
```

## Architecture notes

- **`automerge/schema.cue` is shared by both validators.** `validate.sh` checks the whole list against `#Tokenlist`; the Go bot embeds the file (`//go:embed`) and checks each added entry against `#StrictTokenInfo`. It also contains whitelists (`#SymbolWhitelist`, `#NameWhitelist`) of grandfathered tokens that violate the naming rules — new tokens must not need to be added there.
- **Automerge flow** (`automerge/automerge.go`, run hourly by `.github/workflows/automerge_new.yml`): for each open PR, parse the diff (`parseDiff`/`processTokenlist`), reject any PR with deleted/modified lines or touching other files, validate with the strict CUE schema, verify logo URI/asset, CoinGecko ID and Twitter handle, reject duplicate addresses/names (`knownAddrs`/`knownNames`) and blacklisted tokens, then commit the additions onto an `automerge-pending` branch which the workflow merges into `main`. Results are reported back to the PR as check runs/comments.
- **`automerge/parser/`** is a deliberately tolerant JSON-fragment parser: PR diffs contain partial JSON (fragments of the token array), so it normalizes brackets/truncation before decoding into `Token` structs. Its tests are in `parser_test.go`.
- **Release**: pushes to `main` run `.github/workflows/main.yml`: `yarn test`, then `standard-version --patch`, push tags, purge the jsDelivr CDN cache for the JSON, and `npm publish`. Don't bump the version in `package.json` by hand.

## Conventions

- Token PR rules (`.github/PULL_REQUEST_TEMPLATE.md`): only add entries to `solana.tokenlist.json`, never modify or delete existing ones (the automerger rejects PRs with deleted lines), with each entry having address, name, symbol, logo (under `assets/mainnet/<mint>/`), homepage and, if available, CoinGecko ID.
- Prettier uses single quotes; eslint config includes `eslint-plugin-functional` (immutability rules) — run `yarn fix` before committing. Commits follow commitizen/conventional-commit style.
