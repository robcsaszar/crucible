# AGENTS.md

## Mission

This repo publishes the `crucible` skill: a single skill under `skills/crucible/`. There is no build, no tests, no runtime of its own. The deliverable is the contents of that one directory. Changes should be judged by: would a stranger who installs this skill into their own project get a correct, generically useful audit workflow out of it, one that doesn't assume Orakl's codebase, stack, or conventions?

## Judgment boundaries

ASK:
- Ask before removing the skill or restructuring its phases; this repo has exactly one skill and no fallback if it regresses.

ALWAYS:
- Don't let the skill's content drift toward assumptions specific to any one codebase; it needs to read as generic audit guidance for any project.
- Changes to `skills/crucible/scripts/validate-findings.cjs` need a matching update to [`SAFETY.md`](SAFETY.md) in the same change.

## Releasing

Releases are cut by the **Release** workflow (`.github/workflows/release.yml`), never by hand. It is a manual `workflow_dispatch` with one input, `tag`, and it releases the commit at the tip of the branch it is run on. Run it on `main`.

The workflow, in order:
1. Rejects a tag that is not `vX.Y.Z`, or that already exists.
2. Fails unless `version` in `.claude-plugin/plugin.json` and `plugins[0].version` in `.claude-plugin/marketplace.json` both equal `X.Y.Z`.
3. Takes the `## [X.Y.Z]` block from `CHANGELOG.md` as the release notes, and fails if there is none.
4. Creates the tag at the checked-out commit and publishes the GitHub release with those notes.

Nothing is created until every check passes, so a failed run leaves nothing to clean up.

To prepare a release, in one PR:
- Set the same new version in `.claude-plugin/plugin.json` and `.claude-plugin/marketplace.json`.
- Add a `## [X.Y.Z] - YYYY-MM-DD` block at the top of `CHANGELOG.md`, and its `[X.Y.Z]: …` link reference at the bottom.
- Merge to `main`.

Then run the workflow on `main` with `tag=vX.Y.Z`, from the Actions tab (**Release → Run workflow**) or from a shell:

```sh
gh workflow run release.yml --ref main -f tag=vX.Y.Z
```

Afterwards, confirm the release exists and its notes match the changelog block.

NEVER:
- Never push a tag or create a release outside the workflow. A hand-made tag makes the workflow refuse that version, and a hand-written release skips the changelog and version checks.
- Never work around a failed run by hand-writing notes or skipping a check. Fix the changelog or the manifests, merge, and re-run.
