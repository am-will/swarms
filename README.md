<img width="1024" height="559" alt="image" src="https://github.com/user-attachments/assets/778c9168-b39a-4b2f-8a4c-1e06ee00d98c" />

# Swarms

Multi-agent orchestration for Claude Code and Codex. Plan with explicit dependencies, execute in parallel waves, with an orchestrator that maintains context and verifies work.

## The Problem with Simple Loops

A popular pattern for AI coding agents is the bash loop: spawn an agent, have it find the next task, execute it, repeat. It mirrors how humans work, logical and straightforward.

But LLMs aren't humans. Each new agent session starts fresh. It has to:
- Re-read the same prompt as every previous iteration
- Discover the current project state through exploration
- Figure out what comes next by inspecting previous work
- Hope the last agent didn't cut corners

There's no continuity. No one checking work. No context passed between agents. Each iteration burns tokens re-discovering what the orchestrator already knows.

## A Different Approach

Swarms takes lessons from simple loops and more complex multi-agent systems, packaging them into something effective and beginner-friendly.

The key insight: **an orchestrator that plans the work should also coordinate the execution**. It already has the complete picture: codebase research, documentation, clarified requirements, a reviewed plan. Why throw that away?

### How It Works

**Phase 1: Planning** (`/swarm-planner`)

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/27fc3022-bf7e-41b8-a805-0e3092d99fcf" />


The planner researches your codebase, fetches current library docs via Context7, asks clarifying questions when it detects ambiguity, and produces a dependency-aware plan. Before finishing, it spawns a subagent to review the plan for gaps.

The critical addition: **explicit task dependencies**. Every task declares what it depends on, enabling the orchestrator to know exactly which tasks can run in parallel at any moment.

**Phase 2: Execution** (`/parallel-task`)

The orchestrator (same session that planned) parses unblocked tasks and launches subagents in parallel. Because it planned the work, it can provide each subagent with:
- The plan file and structure
- Exactly which files to work on
- The correct filenames/paths for any dependencies and/or previous tasks
- Relevant context from the plan
- What adjacent tasks are doing
- Current project state with detailed logs

This dramatically reduces the tokens each subagent spends on discovery. Instead of grepping, searching, and exploring, they know exactly where to go and what to build.

When subagents complete, they:
1. Run validation/tests if feasible
2. Commit their work (never push: other agents are working in parallel)
3. Update the plan with status, work log, and files modified
4. Return a summary to the orchestrator

The orchestrator then **verifies the work** before moving to the next wave. If something went wrong, it catches and fixes it immediately, not three tasks later when everything is tangled.

This continues wave by wave until all tasks are complete.

## Why Not Just Use Ralph Loops or Gas Town?

**vs. Ralph Loops**: The orchestrator maintains context across the entire project. It checks work, provides upfront context to subagents, and catches issues between waves. Simple loops have no oversight. Each agent checks its own work and hopes for the best.

**vs. Complex Multi-Agent Systems**: Systems like Gas Town use families of specialized agents (context managers, commit handlers, conflict resolvers, etc.) working independently. Powerful, but complex and token-hungry. Swarms trades some parallelism for simplicity. Waves execute sequentially, but within each wave, tasks run in parallel. No scripts, no complex setup, no conflict resolution needed because dependencies are explicit.

**The tradeoff**: Time. Swarms works in phases. A more complex system might have dozens of agents working simultaneously. But you'll spend fewer tokens, encounter fewer conflicts, and can actually understand what's happening.

## Skills

| Skill | Description |
|-------|-------------|
| [swarm-planner](./skills/swarm-planner/) | Dependency-aware planning with codebase research, doc fetching, and subagent review |
| [parallel-task](./skills/parallel-task/) | Wave-based execution with context handoff and work verification |
| [context7](./skills/context7/) | Fetches up-to-date library documentation via Context7 API |

## Installation

### Claude Code

```bash
npx skills add am-will/swarms --skill swarm-planner -g -a claude-code -y
npx skills add am-will/swarms --skill parallel-task -g -a claude-code -y
npx skills add am-will/swarms --skill context7 -g -a claude-code -y
```

### Codex

```bash
npx skills add am-will/swarms --skill swarm-planner -g -a codex -y
npx skills add am-will/swarms --skill parallel-task -g -a codex -y
npx skills add am-will/swarms --skill context7 -g -a codex -y
```

## Usage

I strongly recommend you download the Context7 Skill, or use their MCP server. To use the Context7 skill, you must add your Context7 API key to .env file in the /context7/ skill folder. There is an .env.example file you can edit.

### Planning

**For Codex Users:**
The swarm-planner skill works best when used in PLAN Mode. If you don't have plan mode enabled, it will still work, but this allows you to take advantage of Codex's new Request User Input tool, which is currently only available in Plan Mode. 

**Codex instructions:**

```
1. Switch to Plan Mode
2. Provide the prompt of the product/feature you want to build and call the $swarm-planner skill directly in the prompt
3. Answer any questions it asks during the planning
4. At the end of the plan, when it asks you if you want to implement this plan, press Esc
5. Switch out of Plan Mode by pressing SHIFT+TAB
6. SAVE THE PLAN to your repo. [feature/product]-plan.md
7. Continue on to Execution phase
```

**Claude Code instructions:**

```
1. You don't need to switch to plan mode. You can just run the skill with your prompt.
2. /swarm-planner
3. Answer any questions it asks during planning
4. Continue on to Execution phase
```

The planner will:
1. Research your codebase architecture
2. Fetch current docs for libraries via Context7
3. Ask clarifying questions (with recommendations) when ambiguity exists
4. Generate a plan with explicit task dependencies
5. Spawn a subagent to review for gaps
6. Save to `<topic>-plan.md`

**NOTE: Codex in Plan Mode cannot save the plan to a file. Esc out, switch to Code Mode, and ask it to save the file to your repo. Proceed to execution**

### Execution

To run the swarms, simply say implement the @plan.md with the $parallel-task skill.

```
Prompt: Run the plan with the parallel task skill
```

## Dependency Format

Tasks declare dependencies explicitly:

```
T1: [depends_on: []] Create database schema
T2: [depends_on: []] Install packages
T3: [depends_on: [T1]] Create repository layer
T4: [depends_on: [T1]] Create service interfaces
T5: [depends_on: [T3, T4]] Implement business logic
T6: [depends_on: [T2, T5]] Add API endpoints
```

This produces execution waves:

| Wave | Tasks | Runs When |
|------|-------|-----------|
| 1 | T1, T2 | Immediately |
| 2 | T3, T4 | After T1 |
| 3 | T5 | After T3, T4 |
| 4 | T6 | After T2, T5 |

## Plan Structure

Each task in the plan includes:

```markdown
### T1: Create User Model
- **depends_on**: []
- **location**: src/models/user.ts
- **description**: Define User entity with required fields
- **validation**: Unit tests pass
- **status**: Not Completed
- **log**: [updated by subagent on completion]
- **files edited/created**: [updated by subagent on completion]
```

The `status`, `log`, and `files` fields are updated by subagents as work completes, giving the orchestrator (and you) visibility into progress.

## Example Session

```
> User: Add authentication to my Express app using the swarm-planner skill.
-or-
> /swarm-planner add authentication to my express app.
-or-
> Use swarm planner skill to help me build a b2b website to sell AI agents


[researches codebase, fetches Express/Passport/JWT docs]
[asks: "JWT or session-based? Where should tokens be stored?"]
[generates auth-plan.md with 8 dependency-ordered tasks]
[subagent reviews plan, suggests adding rate limiting task]
[updates plan, saves]

User: looks good, execute it

> /parallel-task auth-plan.md
-or-
> execute the plan using parallel task skill
 
Wave 1: Launching T1 (user schema), T2 (install packages) in parallel...
  T1 complete - committed, plan updated
  T2 complete - committed, plan updated
  Verifying wave 1... OK

Wave 2: Launching T3 (auth service), T4 (middleware) in parallel...
  [continues until all tasks complete]

Summary: 8/8 tasks completed
Files modified: src/models/user.ts, src/middleware/auth.ts, ...
```

## License

MIT
