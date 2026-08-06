# Safety notes

`crucible` ships one small validator script: `scripts/validate-findings.cjs`. This document describes what it does.

## `scripts/validate-findings.cjs`

Reads a findings JSON file (the path is passed as a command-line argument) and validates every entry against the schema at `assets/findings-schema.json`, which it also reads at runtime. The schema is the single source of truth: the script interprets a small JSON Schema subset (`type`, `properties`, `required`, `additionalProperties: false`, `enum`, `const`, `items`, `minItems`, `oneOf`) with a hand-rolled interpreter, no external validation library.

On top of the structural check, it applies one semantic rule that isn't expressible in the schema subset: when run with `--pre-spec`, any finding with `severity` of `critical` or `high` and `verdict: "confirmed"` must not have `verification: "pending"`. This check only runs when the `--pre-spec` flag is passed; without it, only structural validation runs.

For each finding, errors are printed to stderr with the finding's index, id, and title. The script exits `0` when every finding passes and `1` if any error was found.

## What it doesn't do

Zero dependencies: only Node's built-in `fs` and `path` modules. It only reads the findings file and the schema file, both from paths given to it, either as an argument or resolved relative to its own directory (`assets/findings-schema.json`, one level up from `scripts/`). It never writes, deletes, or modifies any file, and it makes no network calls. Its side effects are limited to console output and its process exit code.
