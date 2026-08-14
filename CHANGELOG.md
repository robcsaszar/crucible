# Changelog

All notable changes to this project are documented here. Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/); versioning follows [SemVer](https://semver.org/).

## [0.6.0] - 2026-08-14

### Changed

- Added a failure branch for `validate-findings.cjs`. A schema rejection still blocks; a validator that cannot run (missing `node`, missing script or schema, usage error) now marks the findings `⚠ unvalidated` in both the findings output and the SPEC handoff instead of silently reading as validated. Both cases exit `1`, so the branch keys on the error text.
- Description rewritten as trigger conditions rather than a phase summary, and given a negative trigger (single-file review, bugs with a known cause). Trigger phrasing removed — the skill sets `disable-model-invocation: true`, so its description is human-facing only.

## [0.5.0] - 2026-07-10

### Added

- Initial release: crucible skill.

[0.5.0]: https://github.com/robcsaszar/crucible/releases/tag/v0.5.0
