# infra-modules

Reusable OpenTofu modules. **Blueprints only — this repo creates nothing.**

Consumed by infrastructure-live repos via `git::` with a pinned `?ref=`.

## Layout

```
<category>/<module>/
├── main.tf
├── variables.tf
├── outputs.tf
├── versions.tf
├── README.md
└── examples/simple/main.tf
```

- Categories mirror the live repo: `networking` `kubernetes` `data-stores`
  `security` `identity` `observability` `ci-cd` `organization`
- **No `modules/` folder and no module at the repo root** — the repo name already
  says "modules", and a module sitting next to the categories makes them meaningless

## Consuming a module

```hcl
terraform {
  source = "git::git@github.com:adi4fab/infra-modules.git//networking/vpc?ref=v1.2.0"
}                                                        ↑↑
                                              double slash = subdirectory
```

⚠️ **Always `?ref=` a tag.** Without it the module changes under the consumer, and
identical code produces different infrastructure on different days.

⚠️ **A module change is not live until a consumer bumps its `?ref=`.** Merging here
publishes a version; it moves no infrastructure.

## Creating a module

```console
make new-module CATEGORY=networking NAME=vpc
```

Scaffolds every required file, including the README with its docs marker. Nothing
to remember, and no module can end up without docs.

## Internal composition — relative paths only

```hcl
module "subnets" {
  source = "../../networking/additional-subnets"   # ✅
}
```

**Never** `git::` back into this repo. A module pinned to an old version of its own
sibling is invisible and unfixable at scale — `make check-self-ref` fails the build.

## Releasing

Fully automated, and **tag-only**.

| Commit prefix | Bump |
|---|---|
| `fix:` | patch |
| `feat:` | minor |
| `BREAKING CHANGE:` | **major** |

- The version lives in **git tags**, not in a file — `version_provider = "scm"`
- The release job creates a tag and pushes **only** `refs/tags/*`. It never pushes to
  `main`, so it needs **no bypass** of the branch ruleset
- One tag form: `v1.2.0`
- `major_version_zero = false` — a breaking change is visible in the version number
- The changelog is the **GitHub Release body**, not a committed file, so it cannot drift

### Why no version file

A stored version is **state**. If a tag were ever created by hand, or a bump commit
failed to land, the file and the tags would disagree — and the next automated bump
would then **overwrite an existing tag**, silently changing code a consumer is pinned
to. That is the worst failure available in a pinned-module design.

With no file there is nothing to desync. Tags are also protected by an
`immutable-version-tags` ruleset — they cannot be moved or deleted, by anyone.

## CI

Every PR runs:

- pre-commit — fmt, docs, gitleaks, private keys, commit message shape
- `make validate` — `tofu validate` on **every module and every example**
- `make check-pins` — no `git::` source without `?ref=`
- `make check-self-ref` — no module referencing this repo

`examples/` is the test. It's documentation that cannot go stale, and it needs no
AWS credentials.

## Setup

```console
brew install mise
mise install
pre-commit install
pre-commit install --hook-type commit-msg
git config blame.ignoreRevsFile .git-blame-ignore-revs
```

`make` refuses to run until the hooks are installed — a local error with a pointer,
rather than a CI failure later.

## Why the checks are required, not advisory

A check that runs but does not gate is worse than no check:

- People wait on a red X that does not block
- People merge past a check they should have read
- You pay the runner minutes and ignore the signal

`quality-checks` is a **required status check** on `main`. If it is red, the PR
does not merge.

## Guardrails

- gitleaks + detect-private-key pre-commit hooks
- GitHub secret scanning + push protection
- `main` protected — PR required, no force-push, no deletion, **zero bypass actors**
- Renovate keeps hook pins and provider constraints current
