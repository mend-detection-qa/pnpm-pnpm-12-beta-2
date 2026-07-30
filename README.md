# pnpm12-platform-filtered-optionals

## Probe metadata

| Field           | Value                            |
|-----------------|----------------------------------|
| pm              | pnpm                             |
| pm_version      | 12.0.0-beta.2                    |
| schema_version  | 1.0                              |
| categories      | lockfile_format, install_command |
| pattern         | pnpm12-platform-filtered-optionals |
| target_platform | linux/x64/glibc                  |
| generated_at    | 2026-07-30T00:00:00Z             |

## Feature exercised

Two pnpm 12 Beta 2 behaviours that affect what the
`pnpm-lock.yaml` contains and therefore what Mend SCA
resolves at scan time:

1. **lockfile_format** — pnpm 12 silently rewrites broken
   or corrupted lockfiles rather than aborting with a hard
   error. A Mend UA scan that encounters such a project
   will find a freshly-rewritten, structurally-valid v9
   lockfile; the dependency tree must still resolve
   correctly from it.

2. **install_command** — optional dependencies whose `os`,
   `cpu`, or `libc` fields do not match the current host
   platform are excluded from the lockfile entirely at
   install time. Under pnpm 12 on Linux x64:
   - `@esbuild/linux-x64@0.21.5` IS in the lockfile
     (platform match).
   - `@esbuild/darwin-arm64`, `@esbuild/win32-x64`, and
     all other esbuild platform binaries are NOT in the
     lockfile (filtered out).
   - `fsevents@2.3.3` is NOT in the lockfile (macOS-only
     optional; excluded on Linux x64).

## Dependencies

### Regular (non-optional)

| Package  | Version | Notes                       |
|----------|---------|-----------------------------|
| hono     | 4.4.0   | Zero-dep web framework      |
| zod      | 3.23.8  | Schema validation library   |
| esbuild  | 0.21.5  | Bundler; has platform optionals |

### Optional (platform-filtered)

| Package             | Version | Platform  | In lockfile? |
|---------------------|---------|-----------|--------------|
| @esbuild/linux-x64  | 0.21.5  | linux/x64 | YES          |
| fsevents            | 2.3.3   | macOS     | NO           |

All other esbuild platform binaries (`@esbuild/darwin-arm64`,
`@esbuild/win32-x64`, etc.) are absent from the lockfile
because pnpm 12 filtered them at install time on Linux x64.

## Expected dependency tree summary

Mend SCA scanning on Linux x64 MUST report:

- `hono@4.4.0` — registry, group: main
- `zod@3.23.8` — registry, group: main
- `esbuild@0.21.5` — registry, group: main;
  depends on `@esbuild/linux-x64`
- `@esbuild/linux-x64@0.21.5` — registry, group: main,
  optional: true (platform-matching esbuild binary)

Mend MUST NOT report:
- `fsevents` (absent from lockfile)
- Any non-Linux esbuild binary
  (`@esbuild/darwin-arm64`, `@esbuild/win32-x64`, etc.)

## Mend failure modes

- Platform-non-matching optionals included (Mend scanned
  `package.json` manifest instead of lockfile — the
  manifest does not have optionals for esbuild binaries
  directly, but if Mend resolves esbuild's own
  `optionalDependencies` field from the registry it would
  add all platform binaries).
- `fsevents` included (Mend scanned `optionalDependencies`
  in `package.json` rather than lockfile).
- `optional: true` flag missing on
  `@esbuild/linux-x64`.
- Tree fails to parse because pnpm 12's v9 lockfile
  structure is not recognized.
- Lockfile rewrite behaviour (lockfile_format) causes
  the parser to bail on a structural edge case.

## Resolver note

The Mend pnpm resolver (`PnpmLockCollector`) reads
`pnpm-lock.yaml` via `PnpmParserV9Impl` (lockfileVersion
7.x–9.x). It extracts dependencies from the `snapshots`
section and resolution metadata from `packages`. Platform-
filtered optionals are NOT present in the lockfile, so
Mend must NOT try to add them by reading esbuild's
registry metadata. This probe validates that the resolver
reads the lockfile faithfully without supplementing it
from the registry.

## Mend config

**Bucket A** — `js-pnpm` has no dynamic version detection
from the manifest. This probe ships `.whitesource` with
`scanSettings.versioning` pinning:

- `pnpm: "12.0.0-beta.2"` (exact beta version for
  reproducibility).
- `node: "20.11.1"` (LTS; used for esbuild binary
  selection).

No `whitesource.config` is present, so `configMode` is
`"AUTO"`.

## Files

```
pnpm12-platform-filtered-optionals-20260730-000000/
├── .npmrc               (supported-architectures: linux/x64/glibc)
├── .whitesource         (Bucket A versioning: pnpm + node)
├── expected-tree.json   (ground truth for Mend comparison)
├── package.json         (packageManager: pnpm@12.0.0-beta.2)
├── pnpm-lock.yaml       (lockfileVersion: '9.0', Linux x64)
└── README.md            (this file)
```
