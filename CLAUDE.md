## Rule: Skill Priority

TRIGGER: Any CargoWise task
ACTION: Invoke `cw-coding` skill FIRST, BEFORE any other skill or action.

TRIGGER: Writing, updating, reviewing, fixing, or debugging agent customization files and prompt assets (e.g. `SKILL.md`, `*.prompt.md`, `*.instructions.md`, `*.agent.md`, `copilot-instructions.md`, `AGENTS.md`)
ACTION: Invoke `agent-customization` skill FIRST.

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
- ../CargoWise.Customs - CargoWise customs modules
- ../Dev-Workspaces - Dev workspace orchestration scripts and templates
- ../WTG.AI.Prompts - Instructions, prompts and skills for use with AI Agents

## Rule: Use Repository Paths

TRIGGER: File operations (read, search, modify)
ACTION: Use the repository paths listed above, not assumptions

## Rule: Always use a Dev Workspace For Code Changes

TRIGGER: Any request that requires code changes (regardless of repository), unless the user explicitly instructs otherwise
ACTION: First, check if there is already a Dev-Workspace configured for the branch to be worked on. If not, create or resolve a Dev-Workspace first by following ../Dev-Workspaces/AGENTS.md, then perform code changes in that workspace

<!-- WORKSPACE-RULES:END -->

## Rule: MCP OAuth Recovery

TRIGGER: MCP tool calls fail with "certificate verification error", "unauthorized", "401", or authentication-related errors
ACTION: Run the full browser-based OAuth fix script:
```
C:/Users/Joseph.Poh/AppData/Local/Programs/Python/Python312/python.exe C:/git/GitHub/WiseTechGlobal/cargowise-claude/scripts/fix-claude-mcp-oauth.py
```
Note: A SessionStart hook silently refreshes tokens on startup. This manual step is only needed when the refresh token itself has expired (typically after 12-168h of inactivity).

<!-- rtk-instructions v2 -->
# RTK (Rust Token Killer) - Token-Optimized Commands

## Golden Rule

**Always prefix commands with `rtk`**. If RTK has a dedicated filter, it uses it. If not, it passes through unchanged. This means RTK is always safe to use.

**Important**: Even in command chains with `&&`, use `rtk`:
```bash
# ❌ Wrong
git add . && git commit -m "msg" && git push

# ✅ Correct
rtk git add . && rtk git commit -m "msg" && rtk git push
```

## RTK Commands by Workflow

### Build & Compile (80-90% savings)
```bash
rtk cargo build         # Cargo build output
rtk cargo check         # Cargo check output
rtk cargo clippy        # Clippy warnings grouped by file (80%)
rtk tsc                 # TypeScript errors grouped by file/code (83%)
rtk lint                # ESLint/Biome violations grouped (84%)
rtk prettier --check    # Files needing format only (70%)
rtk next build          # Next.js build with route metrics (87%)
```

### Test (90-99% savings)
```bash
rtk cargo test          # Cargo test failures only (90%)
rtk vitest run          # Vitest failures only (99.5%)
rtk playwright test     # Playwright failures only (94%)
rtk test <cmd>          # Generic test wrapper - failures only
```

### Git (59-80% savings)
```bash
rtk git status          # Compact status
rtk git log             # Compact log (works with all git flags)
rtk git diff            # Compact diff (80%)
rtk git show            # Compact show (80%)
rtk git add             # Ultra-compact confirmations (59%)
rtk git commit          # Ultra-compact confirmations (59%)
rtk git push            # Ultra-compact confirmations
rtk git pull            # Ultra-compact confirmations
rtk git branch          # Compact branch list
rtk git fetch           # Compact fetch
rtk git stash           # Compact stash
rtk git worktree        # Compact worktree
```

Note: Git passthrough works for ALL subcommands, even those not explicitly listed.

### GitHub (26-87% savings)
```bash
rtk gh pr view <num>    # Compact PR view (87%)
rtk gh pr checks        # Compact PR checks (79%)
rtk gh run list         # Compact workflow runs (82%)
rtk gh issue list       # Compact issue list (80%)
rtk gh api              # Compact API responses (26%)
```

### JavaScript/TypeScript Tooling (70-90% savings)
```bash
rtk pnpm list           # Compact dependency tree (70%)
rtk pnpm outdated       # Compact outdated packages (80%)
rtk pnpm install        # Compact install output (90%)
rtk npm run <script>    # Compact npm script output
rtk npx <cmd>           # Compact npx command output
rtk prisma              # Prisma without ASCII art (88%)
```

### Files & Search (60-75% savings)
```bash
rtk ls <path>           # Tree format, compact (65%)
rtk read <file>         # Code reading with filtering (60%)
rtk grep <pattern>      # Search grouped by file (75%)
rtk find <pattern>      # Find grouped by directory (70%)
```

### Analysis & Debug (70-90% savings)
```bash
rtk err <cmd>           # Filter errors only from any command
rtk log <file>          # Deduplicated logs with counts
rtk json <file>         # JSON structure without values
rtk deps                # Dependency overview
rtk env                 # Environment variables compact
rtk summary <cmd>       # Smart summary of command output
rtk diff                # Ultra-compact diffs
```

### Infrastructure (85% savings)
```bash
rtk docker ps           # Compact container list
rtk docker images       # Compact image list
rtk docker logs <c>     # Deduplicated logs
rtk kubectl get         # Compact resource list
rtk kubectl logs        # Deduplicated pod logs
```

### Network (65-70% savings)
```bash
rtk curl <url>          # Compact HTTP responses (70%)
rtk wget <url>          # Compact download output (65%)
```

### Meta Commands
```bash
rtk gain                # View token savings statistics
rtk gain --history      # View command history with savings
rtk discover            # Analyze Claude Code sessions for missed RTK usage
rtk proxy <cmd>         # Run command without filtering (for debugging)
rtk init                # Add RTK instructions to CLAUDE.md
rtk init --global       # Add RTK to ~/.claude/CLAUDE.md
```

## Token Savings Overview

| Category | Commands | Typical Savings |
|----------|----------|-----------------|
| Tests | vitest, playwright, cargo test | 90-99% |
| Build | next, tsc, lint, prettier | 70-87% |
| Git | status, log, diff, add, commit | 59-80% |
| GitHub | gh pr, gh run, gh issue | 26-87% |
| Package Managers | pnpm, npm, npx | 70-90% |
| Files | ls, read, grep, find | 60-75% |
| Infrastructure | docker, kubectl | 85% |
| Network | curl, wget | 65-70% |

Overall average: **60-90% token reduction** on common development operations.
<!-- /rtk-instructions -->