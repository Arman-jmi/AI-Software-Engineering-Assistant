# CodePilot AI

**An AI Software Engineering Assistant built on the OpenAI Agents SDK.**

CodePilot AI carries a real GitHub issue through the full engineering lifecycle — requirements extraction, bug investigation, implementation planning with a generated code patch, code review, executed test verification, documentation, and a human-gated pull request — using six specialized agents and five tools, each actually invoked, not just defined.

> Capstone project — Domain: Software Development · Track: AI Software Engineering Assistant

---

## Table of Contents

- [Overview](#overview)
- [Why Multi-Agent](#why-multi-agent)
- [Architecture](#architecture)
- [Agents](#agents)
- [Tools](#tools)
- [Memory & Structured Outputs](#memory--structured-outputs)
- [Human Approval](#human-approval)
- [Getting Started](#getting-started)
- [Usage](#usage)
- [Project Structure](#project-structure)
- [Requirement Compliance](#requirement-compliance)
- [Limitations](#limitations)
- [Future Scope](#future-scope)
- [License](#license)

---

## Overview

Software teams lose real time context-switching between an issue tracker, the source repository, code review, testing, and documentation. CodePilot AI anchors all of that to one real GitHub issue, in one auditable pipeline, while keeping a human in control of any action that actually writes to the repository.

**Highlights**

- 6 specialized agents + 1 human-gated Pull Request agent
- 5 tools, each actually invoked in the notebook (not just defined)
- A real, executed **SDK handoff** demonstration — not just `handoffs=[...]` configuration
- A deterministic, production-style pipeline used as the official workflow
- `SQLiteSession` memory wired into the live pipeline, not only demoed in isolation
- Structured Pydantic outputs threaded between every stage
- Human approval required (SDK-level `needs_approval=True` + an explicit approval gate) before any GitHub write action
- Interactive repository/issue selection — nothing is hard-coded
- Retry-aware error handling that never leaks credentials and never fakes a result

---

## Why Multi-Agent

A single general-purpose agent could attempt every task, but specialization gives each agent a narrow responsibility, a focused prompt, a restricted toolset, and a predictable structured output — mirroring how a real engineering team is organized, and making each stage independently testable, auditable, and improvable.

---

## Architecture

```text
                          Developer
                              |
                              v
                Interactive Repo / Issue Selection
                              |
                              v
                     CodePilot Orchestrator
                  (SQLiteSession + ProjectState)
                              |
           +------------------+------------------+
           |                                     |
           v                                     v
 SDK Handoff Execution                Deterministic Workflow
 real Runner.run(orchestrator)         official submitted pipeline
           |                                     |
           v                                     v
 Requirements -> Bug Investigation -> Coding (patch) -> Code Review
        -> Testing (executed) -> Documentation
                              |
                              v
                    Human Approval Gate
                     /              \
               APPROVE              REJECT
                  |                    |
     create_github_pull_request     stop safely
       (needs_approval=True)      (no repository write)
```

Two orchestration layers are implemented, on purpose:

| Layer | What it proves | When it's used |
|---|---|---|
| **SDK Handoffs** | The Agents SDK's native delegation mechanism actually working — `Runner.run(orchestrator_agent, ...)` is executed for real, and the trace is built from genuine `HandoffCallItem` / `HandoffOutputItem` entries in `RunResult.new_items` | Demonstration of the mechanism |
| **Deterministic Workflow** | Guaranteed stage order, shared structured state, retry/error handling, reproducibility | The official, submitted pipeline |

---

## Agents

| Agent | Responsibility | Tools | Structured Output |
|---|---|---|---|
| Requirements Analysis | Turn a raw GitHub issue into structured requirements | `get_github_issue` | `RequirementAnalysis` |
| Bug Investigation | Hypothesize root cause from the issue (+ source, if available) | `get_github_issue`, `read_github_file` | `BugInvestigation` |
| Coding Assistant | Generate a concrete proposed patch — not just a plan | `read_github_file`, `get_github_issue` | `CodingPlan` (incl. `proposed_code`) |
| Code Reviewer | Review the *generated* patch for correctness, security, quality | `generate_code_diff` | `CodeReview` |
| Testing | Design a test plan and use the real test runner | `run_python_tests` | `TestReport` (incl. `executed` flag) |
| Documentation Writer | Summarize the whole pipeline accurately | — | `DocumentationOutput` |
| Pull Request *(supporting)* | Open a PR only after explicit human approval | `create_github_pull_request` (`needs_approval=True`) | free-form confirmation |

---

## Tools

Each tool keeps a plain, undecorated implementation function (used to prove real execution) plus an `@function_tool`-wrapped version (used by the agents):

| Tool | Purpose | Approval Required |
|---|---|---|
| `get_github_issue` | Retrieve a real issue's title, description, labels, state | No |
| `read_github_file` | Read a real source file from the target repository | No |
| `generate_code_diff` | Produce a unified diff between original and proposed code | No |
| `run_python_tests` | Execute a Python test script in an isolated subprocess | No — trusted code only |
| `create_github_pull_request` | Open a GitHub Pull Request | **Yes** — `needs_approval=True` |

---

## Memory & Structured Outputs

- **`SQLiteSession`** persists conversation history to disk (`codepilot_memory.db`) so context survives across separate `Runner.run` calls. The **same session** is passed into every stage of the official workflow, not only demonstrated in isolation.
- **`ProjectState`** (a dataclass) carries each stage's typed Pydantic output — `requirements`, `bug_analysis`, `coding_plan`, `code_review`, `test_report`, `documentation` — forward in-process, so later stages receive structured objects instead of re-parsed strings.

---

## Human Approval

No Pull Request is ever created automatically. Two independent layers protect any write action:

1. **SDK level** — `create_github_pull_request` is declared with `needs_approval=True`.
2. **Notebook level** — an explicit `input()`-based approval gate, clearly labeled as the Colab demonstration/fallback mechanism, runs before the Pull Request agent is ever invoked.

---

## Getting Started

### Prerequisites

- Google Colab (recommended) or a local Jupyter environment with internet access
- An [OpenAI API key](https://platform.openai.com/api-keys)
- A [GitHub personal access token](https://github.com/settings/tokens) with `repo` scope (read access is enough unless you intend to actually approve a PR, which needs write access)

### Setup (Google Colab)

1. Open `CodePilot_AI_FINAL_READY.ipynb` in Google Colab.
2. Add two Colab Secrets (🔑 panel, left sidebar):
   - `OPENAI_API_KEY`
   - `GITHUB_TOKEN`
3. Run the notebook top to bottom.

API keys are never hard-coded or printed anywhere in this project.

### Setup (local Jupyter)

```bash
pip install openai-agents PyGithub python-dotenv pydantic
export OPENAI_API_KEY="sk-..."
export GITHUB_TOKEN="ghp_..."
jupyter notebook CodePilot_AI_FINAL_READY.ipynb
```

---

## Usage

1. **Select a repository** — enter any `owner/repo` you have access to when prompted; it's validated live against the GitHub API.
2. **Select an issue** — pick from the listed open issues, or enter an issue number directly.
3. **Watch the pipeline run** — Requirements → Bug Investigation → Coding → Code Review → Testing → Documentation.
4. **Review the human approval gate** — approve or reject the proposed Pull Request.
5. **Check the final dashboard** — every line reflects what actually completed; nothing is reported as done that didn't run.

---

## Project Structure

```text
.
├── CodePilot_AI_FINAL_READY.ipynb   # Main notebook — the full implementation
├── CodePilot_AI_Project_Documentation.pdf   # Full written project documentation
├── CodePilot_AI_Presentation.pptx           # 12-slide project presentation
└── README.md
```

---

## Requirement Compliance

| Requirement | Where it's implemented |
|---|---|
| Business context, stakeholders, problem statement, objectives | Notebook §2 / Docs §2 |
| Agent architecture, roles, handoff flow, tool integration | Notebook §3–4 / Docs §3 |
| 5+ specialized agents | 6 agents + Pull Request agent |
| 5+ tools/APIs, each actually invoked | All 5 tools exercised live in the notebook |
| Agent handoffs — actually executed, not just configured | Notebook §21 — real `Runner.run(orchestrator_agent, ...)` |
| Memory used in the main workflow | `SQLiteSession` passed into every stage of §22 |
| Structured outputs, threaded between stages | Pydantic models in §8, used throughout §22 |
| Human approval before sensitive GitHub actions | §23 — SDK `needs_approval=True` + notebook gate |
| Interactive repository/issue selection | §10–11 |
| Error handling / retries | §13, applied throughout |

---

## Limitations

- `proposed_code` is a generated patch for review — it is never automatically committed, pushed, or merged.
- The executed baseline test is a small, self-contained script chosen to be safe to run automatically; it is not a substitute for the target repository's own test suite.
- `run_python_tests` executes arbitrary Python in a subprocess — only ever give it trusted code.
- Opening a Pull Request requires a `GITHUB_TOKEN` with write permission.
- AI-generated root-cause analysis, code, and review findings require human verification before being trusted.
- `SQLiteSession` persistence is local to wherever the notebook runs.

---

## Future Scope

- Apply the generated patch in an isolated sandbox branch rather than only producing a diff.
- Run the target repository's actual test suite via `run_python_tests`.
- GitHub Actions / CI integration so the pipeline triggers on new issues automatically.
- A dedicated security vulnerability scanning agent.
- Multi-language support beyond Python.
- A vector database for long-term, cross-issue project knowledge.
- A web dashboard in place of a notebook interface.

---

## License

Add your license of choice here (e.g. MIT) before publishing this repository.
