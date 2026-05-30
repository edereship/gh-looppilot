# `gh looppilot` — LoopPilot setup CLI

**English** | [日本語](README.ja.md)

A [GitHub CLI extension](https://docs.github.com/en/github-cli/github-cli/creating-github-cli-extensions)
that turns LoopPilot adoption (otherwise ~461 lines of copy-pasted YAML) into one
command. It generates the thin caller workflows (referencing the reusable
workflows at `team-yubune/loop-pilot@v1`), creates the gate label, suggests a
`CHECK_COMMAND`, lists the manual setup steps, and runs a read-only pre-flight
against your authenticated `gh` session.

Distribution rationale and the Action-vs-CLI release boundary:
[ADR-0001](https://github.com/team-yubune/loop-pilot/blob/main/docs/architecture/adr-0001-cli-distribution.md)
in the main repo. This repo is the canonical home of the extension (a gh
extension must live in a `gh-<name>` repo).

## Install

```bash
gh extension install team-yubune/gh-looppilot
gh looppilot init      # scaffold callers + label + CHECK_COMMAND + manual steps + pre-flight
gh looppilot doctor    # read-only pre-flight only
```

Requires Node ≥ 20 on PATH (the extension is an interpreted Node CLI; the shim
`gh-looppilot` execs `node cli.cjs`).

## Commands

| Command | What it does |
|---|---|
| `gh looppilot init` | Detect the toolchain, write `.github/workflows/looppilot-{init,loop}.yml` (thin callers pinned to `@v1`), create the gate label, suggest `CHECK_COMMAND`, print the manual (non-automatable) steps, then run pre-flight. |
| `gh looppilot doctor` | Run the read-only pre-flight only (= `init --preflight-only`). |

Key flags: `--full-auto` (no gate label), `--same-repo` (`secrets: inherit`),
`--label <name>`, `--check-command <c>`, `--ref <ref>`, `--repo <owner/repo>`,
`--dry-run`, `--force`, `--no-preflight`, `--json` (doctor). See `gh looppilot --help`.

Pre-flight exit codes: `0` = no errors (warnings/unknown allowed), `1` = an error
to fix before the first PR, `2` = the check run itself could not proceed (auth /
repo resolution).

## Pre-flight checks (`doctor`)

Read-only checks against the developer's authenticated `gh` session. Each surfaces
one of the silent-failure classes that otherwise only appear after the first PR.
A check that lacks permission to determine its answer returns `unknown` (never a
silent pass); 403s degrade per-probe.

| Check id | Surfaces | Statuses |
|---|---|---|
| `label.gate` | Gate label missing → no Actions run is generated | ok / error / unknown |
| `secret.anthropicAuth` | Anthropic credential dual-set / unset (repo + org secrets) | ok / error / unknown |
| `codex.connection` | Codex GitHub App connection (inferred from recent bot activity) | ok / **unknown** |
| `secret.loopPilotPushToken` | Required checks / auto-merge active but push token missing | ok / warning / unknown |
| `autoMerge.config` | `LOOPPILOT_AUTO_MERGE=true` but repo "Allow auto-merge" off | ok / error / unknown |
| `secret.codexReviewToken` | `CODEX_REVIEW_REQUEST_TOKEN` missing (recommended) | ok / warning / unknown |
| `toolchain.checkCommand` | CHECK_COMMAND unsafe / inconsistent with the detected toolchain | ok / warning / error |

### `--json` schema

```json
{
  "ok": false,
  "repository": "owner/repo",
  "checks": [
    {
      "id": "secret.loopPilotPushToken",
      "status": "ok|warning|error|unknown",
      "summary": "short human text",
      "details": "actionable detail or null",
      "nextSteps": ["concrete command or UI step"]
    }
  ]
}
```

`ok` is `false` iff any check is `error`. Field order is stable. Exit code mirrors
`ok` (0/1), or `2` when the run could not start.

## Development

```bash
npm install
npm run cli -- doctor --json   # run from source via tsx
npm run check                  # tsc --noEmit + vitest run
npm run bundle                 # rebuild the committed cli.cjs (esbuild)
gh extension install .         # install this local checkout as the extension
```

`cli.cjs` is the committed bundle the installed extension runs (gh does not
`npm install` an interpreted extension). **Rebuild and commit it whenever `src/`
changes** (`npm run bundle`).

Layout: `gh-looppilot` (shim) · `cli.cjs` (committed bundle) · `src/` (TS) ·
`tests/` (vitest).

## License

MIT (same as [team-yubune/loop-pilot](https://github.com/team-yubune/loop-pilot)).
