# Contributing

Thanks for considering a contribution to `crucible`. This repo publishes a single skill; contributions are welcome, but keep changes proportionate to that scope.

## Before you start

- **Bug in the audit workflow?** Open an issue with a concrete example: which phase, what went wrong, what you expected instead.
- **Bigger changes** (new phase, new output artifact, changed schema) are worth raising as an issue before writing code.

## Making a change

1. Fork and branch from `main`.
2. Keep the skill's phase structure intact unless the issue you opened specifically proposes changing it.
3. If your change touches `scripts/validate-findings.cjs` or `assets/findings-schema.json`, update [`SAFETY.md`](SAFETY.md) in the same change.
4. Make sure the skill still reads as generic guidance: no assumptions specific to one codebase or stack.

## Quality bar

The skill should stay generically useful outside the project it was originally built for. A PR that ties its instructions to one repo's conventions, or that regresses the clarity of a phase, will be asked to fix that before merge.

## Questions

Open an issue. There's no separate chat or forum for this project.
