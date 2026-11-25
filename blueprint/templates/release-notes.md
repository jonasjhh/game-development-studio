---
type: release-notes
product: <framework | games/<slug> | harness>
version: X.Y.Z
date: YYYY-MM-DD
tag: <framework/vX.Y.Z | games/<slug>/vX.Y.Z | harness/vX.Y.Z>
godot: <exact version, e.g. 4.7.1-stable>
framework: <for games: pinned framework version>
sim_fingerprint: <for games: hash shown in debug overlay>
save_format: <for games: integer>
net_protocol: <for games: integer>
approved_by: product-owner
---

# <Product> <version>

## Highlights
- <1–3 bullets for humans>

## Breaking changes
<For framework: list each with migration steps. "None" is a valid entry.>
| Change | Affected API/module | Migration |
|---|---|---|

## Added
- … (#PR)

## Changed
- …

## Deprecated
- <API> — replacement: <API>; removal planned in: <version>

## Fixed
- …

## Module maturity changes (framework)
| Module | From → To |
|---|---|

## Compatibility
- Supported Godot range: …
- Games verified against this release (canary results): …
- Save compatibility: <old saves load: yes/no; migration from version N>
- Multiplayer compatibility: <compatible with previous version: yes/no (sim fingerprint changed)>

## Verification summary
- `gdf verify --level release`: <run ID, pass>
- Determinism (Windows/Linux): <pass>
- Performance vs previous release: <table or "within budget">
- Known issues: …

## Upgrade instructions
```
gdf vendor games/<slug> --framework X.Y.Z
gdf verify --scope games/<slug> --level full
```
