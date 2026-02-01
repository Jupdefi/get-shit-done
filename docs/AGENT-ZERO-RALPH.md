# Agent Zero Ralph Loop

A pure Agent Zero implementation of the Ralph Wiggum autonomous loop pattern. No Claude Code required - works with any LLM backend Agent Zero supports.

## Why This Approach

| Claude as Subordinate | Pure Agent Zero |
|-----------------------|-----------------|
| Pays Claude API fees per subordinate | Uses your existing Agent Zero LLM |
| Limited to Claude models | Works with any model (local, API, etc.) |
| Complex orchestration | Simple loop pattern |
| Context resets each call | Persistent agent memory |

## Core Implementation

### ralph_loop.py

```python
"""
Ralph Loop for Agent Zero
Autonomous iteration until task completion

No Claude Code dependency - pure Agent Zero implementation
"""

import asyncio
import time
from pathlib import Path
from dataclasses import dataclass
from typing import Callable, Optional
from enum import Enum

class LoopStatus(Enum):
    RUNNING = "running"
    COMPLETE = "complete"
    BLOCKED = "blocked"
    MAX_ITERATIONS = "max_iterations"
    CANCELLED = "cancelled"

@dataclass
class RalphConfig:
    max_iterations: int = 30
    escape_threshold: int = 5  # Same error N times = blocked
    cooldown_seconds: float = 1.0  # Pause between iterations
    completion_markers: list = None  # Strings that signal completion
    blocked_markers: list = None  # Strings that signal blocked
    cost_limit: float = None  # Optional cost cap

    def __post_init__(self):
        self.completion_markers = self.completion_markers or ["COMPLETE", "DONE", "FINISHED"]
        self.blocked_markers = self.blocked_markers or ["BLOCKED", "STUCK", "FAILED"]


class RalphLoop:
    """
    Self-referential iteration loop for Agent Zero

    The agent executes a task, sees its previous work (files, git history),
    and iterates until completion criteria met.
    """

    def __init__(self, agent, config: RalphConfig = None):
        self.agent = agent
        self.config = config or RalphConfig()
        self.iteration = 0
        self.status = LoopStatus.RUNNING
        self.history = []
        self.error_counts = {}  # Track repeated errors
        self._cancelled = False

    async def execute(self, prompt: str, context: str = "") -> dict:
        """
        Execute Ralph loop until completion or limit reached

        Args:
            prompt: The task to complete (should include completion criteria)
            context: Optional additional context (project files, etc.)

        Returns:
            Dict with status, iterations, and final result
        """

        full_prompt = self._build_prompt(prompt, context)

        while self.iteration < self.config.max_iterations:
            if self._cancelled:
                self.status = LoopStatus.CANCELLED
                break

            self.iteration += 1

            # Execute iteration
            result = await self._execute_iteration(full_prompt)
            self.history.append({
                "iteration": self.iteration,
                "result": result,
                "timestamp": time.time()
            })

            # Check completion
            if self._check_completion(result):
                self.status = LoopStatus.COMPLETE
                break

            # Check blocked
            if self._check_blocked(result):
                self.status = LoopStatus.BLOCKED
                break

            # Check escape threshold (same error repeated)
            if self._check_escape_threshold(result):
                self.status = LoopStatus.BLOCKED
                break

            # Cooldown before next iteration
            await asyncio.sleep(self.config.cooldown_seconds)

        if self.iteration >= self.config.max_iterations:
            self.status = LoopStatus.MAX_ITERATIONS

        return {
            "status": self.status.value,
            "iterations": self.iteration,
            "history": self.history,
            "final_result": self.history[-1]["result"] if self.history else None
        }

    def cancel(self):
        """Cancel the running loop"""
        self._cancelled = True

    def _build_prompt(self, task: str, context: str) -> str:
        """Build the iteration prompt with Ralph instructions"""
        return f"""# Ralph Loop Execution

## Task
{task}

## Context
{context if context else "Check files and git history for prior work."}

## Instructions
1. Check what work has already been done (read files, check git log)
2. Continue from where you left off
3. Execute the next step of the task
4. Run any verification commands
5. If task complete, output: COMPLETE
6. If blocked and cannot proceed, output: BLOCKED with reason
7. Otherwise, describe what you accomplished this iteration

## Completion Criteria
Output one of these markers when appropriate:
- COMPLETE - All task requirements satisfied
- BLOCKED - Cannot proceed, need human intervention

## Current Iteration: {self.iteration + 1}
"""

    async def _execute_iteration(self, prompt: str) -> str:
        """Execute a single iteration using Agent Zero"""

        # Use Agent Zero's message handling
        # This uses whatever LLM is configured (local, API, etc.)
        response = await self.agent.message(prompt)

        # If agent has tool results, include them
        if hasattr(response, 'tool_results'):
            return f"{response.text}\n\nTool Results:\n{response.tool_results}"

        return str(response)

    def _check_completion(self, result: str) -> bool:
        """Check if result contains completion marker"""
        result_upper = result.upper()
        return any(marker in result_upper for marker in self.config.completion_markers)

    def _check_blocked(self, result: str) -> bool:
        """Check if result contains blocked marker"""
        result_upper = result.upper()
        return any(marker in result_upper for marker in self.config.blocked_markers)

    def _check_escape_threshold(self, result: str) -> bool:
        """Check if same error repeated too many times"""
        # Hash the error signature
        error_sig = self._extract_error_signature(result)
        if error_sig:
            self.error_counts[error_sig] = self.error_counts.get(error_sig, 0) + 1
            if self.error_counts[error_sig] >= self.config.escape_threshold:
                return True
        return False

    def _extract_error_signature(self, result: str) -> Optional[str]:
        """Extract error pattern from result for dedup"""
        error_keywords = ["error", "failed", "exception", "traceback"]
        result_lower = result.lower()

        for keyword in error_keywords:
            if keyword in result_lower:
                # Return first 100 chars after error keyword as signature
                idx = result_lower.index(keyword)
                return result[idx:idx+100]

        return None


# Convenience function
async def ralph_loop(agent, task: str, **kwargs) -> dict:
    """
    Simple interface for Ralph loop execution

    Usage:
        result = await ralph_loop(agent, "Build a REST API with tests")
    """
    config = RalphConfig(**kwargs)
    loop = RalphLoop(agent, config)
    return await loop.execute(task)
```

### Integration with GSD

```python
# gsd_ralph.py - GSD + Ralph integration for Agent Zero

from pathlib import Path
from ralph_loop import RalphLoop, RalphConfig

class GSDRalph:
    """
    GSD planning + Ralph execution
    Pure Agent Zero - no Claude Code
    """

    def __init__(self, agent, project_path: str = "."):
        self.agent = agent
        self.project_path = Path(project_path)
        self.planning_dir = self.project_path / ".planning"

    async def execute_plan(self, plan_path: str, max_iterations: int = 30) -> dict:
        """
        Execute a GSD PLAN.md using Ralph loop
        """

        plan_content = Path(plan_path).read_text()

        # Build Ralph-compatible prompt from GSD plan
        prompt = self._plan_to_ralph_prompt(plan_content, plan_path)

        # Configure Ralph
        config = RalphConfig(
            max_iterations=max_iterations,
            completion_markers=["GSD_COMPLETE", "PHASE_COMPLETE", "ALL_TASKS_DONE"],
            blocked_markers=["GSD_BLOCKED", "CANNOT_PROCEED"],
            escape_threshold=5
        )

        # Execute
        ralph = RalphLoop(self.agent, config)
        result = await ralph.execute(prompt)

        # Create summary if complete
        if result["status"] == "complete":
            await self._create_summary(plan_path, result)

        return result

    def _plan_to_ralph_prompt(self, plan: str, plan_path: str) -> str:
        """Convert GSD PLAN.md to Ralph loop prompt"""

        return f"""# GSD Plan Execution

## Plan Location
{plan_path}

## Plan Content
{plan}

## Execution Instructions

For each <task> in the plan:

1. **Read** the task specification
2. **Check** if already completed (look for prior commits, existing files)
3. **Implement** the <action> exactly as specified
4. **Verify** by running the <verify> command
5. **Commit** with message: feat(gsd): <task-name>
6. **Continue** to next task

## Completion Criteria

Output `GSD_COMPLETE` when:
- All <task> elements have been completed
- All <verify> commands pass
- All changes committed to git

Output `GSD_BLOCKED` when:
- Cannot proceed after multiple attempts
- Need human decision or action
- External dependency unavailable

## Progress Tracking

After each task, update the plan file:
- Mark completed tasks with ✓
- Note any deviations in comments

## Current State

Check git log and files to see what's already done.
Continue from the first incomplete task.
"""

    async def _create_summary(self, plan_path: str, result: dict) -> str:
        """Create SUMMARY.md after successful execution"""

        summary_path = Path(plan_path).parent / "SUMMARY.md"

        summary = f"""# Execution Summary

## Status
{result['status'].upper()}

## Iterations
{result['iterations']}

## Execution Log
"""
        for entry in result.get("history", [])[-5:]:  # Last 5 iterations
            summary += f"\n### Iteration {entry['iteration']}\n"
            summary += f"{entry['result'][:500]}...\n"  # Truncate

        summary_path.write_text(summary)
        return str(summary_path)

    async def execute_roadmap(self, phases: list[str] = None) -> dict:
        """
        Execute entire roadmap phase by phase
        """

        roadmap_path = self.planning_dir / "ROADMAP.md"
        if not roadmap_path.exists():
            return {"status": "error", "message": "No ROADMAP.md found"}

        # Find all phase directories
        phases_dir = self.planning_dir / "phases"
        if phases:
            phase_dirs = [phases_dir / p for p in phases]
        else:
            phase_dirs = sorted(phases_dir.iterdir()) if phases_dir.exists() else []

        results = []
        for phase_dir in phase_dirs:
            if not phase_dir.is_dir():
                continue

            # Find plans in phase
            plans = sorted(phase_dir.glob("*PLAN.md"))

            for plan in plans:
                result = await self.execute_plan(str(plan))
                results.append({
                    "phase": phase_dir.name,
                    "plan": plan.name,
                    "result": result
                })

                # Stop if blocked
                if result["status"] == "blocked":
                    return {
                        "status": "blocked",
                        "completed": results[:-1],
                        "blocked_at": results[-1]
                    }

        return {
            "status": "complete",
            "phases": results
        }
```

### Simple Usage

```python
# example_usage.py

from agent_zero import Agent  # Your Agent Zero setup
from gsd_ralph import GSDRalph
from ralph_loop import ralph_loop

async def main():
    # Initialize your Agent Zero (with whatever LLM you want)
    agent = Agent(
        model="ollama/llama3",  # Or any model
        # model="anthropic/claude-3-haiku",  # Cheap Claude option
        # model="openai/gpt-4o-mini",  # Cheap OpenAI option
    )

    # Option 1: Direct Ralph loop
    result = await ralph_loop(
        agent,
        task="Create a Python CLI tool that converts CSV to JSON. Include tests.",
        max_iterations=20
    )
    print(f"Status: {result['status']}, Iterations: {result['iterations']}")

    # Option 2: GSD + Ralph
    gsd = GSDRalph(agent, project_path="./my-project")
    result = await gsd.execute_plan(".planning/phases/01-setup/PLAN.md")
    print(f"Plan complete: {result['status']}")

    # Option 3: Full roadmap execution
    result = await gsd.execute_roadmap()
    print(f"Roadmap complete: {result['status']}")


if __name__ == "__main__":
    import asyncio
    asyncio.run(main())
```

## Configuration Options

```python
config = RalphConfig(
    # Iteration limits
    max_iterations=30,          # Stop after N iterations
    escape_threshold=5,         # Same error N times = blocked

    # Timing
    cooldown_seconds=1.0,       # Pause between iterations

    # Completion detection
    completion_markers=[        # Any of these = success
        "COMPLETE",
        "DONE",
        "ALL_TASKS_FINISHED"
    ],
    blocked_markers=[           # Any of these = stop
        "BLOCKED",
        "STUCK",
        "NEED_HUMAN"
    ],

    # Optional cost tracking (if your LLM wrapper supports it)
    cost_limit=10.0             # Stop if cost exceeds $10
)
```

## Model Recommendations

| Model | Cost | Speed | Quality | Best For |
|-------|------|-------|---------|----------|
| Ollama/Llama3 | Free (local) | Medium | Good | Development, testing |
| Ollama/CodeLlama | Free (local) | Medium | Good for code | Code-heavy tasks |
| Claude Haiku | $0.25/M tokens | Fast | Good | Quick iterations |
| GPT-4o-mini | $0.15/M tokens | Fast | Good | Budget production |
| DeepSeek Coder | Cheap | Fast | Great for code | Code tasks |
| Claude Sonnet | $3/M tokens | Medium | Excellent | Complex tasks |

**Cost comparison for 20-iteration loop:**

| Model | Est. Cost |
|-------|-----------|
| Local (Ollama) | $0 |
| GPT-4o-mini | $0.50-2 |
| Claude Haiku | $1-3 |
| Claude Sonnet | $10-30 |
| GPT-4o | $20-50 |

## Integration with Agent Zero Tools

The Ralph loop uses Agent Zero's existing tool system:

```python
# The agent can use any tools during iterations
agent = Agent(
    model="ollama/llama3",
    tools=[
        "code_execution",    # Run bash/python
        "file_operations",   # Read/write files
        "browser",           # Web access if needed
        "memory_save",       # Persist learnings
    ]
)
```

Each iteration, the agent:
1. Sees the task prompt
2. Checks previous work (files, git)
3. Uses tools to make progress
4. Reports status
5. Loop continues or exits

## Differences from Claude Code Ralph

| Claude Code Ralph | Agent Zero Ralph |
|-------------------|------------------|
| Stop hook intercepts exit | Loop controls iteration |
| Claude-only | Any LLM |
| Plugin system | Python module |
| Session-based | Agent-based |
| `--completion-promise` flag | `completion_markers` config |

## Debugging

```python
# Enable verbose logging
import logging
logging.basicConfig(level=logging.DEBUG)

# Access iteration history
result = await ralph_loop(agent, task)
for entry in result["history"]:
    print(f"Iteration {entry['iteration']}:")
    print(entry["result"][:200])
    print("---")
```

## Best Practices

1. **Clear completion criteria** - Be explicit about what "done" means
2. **Executable verification** - Include commands the agent can run to verify
3. **Git commits** - Each iteration should commit, giving context to the next
4. **Escape hatches** - Always set `max_iterations` to prevent runaway loops
5. **Start small** - Test with 5-10 iterations before going to 30+
