# Exploration Docs Workflow

## Answering questions vs. new research

- **Existing topics**: Read the relevant `exploration/` doc first. Only search `openclaw/src/` or `openclaw/docs/` if the doc doesn't cover the question.
- **New research**: Deliverable is a markdown file in the appropriate subdirectory (e.g. `exploration/agents/topic-name.md`). Run research in background subtasks — never block the main conversation.

## Writing docs

- Update `exploration/README.md` when adding or completing documents
- Every claim must reference a source file (`src/path/to/file.ts:line`) or official doc (`docs/path/to/page.md`) — paths are relative to the `openclaw/` repo root
- Cross-reference `openclaw/docs/` before writing — avoid duplicating existing documentation
- Consult `exploration/external-resources.md` for curated external references before web-searching
- One topic per file
