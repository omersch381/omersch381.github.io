---
layout: post
title: "How I Used MCP Servers and Claude Code to Tackle Designate's Bug Backlog"
description: "Building MCP servers and skilled agents to eliminate context-switching across Launchpad, Gerrit, DevStack, and tox — and systematically reduce a decade-old bug backlog."
date: 2026-04-20
feature_image: images/Bugs_cleaning.png
tags: [openstack, designate, claude-code, mcp, ai]
---

I am Omer Schwartz, currently [Designate](https://docs.openstack.org/designate/latest/) PTL, which is the DNS-as-a-Service project of OpenStack. At the start of Q4 2025, I decided to tackle an issue the upstream community accumulated over the years: the Launchpad bug backlog had grown unwieldy. Bugs dating back to 2015 — some marked high or critical priority — were piling up, many sitting in "New" or "Confirmed" for years with no one triaging them. Some of them were already fixed, but not updated in Launchpad. Community patches fixing some of those bugs were scattered across four repositories with no clear prioritization.

Over the next two quarters (2025Q4 and 2026Q1), I systematically reduced this backlog. The tooling I built with Claude Code's MCP servers and custom slash commands (skilled agents) turned what would have been months of tedious context-switching into a focused, efficient effort.

<!--more-->

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

Here are additional experiments with skilled agents:

### `/update-review-priority-list` - The Backlog Command Center

This skill maintains a prioritized list of all open patches across four Designate repositories, currently tracking 100+ patches.

Before building this, I maintained the priority list using Go scripts that queried the Gerrit API, generated a flat list, and required either manual sorting or manual tagging followed by automated sorting. The Claude Code skill replaced all of that with a command that:

1. Fetches the current release cycle name
2. Queries every open patch across `openstack/designate`, `openstack/python-designateclient`, `openstack/designate-dashboard`, and `openstack/designate-tempest-plugin`
3. Removes merged and abandoned patches
4. Discovers new patches not yet in the list
5. Cross-references `@skip_because(bug=...)` decorators in tempest tests with patches that fix those bugs
6. Builds a dependency map from `Depends-On:` headers between patches
7. Categorizes patches into priority groups: features first, then CI-unblocking, then bugfixes, then "regular" ones

The cross-referencing in step 5 was particularly valuable: when a patch both closes a bug and unblocks a skipped tempest test, that's triple impact per review — exactly the kind of prioritization that makes a backlog effort efficient.

The skill uses `mcp__gerrit__get_patch` and `mcp__gerrit__search_patches` heavily, parallelizing queries using Claude Code's Agent tool with the cheaper Haiku model. The main Opus model orchestrates the workflow and makes judgment calls (prioritization, categorization), while Haiku agents handle the 20+ parallel API queries. This saves meaningful tokens on what is largely mechanical data fetching.

### `/verify-release` - Release Verification

At the end of each release cycle, the Release Management team proposes a release patch in the `openstack/releases` repository. This skill verifies that the commit hashes in that patch match the actual branch tips, and lists any open patches still under review that won't make it into the release if we approve it as-is. It saves some manual verification work.

## Back to the Backlog: What Worked Well

- **FastMCP's simplicity**: Each server is under 200 lines. The `@mcp.tool()` decorator with type-annotated args and a docstring is all you need.
- **Persistent SSH connections**: The DevStack server's lifespan-managed connection avoids reconnecting on every command, making bug reproduction feel instant.
- **Output management**: Truncating large diffs (100k char limit) and stripping tox noise prevents context window bloat.
- **Skills as composition layers**: Raw tools (MCP servers) are simple and reusable. Skills compose them into domain-specific workflows. This separation kept both clean.
- **Haiku for bulk queries**: Using the cheaper model for parallel data fetching while Opus handles reasoning saved meaningful tokens during the priority list updates.

## What Could Be Better

1. **Caching**: The priority list skill queries 100+ patches one at a time. A short-lived cache (even 5 minutes) for patch metadata would cut latency and context / tokens significantly.

2. **Tox runner portability**: The tox runner MCP server is currently hardcoded to the Designate repository. Ideally it would run against whatever the current working directory is, so it could be reused across the other Designate repositories without modification.

3. **Multi-VM support**: The DevStack MCP server is hardcoded to one VM. Testing multipool and grenade upgrade scenarios sometimes needs different VMs, and making the target configurable per-session would help.
