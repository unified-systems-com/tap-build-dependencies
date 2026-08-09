# tap-build-dependencies

**Build-time / bootstrap-tier dependencies for TAP images.** Everything in this
repository installs **below the plugin system** — at image build, before boot,
before secrets resolve — which is precisely why it does not and cannot live in a
`tap-plugin-*` repo: the plugin system is not running yet when this code loads.

## Read this before touching anything

This repository is **supply-chain-critical** and is deliberately loud about it:

- Code here becomes part of TAP images at **build time**. In the case of secret-source
  providers, it is credential-resolution code running inside every instance that
  bakes it.
- TAP core gates which distributions may register secret sources by an explicit
  allowlist (`_ALLOWED_SOURCE_DISTRIBUTIONS` in `tap/secret_sources.py`, checked by
  distribution name *before* the provider module is ever imported). The allowlist
  plus **consumers pinning an exact rev** are the load-bearing install-time
  controls; this repo's rulesets (no force-push, no deletion, protected tags,
  linear history, secret push-protection) are defense-in-depth on top — they gate
  *writes*, not reads.
- Additions here are a security review, not a routine PR. If the addition touches
  credentials in any way, run TAP's `manage-secret` review first.

## How consumers install from here

Pin an exact revision and name the subdirectory — never track a branch:

```
uv pip install "aws-secrets-source @ git+https://github.com/unified-systems-com/tap-build-dependencies@<full-sha>#subdirectory=aws_secrets_source"
```

## Inventory

| Path | Distribution | What it is |
| --- | --- | --- |
| `aws_secrets_source/` | `aws-secrets-source` | AWS Secrets Manager backend for TAP's secret-source seam (`tap.secret_sources` entry point). From its extraction commit: *"Bootstrap-tier secret-source provider… **Not a grid plugin**; installed at build time into the CI base image before secret-gated plugins."* Authenticates via ambient cloud IAM only — a source provider never authenticates via a TAP secret (no resolution recursion). |

## Layout convention

One directory per artifact, each a complete installable distribution
(`pyproject.toml` at its root, tests inside). The repo is a shelf, not a package:
nothing at the top level is importable, and consumers reference artifacts by
`#subdirectory=`.

## Provenance

`aws_secrets_source/` moved here 2026-08-09 from TAP core's in-tree copy
(`plugins/aws_secrets_source/`, byte-identical at the move to the v0.1.0
extraction snapshot formerly at `tap-plugin-aws-secrets-source`, now archived).
Core's in-tree copy remains the live one until the build-bake eviction completes:
image builds install from here at a pinned rev, prove green, and only then does
the in-tree copy get deleted. That sequencing is tracked in TAP core's
`docs/misc/doc-github-org-migration-plan.md`.
