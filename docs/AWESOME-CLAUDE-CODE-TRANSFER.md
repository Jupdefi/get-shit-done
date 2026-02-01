# Awesome Claude Code → Agent Zero Transfer Guide

This guide maps resources from [awesome-claude-code](https://github.com/hesreallyhim/awesome-claude-code) to Agent Zero equivalents.

## Transferability Overview

| Category | Transferable? | How |
|----------|--------------|-----|
| Agent Skills | ✅ Yes | Convert to system prompts or instruments |
| Workflows | ✅ Yes | Adapt as Agent Zero workflow patterns |
| Slash Commands | ✅ Yes | Convert to tools or prompt templates |
| Hooks | ⚠️ Partial | Adapt validation logic to middleware |
| Orchestration | ✅ Yes | Map to subordinate patterns |
| Status Lines | ❌ No | Claude Code terminal specific |
| IDE Integrations | ❌ No | Claude Code specific |

---

## Agent Skills → Agent Zero Instruments

Agent Skills are specialized prompts that give Claude domain expertise. These become Agent Zero instruments or system prompts.

### DevOps Skills

**Source:** [cc-devops-skills](https://github.com/akin-ozer/cc-devops-skills)

**Agent Zero Adaptation:**

```python
# instruments/devops_skill.py
"""
DevOps Engineer skill for Agent Zero
Adapted from cc-devops-skills
"""

DEVOPS_SYSTEM_PROMPT = """
You are a Senior DevOps Engineer with expertise in:
- Infrastructure as Code (Terraform, Pulumi, CloudFormation)
- Container orchestration (Kubernetes, Docker Swarm)
- CI/CD pipelines (GitHub Actions, GitLab CI, Jenkins)
- Cloud platforms (AWS, GCP, Azure)
- Monitoring and observability (Prometheus, Grafana, Datadog)

When asked to create infrastructure:
1. Always use IaC - never manual console changes
2. Follow security best practices (least privilege, encryption)
3. Include monitoring and alerting
4. Document with inline comments
5. Create modular, reusable components

Output Terraform/Pulumi code with:
- Variables for environment-specific values
- Outputs for dependent resources
- State management configuration
- README with usage instructions
"""

class DevOpsSkill:
    def __init__(self, agent):
        self.agent = agent

    async def provision_infrastructure(self, request: str) -> str:
        return await self.agent.call_subordinate(
            message=f"{DEVOPS_SYSTEM_PROMPT}\n\nRequest: {request}",
            reset_context=True
        )
```

### Security Auditing Skills

**Source:** [Trail of Bits Security Skills](https://github.com/trailofbits/skills)

**Agent Zero Adaptation:**

```python
# instruments/security_audit.py

SECURITY_AUDIT_PROMPT = """
You are a security auditor specializing in code review.

For every code file reviewed:
1. Check for OWASP Top 10 vulnerabilities
2. Identify injection points (SQL, XSS, command)
3. Review authentication and authorization logic
4. Check for hardcoded secrets or credentials
5. Analyze cryptographic implementations
6. Review error handling for information leakage

Output format:
- CRITICAL: Immediate exploitation risk
- HIGH: Significant vulnerability
- MEDIUM: Security weakness
- LOW: Best practice violation
- INFO: Recommendation

For each finding:
- Location (file:line)
- Vulnerability type
- Proof of concept (if applicable)
- Remediation steps
"""

class SecurityAudit:
    async def audit_codebase(self, path: str) -> dict:
        # Scan files
        findings = await self.agent.call_subordinate(
            message=f"{SECURITY_AUDIT_PROMPT}\n\nAudit: {path}",
            reset_context=True
        )
        return self._parse_findings(findings)
```

---

## Workflows → Agent Zero Patterns

### RIPER Workflow

**Source:** [claude-code-riper-5](https://github.com/tony/claude-code-riper-5)

The RIPER workflow is: **R**esearch → **I**nnovate → **P**lan → **E**xecute → **R**eview

**Agent Zero Implementation:**

```python
# workflows/riper.py

async def riper_workflow(agent, task: str):
    """
    RIPER: Structured development workflow
    Adapted from claude-code-riper-5
    """

    # R - Research
    research = await agent.call_subordinate(
        message=f"""RESEARCH PHASE

        Task: {task}

        Investigate:
        1. Existing solutions and prior art
        2. Technical constraints and requirements
        3. Dependencies and integrations needed
        4. Potential risks and blockers

        Output: Research summary with findings and recommendations.""",
        reset_context=True
    )

    # I - Innovate
    innovation = await agent.call_subordinate(
        message=f"""INNOVATE PHASE

        Task: {task}
        Research: {research}

        Generate:
        1. Multiple solution approaches (at least 3)
        2. Pros/cons for each approach
        3. Recommended approach with justification
        4. Novel techniques or optimizations

        Output: Innovation proposals with recommended direction.""",
        reset_context=True
    )

    # P - Plan
    plan = await agent.call_subordinate(
        message=f"""PLAN PHASE

        Task: {task}
        Selected Approach: {innovation}

        Create:
        1. Detailed implementation plan
        2. Task breakdown (2-3 atomic tasks)
        3. Dependencies and order of execution
        4. Verification steps for each task
        5. Rollback strategy

        Output: GSD-compatible PLAN.md""",
        reset_context=True
    )

    # E - Execute (with Ralph loop)
    execution = await agent.call_subordinate(
        message=f"""EXECUTE PHASE

        Execute the plan using Ralph loop:
        /ralph-loop "{plan}" --completion-promise "COMPLETE" --max-iterations 25

        For each task:
        1. Implement as specified
        2. Run verification
        3. Commit changes
        4. Continue or retry

        Output: COMPLETE when done, BLOCKED if stuck.""",
        reset_context=True,
        tools=["ralph-wiggum"]
    )

    # R - Review
    review = await agent.call_subordinate(
        message=f"""REVIEW PHASE

        Execution Results: {execution}

        Perform:
        1. Code review for quality and security
        2. Test coverage analysis
        3. Documentation completeness check
        4. Performance assessment
        5. Lessons learned

        Output: Review report with recommendations.""",
        reset_context=True
    )

    return {
        "research": research,
        "innovation": innovation,
        "plan": plan,
        "execution": execution,
        "review": review
    }
```

### Claude Code PM (Project Management)

**Source:** [ccpm](https://github.com/automazeio/ccpm)

**Agent Zero Adaptation:**

```python
# workflows/project_manager.py

class ProjectManager:
    """
    Project management workflow for Agent Zero
    Adapted from ccpm (Claude Code PM)
    """

    def __init__(self, agent, project_path: str = "."):
        self.agent = agent
        self.project_path = project_path

    async def initialize_project(self, name: str, description: str):
        """Create project structure with PM artifacts"""
        return await self.agent.call_subordinate(
            message=f"""Initialize project: {name}

            Create:
            1. .planning/PROJECT.md - Vision, goals, stakeholders
            2. .planning/ROADMAP.md - Milestones and phases
            3. .planning/BACKLOG.md - Feature backlog with priorities
            4. .planning/RISKS.md - Risk register
            5. .planning/DECISIONS.md - Decision log (ADRs)

            Project description: {description}

            Use markdown tables for structured data.
            Include status tracking columns.""",
            reset_context=True
        )

    async def sprint_planning(self, sprint_goal: str, capacity: int = 3):
        """Plan a sprint from backlog"""
        return await self.agent.call_subordinate(
            message=f"""Sprint Planning

            Goal: {sprint_goal}
            Capacity: {capacity} tasks

            1. Read .planning/BACKLOG.md
            2. Select top {capacity} items aligned with goal
            3. Create .planning/sprints/current/SPRINT.md
            4. Break each item into PLAN.md tasks
            5. Update BACKLOG.md (move items to "In Sprint")

            Output: Sprint overview with selected items.""",
            reset_context=True
        )

    async def daily_standup(self):
        """Generate daily status report"""
        return await self.agent.call_subordinate(
            message="""Daily Standup Report

            Analyze:
            1. Git commits since last standup
            2. Current sprint progress
            3. Any blockers in STATE.md

            Output:
            - Yesterday: [completed work]
            - Today: [planned work]
            - Blockers: [any impediments]""",
            reset_context=True
        )

    async def retrospective(self, sprint_id: str):
        """Sprint retrospective analysis"""
        return await self.agent.call_subordinate(
            message=f"""Sprint Retrospective: {sprint_id}

            Analyze .planning/sprints/{sprint_id}/:
            1. Planned vs completed tasks
            2. Time spent per task category
            3. Blockers encountered
            4. Process improvements

            Output:
            - What went well
            - What could improve
            - Action items for next sprint""",
            reset_context=True
        )
```

---

## Slash Commands → Agent Zero Tools

Convert Claude Code slash commands to Agent Zero callable tools.

### /commit → commit_tool.py

```python
# tools/commit_tool.py

from python.helpers.tool import Tool, Response

class CommitTool(Tool):
    """
    Conventional commit with analysis
    Adapted from awesome-claude-code /commit
    """

    async def execute(self, **kwargs):
        message = self.args.get("message", "")
        scope = self.args.get("scope", "")

        # Analyze staged changes
        diff_result = await self.agent.call_tool("code_execution", {
            "command": "git diff --cached --stat"
        })

        # Determine commit type from changes
        commit_type = self._determine_type(diff_result)

        # Format conventional commit
        if scope:
            commit_msg = f"{commit_type}({scope}): {message}"
        else:
            commit_msg = f"{commit_type}: {message}"

        # Execute commit
        result = await self.agent.call_tool("code_execution", {
            "command": f'git commit -m "{commit_msg}"'
        })

        return Response(message=result, break_loop=False)

    def _determine_type(self, diff: str) -> str:
        if "test" in diff.lower():
            return "test"
        elif "fix" in diff.lower() or "bug" in diff.lower():
            return "fix"
        elif "doc" in diff.lower() or "readme" in diff.lower():
            return "docs"
        else:
            return "feat"
```

### /tdd → tdd_tool.py

```python
# tools/tdd_tool.py

class TDDTool(Tool):
    """
    Test-Driven Development enforcement
    Adapted from awesome-claude-code /tdd
    """

    async def execute(self, **kwargs):
        feature = self.args.get("feature", "")
        test_framework = self.args.get("framework", "pytest")

        # RED: Write failing test first
        red_result = await self.agent.call_subordinate(
            message=f"""TDD RED PHASE

            Feature: {feature}
            Framework: {test_framework}

            1. Write a failing test that defines the expected behavior
            2. Run the test - it MUST fail
            3. Commit: test(tdd): red - {feature}

            Output the test code and failure message.""",
            reset_context=True
        )

        # GREEN: Minimal implementation
        green_result = await self.agent.call_subordinate(
            message=f"""TDD GREEN PHASE

            Feature: {feature}
            Test: {red_result}

            1. Write MINIMAL code to make the test pass
            2. No extra features or optimizations
            3. Run test - it MUST pass now
            4. Commit: feat(tdd): green - {feature}

            Output the implementation.""",
            reset_context=True
        )

        # REFACTOR: Clean up
        refactor_result = await self.agent.call_subordinate(
            message=f"""TDD REFACTOR PHASE

            Feature: {feature}
            Implementation: {green_result}

            1. Improve code quality without changing behavior
            2. Remove duplication
            3. Improve naming and structure
            4. Run tests - they MUST still pass
            5. Commit: refactor(tdd): {feature}

            Output the refactored code.""",
            reset_context=True
        )

        return Response(
            message=f"TDD Complete:\n{refactor_result}",
            break_loop=False
        )
```

### /create-pr → pr_tool.py

```python
# tools/pr_tool.py

class PRTool(Tool):
    """
    Pull Request creation with analysis
    Adapted from awesome-claude-code /create-pr
    """

    async def execute(self, **kwargs):
        base = self.args.get("base", "main")
        title = self.args.get("title", "")

        # Analyze changes
        analysis = await self.agent.call_subordinate(
            message=f"""Analyze branch for PR to {base}

            1. List all commits since branching
            2. Summarize changes by category
            3. Identify breaking changes
            4. List files modified
            5. Check test coverage

            Output: PR description with summary, changes, testing notes.""",
            reset_context=True
        )

        # Create PR
        pr_body = f"""## Summary
{analysis}

## Checklist
- [ ] Tests pass
- [ ] Documentation updated
- [ ] No breaking changes (or documented)

---
*Generated by Agent Zero + GSD*
"""

        result = await self.agent.call_tool("code_execution", {
            "command": f'gh pr create --base {base} --title "{title}" --body "{pr_body}"'
        })

        return Response(message=result, break_loop=False)
```

---

## Orchestration Tools → Agent Zero Patterns

### Claude Squad → Parallel Subordinates

**Source:** [claude-squad](https://github.com/smtg-ai/claude-squad)

```python
# orchestration/squad.py

class AgentSquad:
    """
    Parallel agent orchestration
    Adapted from claude-squad
    """

    def __init__(self, agent):
        self.agent = agent
        self.squad = []

    async def spawn_squad(self, tasks: list[dict]):
        """Spawn multiple agents working in parallel"""
        import asyncio

        async def run_agent(task: dict):
            return await self.agent.call_subordinate(
                message=task["prompt"],
                reset_context=True
            )

        # Run all tasks in parallel
        results = await asyncio.gather(*[
            run_agent(task) for task in tasks
        ])

        return dict(zip([t["name"] for t in tasks], results))

    async def divide_and_conquer(self, large_task: str, num_agents: int = 3):
        """Split a large task across multiple agents"""

        # Planning agent divides the work
        division = await self.agent.call_subordinate(
            message=f"""Divide this task into {num_agents} independent subtasks:

            Task: {large_task}

            Requirements:
            1. Each subtask must be completable independently
            2. No dependencies between subtasks
            3. Clear boundaries and deliverables
            4. Roughly equal complexity

            Output: JSON array of subtasks with name and prompt.""",
            reset_context=True
        )

        subtasks = json.loads(division)

        # Execute in parallel
        results = await self.spawn_squad(subtasks)

        # Merge results
        merged = await self.agent.call_subordinate(
            message=f"""Merge these parallel execution results:

            Original task: {large_task}
            Results: {json.dumps(results)}

            1. Integrate all outputs
            2. Resolve any conflicts
            3. Verify completeness
            4. Create unified deliverable""",
            reset_context=True
        )

        return merged
```

### Ralph Orchestrator Pattern

**Source:** [ralph-orchestrator](https://github.com/mikeyobrien/ralph-orchestrator)

```python
# orchestration/ralph_orchestrator.py

class RalphOrchestrator:
    """
    Multi-phase Ralph execution with circuit breakers
    Adapted from ralph-orchestrator
    """

    def __init__(self, agent, config: dict = None):
        self.agent = agent
        self.config = config or {
            "max_iterations": 30,
            "escape_threshold": 5,
            "cost_limit": 50.0  # USD
        }
        self.execution_log = []

    async def execute_roadmap(self, roadmap_path: str):
        """Execute entire roadmap phase by phase with Ralph"""

        # Parse roadmap
        phases = await self._parse_roadmap(roadmap_path)

        for phase in phases:
            result = await self._execute_phase_with_ralph(phase)

            if result["status"] == "BLOCKED":
                # Circuit breaker - stop execution
                return {
                    "status": "BLOCKED",
                    "completed_phases": self.execution_log,
                    "blocked_at": phase,
                    "reason": result["reason"]
                }

            self.execution_log.append({
                "phase": phase["name"],
                "result": result
            })

        return {
            "status": "COMPLETE",
            "phases": self.execution_log
        }

    async def _execute_phase_with_ralph(self, phase: dict):
        """Execute single phase with Ralph loop and safety checks"""

        # Create plan for phase
        plan = await self.agent.call_subordinate(
            message=f"""Create GSD plan for phase: {phase['name']}

            Objective: {phase['objective']}

            Output PLAN.md with 2-3 atomic tasks.
            Each task must have executable <verify> command.""",
            reset_context=True
        )

        # Execute with Ralph
        ralph_result = await self.agent.call_subordinate(
            message=f"""/ralph-loop "{plan}

            Execute all tasks.
            Run <verify> for each.
            Commit after each task.

            On completion: <promise>PHASE_COMPLETE</promise>
            If stuck {self.config['escape_threshold']}+ iterations: <promise>PHASE_BLOCKED</promise>
            " --completion-promise "PHASE_" --max-iterations {self.config['max_iterations']}""",
            reset_context=True,
            tools=["ralph-wiggum"]
        )

        return self._parse_ralph_result(ralph_result)
```

---

## Hooks → Agent Zero Middleware

Claude Code hooks run at lifecycle events. In Agent Zero, implement as middleware or validators.

### TDD Guard

**Source:** [tdd-guard](https://github.com/nizos/tdd-guard)

```python
# middleware/tdd_guard.py

class TDDGuard:
    """
    Enforce TDD principles during execution
    Adapted from tdd-guard hook
    """

    def __init__(self, agent):
        self.agent = agent
        self.test_first_required = True

    async def validate_before_implementation(self, task: dict) -> bool:
        """Check that tests exist before implementation code"""

        if not self.test_first_required:
            return True

        # Check for test file
        test_exists = await self.agent.call_tool("code_execution", {
            "command": f"find . -name '*test*.py' -newer {task['files']}"
        })

        if not test_exists:
            # Warn and require test first
            await self.agent.notify_user(
                "TDD GUARD: No test found for this implementation. "
                "Write failing test first!"
            )
            return False

        return True

    async def validate_after_implementation(self, task: dict) -> bool:
        """Ensure tests pass after implementation"""

        result = await self.agent.call_tool("code_execution", {
            "command": "pytest --tb=short"
        })

        if "FAILED" in result:
            await self.agent.notify_user(
                f"TDD GUARD: Tests failing after implementation:\n{result}"
            )
            return False

        return True
```

### Quality Hooks

**Source:** [claude-code-typescript-hooks](https://github.com/bartolli/claude-code-typescript-hooks)

```python
# middleware/quality_guard.py

class QualityGuard:
    """
    Code quality enforcement
    Adapted from typescript-hooks
    """

    async def pre_commit_check(self, files: list[str]) -> dict:
        """Run quality checks before allowing commit"""

        results = {
            "lint": await self._run_linter(files),
            "format": await self._check_formatting(files),
            "types": await self._check_types(files),
            "security": await self._security_scan(files)
        }

        passed = all(r["passed"] for r in results.values())

        if not passed:
            failures = [k for k, v in results.items() if not v["passed"]]
            return {
                "allowed": False,
                "reason": f"Quality checks failed: {', '.join(failures)}",
                "details": results
            }

        return {"allowed": True, "results": results}

    async def _run_linter(self, files: list[str]) -> dict:
        result = await self.agent.call_tool("code_execution", {
            "command": f"eslint {' '.join(files)}"
        })
        return {"passed": "error" not in result.lower(), "output": result}

    async def _check_formatting(self, files: list[str]) -> dict:
        result = await self.agent.call_tool("code_execution", {
            "command": f"prettier --check {' '.join(files)}"
        })
        return {"passed": "error" not in result.lower(), "output": result}
```

---

## Complete Integration: Agent Zero + Awesome Claude Code

Combine all transferred components:

```python
# main.py - Full Agent Zero with awesome-claude-code features

from orchestration.squad import AgentSquad
from orchestration.ralph_orchestrator import RalphOrchestrator
from workflows.riper import riper_workflow
from workflows.project_manager import ProjectManager
from middleware.tdd_guard import TDDGuard
from middleware.quality_guard import QualityGuard
from instruments.devops_skill import DevOpsSkill
from instruments.security_audit import SecurityAudit

class EnhancedAgentZero:
    """
    Agent Zero enhanced with awesome-claude-code capabilities
    """

    def __init__(self, base_agent):
        self.agent = base_agent

        # Orchestration
        self.squad = AgentSquad(base_agent)
        self.ralph = RalphOrchestrator(base_agent)

        # Workflows
        self.pm = ProjectManager(base_agent)

        # Skills
        self.devops = DevOpsSkill(base_agent)
        self.security = SecurityAudit(base_agent)

        # Quality Guards
        self.tdd_guard = TDDGuard(base_agent)
        self.quality_guard = QualityGuard(base_agent)

    async def build_feature(self, request: str, method: str = "riper"):
        """Build a feature using selected methodology"""

        if method == "riper":
            return await riper_workflow(self.agent, request)
        elif method == "gsd":
            return await self._gsd_workflow(request)
        elif method == "squad":
            return await self.squad.divide_and_conquer(request)
        else:
            raise ValueError(f"Unknown method: {method}")

    async def _gsd_workflow(self, request: str):
        """GSD + Ralph execution"""

        # Plan
        plan = await self.agent.call_subordinate(
            message=f"Create GSD PLAN.md for: {request}",
            reset_context=True
        )

        # Quality gate
        pre_check = await self.quality_guard.pre_commit_check([])
        if not pre_check["allowed"]:
            return {"status": "BLOCKED", "reason": pre_check["reason"]}

        # Execute with Ralph
        return await self.ralph.execute_roadmap(".planning/ROADMAP.md")
```

---

## What Doesn't Transfer

| Resource | Reason |
|----------|--------|
| Status Lines | Terminal UI specific to Claude Code |
| IDE Integrations | VSCode/Neovim extensions for Claude Code |
| Usage Monitors | Track Claude Code specific metrics |
| Alternative Clients | Different interfaces to Claude Code |

---

## Quick Reference

| Awesome Claude Code | Agent Zero Equivalent |
|---------------------|----------------------|
| Agent Skills | System prompts or Instruments |
| Workflows | Workflow patterns with subordinates |
| Slash Commands | Tools (`python/tools/`) |
| Hooks | Middleware/validators |
| Orchestration | Squad pattern with `call_subordinate()` |
| CLAUDE.md | Agent Zero config + prompts |

---

## Resources

- [awesome-claude-code](https://github.com/hesreallyhim/awesome-claude-code)
- [Agent Zero](https://github.com/agent0ai/agent-zero)
- [Get Shit Done](../README.md)
- [Ralph Wiggum](https://github.com/anthropics/claude-code/tree/main/plugins/ralph-wiggum)
