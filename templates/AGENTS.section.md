<!--
  Paste this section into your target repo's AGENTS.md.
  The loop skills read it at runtime to learn HOW to test/lint/build YOUR
  project. These commands are the single source of truth — the skills never
  hard-code them. Replace the placeholders with your project's real commands.
-->

## Loop Commands

Commands the loop-coder / loop-reviewer skills run to verify changes. Use the
exact form an agent can execute non-interactively from the repo root.

- **test**: `<your test command>`        <!-- e.g. pytest -q  |  npm test  |  go test ./... -->
- **lint**: `<your lint command>`        <!-- e.g. ruff check . |  npm run lint  |  golangci-lint run -->
- **build**: `<your build command>`      <!-- e.g. npm run build |  make build  | (omit if none) -->
- **typecheck**: `<optional>`            <!-- e.g. mypy . |  npm run typecheck  | (optional) -->

### Conventions (optional but recommended)

A few project-specific rules the reviewer should enforce and the coder should
follow — the "we don't do it like this because of that one incident" knowledge:

- <convention 1>
- <convention 2>
