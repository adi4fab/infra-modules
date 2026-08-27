# infra-modules

Reusable OpenTofu modules. **Blueprints only — this repo creates nothing.**

Consumed by `infrastructure-live` repos via `git::` with a pinned `?ref=`.

## How a module is consumed

```hcl
terraform {
  source = "git::git@github.com:adi4fab/infra-modules.git//modules/vpc?ref=v1.2.0"
}                                                        ↑↑
                                              double slash = subdirectory
```

⚠️ **Always `?ref=` a tag.** Without it the module changes under the consumer, and
identical code produces different infrastructure on different days.

## Layout

```
modules/
├── vpc/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   └── README.md
└── <next-module>/
```

- One folder per module
- Every module: `main.tf` · `variables.tf` · `outputs.tf` · `README.md`
- Modules never hardcode account ids, regions or environment names — those are inputs

## Releasing

```console
git tag v1.2.0
git push origin v1.2.0
```

- **SemVer.** Breaking input/output change → major bump
- Consumers upgrade by editing `?ref=` — one line, reviewable, revertable
- A module change is a **release**, not a silent edit

## Local development

Sourcing by tag means you must tag before you can use. While developing, override it:

```console
TERRAGRUNT_SOURCE=/path/to/infra-modules//modules/vpc terragrunt plan
```

⚠️ Do **not** work around this by putting a relative path in `source`. It works fine,
nobody changes it back, and you lose versioning permanently.

## Setup

```console
brew install mise
mise install
pre-commit install
```

## Guardrails

- **gitleaks** + **detect-private-key** pre-commit hooks
- **GitHub secret scanning + push protection** (free on public repos)
- **`main` is protected** — PR required, no force-push, no deletion, no bypass
- `.gitignore` blocks `.env`, `*.pem`, `*.key`, kubeconfigs, tfstate
