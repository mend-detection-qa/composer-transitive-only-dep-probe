# transitive-only-dep

## Feature exercised

This probe exercises Mend SCA's ability to detect packages that are never declared in the root `composer.json` but are reachable only through a transitive chain of two or more intermediate packages (depth >= 3 from root).

## Probe metadata

- Pattern: `transitive-only-dep`
- Package manager: Composer >= 2.6
- PHP minimum: >= 8.1
- Direct dependency: `symfony/console ^6.0` (resolved to v6.4.36)
- Total packages in lockfile: 9 (1 direct, 8 transitive)
- Deepest transitive chain observed: 3 levels

## Dependency layers

### Layer 1 — Direct (declared in composer.json)

| Package | Version |
|---|---|
| symfony/console | v6.4.36 |

### Layer 2 — Transitive via symfony/console

`symfony/console` requires all of the following in its lockfile `require` block:

| Package | Version | Parent |
|---|---|---|
| symfony/deprecation-contracts | v3.6.0 | symfony/console |
| symfony/polyfill-mbstring | v1.37.0 | symfony/console |
| symfony/service-contracts | v3.6.1 | symfony/console |
| symfony/string | v7.4.8 | symfony/console |

### Layer 3 — Transitive via layer-2 packages (transitive-only at depth >= 2)

| Package | Version | Parent (layer-2) | Path from root |
|---|---|---|---|
| psr/container | 2.0.2 | symfony/service-contracts | root -> symfony/console -> symfony/service-contracts -> psr/container |
| symfony/polyfill-ctype | v1.37.0 | symfony/string | root -> symfony/console -> symfony/string -> symfony/polyfill-ctype |
| symfony/polyfill-intl-grapheme | v1.37.0 | symfony/string | root -> symfony/console -> symfony/string -> symfony/polyfill-intl-grapheme |
| symfony/polyfill-intl-normalizer | v1.37.0 | symfony/string | root -> symfony/console -> symfony/string -> symfony/polyfill-intl-normalizer |

Note: `symfony/deprecation-contracts` also appears as a child of `symfony/service-contracts` and `symfony/string`, but its shortest path is depth 2 (directly via symfony/console). It is not counted as depth-3-only.

## Assertion target (depth-3 transitive-only)

**`psr/container` version `2.0.2`** is the primary assertion target.

- It is NOT declared anywhere in root `composer.json`.
- Its only path to root is: `root -> symfony/console -> symfony/service-contracts -> psr/container`.
- Mend must:
  1. Detect `psr/container` in the dependency tree at all.
  2. Link it to parent `symfony/service-contracts` (NOT directly to `symfony/console`).
  3. Report it as `scope: require` (production), `source: registry` (Packagist).

## Failure modes targeted

- Missing transitive dep at depth 3 (`psr/container` silently dropped).
- Wrong parent link for `psr/container` (linked to root or to `symfony/console` instead of `symfony/service-contracts`).
- Transitive-only packages incorrectly marked as direct.
- Layer-2 transitives missing (`symfony/string`, `symfony/service-contracts` not detected).