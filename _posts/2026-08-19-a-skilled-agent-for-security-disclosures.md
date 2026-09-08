---
layout: post
title: "A Skilled Agent for Security Disclosures"
description: "Handling a growing queue of embargoed security bugs as PTL, the standard VMT questionnaire behind every disclosure, and the skilled agent I built to answer it."
date: 2026-08-19
tags: [openstack, designate, claude-code, mcp, ai, security]
---

In my [previous posts]({{ site.baseurl }}/#skilled-agents-ptl-workflows), I covered the MCP servers and skilled agents I built to reduce Designate's bug backlog and streamline recurring PTL work: review prioritization, release verification, context-aware linting. Those are all public, low-stakes, undo-able workflows. This post is about a different category entirely: embargoed security bugs, where the cost of a wrong or slow answer is much higher, and where I don't get to just iterate in the open.

## The Growing Security Queue

In practice, as PTL I'm usually the one who ends up reviewing it first whenever someone — an operator, a researcher, a fellow contributor — reports something that looks like a security issue. That queue doesn't behave like the regular bug backlog. Reports are infrequent enough that you don't get to build a comfortable, memorized routine for handling them, but each one might be high-stakes enough that you can't afford to wing it either. And once you're in one, the Vulnerability Management Team (VMT) process isn't optional structure — it's the thing that keeps the disclosure coordinated across the reporter, the security team, and everyone downstream who needs advance notice.

But infrequent doesn't mean unstructured. Once I was actually inside a report, I noticed I was answering nearly the same set of questions every single time, just applied to different code paths. So the queue has it both ways: too rare to ever become routine, but repetitive enough underneath that I was re-deriving the same answers from scratch each time. That combination felt like exactly the kind of thing worth turning into a skilled agent — the same instinct that drove the backlog and review-priority work, just aimed at a much more sensitive workflow.

## The Questions Behind Every Disclosure

Once a report is confirmed as a real security concern, the conversation with the security team converges on the same handful of questions, regardless of which component is affected:

1. **Can you reproduce it?**
2. **What fix(es) would you recommend?** Sometimes more than one, if the report turns out to have more than one independent root cause.
3. **Which branches and versions are affected?** And critically — will the fix even apply cleanly to each currently maintained stable branch, or does something else need to land first?
4. **Should this stay embargoed, or move to public security?** And if there's more than one fix, should they ship together or separately?
5. **What severity classification does this deserve?**
6. **What follow-up work, public or private, should happen after the immediate fix ships?**

## Building a Skilled Agent

So I built a skilled agent whose whole job is to work through that questionnaire for me, on the next report. Given a (public) bug reference, a (public) patch reference, or just a pasted report, it:

- Reads the actual source itself and forms its own theory of the root cause, rather than trusting the reporter's framing
- Searches the rest of the codebase for the same pattern — the same missing check, the same unscoped lookup — since the case above made it obvious that a reported instance is rarely guaranteed to be the only one
- Stops and asks for explicit permission before changing my VM state, mentioning exactly what it's about to run and what it might expose
- If confirmed, it reproduces it live, capturing real output rather than a predicted one
- Walks git history backward from master to pin down the commit that introduced the bug, then checks each maintained stable branch to establish the full affected range
- Checks each stable branch for a clean apply and, if one wouldn't apply cleanly, specifically hunts for the prerequisite patch that would need to land first
- Drafts fix recommendations as a written explanation with an illustrative code sketch, assigns a severity classification, and recommends a disclosure track

Its entire output is a single labeled draft — "for your review, not yet posted anywhere" — that I read, edit, and decide whether to post myself. It never runs `git commit`, `git review`, or anything that touches a tracker. For something this sensitive, the agent's job is to shorten my path to a well-supported answer, not to make the disclosure decision for me.

## What Worked Well

- **Treating the VMT questionnaire as the actual spec.** Once I had the recurring questions written down explicitly, building the agent around them directly — rather than a generic "investigate this bug" prompt — made its output land in the shape the security team actually needs.
- **Building permission-gating into the workflow itself, instead of relying on myself to remember to ask each time.** Embargoed reproductions are exactly the place where "I'll remember to ask first" is the wrong thing to depend on.
- **Keeping it stateless, on purpose.** Every case has different preconditions and impact, and I didn't want a severity call from one ticket quietly anchoring the next one. It's a deliberate trade-off against convenience.

## What Could Be Better

1. **The actual embargo choreography — disclosure date, downstream stakeholder notifications, CVE requests — is still entirely manual**, and I think it should stay that way for now. That's coordination with real people on a timeline, not investigation, and it doesn't feel like something to hand to an agent yet.
