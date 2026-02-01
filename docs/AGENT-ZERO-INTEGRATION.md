# Integrating Get Shit Done with Agent Zero

This guide explains how to give your Agent Zero setup "Get Shit Done" (GSD) capabilities for structured, spec-driven development.

## Overview

| System | Strength | Integration Value |
|--------|----------|-------------------|
| **GSD** | Context engineering, spec-driven planning, atomic execution plans | Provides structured development methodology |
| **Agent Zero** | Multi-agent orchestration, dynamic tool creation, persistent memory | Provides execution runtime and agent hierarchy |

**Goal:** Combine GSD's planning discipline with Agent Zero's execution capabilities.

---

## Integration Approaches

### Approach 1: GSD as an MCP Server (Recommended)

Agent Zero supports MCP (Model Context Protocol) as both client and server. Create an MCP server that exposes GSD's core capabilities as tools.

**Architecture:**
```
Agent Zero (MCP Client)
    ↓
GSD MCP Server
    ├── gsd_new_project()     → Creates PROJECT.md via guided discovery
    ├── gsd_create_roadmap()  → Generates phased ROADMAP.md
    ├── gsd_plan_phase()      → Creates executable PLAN.md
    ├── gsd_get_context()     → Returns current project state
    └── gsd_parse_plan()      → Extracts tasks from PLAN.md
```

**Implementation:**

```python
# python/tools/gsd_tool.py
from python.helpers.tool import Tool, Response
from pathlib import Path
import json

class GsdTool(Tool):
    """
    Agent Zero tool for Get Shit Done methodology.
    Provides structured project planning and execution.
    """

    async def execute(self, **kwargs):
        method = self.args.get("method", "get_context")
        project_path = self.args.get("project_path", ".")

        if method == "get_context":
            return await self._get_project_context(project_path)
        elif method == "parse_plan":
            plan_path = self.args.get("plan_path")
            return await self._parse_plan(plan_path)
        elif method == "create_task_prompt":
            return await self._create_task_prompt(self.args)
        elif method == "update_state":
            return await self._update_state(project_path, self.args)
        else:
            return Response(message=f"Unknown method: {method}", break_loop=False)

    async def _get_project_context(self, project_path: str) -> Response:
        """Load GSD project context (PROJECT.md, ROADMAP.md, STATE.md)"""
        planning_dir = Path(project_path) / ".planning"
        context = {}

        for doc in ["PROJECT.md", "ROADMAP.md", "STATE.md"]:
            doc_path = planning_dir / doc
            if doc_path.exists():
                context[doc] = doc_path.read_text()

        # Load codebase maps if brownfield
        codebase_dir = planning_dir / "codebase"
        if codebase_dir.exists():
            context["codebase"] = {}
            for map_file in codebase_dir.glob("*.md"):
                context["codebase"][map_file.stem] = map_file.read_text()

        return Response(
            message=json.dumps(context, indent=2),
            break_loop=False
        )

    async def _parse_plan(self, plan_path: str) -> Response:
        """Parse a PLAN.md into structured tasks for Agent Zero execution"""
        import re

        plan_content = Path(plan_path).read_text()
        tasks = []

        # Extract tasks using XML-style parsing
        task_pattern = r'<task type="([^"]+)"[^>]*>(.*?)</task>'
        matches = re.findall(task_pattern, plan_content, re.DOTALL)

        for task_type, task_content in matches:
            task = {"type": task_type}

            # Extract task fields
            for field in ["name", "files", "action", "verify", "done"]:
                field_match = re.search(f'<{field}>(.*?)</{field}>', task_content, re.DOTALL)
                if field_match:
                    task[field] = field_match.group(1).strip()

            # Handle checkpoint-specific fields
            for field in ["what-built", "how-to-verify", "resume-signal", "decision", "options"]:
                field_match = re.search(f'<{field}>(.*?)</{field}>', task_content, re.DOTALL)
                if field_match:
                    task[field.replace("-", "_")] = field_match.group(1).strip()

            tasks.append(task)

        return Response(
            message=json.dumps({"tasks": tasks, "total": len(tasks)}, indent=2),
            break_loop=False
        )

    async def _create_task_prompt(self, args: dict) -> Response:
        """Create a focused prompt for executing a single GSD task"""
        task = args.get("task", {})
        context = args.get("context", "")

        prompt = f"""# Task Execution

## Objective
{task.get('name', 'Execute task')}

## Files to Modify
{task.get('files', 'As needed')}

## Action
{task.get('action', 'Complete the task as specified')}

## Verification
{task.get('verify', 'Ensure task completes successfully')}

## Done Criteria
{task.get('done', 'Task is complete')}

## Project Context
{context[:2000] if context else 'No additional context'}

## Instructions
1. Execute the action as specified
2. Modify only the listed files unless absolutely necessary
3. Run verification command when complete
4. Report success or blockers encountered
"""
        return Response(message=prompt, break_loop=False)

    async def _update_state(self, project_path: str, args: dict) -> Response:
        """Update STATE.md with execution results"""
        state_path = Path(project_path) / ".planning" / "STATE.md"
        update = args.get("update", "")

        if state_path.exists():
            current = state_path.read_text()
            # Append to execution log section
            if "## Execution Log" in current:
                current = current.replace(
                    "## Execution Log",
                    f"## Execution Log\n\n{update}"
                )
            else:
                current += f"\n\n## Execution Log\n\n{update}"
            state_path.write_text(current)

        return Response(message="State updated", break_loop=False)
```

---

### Approach 2: GSD Instrument

Agent Zero's Instruments system allows custom procedures. Create a GSD Instrument that Agent Zero can call.

**File:** `instruments/gsd_planner.py`

```python
"""
GSD Planning Instrument for Agent Zero

Provides structured project planning following Get Shit Done methodology.
Agent Zero can invoke this to create executable development plans.
"""

from pathlib import Path
from datetime import datetime

class GSDPlanner:
    """Implements GSD planning methodology for Agent Zero"""

    def __init__(self, project_path: str = "."):
        self.project_path = Path(project_path)
        self.planning_dir = self.project_path / ".planning"

    def initialize_project(self, name: str, description: str, requirements: list[str]) -> dict:
        """
        Initialize a GSD project structure.

        Creates:
        - .planning/PROJECT.md
        - .planning/config.json
        """
        self.planning_dir.mkdir(exist_ok=True)

        project_md = f"""# {name}

## What This Is

{description}

## Core Value

[To be defined - what's the ONE thing that must work?]

## Requirements

### Active

{chr(10).join(f'- [ ] {req}' for req in requirements)}

### Out of Scope

(None defined yet)

## Constraints

- **Stack**: [To be defined]
- **Timeline**: [To be defined]

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|

---
*Created: {datetime.now().strftime('%Y-%m-%d')} via Agent Zero + GSD*
"""

        (self.planning_dir / "PROJECT.md").write_text(project_md)
        (self.planning_dir / "config.json").write_text('{"mode": "interactive"}')

        return {
            "status": "initialized",
            "path": str(self.planning_dir),
            "files_created": ["PROJECT.md", "config.json"]
        }

    def create_plan(self, phase_name: str, objective: str, tasks: list[dict]) -> str:
        """
        Create an executable PLAN.md for a development phase.

        Args:
            phase_name: Name of the phase (e.g., "01-auth-setup")
            objective: What this phase accomplishes
            tasks: List of task dicts with keys: name, files, action, verify, done

        Returns:
            Path to created PLAN.md
        """
        phases_dir = self.planning_dir / "phases" / phase_name
        phases_dir.mkdir(parents=True, exist_ok=True)

        task_xml = []
        for i, task in enumerate(tasks, 1):
            task_xml.append(f'''<task type="auto">
  <name>Task {i}: {task.get('name', 'Unnamed task')}</name>
  <files>{task.get('files', 'TBD')}</files>
  <action>{task.get('action', 'Complete the task')}</action>
  <verify>{task.get('verify', 'Manual verification')}</verify>
  <done>{task.get('done', 'Task complete')}</done>
</task>''')

        plan_md = f"""---
phase: {phase_name}
type: execute
created: {datetime.now().isoformat()}
agent: agent-zero
---

<objective>
{objective}
</objective>

<context>
@.planning/PROJECT.md
@.planning/ROADMAP.md
</context>

<tasks>
{chr(10).join(task_xml)}
</tasks>

<verification>
- [ ] All tasks completed successfully
- [ ] Code compiles/runs without errors
- [ ] Changes committed to git
</verification>

<success_criteria>
- All tasks marked complete
- Verification checks pass
</success_criteria>
"""

        plan_path = phases_dir / "PLAN.md"
        plan_path.write_text(plan_md)

        return str(plan_path)

    def get_next_task(self, plan_path: str) -> dict | None:
        """
        Parse a PLAN.md and return the next uncompleted task.

        Returns task dict or None if all complete.
        """
        import re

        plan = Path(plan_path).read_text()
        tasks = []

        for match in re.finditer(r'<task type="([^"]+)"[^>]*>(.*?)</task>', plan, re.DOTALL):
            task_type, content = match.groups()
            task = {"type": task_type, "raw": content}

            for field in ["name", "files", "action", "verify", "done"]:
                field_match = re.search(f'<{field}>(.*?)</{field}>', content, re.DOTALL)
                if field_match:
                    task[field] = field_match.group(1).strip()

            # Check if marked complete (look for ✓ or [x])
            if "✓" not in task.get("name", "") and "[x]" not in content.lower():
                tasks.append(task)

        return tasks[0] if tasks else None

    def create_summary(self, phase_name: str, accomplishments: list[str],
                       files_changed: list[str], decisions: list[str] = None) -> str:
        """Create SUMMARY.md after phase completion"""
        phases_dir = self.planning_dir / "phases" / phase_name

        summary = f"""---
phase: {phase_name}
completed: {datetime.now().isoformat()}
agent: agent-zero
---

# Phase Summary: {phase_name}

## Accomplishments

{chr(10).join(f'- {a}' for a in accomplishments)}

## Files Created/Modified

{chr(10).join(f'- `{f}`' for f in files_changed)}

## Decisions Made

{chr(10).join(f'- {d}' for d in (decisions or ['None']))}

## Issues Encountered

(None logged)

## Next Phase Readiness

Ready to proceed.
"""

        summary_path = phases_dir / "SUMMARY.md"
        summary_path.write_text(summary)
        return str(summary_path)
```

---

### Approach 3: GSD Prompts in Agent Zero System

Integrate GSD's methodology directly into Agent Zero's system prompts.

**File:** `prompts/gsd/agent.system.md`

```markdown
# Agent Zero + Get Shit Done

You are an AI agent enhanced with the Get Shit Done (GSD) methodology for structured development.

## GSD Principles

1. **Plans ARE Prompts** - Never execute vague instructions. Create explicit PLAN.md files first.
2. **Atomic Tasks** - Each task should be 15-60 minutes of work with clear done criteria.
3. **Context Engineering** - Maintain PROJECT.md, ROADMAP.md, STATE.md for persistent memory.
4. **Per-Task Commits** - Commit after each completed task with format: `{type}({phase}): {task-name}`

## Development Workflow

When asked to build something:

1. **Initialize** (if no .planning/ exists):
   - Ask clarifying questions about vision, requirements, constraints
   - Create `.planning/PROJECT.md` with gathered context
   - Create `.planning/ROADMAP.md` with phased approach

2. **Plan** (before any implementation):
   - Create a PLAN.md with 2-3 atomic tasks
   - Each task must have: name, files, action, verify, done
   - Never proceed without an executable plan

3. **Execute**:
   - Work through tasks sequentially
   - Commit after each task
   - Log deviations in STATE.md
   - Create SUMMARY.md when plan complete

4. **Iterate**:
   - Review with user at checkpoints
   - Update roadmap based on learnings
   - Move to next phase

## Task Format

Every task you create must follow this structure:

```xml
<task type="auto">
  <name>Task N: [Descriptive Name]</name>
  <files>[Exact file paths]</files>
  <action>[Specific implementation - what to do and what to avoid]</action>
  <verify>[Executable verification command or check]</verify>
  <done>[Measurable completion criteria]</done>
</task>
```

## Memory Integration

Store GSD artifacts in your persistent memory:
- Key: `gsd:project:{project_name}` → PROJECT.md content
- Key: `gsd:roadmap:{project_name}` → ROADMAP.md content
- Key: `gsd:state:{project_name}` → Current execution state

## Deviation Handling

During execution:
- **Auto-fix bugs** - Fix immediately, log in SUMMARY
- **Auto-add critical** - Security gaps must be addressed
- **Ask about architectural** - Major changes require approval
- **Log enhancements** - Nice-to-haves go to ISSUES.md

## Checkpoints

Pause and wait for human input when:
- Visual/UX verification needed (can't be automated)
- Architectural decision required (affects downstream work)
- Truly unavoidable manual action (no CLI/API available)

For everything else, automate it.
```

---

### Approach 4: Hybrid Multi-Agent Architecture

Use Agent Zero's hierarchical agent structure with GSD-specialized agents.

```
Human User
    ↓
Agent Zero (Orchestrator)
    ├── GSD Planner Agent (subordinate)
    │   └── Creates PROJECT.md, ROADMAP.md, PLAN.md
    ├── GSD Executor Agent (subordinate)
    │   └── Executes tasks from PLAN.md, commits code
    ├── GSD Reviewer Agent (subordinate)
    │   └── Runs verification, creates SUMMARY.md
    └── Codebase Agent (subordinate)
        └── Maintains codebase maps, answers architecture questions
```

**Implementation in Agent Zero:**

```python
# In your main agent interaction, spawn specialized subordinates

async def gsd_workflow(agent, user_request):
    """Execute full GSD workflow with specialized agents"""

    # 1. Planning Phase - spawn planner subordinate
    planner_result = await agent.call_subordinate(
        message=f"""You are a GSD Planner agent.

User request: {user_request}

Your job:
1. If no .planning/PROJECT.md exists, ask questions to understand:
   - What are we building?
   - Who is it for?
   - What are the requirements?
   - What are the constraints?

2. Create .planning/PROJECT.md with the gathered context

3. Create .planning/ROADMAP.md breaking work into phases

4. Create a PLAN.md for the first phase with 2-3 atomic tasks

Return the path to the PLAN.md when ready for execution.""",
        reset_context=True
    )

    plan_path = extract_plan_path(planner_result)

    # 2. Execution Phase - spawn executor subordinate
    executor_result = await agent.call_subordinate(
        message=f"""You are a GSD Executor agent.

Execute the plan at: {plan_path}

For each task:
1. Read the task specification
2. Implement exactly as specified
3. Run the verification command
4. Commit with message: feat(phase): task-name
5. Move to next task

Report completion status for each task.""",
        reset_context=True
    )

    # 3. Review Phase - spawn reviewer subordinate
    reviewer_result = await agent.call_subordinate(
        message=f"""You are a GSD Reviewer agent.

Review the completed work from: {plan_path}

1. Verify all tasks completed successfully
2. Run overall verification checks
3. Create SUMMARY.md with:
   - Accomplishments
   - Files changed
   - Decisions made
   - Issues encountered
4. Update STATE.md with current position

Return the summary.""",
        reset_context=True
    )

    return reviewer_result
```

---

## What Needs Adapting

### 1. Execution Runtime

| GSD (Claude Code) | Agent Zero Equivalent | Adaptation Needed |
|-------------------|----------------------|-------------------|
| `Task` tool (subagents) | `call_subordinate()` | Map subagent spawning to subordinate calls |
| `Bash` tool | `code_execution_tool` | Already compatible |
| `Read/Write/Edit` | File system access | Already compatible |
| `Glob/Grep` | Search tools | May need custom tool |
| Fresh 200k context | `reset_context=True` | Use on subordinate calls |

### 2. State Persistence

| GSD Artifact | Agent Zero Storage | Adaptation |
|--------------|-------------------|------------|
| PROJECT.md | `memory_save` + files | Dual storage for search + persistence |
| ROADMAP.md | `memory_save` + files | Index phases in memory |
| STATE.md | `memory_save` + files | Real-time state updates |
| SUMMARY.md | Files only | Created per-phase |

### 3. Prompt Format

GSD uses XML-structured prompts. Agent Zero can parse these directly:

```python
# Add to Agent Zero's tool parsing
def parse_gsd_task(task_xml: str) -> dict:
    """Parse GSD task XML into Agent Zero action format"""
    import re

    task = {}
    for field in ["name", "files", "action", "verify", "done"]:
        match = re.search(f'<{field}>(.*?)</{field}>', task_xml, re.DOTALL)
        if match:
            task[field] = match.group(1).strip()

    return {
        "tool": "code_execution_tool",
        "args": {
            "runtime": "terminal",
            "instructions": task.get("action", ""),
            "files": task.get("files", "").split(", "),
            "verify_command": task.get("verify", ""),
            "done_criteria": task.get("done", "")
        }
    }
```

### 4. Checkpoint Handling

GSD checkpoints map to Agent Zero's user interaction:

| GSD Checkpoint | Agent Zero Handler |
|----------------|-------------------|
| `checkpoint:human-verify` | `notify_user` + wait for response |
| `checkpoint:decision` | Present options via `notify_user`, await choice |
| `checkpoint:human-action` | Explain action needed, wait for completion signal |

---

## Quick Start

### Option A: Minimal Integration (System Prompt Only)

1. Copy the GSD system prompt to `prompts/gsd/agent.system.md`
2. Update `initialize.py` to load GSD prompts
3. Agent Zero now follows GSD methodology for development tasks

### Option B: Tool Integration

1. Add `gsd_tool.py` to `python/tools/`
2. Add `gsd_planner.py` to `instruments/`
3. Agent Zero can now call GSD planning functions

### Option C: Full Integration (MCP)

1. Create standalone GSD MCP server
2. Configure Agent Zero as MCP client
3. Full GSD workflow available as external tools

---

## File Structure After Integration

```
agent-zero/
├── python/
│   ├── tools/
│   │   └── gsd_tool.py          # GSD tool for Agent Zero
│   └── instruments/
│       └── gsd_planner.py       # GSD planning instrument
├── prompts/
│   └── gsd/
│       ├── agent.system.md      # GSD-enhanced system prompt
│       ├── planning.md          # Planning guidelines
│       └── execution.md         # Execution guidelines
└── {your-project}/
    └── .planning/               # GSD artifacts (created per project)
        ├── PROJECT.md
        ├── ROADMAP.md
        ├── STATE.md
        └── phases/
            └── 01-phase-name/
                ├── PLAN.md
                └── SUMMARY.md
```

---

## Benefits of Integration

1. **Structured Planning** - No more ad-hoc development; every feature has a plan
2. **Context Persistence** - PROJECT.md and STATE.md survive across sessions
3. **Atomic Execution** - 2-3 task plans prevent context window degradation
4. **Verifiable Progress** - Each task has explicit done criteria
5. **Git History** - Per-task commits enable precise rollback
6. **Brownfield Support** - Codebase mapping works with existing projects
7. **Multi-Agent Synergy** - Agent Zero's hierarchy + GSD's planning = reliable execution

---

## Further Resources

- [Get Shit Done Documentation](../README.md)
- [GSD Plan Format Reference](../get-shit-done/references/plan-format.md)
- [GSD Checkpoint Types](../get-shit-done/references/checkpoints.md)
- [Agent Zero GitHub](https://github.com/agent0ai/agent-zero)
- [MCP Specification](https://modelcontextprotocol.io/)
