# Changelog

All notable changes to `xdsl-manager` are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
repository adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

Every release recorded below names a tag that exists in this repository, and its date is the date of
the tag: the tagger date where the tag is annotated, and the tagged commit's date for `v1.0.0`,
which is a lightweight tag carrying no tagger date of its own. A `git describe --tags` string, which
appends a commit count and an abbreviated hash to the nearest tag, denotes a position a number of
commits past a tag rather than a release, so no such string is recorded here as a release. Work that
is not yet tagged is listed under `Unreleased`. This file was introduced after every release below
was tagged, so each release entry is compiled from the non-merge commits in the range its tag
closes.

The repository additionally carries the release-candidate tags `RC1.6.0a` (2025-08-19), `RC1.5.0a`
(2025-07-03), `RC1.4.0a` (2025-03-18), `RC1.3.0a` (2025-01-28), `RC1.2.0a` (2024-11-13), `RC1.1.0b`
(2024-10-18) and `RC1.1.0` (2024-09-24). These are pre-release markers rather than releases, so they
are named here for completeness and have no release heading of their own. Each marks the same commit
as the release of the same number, except `RC1.1.0`, which marks an earlier commit inside the range
that `v1.1.0` closes.

Two version numbers travel with this repository, and they are not the same kind of thing:

- **Release tag** - the point in this repository's history that a consumer checks out, recorded as
  the release headings in this file. The current tag is **v1.6.0**.
- **HAL schema version** - the version of the JSON HAL contract itself, carried by the
  `schemaVersion` definition in `hal_schema/xdsl_hal_schema.json` as the constant **0.0.1**. The
  schema states that this value must not be modified, because HAL operation depends on the client
  and the vendor server agreeing on it. It is advanced independently of the release tag.

## Unreleased

Changes that are merged but not yet part of a tagged release. They cover the commits after the
`v1.6.0` tag; run `git log v1.6.0..` in a checkout of this repository to list them. They are three
repository-maintenance commits that change only files under `.github/`, and the HAL documentation
work.

The HAL documentation work is documentation only. No functional change was made: no schema,
fixture, configuration file, data model, source file or build file was modified, and the JSON HAL
contract is unchanged.

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
- `.github/workflows/fossid_integration_stateless_diffscan_target_repo.yml` - a FossID stateless
  diff-scan GitHub Actions workflow. It runs when a pull request from a branch of this repository,
  not from a fork, is opened, synchronized or reopened, and calls the shared
  `rdkcentral/build_tools_workflows` FossID workflow at `1.0.0`
  ([`e3aee0b`](https://github.com/rdkcentral/xdsl-manager/commit/e3aee0b86d2d0ff517707f57baeafaaff54d9cd0),
  [#21](https://github.com/rdkcentral/xdsl-manager/pull/21)).

### Changed

- The repository README - replaced the single-line stub with a manager overview: what the component
  is, the JSON HAL contract it speaks and the five values that fix that contract on a device, the
  four TR-181 families the schema exposes, the repository layout, the four shipped example messages
  that cannot be used as they stand, and what the build files do and do not establish. It links to
  both documentation pages and remains a regular file rather than a symlink, because this
  repository's landing page is a manager overview and not a HAL specification.
- `.github/workflows/cla.yml` - the Contributor License Agreement workflow now declares its token
  permissions explicitly and calls the shared `rdkcentral/cmf-actions` CLA workflow at `v1` instead
  of `main`
  ([`c786824`](https://github.com/rdkcentral/xdsl-manager/commit/c786824e98af5037b13d24b420d4fcaac850124f),
  [#20](https://github.com/rdkcentral/xdsl-manager/pull/20)).
- `.github/CODEOWNERS` - the default reviewer for every path changed from
  `@rdkcentral/wanmanager-owners` to `@rdkcentral/wanmanager-maintainers`
  ([`ad1cba9`](https://github.com/rdkcentral/xdsl-manager/commit/ad1cba96c972a9f4a914fe7782ed2918363e083c),
  [#19](https://github.com/rdkcentral/xdsl-manager/pull/19)).

## 1.6.0 - 2025-11-06

Release tagged [`v1.6.0`](https://github.com/rdkcentral/xdsl-manager/releases/tag/v1.6.0).
Its changes since `v1.5.0` are the range
[`v1.5.0...v1.6.0`](https://github.com/rdkcentral/xdsl-manager/compare/v1.5.0...v1.6.0).

### Added

- Contributor License Agreement check as the `.github/workflows/cla.yml` GitHub Actions workflow,
  which calls the shared `rdkcentral/cmf-actions` CLA workflow on pull requests and comments
  ([`b712ce0`](https://github.com/rdkcentral/xdsl-manager/commit/b712ce0bb2f35ac1a91e6bc93fb50c79d1ff6aa1),
  [#18](https://github.com/rdkcentral/xdsl-manager/pull/18)).

## 1.5.0 - 2025-08-05

Release tagged [`v1.5.0`](https://github.com/rdkcentral/xdsl-manager/releases/tag/v1.5.0).
Its changes since `v1.4.0` are the range
[`v1.4.0...v1.5.0`](https://github.com/rdkcentral/xdsl-manager/compare/v1.4.0...v1.5.0).

### Removed

- The D-Bus path message handler `CcspComp_path_message_func` and its
  `CCSP_Message_Bus_Register_Path` registration, from
  `source/RdkXdslManager/ssp_messagebus_interface.c`. The change was raised as an internal build
  fix (CTB-3889_3075)
  ([`40134d9`](https://github.com/rdkcentral/xdsl-manager/commit/40134d9e0ee667b31443274f6e7da604ed0cd299),
  [#15](https://github.com/rdkcentral/xdsl-manager/pull/15)).

## 1.4.0 - 2025-05-23

Release tagged [`v1.4.0`](https://github.com/rdkcentral/xdsl-manager/releases/tag/v1.4.0).
Its changes since `v1.3.0` are the range
[`v1.3.0...v1.4.0`](https://github.com/rdkcentral/xdsl-manager/compare/v1.3.0...v1.4.0).

### Added

- `.github/CODEOWNERS`, which makes `@rdkcentral/wanmanager-owners` the default reviewer for every
  path in the repository (CMFSUPPORT-2402)
  ([`0eef7d5`](https://github.com/rdkcentral/xdsl-manager/commit/0eef7d5452e9cade7541d3abb232f2f6b8d62ae4),
  [#17](https://github.com/rdkcentral/xdsl-manager/pull/17)).

## 1.3.0 - 2025-02-12

Release tagged [`v1.3.0`](https://github.com/rdkcentral/xdsl-manager/releases/tag/v1.3.0).
Its changes since `v1.2.0` are the range
[`v1.2.0...v1.3.0`](https://github.com/rdkcentral/xdsl-manager/compare/v1.2.0...v1.3.0).

### Added

- xTU-R and xTU-C identification parameters under `Device.DSL.Line.{i}.` (RDKBDEV-2903):
  `XTURVersion`, `XTURSerial`, `XTURVendorId`, `XTURSystemVendorId`, `XTURVendorSpecific`,
  `XTURSystemVendor`, `XTURSystemVendorSpecific`, `XTURSystemCountry`, `XTUCVersion`,
  `XTUCSerial`, `XTUCVendorSpecific`, `XTUCSystemVendor`, `XTUCSystemVendorSpecific` and
  `XTUCSystemCountry`, in `config/RdkXdslManager.xml`, `hal_schema/xdsl_hal_schema.json` and the
  data-model handlers; `XTURVendor` changes from `string(9)` to `hexBinary(4:4)`. The schema's
  parameter list references `dslLineXTURVersion` and `dslLineXTURSerial` without defining them, a
  defect recorded under `Contract Defects` in the xDSL HAL Detailed API Reference
  ([`1e44f73`](https://github.com/rdkcentral/xdsl-manager/commit/1e44f7335226f76b11534cbdaead20901cf0c04a),
  [#6](https://github.com/rdkcentral/xdsl-manager/pull/6)).
- The build version is logged at startup (RDKB-58019): `configure.ac` captures the output of
  `git describe --tags --always --dirty` as `GIT_VERSION`, the `xdslmanager` build passes it as a
  compile definition, and `ssp_main.c` logs it once the logger is initialised
  ([`3a7e845`](https://github.com/rdkcentral/xdsl-manager/commit/3a7e845290233ac0329148837b10a5b40ab55891),
  [#14](https://github.com/rdkcentral/xdsl-manager/pull/14)).
- Error logging on the failure paths of the manager, the HAL client, the DSL report and the WebPA
  report interface (RDKBDEV-2943), with several existing failure messages raised to error level.
  `DmlXdslGetLowerLayersInstanceInOtherAgent` now also returns a failure, rather than success, when
  the other agent reports no entries
  ([`93b05d0`](https://github.com/rdkcentral/xdsl-manager/commit/93b05d0b77fc9610f77387727d3f376721fef0e3),
  [#9](https://github.com/rdkcentral/xdsl-manager/pull/9)).

### Fixed

- `Device.DSL.Line.1.Enable` could not be set to `false` (RDKBDEV-2852). `DmlXdslLineSetEnable`
  now sends the value to the HAL as `TRUE` or `FALSE` rather than `1` or `0`, and
  `Line_SetParamBoolValue` passes the line's instance number rather than the instance number minus
  one
  ([`0a618e2`](https://github.com/rdkcentral/xdsl-manager/commit/0a618e244c549877ff2372165a63854cfb246750),
  [`520c4cb`](https://github.com/rdkcentral/xdsl-manager/commit/520c4cbc6f30b1ec6aacc481f5c44f699b8862bd),
  [`a96de6f`](https://github.com/rdkcentral/xdsl-manager/commit/a96de6fa1104fead36e5abef7d39af43e4476909),
  [#3](https://github.com/rdkcentral/xdsl-manager/pull/3)).
- `UINT` and `ULONG` data-model values stopped updating once they passed `2^31 - 1`
  (RDKBDEV-2900). Values received from the HAL are now parsed with `strtoull` instead of `atoi` or
  `atol` in `xdsl_hal.c` and `xtm_internal.c`
  ([`3aa7020`](https://github.com/rdkcentral/xdsl-manager/commit/3aa70206bdae1db288f3047d540d88ae6b194c71),
  [#12](https://github.com/rdkcentral/xdsl-manager/pull/12)).

## 1.2.0 - 2025-01-14

Release tagged [`v1.2.0`](https://github.com/rdkcentral/xdsl-manager/releases/tag/v1.2.0).
Its changes since `v1.1.0` are the range
[`v1.1.0...v1.2.0`](https://github.com/rdkcentral/xdsl-manager/compare/v1.1.0...v1.2.0).

### Fixed

- `Device.DSL.Channel.{i}.Enable` always read `false` (RDKBDEV-2883). `xdsl_hal_dslGetChannelInfo`
  now maps the `Enable` value the HAL returns
  ([`778732a`](https://github.com/rdkcentral/xdsl-manager/commit/778732aaaabbbb1adf990f263c49b9521603b7ff),
  [#4](https://github.com/rdkcentral/xdsl-manager/pull/4)).

## 1.1.0 - 2024-10-31

Release tagged [`v1.1.0`](https://github.com/rdkcentral/xdsl-manager/releases/tag/v1.1.0).
Its changes since `v1.0.0` are the range
[`v1.0.0...v1.1.0`](https://github.com/rdkcentral/xdsl-manager/compare/v1.0.0...v1.1.0).

### Changed

- JSON HAL requests and responses are logged on a single line, with `JSON_C_TO_STRING_SPACED`
  instead of `JSON_C_TO_STRING_PRETTY` (RDKBDEV-2901)
  ([`ef75776`](https://github.com/rdkcentral/xdsl-manager/commit/ef75776f3670aa425d1560fcdbcd380fa7a4f87e),
  [#7](https://github.com/rdkcentral/xdsl-manager/pull/7)).

### Fixed

- An `OverrideTTL` set through RFC after a factory reset was reset to 0, because DSL reporting
  started before its reporting periods were set (SHARMAN-2524). `StartXdslReporting` now waits,
  polling every two seconds, until both the override and the default reporting periods are set
  ([`f7b197f`](https://github.com/rdkcentral/xdsl-manager/commit/f7b197fecd438da2e7c5be380d71549443b72002),
  [`9543540`](https://github.com/rdkcentral/xdsl-manager/commit/95435401fd98ee976b6958e67b587e1efa3977d4),
  [#5](https://github.com/rdkcentral/xdsl-manager/pull/5)).
- `RdkXdslManager` crashed on 64-bit builds (RDKB-56987): functions such as `ATMLink_GetEntry` and
  `PTMLink_GetEntry` were called without a declaration in scope, so their pointer results were
  truncated to 32 bits. The missing declarations and includes are added
  ([`dd5a070`](https://github.com/rdkcentral/xdsl-manager/commit/dd5a070d57cd4341d5d7c83e2dfc334ae145e718),
  [#8](https://github.com/rdkcentral/xdsl-manager/pull/8)).

## 1.0.0 - 2024-06-07

First tagged release, at tag
[`v1.0.0`](https://github.com/rdkcentral/xdsl-manager/releases/tag/v1.0.0). It publishes
`RdkXdslManager` on GitHub together with the history the component accumulated on the RDK `stable2`
branch from 2021, merged through [#1](https://github.com/rdkcentral/xdsl-manager/pull/1). The
open-sourcing commit (CMFSUPPORT-1695) replaced the licence header of the component's files with the
Apache 2.0 form
([`c406e0e`](https://github.com/rdkcentral/xdsl-manager/commit/c406e0eb366db6ed2c67b867e2386d0a779dfebc)).

### Added

- The `RdkXdslManager` component (RDKB-34410): the CCSP process, the TR-181 `Device.DSL.` data model
  (`Line`, `Channel`, `Diagnostics`, `X_RDK_Report` and `X_RDK_NLNM`) with `Device.PTM.Link` and
  `Device.ATM.Link` in `config/RdkXdslManager.xml`, the JSON HAL client integration and its
  configuration file `config/xdsl_manager_conf.json`, the HAL contract
  `hal_schema/xdsl_hal_schema.json` with its example messages, and the Apache 2.0 `LICENSE`,
  `COPYING` and `NOTICE`
  ([`f7db450`](https://github.com/rdkcentral/xdsl-manager/commit/f7db450d4eb6eb398dc3ce0898fc8b91c9954520)).
- `Device.DSL.Line.{i}.TestParams.` (RDKBDEV-649)
  ([`b13a8e2`](https://github.com/rdkcentral/xdsl-manager/commit/b13a8e2c717a6ee105f814a49dd6c51c4494ee31)),
  and the `CurrentDayLinkRetrain`, `QuarterHourLinkRetrain` and `EchotoNoiseRatio` parameters in the
  HAL schema (SKYH4-6289)
  ([`3cec334`](https://github.com/rdkcentral/xdsl-manager/commit/3cec3343121cfd38da957b82a6c55cc206a772a0)).
- Configurable DSL WAN interfaces (RDKB-43937, RDKB-46826): with
  `FEATURE_RDKB_CONFIGURABLE_WAN_INTERFACE` the WAN interface is named `vdsl0` for VDSL and
  `pppoa0` for ADSL rather than `erouter0`, and VLAN creation waits for the PTM interface to come up
  ([`ab2ff60`](https://github.com/rdkcentral/xdsl-manager/commit/ab2ff60d0cfc6d8760d3330aeefa0564baf75b8e),
  [`89ae966`](https://github.com/rdkcentral/xdsl-manager/commit/89ae96660360109ce0f5a2028adb7f25c227b028)).
- A `gtest` unit-test harness for the manager (RDKB-38102)
  ([`b3ab174`](https://github.com/rdkcentral/xdsl-manager/commit/b3ab1740165ef4284867dd1420c9c319e1ad294f)).

### Changed

- The manager's architecture and code flow were reworked into a unified DSL Manager implementation
  with an event queue (RDKB-48044)
  ([`8b3b8ee`](https://github.com/rdkcentral/xdsl-manager/commit/8b3b8eec8b878abb3760fab6cba674304c934204)),
  aligned with the converged WAN Manager 1.5 data model (RDKB-48263) and the converged VLAN Manager
  (RDKB-48040)
  ([`f90b7d6`](https://github.com/rdkcentral/xdsl-manager/commit/f90b7d6ea26c1c392d862fc547ee1f57ab8946ae),
  [`83a09e7`](https://github.com/rdkcentral/xdsl-manager/commit/83a09e7450919c822de8eabd50b914554a396454)),
  with the DSL driver initialisation separated from the service start-up (SKYH4-6760)
  ([`8331961`](https://github.com/rdkcentral/xdsl-manager/commit/8331961bf467247d4c0496330c9ffaf743c1d0d9)).
- The ATM and PTM data models are read-only: their set handlers were removed, link creation no
  longer goes through bus calls, and `Link.{i}.Enable` became read-only
  ([`c937b98`](https://github.com/rdkcentral/xdsl-manager/commit/c937b98a6a99649d3bb62f934819039e9aef4408),
  [`03d781b`](https://github.com/rdkcentral/xdsl-manager/commit/03d781bc8ede78f64e9c576e35e69776f152bc87),
  [`a6ff03e`](https://github.com/rdkcentral/xdsl-manager/commit/a6ff03e5a7469d039cd7b64b61ecfd85f39e270a)).
- The process runs at non-root privilege, under the non-root blocklist mechanism
  ([`bdee591`](https://github.com/rdkcentral/xdsl-manager/commit/bdee5916eaf45a350ec36b19720c6f698771eeb0),
  [`d030f9c`](https://github.com/rdkcentral/xdsl-manager/commit/d030f9c6c480ce454ea0374c998d1619e21d4ed0)).
- Line and channel information is read without platform-dependent flags (RDKB-47343)
  ([`c378bbd`](https://github.com/rdkcentral/xdsl-manager/commit/c378bbda87c95c5f399bf4783892c28993514ae9)),
  and logging was reduced and corrected: full line information is logged at debug level, and WAN
  Agent references read WAN Manager
  ([`84757ca`](https://github.com/rdkcentral/xdsl-manager/commit/84757ca2ee81ee711f2cfee6bf5ecb16c5fba849),
  [`07e3136`](https://github.com/rdkcentral/xdsl-manager/commit/07e3136dd19310c527fb2e390d1e30dd6ef49076),
  [`9f4eac7`](https://github.com/rdkcentral/xdsl-manager/commit/9f4eac795618d7e8fe1469cbfdf894bd3098c1ed)).
- Build and code clean-up: obsolete Windows support, the legacy `Makefile`, legacy `AM_CFLAGS`,
  unused declarations and an unused logging variable were dropped, `AnscEqualString()` was replaced
  with `strcmp()`, the `syscfg` header is included explicitly without a `syscfg_init()` call,
  `libparodus.h` is included more robustly, and source files use Unix line endings
  ([`702c488`](https://github.com/rdkcentral/xdsl-manager/commit/702c488a06ea1431efff5f24505f6cdbfe010a3f),
  [`dc6baa7`](https://github.com/rdkcentral/xdsl-manager/commit/dc6baa7dcacafa697019aafd644f9a4dde7e2020),
  [`b27e0b6`](https://github.com/rdkcentral/xdsl-manager/commit/b27e0b6056afd080e2f052f0de24d443bd258cbb),
  [`239eae7`](https://github.com/rdkcentral/xdsl-manager/commit/239eae709e40b50ade8ff9fa9c84c98931ee3959),
  [`4ef7a64`](https://github.com/rdkcentral/xdsl-manager/commit/4ef7a640bb648aa0dd93bef4a703f2aff4d7fd02),
  [`10517d6`](https://github.com/rdkcentral/xdsl-manager/commit/10517d65be4940b0898697094e3daa886a958a60),
  [`4987d6a`](https://github.com/rdkcentral/xdsl-manager/commit/4987d6a4671dbf2007c012257eaad7ff57330cf7),
  [`bb354df`](https://github.com/rdkcentral/xdsl-manager/commit/bb354df70c7919c181d4f161bd186e1d52a6e865),
  [`f967cdc`](https://github.com/rdkcentral/xdsl-manager/commit/f967cdcadac71d893cb875be7601dc98f524de1b),
  [`336abe1`](https://github.com/rdkcentral/xdsl-manager/commit/336abe1ed256e4ae33667e857aa91e627917cf6f),
  [`ba1a948`](https://github.com/rdkcentral/xdsl-manager/commit/ba1a948b6d16088e09a5e63bd5ff88e8c913b7a5)).

### Fixed

- 64-bit crashes: `DmlXdslGetLowerLayersInstanceInOtherAgent` takes its instance number as `ULONG`
  (RDKB_55149, merged through [#2](https://github.com/rdkcentral/xdsl-manager/pull/2))
  ([`704aed7`](https://github.com/rdkcentral/xdsl-manager/commit/704aed7405118e3eef222164bd1e617d74131ea8)),
  and the Broadcom SDK fixes for a crash in the DSL `GetParamValues` data-model calls, which check
  that a value was returned before using it (RDKB-52374, applied, reverted and re-applied)
  ([`ec18ffc`](https://github.com/rdkcentral/xdsl-manager/commit/ec18ffcd67f64b387cd37ce12d4aa4fb8a8f10cc),
  [`bf761cc`](https://github.com/rdkcentral/xdsl-manager/commit/bf761cc64eb6f39ab9a906ea03bcccfb015a31a9),
  [`84a5677`](https://github.com/rdkcentral/xdsl-manager/commit/84a567703c587269a5f9dd470f1b7e9b3dde715f)).
- DSL report values: reports from the cloud returning 0 (SHARMAN-611, SKYH4-6859), the report not
  being sent at once after `OverrideTTL` is set (RDKB-44034), a new `ReportingPeriod` not taking
  effect (RDKB-44086), and the `XTS_Used` field of the report schema (SHARMAN-1165)
  ([`5c660f2`](https://github.com/rdkcentral/xdsl-manager/commit/5c660f2b7406c59d9cbdb15822c6414509adbe40),
  [`36f9d0f`](https://github.com/rdkcentral/xdsl-manager/commit/36f9d0fb7bed26e00dce9f2b00242b215a60fb4f),
  [`f40c6da`](https://github.com/rdkcentral/xdsl-manager/commit/f40c6dad04574fc62b8e20a91b29f080e9311e04),
  [`dda3ed6`](https://github.com/rdkcentral/xdsl-manager/commit/dda3ed63cf14e483ac09e8bd8409eb6ffa01c39f),
  [`4ac21e8`](https://github.com/rdkcentral/xdsl-manager/commit/4ac21e868b81bc193bc25a96a216a9ca260d99e7)).
- WAN connectivity after line events: the manager handles an ADSL line reconnect and a switch
  between ADSL and VDSL (ADAHUB-1622, RDKB-44010), retries an XTM interface creation that failed
  after a reboot (SKYH4-7018, SKYH4-7044), creates its event message queue when it does not yet
  exist (SKYH4-7118), reports the initializing and establishing-link states to WAN Manager
  (RDKB-49965), and reads the initial XTM link state from the HAL before enabling a link after a
  restart (RDKB-49751)
  ([`741829a`](https://github.com/rdkcentral/xdsl-manager/commit/741829ae2b44c6a840de035d324ef8a33d519858),
  [`80e8725`](https://github.com/rdkcentral/xdsl-manager/commit/80e8725c3e68a7a2672936ca11d3a9639e774c12),
  [`5ef8b3a`](https://github.com/rdkcentral/xdsl-manager/commit/5ef8b3a9b4fa42ae0130158ec942ca598cc00ffb),
  [`3919152`](https://github.com/rdkcentral/xdsl-manager/commit/39191529f73c71232edc4fbd97a97d8282486630),
  [`ce0f572`](https://github.com/rdkcentral/xdsl-manager/commit/ce0f5727e2bb5551d88a688093ca80e4c938fe9a),
  [`0055dac`](https://github.com/rdkcentral/xdsl-manager/commit/0055dacce386071c0da8a03cfe85cc7e90ec1299)).
- Data-model values: `LineEncoding` and `PowerManagementState` are mapped from the strings the HAL
  returns (RDKBDEV-668, RDKBDEV-671), channel statistics are requested by channel number so the
  link-retrain counters update (ADAHUB-1533), and `Line.{i}.Upstream` defaults to `true` because a
  DSL line serves only as a WAN interface (RDKB-53068)
  ([`b11d1f1`](https://github.com/rdkcentral/xdsl-manager/commit/b11d1f161c882ee4292ba57e026f23e763e20fb0),
  [`aff51a8`](https://github.com/rdkcentral/xdsl-manager/commit/aff51a8699ab0cadeb0924c24ab90a78a2feb55f),
  [`aaef143`](https://github.com/rdkcentral/xdsl-manager/commit/aaef143e9597b9616c2c84b6de2ce5d172744faa),
  [`5f10a80`](https://github.com/rdkcentral/xdsl-manager/commit/5f10a807d6886a779a7c8395b9ca6815cf23fddc)).
- A `NULL` dereference in `ATMLink_GetEntry` (SHARMAN-1597) and issues reported by Coverity
  (RDKB-37788, RDKB-50135)
  ([`c3ce702`](https://github.com/rdkcentral/xdsl-manager/commit/c3ce702640a91aaff13a9dc4f787702a61d17555),
  [`42ad397`](https://github.com/rdkcentral/xdsl-manager/commit/42ad39775822fa63a5bbca5afc5cb80393b7c30b),
  [`fd8b2bb`](https://github.com/rdkcentral/xdsl-manager/commit/fd8b2bbdbc20b62b1c19a864f7d1a6d21883e0d9)).
