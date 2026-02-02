# Swarms

Multi-agent orchestration skills for Claude Code and Codex. Plan tasks with explicit dependencies, then execute them in parallel across multiple AI agents.

## Skills

| Skill | Description |
|-------|-------------|
| [swarm-planner](./skills/swarm-planner/) | Creates dependency-aware implementation plans optimized for parallel execution |
| [parallel-task](./skills/parallel-task/) | Executes plans by launching subagents in dependency-ordered waves |
| [context7](./skills/context7/) | Fetches up-to-date library documentation via Context7 API |

## How It Works

### 1. Plan with Dependencies

`/swarm-planner` creates plans where each task declares what it depends on:

```
T1: [depends_on: []] Create database schema
T2: [depends_on: []] Install packages
T3: [depends_on: [T1]] Create repository layer
T4: [depends_on: [T1]] Create service interfaces
T5: [depends_on: [T3, T4]] Implement business logic
T6: [depends_on: [T2, T5]] Add API endpoints
```

Tasks with empty or satisfied dependencies can run in parallel.

### 2. Execute in Waves

`/parallel-task` reads the plan and launches agents in waves:

| Wave | Tasks | Runs When |
|------|-------|-----------|
| 1 | T1, T2 | Immediately (no dependencies) |
| 2 | T3, T4 | After T1 completes |
| 3 | T5 | After T3 and T4 complete |
| 4 | T6 | After T2 and T5 complete |

Each wave runs all unblocked tasks in parallel, maximizing throughput while respecting dependencies.

## Installation

### Claude Code

```bash
# Install both skills
cp -r skills/swarm-planner ~/.claude/skills/
cp -r skills/parallel-task ~/.claude/skills/

# Or via codexskills CLI
npx @am-will/codexskills --user am-will/swarms/skills/swarm-planner
npx @am-will/codexskills --user am-will/swarms/skills/parallel-task
```

### Codex

```bash
cp -r skills/swarm-planner ~/.codex/skills/
cp -r skills/parallel-task ~/.codex/skills/
```

## Usage

### Planning

```
/swarm-planner

# The planner will:
# 1. Research your codebase
# 2. Fetch docs for any libraries (via Context7)
# 3. Ask clarifying questions
# 4. Generate a dependency-aware plan
# 5. Have a subagent review for gaps
# 6. Save to <topic>-plan.md
```

### Execution

```
/parallel-task auth-plan.md           # Run full plan
/parallel-task auth-plan.md T1 T2 T4  # Run specific tasks
```

## Plan Format

Plans use this structure for each task:

```markdown
### T1: Create User Model
- **depends_on**: []
- **location**: src/models/user.ts
- **description**: Define User entity with fields...
- **validation**: Unit tests pass
- **status**: Not Completed
- **log**: [filled by executor]
- **files edited/created**: [filled by executor]
```

The executor updates `status`, `log`, and `files` as work completes.

## Workflow Example

```
User: Add authentication to my Express app

> /swarm-planner

Planner: [researches codebase, fetches Express/JWT docs]
         [asks: "OAuth or JWT? Session storage preference?"]
         [generates auth-plan.md with 8 tasks]

User: looks good, run it

> /parallel-task auth-plan.md

Executor: Wave 1 - Launching T1 (schema), T2 (packages) in parallel...
          Wave 1 complete. Wave 2 - Launching T3, T4, T5 in parallel...
          [continues until all tasks complete]

Summary: 8/8 tasks completed. Files modified: [list]
```

## Key Features

- **Explicit dependencies** - No implicit ordering; parallelization is maximized
- **Atomic tasks** - Each task is independently executable
- **Progress tracking** - Plans update with logs as work completes
- **Validation** - Each task includes acceptance criteria
- **Review step** - Subagent reviews plans before execution
- **Wave-based execution** - Respects dependencies while maximizing parallelism

## License

MIT
