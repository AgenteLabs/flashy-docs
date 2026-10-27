# Local Setup: Building From the Sibling Checkouts

How to get the four packages onto a machine so the examples and your own code
can import them. The honest premise first: **two of the four are on the public
npm registry and two are not**, so a plain `npm install` of all four fails on
a clean machine, and the working path is to check the repositories out side by
side and install from disk.

Measured against the checkouts named below on 2026-10-10: flashy-ledger
`d2a9c82`, flashy-rails `1fff537`, magician `8163822`, flashyid `98adaf1`,
flashy-examples `50b0fb6` (each on branch `claude/dreamy-bell-2e5nq3`). The
registry column was measured the same day with `npm view <package> version`
against `https://registry.npmjs.org/`.

## What each manifest says, and what the registry serves

| Package | Checkout | What the manifest declares | On the public registry (2026-10-10) |
|---|---|---|---|
| `@flashyos/ledger` | `flashy-ledger` (public) | `1.0.0`; `publishConfig.registry` is `https://registry.npmjs.org/`, access `public`; a tag-triggered publish workflow exists. | **No** — `npm view` answered 404. Versions up to 0.8.0 went to GitHub Packages. |
| `@flashylabs/rails` | `flashy-rails` (private) | `1.0.0`; depends on the ledger as `file:vendor/ledger` (a vendored `1.0.0` build). No `publishConfig`. | **No** — 404. |
| `@magician-network/core` | `magician/packages/core` (private) | A workspace package, version `0.1.0`, no `publishConfig`. | **Yes** — `0.1.0`. |
| `@flashyid/sdk` | `flashyid/packages/sdk` (private) | `0.2.0`; `publishConfig.access: public`; a tag-triggered publish workflow exists. | **Yes** — `0.1.1`. The `0.2.0` the checkout declares is unpublished; the [FlashyID API](../api/flashyid-api.md) page marks what changed. |

`flashy-examples/package.json` declares `@flashyos/ledger` and
`@flashylabs/rails` at `^1.0.0`, `@magician-network/core` at `^0.1.0` and
`@flashyid/sdk` at `^0.1.1` — ranges the versions above satisfy, so the day
the ledger and rails publish, the same `npm install` resolves them with no
edit. Its README's install section points at sibling checkouts (`npm install
file:../flashy-ledger …`), which is the path below.

## Prerequisites

- **Node 22.** `flashy-ledger` and `flashy-rails` declare `engines.node >=22`;
  `@flashyid/sdk` declares `>=18`; this repository's own tools need 22.
- **Corepack enabled** if you will regenerate `flashy-ledger`'s lockfile: its
  `packageManager` pins npm `10.9.8`, and the pin is only enforced through
  Corepack (`corepack enable`).
- git.

## 1. Clone side by side

```bash
mkdir flashy && cd flashy
git clone https://github.com/AgenteLabs/flashy-ledger
git clone https://github.com/AgenteLabs/flashy-rails
git clone https://github.com/AgenteLabs/magician
git clone https://github.com/AgenteLabs/flashyid
git clone https://github.com/AgenteLabs/flashy-examples
git clone https://github.com/AgenteLabs/flashy-docs
```

Before reading anything as "what ships", ask each checkout which branch is its
default — `main` is not necessarily it:

```bash
git -C flashy-ledger symbolic-ref --short refs/remotes/origin/HEAD
```

## 2. Build each package

```bash
# Ledger: TypeScript -> dist/ (ESM) and dist-cjs/
cd flashy-ledger && npm ci && npm run build && npm test && cd ..

# Rails: plain ESM, nothing to compile; npm install pulls the vendored ledger
cd flashy-rails && npm install && npm test && cd ..

# Magician: a workspace; the root build orders core, cli, then the site
cd magician && npm install && npm run build && cd ..

# FlashyID SDK: its own manifest under packages/sdk
cd flashyid/packages/sdk && npm install && npm run build && cd ../../..
```

Each `npm test` above is the repository's own suite; a red one is a fact about
that checkout, not about your setup.

## 3. Point the examples at the builds

```bash
cd flashy-examples
npm install \
  file:../flashy-ledger \
  file:../flashy-rails \
  file:../magician/packages/core \
  file:../flashyid/packages/sdk
```

`file:` installs link the built packages by path, so a rebuild in a sibling is
picked up without reinstalling. Rails resolves its own ledger from
`vendor/ledger`, not from the sibling — two copies of the ledger will be in
play, which is how the rails repository itself runs.

The two published packages can come from the registry instead of a checkout
— `npm install @magician-network/core @flashyid/sdk` — at the cost of reading
`0.1.1` of the SDK rather than the `0.2.0` a checkout builds.

## 4. What runs at these commits

- Examples `11-intentmesh`, `12-rites`, `13-aao-validation` and
  `14-mesh-consumer` are marked `standalone: true` in
  `examples/manifest.json` and run with no package installed at all:
  `npm run test:standalone`.
- Examples `01`–`10` are marked `runs: "needs-packages"` in the manifest and
  import only names the packages export. At `50b0fb6` (commit `5ef78d1`,
  *examples 01-10 rewritten against the real APIs — every example runs*) all
  ten run against the sibling builds above; the guides here (for example the
  [Combined Workflow](combined-workflow.md)) and those examples are written
  against the same exports, and the [API reference](../api/ledger-api.md)
  pages list them.

## 5. Check this repository

```bash
cd flashy-docs
npm test          # relative links resolve; JavaScript samples parse
npm run status    # regenerates docs/STATUS.md from the sibling manifests
```

`npm test` needs no install: both checkers import only `node:` builtins.

## Related

- [Ledger API](../api/ledger-api.md) · [Rails API](../api/rails-api.md) ·
  [Magician API](../api/magician-api.md) · [FlashyID API](../api/flashyid-api.md)
- [Status](../STATUS.md) — what the sibling checkouts declared when it was last generated
