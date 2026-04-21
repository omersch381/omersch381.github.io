---
layout: post
title: "Skilled Agents for PTL Workflows"
description: "A few ideas for useful Claude Code skilled agents — context-aware linting, review prioritization, and release verification."
date: 2026-04-21
tags: [openstack, designate, claude-code, mcp, ai]
---

In my [previous post]({{ site.baseurl }}/mcp-servers-claude-code-designate-backlog), I described the MCP servers I built to bring Launchpad, Gerrit, DevStack, and tox into Claude Code, and how they helped reduce Designate's bug backlog. The MCP servers provide individual tools — fetching a bug, searching for patches, running tests. But the real leverage came from composing them into multi-step workflows using Claude Code's custom slash commands (skilled agents).

This post covers three skills that address recurring PTL responsibilities: prioritizing code reviews across multiple repositories, verifying releases at the end of each cycle, and context-aware linting.

## `/update-review-priority-list` - The Backlog Command Center

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

## `/verify-release` - Release Verification

At the end of each release cycle, the Release Management team proposes a release patch in the `openstack/releases` repository. This skill verifies that the commit hashes in that patch match the actual branch tips, and lists any open patches still under review that won't make it into the release if we approve it as-is. It saves some manual verification work.

## `/lint` - Context-Aware Pre-Commit

As I mentioned in the [previous post]({{ site.baseurl }}/mcp-servers-claude-code-designate-backlog), this skill runs context-aware linting — only the checks that are relevant to what changed:

- **pep8** always runs.
- **Unit and functional tests** only run if source code was changed. The skill first runs the tests most closely related to the change, then expands to the surrounding test class or the full suite if needed.
- **Docs and api-ref** checks only run if documentation files were modified.
- **Repository style checks** catch Designate-specific violations that flake8 doesn't enforce.

## What Worked Well

- **Skills as composition layers**: Raw tools (MCP servers) are simple and reusable. Skills compose them into domain-specific workflows. This separation kept both clean.
- **Haiku for bulk queries**: Using the cheaper model for parallel data fetching while Opus handles reasoning saved meaningful tokens during the priority list updates.

## What Could Be Better

1. **Caching**: The priority list skill queries 100+ patches one at a time. A short-lived cache (even 5 minutes) for patch metadata would cut latency and context / tokens significantly.
