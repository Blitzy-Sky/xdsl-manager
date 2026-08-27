# Changelog

All notable changes to `xdsl-manager` are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
repository adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

Every release recorded below names a tag that exists in this repository, and its date is the date of
the tag. A `git describe --tags` string, which appends a commit count and an abbreviated hash to the
nearest tag, denotes a position a number of commits past a tag rather than a release, so no such
string is recorded here as a release. Work that is not yet tagged is listed under `Unreleased`.

Two version numbers travel with this repository, and they are not the same kind of thing:

- **Release tag** - the point in this repository's history that a consumer checks out, recorded in
  the release table under `Releases before this file`. The current tag is **v1.6.0**.
- **HAL schema version** - the version of the JSON HAL contract itself, carried by the
  `schemaVersion` definition in `hal_schema/xdsl_hal_schema.json` as the constant **0.0.1**. The
  schema states that this value must not be modified, because HAL operation depends on the client
  and the vendor server agreeing on it. It is advanced independently of the release tag.

## Unreleased

Changes that are merged but not yet part of a tagged release. They cover the commits after the
`v1.6.0` tag; run `git log v1.6.0..` in a checkout of this repository to list them.

All of it is documentation. No functional change was made: no schema, fixture, configuration file,
data model, source file or build file was modified, and the JSON HAL contract is unchanged.

### Added

- [docs/pages/halSpec.md](https://github.com/rdkcentral/xdsl-manager/blob/609617f9f3b2648175851e9b6c0ca4857deb8404/docs/pages/halSpec.md) - the xDSL HAL interface specification. It covers the architecture and the
  component runtime execution requirements, the non-functional requirements, and the interface API
  documentation including the action vocabulary, the object tree and the protocol contract. The
  repository had no HAL documentation before this entry.
- [docs/pages/halSpecDetailed.md](https://github.com/rdkcentral/xdsl-manager/blob/609617f9f3b2648175851e9b6c0ca4857deb8404/docs/pages/halSpecDetailed.md) - the per-parameter reference for the JSON HAL, which is to
  `hal_schema/xdsl_hal_schema.json` what inline Doxygen is to a C header. It documents all 369
  parameter definitions with their type, constraint, access and derived meaning, expands the 20
  definitions that use pattern alternation so every logical path is covered, and adds the object
  index, the enumeration appendix, one worked message exchange per protocol workflow, the error and
  event models, and the known contract defects.
- `docs/generate_docs.sh` - the documentation build entry point, pinning the `hal-doxygen` generator
  to tag `1.2.0` and building under the project name `RDK-B xDSL HAL`.
- `docs/.gitignore` - excludes the generator clone and the generated site, `build` and `output`,
  together with `node_modules`, from version control.
- Five symlinks under `docs/pages/` - `CHANGELOG`, `CONTRIBUTING`, `COPYING`, `LICENSE` and `NOTICE`,
  each carrying a `.md` suffix - pointing at the corresponding files at the repository root, so the
  release history, the contribution route and the legal texts render alongside the HAL documentation in
  the generated Doxygen site.
- This changelog, the release-history record for the repository and the source for the
  `Version History` topic of the HAL specification. The repository had no changelog before this
  entry.

### Changed

- The repository README - replaced the single-line stub with a manager overview: what the component
  is, the JSON HAL contract it speaks and the five values that fix that contract on a device, the
  four TR-181 families the schema exposes, the repository layout, the four shipped example messages
  that cannot be used as they stand, and what the build files do and do not establish. It links to
  both documentation pages and remains a regular file rather than a symlink, because this
  repository's landing page is a manager overview and not a HAL specification.

## Releases before this file

This changelog was introduced during the HAL documentation work recorded under `Unreleased`, after
the releases below were already tagged. Their contents are not restated here, because doing so would
mean asserting release notes this repository never wrote; the authoritative record for each is the
commit range the tag closes.

| Tag | Date |
|---|---|
| `v1.6.0` | 2025-11-06 |
| `v1.5.0` | 2025-08-05 |
| `v1.4.0` | 2025-05-23 |
| `v1.3.0` | 2025-02-12 |
| `v1.2.0` | 2025-01-14 |
| `v1.1.0` | 2024-10-31 |
| `v1.0.0` | 2024-06-07 |

The repository additionally carries the release-candidate tags `RC1.6.0a` (2025-08-19), `RC1.5.0a`
(2025-07-03), `RC1.4.0a` (2025-03-18), `RC1.3.0a` (2025-01-28), `RC1.2.0a` (2024-11-13), `RC1.1.0b`
(2024-10-18) and `RC1.1.0` (2024-09-24). These are pre-release markers rather than releases, so they
are named here for completeness and are not rows in the table above.
