---
name: dilbert
description: Explain a pull request, branch, commit range, or uncommitted diff as a document someone non-technical can actually follow — plain words, full context, real code quoted, every term defined before use. Use when asked to explain/walk through/break down a PR, to prepare to review a PR with someone, or when someone says "explain PR N" or invokes /dilbert.
allowed-tools:
  - Read
  - Grep
  - Glob
  - Bash
  - Agent
  - Write
---

# dilbert

Produce a walkthrough document for a code change that a **non-technical reader can follow end to
end**, without losing anything a technical reader needs.

The name is the test. Imagine your company made a bad hire and took on Dilbert's pointy-haired
boss. He is not stupid — he is impatient, he skims, he does not know what a slope is, he has never
heard of a dataclass, and he will nod along while understanding nothing rather than admit he is
lost. **If he falls off, the document is wrong.** That is the bar, and it is a hard one, because
the document still has to be complete enough that the engineer reviewing the PR uses it as their
map.

**This document is also what people review the change from.** It carries the diff inside it —
generously quoted, in order, with nothing load-bearing left out — *without* giving up a word of the
plain-English explanation wrapped around it. Two full layers, one document. See
**The two layers** below; it is the rule that everything else serves.

But it **explains, it does not grade.** No findings, no nitpicks, no verdict, no approve/reject
recommendation. Give the reviewer everything they need to form a judgement, and then let them form
it. Separate review files own the opinions.

---

## The four things that make it work

Everything in this skill serves these. When a rule below seems tedious, it is serving one of these.

### 1. Utter simplicity

One idea per sentence. If a sentence has three clauses held apart by dashes, it is three sentences
wearing a coat. Break it up.

Short paragraphs. A wall of text is a wall he walks away from.

Never write a sentence whose meaning depends on a sentence the reader has not read yet.

### 2. No codewords

A "codeword" is any term that carries meaning only inside this codebase or this discipline. They
are invisible to the person writing them and opaque to everyone else.

- **Every term is defined before its first use.** No exceptions. Not "defined nearby" — defined
  *earlier*, in the glossary.
- **Substitute plain words for code type names**, and give the mapping once in a table:
  `bool` → switch, `categorical` → pick-one, `numeric` → dial. Then use the plain word everywhere.
  The mapping table is what keeps it honest — the reader can always get back to the code.
- **Cap the glossary at about ten words.** If a change seems to need twenty, the explanation is
  wrong, not the change. Find the smaller set of ideas underneath.
- Words that look plain and are not, and must be defined or replaced: *edge, step, net, signed,
  orient, slope, split, weighted, append-only, seam, port, adapter, composition root, acquisition,
  idempotent, hydrate, marshal, upstream, downstream*.
- Delete the ones that add nothing. "UCB or Thompson sampling, and **the difference does not matter
  for this PR**" is a complete and honest sentence. Saying which is which would have cost the
  reader two paragraphs and bought them nothing.

### 3. Comprehensive context

The reader knows nothing. Act like it, every single time, without ever sounding like you are
talking down.

Before *what changed*, always establish:

- What the whole system is for, in one paragraph, in the opening.
- What the file is for, in two sentences, at the head of its section.
- What the function did **before** the change, and why anyone wanted it.
- Why the change matters — what actually goes wrong in the world if it is not made. Money, wasted
  runs, corrupted data, a person misled. Name the consequence.

A bug is never just "this line was wrong". It is a story: what was believed, what was actually
true, what the gap caused. Tell it in that order.

### 4. Quote the real code — enough to review from

**This document is the main document used to review the change.** Not a companion to the diff — the
thing the reviewer actually works from. Someone should be able to read it and legitimately approve
or reject the PR without opening GitHub. That is a higher bar than "illustrate the point with a
snippet", and it is the bar.

So: **quote generously.** When in doubt, quote. A reviewer cannot review code they cannot see.

**Quote actual code. Never paraphrase into pseudocode.** Pseudocode is where bugs hide — it
launders the real thing into the thing you assumed it was.

Coverage rules:

- **Every changed function gets real code.** Not a description of it. Code.
- **Anything whose behaviour changed gets before *and* after.** A reviewer's question is always
  "what was it, and what is it now" — answer both, side by side, labelled *Before* and *After*.
- **Quote deletions.** Removed code is invisible in a summary and is frequently the most important
  part of a change. A reviewer cannot approve the removal of something they never saw.
- **Quote the real comments** when a comment is the argument — especially a comment that admits a
  tradeoff, marks something temporary, or explains a refusal.
- **Anchor everything.** Give `path/to/file.py:120` style references so the reader can jump
  straight there.
- If a hunk is too long to quote whole, quote the load-bearing lines and say plainly what the
  elided part does. Never silently skip a hunk.

The rule that keeps all that code readable by someone who cannot read code:

> **The sentence directly above every code box says the same thing in English.**
> Skip every box and lose nothing.

State that rule explicitly in the document, near the top. Then honour it in every single instance —
a box without its English sentence above it is a defect. This rule is what lets quoting scale: more
code costs the non-technical reader nothing, because for them the code boxes are not there.

Two readers, one document. He reads the sentences. The reviewer reads the boxes.

---

## The two layers — and neither gives ground

This is the rule that everything else hangs off, and it is the easiest one to break by accident.

The document has **two complete layers**, and they are both full-size:

| layer | who it is for | what it is |
|---|---|---|
| **The plain-English layer** | the pointy-haired boss | opening, glossary, story, one-line summary per file, and the English sentence above every code box |
| **The code layer** | the reviewer | quoted before-and-after code, deletions, comments, line anchors |

**Adding the code layer must not shrink the plain-English layer. Not by a sentence.**

The tempting failure is obvious once you name it: told to quote generously, the document quietly
turns into an annotated diff. The prose thins into captions. The glossary shrinks. The opening
becomes a paragraph instead of a page. Each individual step feels reasonable. The result is
something no non-technical person can read, which was the entire reason the document existed.

The opposite failure is the older one: a beautiful essay a reviewer cannot work from.

**Refuse both.** The plain-English layer is written exactly as if no code were being quoted at all —
full opening, full glossary, full story, full reasoning in every file section. Then the code is
added *underneath* it. Not instead of it, not woven through it until neither survives.

Test for it directly:

- **Delete every code box.** Is the document still a complete, readable explanation that stands on
  its own? It must be. If it now reads like captions with the pictures missing, the prose layer got
  eaten — restore it.
- **Read only the code boxes.** Is the change fully visible? It must be.

Both must pass. A document that fails the first is a diff. A document that fails the second is an
essay. This skill produces neither.

---

## Step 1 — Work out what you are explaining

Accepted targets:

| invocation | means |
|---|---|
| `/dilbert 60` or a GitHub URL | that pull request |
| `/dilbert some/branch-name` | that branch against its merge base with the repo's integration branch |
| `/dilbert abc123..def456` | that commit range |
| `/dilbert` with no argument | the uncommitted working tree diff |

**Gotcha that will bite you: `gh pr diff --patch` emits one patch per commit.** Files appear
multiple times and you will describe a design that a later commit replaced. Always get the combined
diff from git instead:

```bash
gh pr view <N> --json number,title,body,headRefName,baseRefName,commits,additions,deletions,changedFiles
git fetch origin <head> <base>
git diff --stat origin/<base>...origin/<head>
git diff origin/<base>...origin/<head> -- <paths>
```

Never switch the user's branch or touch their working tree. Read everything through `git diff`,
`git show <ref>:<path>`, and `gh`.

## Step 2 — Ask where it goes

One question, two parts, before any writing:

1. **Destination path** — propose a sensible default from what the repo looks like, and offer the
   user's `todos/PR-<N>-<slug>/` convention if a folder like that exists. Create the folder if
   needed.
2. **Format** — org mode or markdown.

If org: match the house header style — `#+TITLE:`, `#+DATE:`, `#+AUTHOR:`, `#+LINK:`, `#+STATUS:`,
`* Headings`, `=verbatim=`, `*bold*`, `/italic/`, org tables, `#+begin_src` / `#+begin_quote`.

## Step 3 — Gather, and verify against the source

**Read the actual files at the head of the change, not only the diff.** This is not optional and it
is the step that makes the document trustworthy.

A diff tells you what lines moved. It does not tell you what the code now *is*. And a PR
description tells you what the author intended three commits ago.

Check, and note every mismatch:

- Does the PR body describe the design the code actually implements? On PR #60 it did not — the
  body described a "store it twice, pre-flipped" design that a later commit deleted entirely. A
  reader following the description would have been lost for ten minutes.
- Do the commit messages agree with each other, or does a later one reverse an earlier one?
- Do the code comments describe the code they sit on?
- Are there deleted functions whose removal is the actual point? Deletions are frequently the best
  part of a change and diffs bury them.
- Do the test names match what the tests assert?

Also read the repo's `CLAUDE.md` if it has one, so the document can explain the house rules the
change is obeying rather than guessing at intent.

**Scale:** read sequentially up to about 25 changed files. Above that, fan out subagents over file
groups and synthesise — and when you do, verify any load-bearing claim a subagent makes against the
source yourself before it goes in the document.

## Step 4 — Find the spine before writing a word

Do not start writing until you can finish this sentence:

> **This whole change is really about: ______.**

One idea. Not three. Nearly every change has one, and finding it is most of the work.

Then, if the idea has a physical shape, open the document with it. A light switch with a note under
it that says "flipping this made the room better" — useless, because it never says which way. That
single image carried an entire PR about signed measurements, and it cost the reader no vocabulary
at all.

An analogy is **welcome, not required.** Do not strain for one. A forced analogy is worse than
none, because the reader now has to hold a bad metaphor *and* the real thing. When nothing fits,
open with the bug stated plainly in four sentences instead. What is required is that the opening
section stands alone and is understandable by someone who knows nothing.

Where a change is genuinely several fixes, say so plainly and number them. Do not pretend to a
unity that is not there.

## Step 5 — Write it to this shape

Every section below is required.

**1. `Read this part even if you read nothing else`**
The opening. Analogy if one fits, plain statement if not. What the system does, what went wrong,
what it cost, what the fix is. No jargon at all — not one term. Ends with "everything below is the
same idea in more detail." Someone must be able to read only this and come away genuinely
understanding the change.

**2. `How to read this file`**
Who it is for. The three rules: read the glossary first; every code box is optional because the
sentence above it says the same thing; each file section opens with a one-line summary in bold, so
skimming those alone gives the whole change.

**3. `The glossary`**
About ten words. Every term the document uses. Include the plain-word substitution table for code
type names. Define each by what it *does* and why anyone cares — the router is not "the escalation
component", it is "the thing that decides whether to spend money on an AI round".

**4. `The story`**
What the system does all day, as numbered steps. Then the bug, told as a story with real numbers
and a concrete named example carried all the way through. Then the cost. Then the fix in one
sentence, with its consequences listed.

**5. `Contradictions`** *(only when there are any)*
Where the PR description, commit messages, or comments disagree with the code. Say plainly which
one to trust and why. This section saves the reader the most time of anything in the document.

**6. `Reading order`**
A table: file, what it is, and the order to read them in. Order by dependency, so each file makes
the next one make sense — usually: the data, then the things that write it, then the shared logic,
then the things that read it, then the wiring, then the tests. Close with the one-sentence shape of
the whole change.

**7. One section per file, in that order**

Each opens with a bolded one-line summary, then what this file is in two plain sentences, then a
subsection per changed function. Each of those carries, in this order:

1. What the function did **before**, and why anyone wanted that.
2. The **before** code, quoted, with its `file.py:line` anchor.
3. What changed and why — the reasoning, in plain words.
4. The **after** code, quoted.
5. What that buys, stated as a consequence in the world.

Mark new things as new and deleted things as deleted, and **quote the deleted code** — a reviewer
approving a deletion needs to see what is going away. A removed workaround, quoted alongside its
own apologetic comment, is often the best paragraph in the document.

This is the longest part of the document by far, and it should be. It is the part being reviewed.

**8. `The tests`**
Read the test names as sentences and treat them as the change's own summary of itself. Group them
by what they prove. Call out any deleted test — deleting the tests that pinned an old workaround,
in the commit that removes the workaround, is the correct move and worth naming.

**No review section.** No findings, no nitpicks, no verdict. Separate review files own that job.

### Voice

Dry, plain, occasionally amused. "Almost funny." "The file with the bug in its name barely
changed." "Loudly unsure beats quietly wrong."

The voice earns attention through a long document. It never costs accuracy, never softens a
problem, and never adds a word for the sake of a joke. If charm and clarity pull apart, clarity
wins without hesitation.

Never condescend. Simple is not the same as dumbed down — every fact stays, only the vocabulary
changes.

## Step 6 — Run the Dilbert test, and report what it found

**Mandatory. Do not skip it. Draft one is measurably worse than draft two, every time.**

Re-read the finished document from the top, as him. Not a vibe check — find **specific line
numbers** where he stops understanding. He falls off at:

- A term used before it was defined. Grep your own glossary against the document to catch these.
- Arithmetic he cannot follow. Any division, any negative number, any sign.
- A sentence with more than one idea in it.
- A code box with no English sentence above it.
- A section that assumes he has opened the file.
- Jargon that crept in below the fold, where you stopped being careful.
- Anything that needs him to remember something from eight paragraphs ago.

Fix every one. Then **report the failures in chat**, with line numbers and what you did — the user
wants to see the reasoning, not just a quietly improved file.

### The second audit: are both layers intact?

Two checks, and the document ships only when both pass.

**Check one — delete every code box in your head. Does the document still stand?**
Full opening, full glossary, the story with its real numbers, and reasoning in every file section
that explains itself without the code. If what remains reads like captions with the pictures
missing, the prose layer got eaten by the quoting. Restore it. This check comes first on purpose —
it is the one that gets skipped.

**Check two — read only the code boxes. Is the whole change visible?**
Walk the diff's file list against the document and ask:

- Is every changed file present?
- Is every changed function quoted, not merely described?
- Is every behaviour change shown **before and after**?
- Is every deletion visible?
- Would a reviewer have to open the diff to answer an obvious question?

**If the answer to that last one is yes anywhere, quote more.** The document is not finished while
a reviewer still needs the other screen.

Then say honestly where he still cannot follow, and why that is acceptable. The deep file sections
are written for the engineer; what matters is that he never *silently* falls off with a floor still
under him.

---

## Quick reference

- Combined diff via `git diff base...head`, never `gh pr diff --patch`
- Read the source at head, not just the diff — descriptions go stale, code does not
- Read the repo's `CLAUDE.md`
- Never switch branches or touch the working tree
- Ask destination and format up front, in one question
- Every term defined before first use; about ten words; plain-word substitutes with a mapping table
- **Two full layers, neither shrinks.** The plain-English layer is written as if no code were being
  quoted; the code is added underneath it, never instead of it
- **This is also the document people review from** — quote generously, before and after, deletions
  included, with `file.py:line` anchors. If a reviewer would need the diff, quote more
- Every code box has its English sentence directly above it — that is what lets the quoting scale
- Delete every box: still a complete explanation? Read only the boxes: whole change visible? Both
- Explain, never grade. No findings, no verdict
- One-line bold summary at the head of every file section
- Contradictions get their own section and a "trust the code" verdict
- Tests read as sentences
- Two mandatory audits: can he follow it, and can it be reviewed from. Findings reported in chat
