# Agent Zero + Ralph Wiggum + GSD Quick Start

The ultimate autonomous development stack: Agent Zero orchestrates, GSD plans, Ralph executes.

```
┌─────────────────────────────────────────────────────┐
│                    Agent Zero                        │
│              (Orchestrator + Memory)                 │
│                                                      │
│  ┌─────────────────┐      ┌─────────────────┐       │
│  │   GSD Planner   │ ───▶ │  Ralph Executor │       │
│  │                 │      │                 │       │
│  │ • PROJECT.md    │      │ • Loop until    │       │
│  │ • ROADMAP.md    │      │   complete      │       │
│  │ • PLAN.md       │      │ • Auto-retry    │       │
│  │ • Checkpoints   │      │ • Git commits   │       │
│  └─────────────────┘      └─────────────────┘       │
└─────────────────────────────────────────────────────┘
```

## Setup Checklist

### 1. Agent Zero Setup

```bash
# Clone Agent Zero
git clone https://github.com/agent0ai/agent-zero.git
cd agent-zero

# Install dependencies
pip install -r requirements.txt

# Configure your LLM (Claude recommended)
cp .env.example .env
# Edit .env with your API keys
```

### 2. Add GSD Tools to Agent Zero

```bash
# Copy GSD tool
cp /path/to/gsd_tool.py python/tools/gsd_tool.py

# Copy GSD instrument
cp /path/to/gsd_planner.py python/instruments/gsd_planner.py

# Add GSD prompts
mkdir -p prompts/gsd
cp /path/to/gsd-prompts/* prompts/gsd/
```

**Minimal `python/tools/gsd_tool.py`:**

```python
from python.helpers.tool import Tool, Response
from pathlib import Path
import json
import re

class GsdTool(Tool):
    async def execute(self, **kwargs):
        method = self.args.get("method", "get_context")
        project_path = Path(self.args.get("project_path", "."))
        planning_dir = project_path / ".planning"

        if method == "get_context":
            context = {}
            for doc in ["PROJECT.md", "ROADMAP.md", "STATE.md"]:
                path = planning_dir / doc
                if path.exists():
                    context[doc] = path.read_text()
            return Response(message=json.dumps(context), break_loop=False)

        elif method == "parse_plan":
            plan_path = Path(self.args.get("plan_path"))
            plan = plan_path.read_text()
            tasks = []
            for match in re.finditer(r'<task[^>]*>(.*?)</task>', plan, re.DOTALL):
                task = {}
                for field in ["name", "action", "verify", "done"]:
                    m = re.search(f'<{field}>(.*?)</{field}>', match.group(1), re.DOTALL)
                    if m:
                        task[field] = m.group(1).strip()
                tasks.append(task)
            return Response(message=json.dumps(tasks), break_loop=False)

        elif method == "create_ralph_prompt":
            plan_path = self.args.get("plan_path")
            prompt = f"""Execute GSD plan: {plan_path}

For each <task>:
1. Implement <action>
2. Run <verify> - retry on failure
3. Commit: feat(phase): task-name

When ALL tasks pass: <promise>COMPLETE</promise>
If stuck 5+ iterations: <promise>BLOCKED</promise>"""
            return Response(message=prompt, break_loop=False)

        return Response(message=f"Unknown method: {method}", break_loop=False)
```

### 3. Install Ralph Wiggum (for Claude Code subordinates)

```bash
# If using Claude Code as a subordinate executor
claude "/plugin marketplace add anthropics/claude-code"
claude "/plugin install ralph-wiggum@claude-plugins-official"
```

### 4. Create the Orchestration Workflow

Add to your Agent Zero main loop or create `workflows/gsd_ralph.py`:

```python
async def autonomous_build(agent, user_request: str):
    """
    Full autonomous workflow:
    1. Agent Zero understands request
    2. GSD creates structured plan
    3. Ralph loops until complete
    """

    # Phase 1: Project Setup (if needed)
    context = await agent.call_tool("gsd_tool", {
        "method": "get_context",
        "project_path": "."
    })

    if not context or "PROJECT.md" not in context:
        # Initialize project
        await agent.call_subordinate(
            message=f"""Initialize a GSD project for: {user_request}

            Ask clarifying questions, then create:
            1. .planning/PROJECT.md - project vision and requirements
            2. .planning/ROADMAP.md - phased breakdown
            3. .planning/STATE.md - execution tracking

            Return "INITIALIZED" when done.""",
            reset_context=True
        )

    # Phase 2: Create Plan
    plan_result = await agent.call_subordinate(
        message=f"""Create a GSD plan for the next phase.

        Request: {user_request}

        Create .planning/phases/current/PLAN.md with:
        - <objective> what we're building
        - 2-3 <task> elements, each with:
          - <name> descriptive title
          - <action> specific implementation steps
          - <verify> executable test command
          - <done> measurable completion criteria

        Return the plan path when complete.""",
        reset_context=True
    )

    plan_path = extract_path(plan_result)  # Parse from response

    # Phase 3: Execute with Ralph
    ralph_prompt = await agent.call_tool("gsd_tool", {
        "method": "create_ralph_prompt",
        "plan_path": plan_path
    })

    execution_result = await agent.call_subordinate(
        message=f"""/ralph-loop "{ralph_prompt}" --completion-promise "COMPLETE" --max-iterations 25

        Execute autonomously until all tasks pass or blocked.
        Create SUMMARY.md when done.""",
        reset_context=True,
        tools=["ralph-wiggum", "bash", "file_operations"]
    )

    # Phase 4: Update State
    await agent.call_subordinate(
        message=f"""Update .planning/STATE.md with execution results.

        Results: {execution_result}

        Add entry to execution log with:
        - Timestamp
        - Tasks completed
        - Any blockers encountered
        - Next recommended action""",
        reset_context=True
    )

    return execution_result


def extract_path(result: str) -> str:
    """Extract file path from subordinate response"""
    import re
    match = re.search(r'\.planning/[^\s]+\.md', result)
    return match.group(0) if match else ".planning/phases/current/PLAN.md"
```

## Usage Examples

### Example 1: Build a Feature

```
You: Build a REST API for user authentication with JWT tokens

Agent Zero:
├── Asks clarifying questions (password reset? OAuth?)
├── Creates PROJECT.md with requirements
├── Creates ROADMAP.md: Phase 1 (Auth), Phase 2 (Tests), Phase 3 (Docs)
├── Creates PLAN.md for Phase 1 with 3 tasks
├── Spawns Ralph loop
│   ├── Iteration 1: Creates user model, fails test
│   ├── Iteration 2: Fixes model, creates routes, fails test
│   ├── Iteration 3: Fixes routes, all tests pass
│   └── Outputs COMPLETE
├── Creates SUMMARY.md
└── Updates STATE.md

You: [Reviews and approves]

Agent Zero: Proceeding to Phase 2...
```

### Example 2: Fix a Bug

```
You: Fix the login timeout issue in production

Agent Zero:
├── Reads existing PROJECT.md context
├── Creates quick PLAN.md (single task)
├── Ralph executes:
│   ├── Analyzes logs
│   ├── Identifies session config issue
│   ├── Implements fix
│   ├── Runs test suite
│   └── COMPLETE
└── Commits: fix(auth): increase session timeout to 24h
```

### Example 3: Large Refactor

```
You: Migrate from REST to GraphQL

Agent Zero:
├── Creates 5-phase ROADMAP
├── For each phase:
│   ├── Plans 2-3 tasks
│   ├── Ralph executes autonomously
│   ├── Commits per-task
│   └── Updates STATE.md
└── Total: 47 commits over 12 Ralph loops
```

## Configuration

### Agent Zero Settings

```python
# config.py
GSD_CONFIG = {
    "planning_dir": ".planning",
    "max_tasks_per_plan": 3,
    "ralph_max_iterations": 30,
    "ralph_escape_threshold": 5,
    "auto_commit": True,
    "commit_format": "{type}({phase}): {task}"
}
```

### Ralph Wiggum Defaults

```bash
# Set in your environment or Agent Zero config
RALPH_MAX_ITERATIONS=30
RALPH_COMPLETION_PROMISE="COMPLETE"
RALPH_BLOCKED_PROMISE="BLOCKED"
```

## Workflow Modes

### Mode 1: Full Autonomous (Walk Away)

```python
# Agent Zero handles everything
await autonomous_build(agent, "Build a todo app with React and FastAPI")
# Come back in an hour, project is built
```

### Mode 2: Planning Review (Checkpoints)

```python
# Plan, review, then execute
plan = await create_plan(agent, request)
print(plan)  # Human reviews
if approved:
    await ralph_execute(agent, plan)
```

### Mode 3: Incremental (Phase by Phase)

```python
# Execute one phase at a time
for phase in roadmap.phases:
    plan = await plan_phase(agent, phase)
    result = await ralph_execute(agent, plan)
    human_review(result)  # Checkpoint between phases
```

## Troubleshooting

| Issue | Cause | Fix |
|-------|-------|-----|
| Ralph loops forever | No clear `<verify>` | Add executable test commands |
| Context lost between iterations | Not using git | Ensure per-task commits |
| Agent Zero loses project state | Missing STATE.md | Check GSD tool creates it |
| High token costs | Too many iterations | Lower `--max-iterations`, better prompts |
| Tasks too large | Plan scope creep | Enforce 2-3 tasks per plan |

## Cost Estimates

| Workflow | Estimated Cost (Claude) |
|----------|------------------------|
| Simple feature (5 iterations) | $5-15 |
| Medium feature (15 iterations) | $15-40 |
| Large refactor (30+ iterations) | $50-150 |
| Full project (multiple phases) | $100-500+ |

Set `--max-iterations` based on your budget.

## Next Steps

1. **Install the stack** - Follow setup checklist above
2. **Test with small task** - "Add a health check endpoint"
3. **Tune iteration limits** - Find your sweet spot
4. **Add custom prompts** - Optimize for your codebase
5. **Scale up** - Tackle larger features autonomously

---

*This stack transforms development from "write code" to "describe intent and review results."*
