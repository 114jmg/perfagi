# Ledger of edits to upstream files (reconciled at every merge)

Format: `file | intent | expected conflict if upstream touches the same file`

> This file is the accounting for the follow-upstream strategy. Every edit to a file that exists upstream gets a row here; new files need no row (they never conflict). Read this file before merging upstream — conflicts can only appear in files listed here.

## Recorded edits

| Date | File | Intent | Expected conflict point |
|---|---|---|---|
| (empty) | | | |

## Planned entries (not yet applied)

| Date | File | Intent | Expected conflict point |
|---|---|---|---|
| pending | `backend/pkg/templates/templates.go` | register the `performer` PromptType and its variable allowlist | same-region hunk when upstream adds a PromptType |
| pending | `backend/pkg/tools/registry.go` | register perf tool-name constants (run_experiment and friends) | when upstream adds tool constants |
| pending | `backend/pkg/graph/models_gen.go` | add a performance entry to the AgentsPrompts enum | when upstream regenerates for agent-list changes (gqlgen) |
| pending | `backend/pkg/providers/providers.go` | hang the performer delegation tool off primary_agent's tool table | when upstream restructures specialist delegation |
| pending | `backend/pkg/database/models.go` | MsgchainType role constant for the performer | when upstream extends the role enum |
| pending | `backend/pkg/templates/prompts/` | new performer prompt files (new files, listed because the directory is upstream) | file-level, essentially conflict-free |
| pending | `backend/pkg/graph/schema.graphqls` | AgentsPrompts enum value; regenerate gqlgen + swag + `pnpm run graphql:generate` | same-region hunk when upstream edits the enum |
| pending | `frontend/src/pages/settings/settings-providers.tsx`-style registration points | any UI registration of the new agent type | mirrored upstream UI changes |
