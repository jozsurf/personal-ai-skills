## Rule: Skill Priority
TRIGGER: Any CargoWise task
ACTION: Invoke `cw-coding` skill FIRST, BEFORE any other skill or action.

TRIGGER: Questions about how features/concepts work, finding implementations, exploring patterns, or understanding business logic (e.g., "how does X work", "understand Y", "where is Z implemented", "find examples of")
ACTION: Invoke `code-research` skill FIRST to dispatch parallel search agents.

## Rule: Post-Change Verification
TRIGGER: Any code change
ACTION: After making code changes, always:
1. Run related unit tests using `dotnet test` scoped to the affected test projects
2. Run `dotnet format style` on the changed files to catch style issues (including unused usings)
3. Run `dotnet format analyzers` on the changed files to catch analyzer violations
Both `style` and `analyzers` must be run — they cover different classes of issues.

<!-- WORKSPACE-RULES:START -->
## Workspace Repositories
The following repositories are part of this workspace:
- ../CargoWise - CargoWise main application
- ../CargoWise.Shared - Cargowise shared components
- ../CargoWise.Customs
- ../Dev-Workspaces
- ../issue.Triage.Agent
- ../WTG.AI.Prompts

## Rule: Use Repository Paths
TRIGGER: File operations (read, search, modify)
ACTION: Use the repository paths listed above, not assumptions
<!-- WORKSPACE-RULES:END -->
