# Cadence: Autonomous Agent Orchestration

Cadence is a language-agnostic orchestration framework for running coding agents autonomously on tracked work. This guide helps AI coding agents understand the project structure, conventions, and how to contribute effectively.

## Project Philosophy

Cadence transforms project work from **agent supervision** to **work management**:

1. **Unified tracker**: Cadence polls Linear continuously for work in active states.
2. **Isolated execution**: Each issue gets its own workspace and isolated Codex session.
3. **Autonomous workflows**: Agent behavior is driven by a repo-versioned `WORKFLOW.md` file that combines configuration + prompt.
4. **Stateless orchestration**: The system reconciles state via the tracker on restart; no persistent database required.
5. **Observable operations**: Agents provide proof of work (CI status, PR review feedback, complexity analysis) as they progress.

## Project Structure

```
cadence/
  .codex/skills/          # Shared skills for agent sessions (linear, commit, push, pull, land, debug)
  .github/                # GitHub workflows, PR templates
  SPEC.md                 # Language-agnostic specification for Cadence implementations
  README.md               # Project overview and quickstart
  elixir/                 # Reference Elixir/OTP implementation
    AGENTS.md             # Elixir-specific agent guidance
    WORKFLOW.md           # Example workflow: YAML config + Liquid prompt template
    lib/                  # Main implementation
    test/                 # Test suite (100% coverage enforced)
    Makefile              # Build and validation commands
    README.md             # Elixir setup and runtime instructions
  docs/                   # Architecture and operational docs
    logging.md            # Logging conventions and context fields
    token_accounting.md   # Token budgeting for agent sessions
```

## Architecture Highlights

### Core Components (per language implementation)

| Component | Purpose |
|-----------|---------|
| **Orchestrator** | GenServer/event loop polling the tracker; dispatches work with bounded concurrency and exponential backoff retry |
| **Agent Runner** | Executes a single issue in an isolated workspace; manages Codex app-server session lifecycle |
| **Workflow Engine** | Loads config and prompt from WORKFLOW.md (YAML front matter + Liquid templating) |
| **Workspace Manager** | Creates per-issue isolation; enforces safety rules (e.g., workspaces stay under configured root) |
| **Tracker Adapter** | Abstraction for issue tracker (Linear, memory for testing, etc.) |
| **Prompt Builder** | Renders agent prompts from issue context using templates |

### Shared Skills (in `.codex/skills/`)

Agents use these skills during app-server sessions to perform common workflows:

| Skill | Purpose |
|-------|---------|
| **`linear`** | Raw GraphQL for Linear operations (comment editing, uploads); uses Symphony's injected `linear_graphql` tool |
| **`commit`** | Create conventional-format commits from session work history |
| **`push`** | Publish branch updates to remote; handles tracking branch setup |
| **`pull`** | Fetch and merge latest changes; resolve conflicts with origin/main |
| **`land`** | Finalize PR workflow: monitor conflicts, resolve, watch CI, squash-merge when green |
| **`debug`** | Troubleshooting guidance for common orchestration issues |

See [`.codex/skills/`](.codex/skills/) for detailed skill documentation.

## Key Conventions (All Implementations)

### Specification Alignment

- All language implementations MUST follow [SPEC.md](SPEC.md) normatively.
- Implementations MAY add features not in the spec, but MUST NOT conflict with spec behavior.
- If implementation changes meaningfully alter intended behavior, update the spec in the same PR.

### Workspace Safety (Critical)

- **Never** run agent sessions (Codex turns) with `cwd` in the source repository.
- Workspaces MUST stay under the configured workspace root (see WORKFLOW.md).
- Validate workspace paths before executing agent code.

### Configuration & Secrets

- Runtime config is loaded from `WORKFLOW.md` front matter (YAML) + Liquid templates.
- Store secrets in environment variables; never commit sensitive data.
- Provide clear error messages if required secrets are missing (e.g., `LINEAR_API_KEY`).

### Logging & Observability

- Log structured context for every issue-related operation: `issue_id` (UUID), `issue_identifier` (human-readable key).
- Log session context for agent operations: `session_id`, `attempt_number`, `turn_number`.
- Follow [docs/logging.md](docs/logging.md) for context field standards.

### State Management

- Preserve orchestrator state semantics: single source of truth for dispatch, retries, reconciliation.
- Use exponential backoff for transient failures (base: 10s).
- Reconcile state via the tracker on restart; no persistent database required.

### Prompt & Workflow Contract

WORKFLOW.md structure (all implementations MUST support):

```markdown
---
tracker:
  kind: linear                     # Tracker type
  project_slug: "..."              # Project identifier
  active_states: [...]             # States considered "work to do"
  terminal_states: [...]           # States where orchestration stops
agent:
  max_concurrent_agents: 10        # Concurrency limit
  max_turns: 20                    # Max agent turns per issue
codex:
  command: "codex app-server ..."  # Codex invocation
  approval_policy: never           # Agent approval strategy
---

You are working on a Linear ticket `{{ issue.identifier }}`

[Liquid template with issue context]
```

Available template variables: `{{ issue.identifier }}`, `{{ issue.title }}`, `{{ issue.state }}`, `{{ issue.labels }}`, `{{ issue.url }}`, `{{ issue.description }}`, `{{ attempt }}`.

## Implementation Guidance

### For Cadence Language Implementations

If you are implementing Cadence in a new language:

1. **Understand the spec first**: Read [SPEC.md](SPEC.md) thoroughly; it defines the contract.
2. **Start minimal**: Implement core components in order: Tracker adapter → Config loader → Workspace manager → Orchestrator → Agent runner.
3. **Test early**: Create unit tests for each component; integration tests for orchestrator reconciliation.
4. **Validate workspace safety**: Write tests that verify workspaces stay under the root; agent cwd is never in source repo.
5. **Document your conventions**: Create `YOUR_LANGUAGE/AGENTS.md` (similar to [elixir/AGENTS.md](elixir/AGENTS.md)) covering language-specific patterns.
6. **Create a WORKFLOW.md example**: Your implementation should include a working example workflow.
7. **Update SPEC.md** if you discover necessary clarifications.

See [elixir/README.md](elixir/README.md) for a concrete reference implementation.

### For Cadence Feature Development

1. **Align with spec**: Proposed changes to core behavior MUST be spec-aligned.
2. **Update docs**: If behavior changes, update relevant docs in the same PR.
3. **Test thoroughly**: Use the validation gates (format check, lint, coverage, type checking).
4. **Keep workspaces safe**: Any new feature touching workspace/agent execution must preserve isolation guarantees.

## Development Workflow

### Elixir Implementation (`elixir/` directory)

**Quick start:**
```bash
cd elixir
mise trust && mise install          # Set up Elixir/OTP 28 via mise
mix setup                           # Install dependencies
make all                            # Run full validation (format → lint → coverage → dialyzer)
mix build                           # Create escript
./bin/symphony WORKFLOW.md          # Run the orchestrator
```

**Quality gates** (must pass before handoff):
```bash
make all                            # Format check, lint, 100% coverage, dialyzer
make ci                             # Full CI pipeline
mix pr_body.check --file PR_BODY.md # Validate PR body format
```

**Key commands:**
- `mix test` — Run unit tests
- `mix coverage` — Run tests with 100% coverage enforcement
- `make fmt` — Format code
- `make lint` — Specs check + credo linting

**Spec validation** (all public functions require spec):
```bash
mix specs.check
```

See [elixir/AGENTS.md](elixir/AGENTS.md) for Elixir-specific rules and patterns.

### Implementing Skills

Skills are reusable workflows for agent sessions. To create a new skill:

1. Create `.codex/skills/YOUR_SKILL_NAME/SKILL.md`
2. Add YAML frontmatter with `name` and `description`
3. Document the skill's purpose, preconditions, steps, and examples
4. Reference it in `WORKFLOW.md` or relevant agent guidance

See existing skills in [`.codex/skills/`](.codex/skills/) for examples.

## Common Workflows

### Deploying a Cadence Instance

1. Choose a language implementation (Elixir reference or your own).
2. Set up the codebase in your repo per implementation docs.
3. Configure `WORKFLOW.md`:
   - Set `tracker.project_slug` to your Linear project slug.
   - Adjust `agent.max_concurrent_agents` and `max_turns` for your needs.
   - Customize the Liquid-templated prompt section.
4. Set `LINEAR_API_KEY` environment variable.
5. Run the orchestrator: `./bin/symphony WORKFLOW.md` (Elixir) or language-specific equivalent.

### Handling Agent Failures

When an agent fails on an issue:

1. **Check logs**: Review orchestrator and agent logs under `./log` (Elixir: `--logs-root` flag).
2. **Inspect the workspace**: Examine the issue workspace to see what happened (e.g., test failures, compile errors).
3. **Understand the retry policy**: The orchestrator automatically retries with exponential backoff.
4. **Fix the blocker**: Often issues are environment setup, missing secrets, or edge cases in the WORKFLOW.md prompt.
5. **Restart the orchestrator**: State reconciles via the tracker on restart; you can resume issue processing.

### Extending the Workflow

The `WORKFLOW.md` Liquid template allows dynamic prompt generation. Common patterns:

```liquid
{% if issue.labels contains "bug" %}
  This is a bug report. Reproduce first, then fix.
{% elsif issue.labels contains "feature" %}
  This is a feature request. Start with design and test plan.
{% endif %}

{% if attempt > 1 %}
  Continuation attempt #{{ attempt }}. Resume from current workspace state.
{% endif %}
```

Customize the prompt to reflect your team's development philosophy and issue handling practices.

## Troubleshooting

### Workspace Path Issues

**Problem**: "Workspace outside configured root"

**Solution**: Verify `workspace.root` in WORKFLOW.md; ensure it's an absolute path and writable; check workspace cleanup hooks don't interfere.

### Missing Linear Auth

**Problem**: "LINEAR_API_KEY not found"

**Solution**: Set `export LINEAR_API_KEY=$(gh secret view LINEAR_API_KEY)` or similar; ensure Linear API token has project/issue read permissions.

### Agent Stuck in Loop

**Problem**: Agent processes but doesn't transition issue states

**Solution**: Check that agent has Linear mutation permissions; review WORKFLOW.md error handling for Linear API calls; check agent's state transition logic matches workflow expectations.

### CI Failures in Merged PR

**Problem**: Agent lands PR but CI fails in main

**Solution**: Review the land skill preconditions; ensure agent ran `make all` locally before landing; adjust `max_turns` if agent doesn't have enough turns to fix CI issues.

## Related Documentation

- [SPEC.md](SPEC.md) — Normative specification (all implementations)
- [README.md](README.md) — Project overview
- [elixir/README.md](elixir/README.md) — Elixir setup and runtime
- [elixir/AGENTS.md](elixir/AGENTS.md) — Elixir-specific conventions
- [docs/logging.md](docs/logging.md) — Logging context standards
- [docs/token_accounting.md](docs/token_accounting.md) — Token budgeting for sessions
- [.codex/skills/](.codex/skills/) — Shared skill documentation

## Questions for Agents

When contributing to Cadence, keep these questions in mind:

1. **Does this preserve the spec?** If spec changes are needed, propose them.
2. **Is workspace safety maintained?** Can agents execute in isolation?
3. **Is state reconciliation preserved?** Does the orchestrator still work correctly on restart?
4. **Are docs updated?** Do SPEC.md, README.md, and language-specific guides reflect the change?
5. **Are tests comprehensive?** Is coverage maintained or improved?
6. **Is the change minimal?** Avoid scope creep; file follow-up issues for improvements.

---

**Last updated**: 2026-05-14  
**Contact**: See GitHub repository for issue tracker and discussions.
