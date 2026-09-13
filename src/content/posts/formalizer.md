---
title: "I Tried to Build an AI Formalizer for Mathematics"
description: "An attempt to turn math papers into Lean proofs, and what happens when you design a framework before understanding the work."
pubDatetime: 2026-09-04T00:00:00+02:00
draft: false
tags:
  - ai
  - lean
  - formal verification
  - side projects
---

<figure class="formalizer-figure formalizer-art">
<img src="/images/formalizer/mathematical-machine.webp" width="1536" height="1024" alt="An elaborate imaginary machine fed with loose papers, watched by a small figure; a blue and copper etching." loading="eager" fetchpriority="high">
<figcaption>A machine for a problem I had yet to understand. AI-generated illustration.</figcaption>
</figure>

In spring 2026, while working on my [Bachelor's thesis on automated theorem proving with AI](https://apd.uw.edu.pl/diplomas/251931), I spent almost two months building **Formalizer**. The idea was simple: upload a mathematics paper, translate it into Lean, and find out whether it was correct.

[Lean](https://lean-lang.org/) is a proof assistant that mechanically checks proofs written in its formal language. In our thesis experiments, even relatively inexpensive models could prove difficult, already-formalized theorems. I wanted to take the next step: start with a paper instead of a prepared Lean statement.

I got as far as toy examples. Along the way, I built several interfaces, changed the architecture repeatedly, and eventually deleted much of it.

## What Lean actually guarantees

My original plan was a mathematical oracle: **PDF in, verified Lean project out**, with a verdict on the paper.

The problem is that proving a statement and translating it faithfully are different tasks. An agent can add an assumption, omit a case, or choose a definition that changes the meaning. Lean can then check a perfectly valid proof of the wrong theorem. Failure to find a proof is equally inconclusive: the paper might be correct and the agent simply stuck.

<figure class="formalizer-figure formalizer-diagram formalizer-boundary">
<svg class="formalizer-boundary-layout" viewBox="0 0 620 150" role="img" aria-labelledby="boundary-title boundary-desc">
<title id="boundary-title">From a paper's theorem to a checked Lean proof</title>
<desc id="boundary-desc">Human review checks the translation from the paper to a Lean statement. Lean mechanically checks the proof against that statement.</desc>
<defs><marker id="boundary-arrow" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto"><path d="M1 1 L8 5 L1 9" /></marker></defs>
<rect class="boundary-paper" x="1" y="15" width="125" height="120" rx="12" />
<text class="boundary-label" x="21" y="45">PAPER</text>
<text class="boundary-text" x="21" y="81">Theorem</text>
<text class="boundary-note" x="207" y="56" text-anchor="middle">Translate</text>
<path class="boundary-translation" d="M144 75 H269" marker-end="url(#boundary-arrow)" />
<text class="boundary-note boundary-muted" x="207" y="100" text-anchor="middle">Human review</text>
<rect class="boundary-lean" x="288" y="15" width="331" height="120" rx="12" />
<text class="boundary-label boundary-accent" x="308" y="45">LEAN</text>
<text class="boundary-text" x="308" y="81">Statement</text>
<path class="boundary-proof" d="M406 75 H528" marker-end="url(#boundary-arrow)" />
<text class="boundary-text" x="547" y="81">Proof</text>
<text class="boundary-note boundary-accent" x="308" y="115">Mechanically checked</text>
</svg>
<figcaption>A checked proof is only as useful as the translation of its statement.</figcaption>
</figure>

This meant I needed someone who understood the mathematics to review the translation. But asking them to inspect a large, messy AI-generated Lean repository did not seem much easier than asking them to construct the formalization themselves.

I borrowed an idea from collaborative formalization projects: a **blueprint**, a human-readable account of the definitions, lemmas, and proof steps, linked to Lean declarations and organized as a dependency graph. [Terence Tao's tour of a blueprint](https://terrytao.wordpress.com/2023/11/18/formalizing-the-proof-of-pfr-in-lean4-using-blueprint-a-short-tour/) explains the approach through a real project.

In Formalizer, each node held an informal statement, a proof strategy, and its dependencies. A human could inspect the mathematical structure before reading the Lean translation. Agents could use the same graph to divide the work and prove nodes in dependency order.

## Review does not fit into three phases

I initially planned three stages, with human approval between them: build the blueprint, translate its statements into Lean, then prove them. Catching mistakes early mattered because even small theorems could burn many tokens on reasoning, library search, compilation, and retries.

But some bad translations only became apparent during proof attempts. A definition could compile yet be awkward to use with Mathlib, Lean's mathematics library. Fixing it could force changes to several later statements. The workflow needed to move backwards, and my state machine kept growing to accommodate that. I kept redesigning the workflow, but I hadn’t formalized enough mathematics myself to know which design would work.

I started thinking of the project as a “Cursor for mathematicians”: a tool where the user could inspect the graph, edit statements, and ask agents to work on individual nodes. I built a web UI around this idea, hoping the blueprint would make reviewing the agent’s work easier.

<figure class="formalizer-figure formalizer-wide formalizer-ui">
<img src="/images/formalizer/ui-with-chat.png" width="1280" height="720" alt="Restored Formalizer interface showing an informal dependency graph, a selected lemma, and a staged question-and-answer exchange in the node chat." loading="lazy">
<figcaption>The original Formalizer review interface, restored from source history. The blueprint and chat are staged illustrations, nothing here was checked in Lean.</figcaption>
</figure>


## Letting agents edit

One design change was especially useful. Initially, agents returned candidate files as structured output, and only the orchestrator could write them. This gave me control over changes, but put the orchestrator in the middle of the agent's edit–compile–inspect loop.

I switched to isolated **git worktrees**. An agent could edit real files, invoke Lean, and retry inside its own checkout. Its work could then be validated and reviewed as a diff before merging into the main project.


Compilation was only one check. Validators also restricted which files could change, rejected remaining proof placeholders, and checked that a proof agent preserved the approved theorem statement. Otherwise, it could make its task easier by changing what it was supposed to prove.

I then built a proof-of-concept GitHub App to review changes through pull requests. That gave me another interface to maintain before I’d managed to formalize a real paper.

<figure class="formalizer-figure formalizer-diagram formalizer-workflow">
<svg class="formalizer-branch" viewBox="0 0 520 270" role="img" aria-labelledby="worktree-title worktree-desc">
<title id="worktree-title">Iterate in isolation. Review before merging.</title>
<desc id="worktree-desc">The approved project continues along the top line. A worktree branches below it, where an agent repeats editing and compiling. A diff goes through checks and human review before merging back into the project.</desc>
<defs><marker id="branch-arrow" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse"><path d="M 1 1 L 8 5 L 1 9" /></marker></defs>
<text x="24" y="28" class="branch-label">APPROVED PROJECT</text>
<path class="branch-main" d="M24 57 H496" />
<path class="branch-route" d="M64 57 C64 115 76 171 92 171" />
<path class="branch-route" d="M427 128 C449 110 456 96 456 78 V65" marker-end="url(#branch-arrow)" />
<circle class="branch-dot" cx="64" cy="57" r="5" /><circle class="branch-dot" cx="456" cy="57" r="5" />
<text x="488" y="28" text-anchor="end" class="branch-label branch-accent">MERGE</text>
<rect class="branch-worktree" x="92" y="113" width="220" height="139" rx="12" />
<text x="110" y="139" class="branch-label">ISOLATED WORKTREE</text>
<text x="123" y="178" class="branch-text">Edit</text><text x="215" y="178" class="branch-text">Compile</text>
<path class="branch-loop" d="M164 171 H201" marker-end="url(#branch-arrow)" />
<path class="branch-loop" d="M251 191 C251 221 139 221 139 200 V189" marker-end="url(#branch-arrow)" />
<text x="195" y="239" text-anchor="middle" class="branch-note">Agent retries</text>
<path class="branch-route" d="M312 171 H350" marker-end="url(#branch-arrow)" />
<rect class="branch-review" x="357" y="128" width="141" height="85" rx="10" />
<text x="427" y="158" text-anchor="middle" class="branch-text">Checks</text>
<text x="427" y="185" text-anchor="middle" class="branch-text">Human review</text>
<text x="334" y="151" text-anchor="middle" class="branch-note">Diff</text>
</svg>
<figcaption>Agents iterate in a worktree; checked, human-approved changes merge back into the project.</figcaption>
</figure>

## Removing most of the application

As my thesis deadline approached, I tried to get the full pipeline working reliably. Managing the context and prompts of many specialized agents had become a problem of its own. Orchestration kept breaking on edge cases.

I removed the web UI, terminal UI, GitHub integration, background jobs, and several agent roles, leaving a small CLI. I also changed statement formalization from separate per-node operations to one project-wide pass: independent agents chose incompatible representations, or needed to read the whole project anyway. Once the shared definitions and statements were in place, proofs could still be attempted individually.

The remaining workflow was:

<figure class="formalizer-figure formalizer-pipeline">
<svg viewBox="0 0 620 72" role="img" aria-labelledby="pipeline-title">
<title id="pipeline-title">Paper → cleaned source → blueprint → Lean skeleton → proofs</title>
<defs><marker id="pipeline-arrow" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto"><path d="M1 1 L8 5 L1 9" /></marker></defs>
<rect x="1" y="10" width="76" height="52" rx="8" />
<rect x="109" y="10" width="116" height="52" rx="8" />
<rect x="257" y="10" width="100" height="52" rx="8" />
<rect x="389" y="10" width="116" height="52" rx="8" />
<rect x="537" y="10" width="82" height="52" rx="8" />
<path class="pipeline-link" d="M85 36 H101" marker-end="url(#pipeline-arrow)" />
<path class="pipeline-link" d="M233 36 H249" marker-end="url(#pipeline-arrow)" />
<path class="pipeline-link" d="M365 36 H381" marker-end="url(#pipeline-arrow)" />
<path class="pipeline-link" d="M513 36 H529" marker-end="url(#pipeline-arrow)" />
<text x="39" y="41">Paper</text>
<text x="167" y="41">Cleaned source</text>
<text x="307" y="41">Blueprint</text>
<text x="447" y="41">Lean skeleton</text>
<text x="578" y="41">Proofs</text>
</svg>
</figure>

It could parse and formalize some toy examples end to end. The surviving test uses a small theorem about two-step paths in a graph on natural numbers. I never completed a real paper, and I never established that the pipeline was meaningfully better than one capable coding agent with access to Lean.

I also wasn’t sure who would use it. The reviewer needed enough mathematical and Lean knowledge to judge the translation. Those users already had ways to work on formalizations; I had not shown why they would prefer mine. At that point, I stopped.

I still think the blueprint was worth exploring. If I restarted, I would first work through a small formalization myself, with a coding agent, and study how existing projects were organized. Only then would I try to automate parts of that work and compare the result against the simpler workflow.
