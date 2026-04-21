---
layout: post
title: "How I Used MCP Servers and Claude Code to Tackle Designate's Bug Backlog"
description: "Building MCP servers and skilled agents to eliminate context-switching across Launchpad, Gerrit, DevStack, and tox — and systematically reduce Designate's bug backlog."
date: 2026-04-20
feature_image: images/Bugs_cleaning.png
tags: [openstack, designate, claude-code, mcp, ai]
---

I am Omer Schwartz, currently Project Team Lead (PTL) of [Designate](https://docs.openstack.org/designate/latest/), OpenStack's DNS-as-a-Service project. As a PTL, my responsibilities include setting the project's technical direction, coordinating releases, shepherding code reviews, and maintaining the bug backlog. That last one — backlog maintenance — is the focus of this post.

A bug backlog that grows unchecked over time has a compounding effect. When untriaged reports pile up and issues sit in "In Progress" with no visible movement, the tracker starts to feel unreliable. Users and operators are less likely to file detailed reports if it's not clear that existing ones are being looked at, and contributors may hesitate to pick up bugs when there's no prioritization to guide them. Over time, an unmaintained backlog can quietly erode trust in the project's responsiveness.

Designate's Launchpad tracker had reached that point. Over 150 open bugs across all states, many untouched for years — some marked high or critical priority, some already fixed upstream but never updated in Launchpad. Community patches fixing some of those bugs were scattered across four repositories with no clear prioritization.

Over two quarters (2025Q4 and 2026Q1), I systematically reduced this backlog. The tooling I built with Claude Code's MCP servers and custom slash commands (skilled agents) saved significant time and money by eliminating the constant context-switching between Launchpad, Gerrit, SSH sessions, and local testing.

## The Results

Addressing a bug here means either submitting a fix for review on Gerrit, or closing it after triage (duplicates, already fixed upstream, or obsolete). Many of the submitted fixes are still awaiting review due to limited reviewer capacity — the investigation and fix work is done, but the Launchpad status may still show "In Progress" until the patch merges.

**Q4 2025** — Starting point: **156 open bugs**. By the end of the quarter, 40 bugs were addressed (**26%**): 28 had fixes submitted for review and 13 were closed after triage.

**Q1 2026** — Starting point: **116 open bugs**, plus 10 new bugs opened during the quarter (126 total). 42 bugs were addressed (**33%**): 17 fixed and 25 closed after triage. Similar triage and fix efforts were carried out in python-designateclient and designate-dashboard as well.

**Across both quarters**, the backlog went from 156 to 84 open bugs — a **46% reduction**.

## The Backlog Problem

Designate's Launchpad tracker tells the story in numbers. Before the effort began, there were well over 100 open bugs across all states -- 24 in "New" (many untriaged, with "Undecided" importance), 12 in "Confirmed", 60 in "In Progress", 8 in "Triaged" -- spanning nearly a decade of reports. Many "In Progress" bugs had been sitting there for years with no actual progress.

The work wasn't just about writing fixes. It involved:

1. **Triaging**: Reading each bug, reproducing it, setting importance, assigning ownership
2. **Cross-referencing**: Matching bugs to existing Gerrit patches, some of which had been open for months or years
3. **Fixing**: Writing new patches for unaddressed bugs
4. **Prioritizing reviews**: Organizing more than 90% of those open patches across four repos by impact
5. **Testing**: Verifying fixes on a DevStack VM when possible before submitting
6. **Unblocking CI**: Identifying which patches fix bugs that cause tempest tests to be skipped

Each of these steps touched different systems: Launchpad for bugs, Gerrit for patches, SSH for the DevStack VM, tox for local testing. The constant context-switching between browser tabs, terminals, and SSH sessions was the real bottleneck -- not necessarily the work itself. Can you imagine the amount of context (tokens) Claude would have consumed for so many different bugs?

## The Solution: MCP Servers + Skilled Agents

I built four MCP servers to bring all of these systems into Claude Code, and three slash-command skills that compose them into multi-step workflows. The total code is under 900 lines of Python. Obviously Claude Code did most of the coding :)

Before I had MCP servers, every session with Claude involved a discovery phase: it would try a `curl` command against the Launchpad or Gerrit API, get the response format wrong, adjust, try again — sometimes taking several attempts before landing on the right query. Claude started fresh each session, and even with guidance it spent a meaningful chunk of tokens just figuring out how to talk to these services. Multiply that by dozens of bugs across several APIs, and the token cost would have added up fast. With the MCP servers properly configured, Claude calls the right tool with the right parameters on the first try. No fumbling, no retries, no wasted context. The token savings alone made the investment in writing these servers worthwhile.

### The Launchpad MCP Server: Bug Fetching at Scale

This server was the entry point for the backlog reduction. It wraps Launchpad's REST API with three read-only tools: `search_bugs` to filter by project, status, and importance; `get_bug` to pull up full details including per-project task status, assignee, and tags; and `get_bug_comments` to read the discussion history.

A typical triage session started with me giving Claude a bug URL, then having it read through the details. With the full description, comments, and task status available in a single call, I could quickly decide: is this still relevant? Do we have a duplicate bug for it or similar ones? What's the right importance? Is there already a patch for this? (using the Gerrit MCP server). Through this workflow, I triaged dozens of bugs — setting importance, confirming or closing stale reports, and linking them to the Gerrit patches that would fix them.

### The Gerrit MCP Server: Connecting Patches to Bugs

The Gerrit server got the heaviest use during the backlog effort. It wraps OpenDev's Gerrit REST API with five tools: `search_patches`, `get_patch`, `get_patch_content`, `get_patch_comments`, and `get_patch_history`.

The key workflow was cross-referencing: for each triaged bug, Claude searched Gerrit for existing patches — both pending reviews and, even more helpfully, already-merged ones.

### The DevStack SSH MCP Server: Reproducing and Verifying

Many of the backlog bugs needed reproduction before they could be triaged or fixed. The DevStack SSH server maintains a persistent SSH connection to my test VM and lets Claude execute commands, restart services, deploy files, and read logs -- all in the flow of a conversation.

```python
@asynccontextmanager
async def lifespan(server):
    conn = await asyncssh.connect(
        VM_HOST, known_hosts=None, agent_forwarding=True,
    )
    server._ssh_conn = conn
    try:
        yield
    finally:
        conn.close()
        await conn.wait_closed()
```

The server handles DevStack-specific setup -- sourcing the OpenRC credentials and activating the virtual environment -- so every command starts from a clean, authenticated state:

```python
OPENRC = "source /opt/stack/devstack/openrc admin admin"
VENV = "source /opt/stack/data/venv/bin/activate"

async def _run(cmd: str, openrc: bool = False, venv: bool = False) -> str:
    preamble = []
    if venv:
        preamble.append(VENV)
    if openrc:
        preamble.append(OPENRC)
    full_cmd = " && ".join(preamble + [cmd]) if preamble else cmd
    conn = mcp._ssh_conn
    result = await conn.run(full_cmd, check=False)
    ...
```

A typical bug investigation looked like this: Claude would pull up the bug from Launchpad, find the relevant code path locally, write a reproducer script, deploy it to the VM, run it, and check the logs. After we decided on a fix, it restarted the relevant Designate services, re-executed the reproducer, and confirmed the fix worked — sometimes by tailing the relevant service logs.

### The Tox Runner MCP Server: Quality Before Submission

Every bug fix needs to pass tests and linters before it can be submitted. The tox runner executes environments locally with smart output filtering -- it strips most irrelevant Python deprecation warnings. Those warnings matter and deserve attention, but we address them separately each cycle — not while triaging bugs. Without filtering, test output is swamped with upstream library warnings that waste context window space. This small, domain-specific decision made a big difference when processing dozens of patches.

## Skilled Agents: The Backlog Reduction Workflows

The MCP servers provide individual tools. The real leverage came from composing them into multi-step workflows using Claude Code's custom slash commands. I'm still exploring what's possible here and probably have a lot more to learn, but even these early skills have already proven their value.

### `/lint` - Pre-Commit Quality Gate

The linting step needs to be fast. This skill runs multiple checks, but (hopefully) only what's needed:

- **pep8** always runs.
- **Unit and functional tests** only run if source code was changed. The skill first runs the tests most closely related to the change, then expands to the surrounding test class or the full suite if needed.
- **Docs and api-ref** checks only run if documentation files were modified.
- **Repository style checks** catch Designate-specific violations that flake8 doesn't enforce.

I also built skills for review prioritization and release verification, which I plan to cover in a future post.

## Back to the Backlog: What Worked Well

- **FastMCP's simplicity**: Each server is under 200 lines. The `@mcp.tool()` decorator with type-annotated args and a docstring is all you need.
- **Persistent SSH connections**: The DevStack server's lifespan-managed connection avoids reconnecting on every command, making bug reproduction feel instant.
- **Output management**: Truncating large diffs (100k char limit) and stripping tox noise prevents context window bloat.
- **Skills as composition layers**: Raw tools (MCP servers) are simple and reusable. Skills compose them into domain-specific workflows. This separation kept both clean.
- **Haiku for bulk queries**: Using the cheaper model for parallel data fetching while Opus handles reasoning saved meaningful tokens during the priority list updates.

## What Could Be Better

1. **Tox runner portability**: The tox runner MCP server is currently hardcoded to the Designate repository. Ideally it would run against whatever the current working directory is, so it could be reused across the other Designate repositories without modification.

2. **Multi-VM support**: The DevStack MCP server is hardcoded to one VM. Testing multipool and grenade upgrade scenarios sometimes needs different VMs, and making the target configurable per-session would help.
