# Onboarding

Installing the two skills, running each one for the first time, and knowing what
you are looking at when it answers.

Ten minutes, most of it reading output.

---

## Contents

- [1. Install](#1-install)
- [2. Verify](#2-verify)
- [3. First run: dilbert](#3-first-run-dilbert)
- [4. First run: lamdan](#4-first-run-lamdan)
- [5. What each one refuses to do](#5-what-each-one-refuses-to-do)
- [6. Where they fit around a change](#6-where-they-fit-around-a-change)
- [7. Hacking on them](#7-hacking-on-them)

---

## 1. Install

In Claude Code:

```
/plugin marketplace add SYKhayyat/claude-skills
/plugin install lamdan@claude-skills
/plugin install dilbert@claude-skills
```

They are independent. Install either one on its own.

The first command registers this repository as a plugin marketplace; the second
and third install from it. Adding the marketplace does not install anything by
itself.

### Requirements

| | |
|---|---|
| Claude Code | With plugin support. `/plugin` must be a command your build knows. |
| `git` | Both skills read history and diffs. |
| `gh` CLI, authenticated | **dilbert only**, and only for `/dilbert <PR number>` or a GitHub URL. Branch, commit-range and working-tree modes need only `git`. |

Both skills declare `Read`, `Grep`, `Glob`, `Bash`, `Agent` and `Write` in
`allowed-tools`. If your setup restricts any of those, see
[TROUBLESHOOTING](TROUBLESHOOTING.md#tool-permissions).

## 2. Verify

```
/plugin
```

You should see `lamdan` and `dilbert` listed as installed. If the marketplace
added but the plugins do not appear, see
[TROUBLESHOOTING](TROUBLESHOOTING.md#installation).

Then just type `/lamdan` or `/dilbert` — if the slash command completes, you are
done.

## 3. First run: dilbert

Start here rather than with lamdan. It is the gentler of the two and it produces
something you can immediately judge.

Point it at a change you already understand:

```
/dilbert                    # the uncommitted working tree diff
```

Or a real one:

| You type | It explains |
|---|---|
| `/dilbert 60` or a GitHub URL | that pull request |
| `/dilbert some/branch-name` | that branch against its merge base with the repo's integration branch |
| `/dilbert abc123..def456` | that commit range |
| `/dilbert` | the uncommitted working tree diff |

### What you get, and why it looks like that

A walkthrough document a **non-technical reader can follow end to end**, that is
still complete enough for the engineer reviewing the PR to use as their map.
Two full layers in one document — that is the constraint everything else in the
skill serves.

The name is the test. The imagined reader is Dilbert's pointy-haired boss: not
stupid, but impatient, a skimmer, does not know what a slope is, has never heard
of a dataclass, and will nod along while understanding nothing rather than admit
he is lost. If he falls off, the document is wrong.

Four rules produce the style, and they are worth knowing so the output does not
read as dumbed-down when it is actually disciplined:

1. **Utter simplicity.** One idea per sentence. No sentence whose meaning depends
   on a sentence the reader has not reached yet.
2. **No codewords.** Every term defined before its first use — *earlier*, in the
   glossary, not "nearby". Code type names get plain-word substitutes with the
   mapping given once in a table, so the reader can always get back to the code.
   The glossary is capped at about ten words: if a change seems to need twenty,
   the explanation is wrong, not the change.
3. **Real code, quoted generously**, in order, with nothing load-bearing left
   out — including the comments, when a comment is the argument.
4. **It explains, it does not grade.** No findings, no nitpicks, no verdict, no
   approve/reject. It gives the reviewer everything needed to form a judgement
   and then lets them form it.

### The one gotcha to know up front

`gh pr diff --patch` emits **one patch per commit**, so files appear repeatedly
and a reader following along sees the same code three times with different
context. The skill uses a combined diff (`git diff base...head`) instead. If you
are adapting the skill, do not "simplify" that back.

## 4. First run: lamdan

```
/lamdan                     # sweep the whole repo, argue about the design
/lamdan src/ingest          # aim it at a path, a PR, or a spec
```

Aim it at something you are about to change, not something you are proud of.
The output is an argument, and an argument about code you have no intention of
touching is a waste of both of you.

### What it is, and what it is not

Not a bug hunt. `/code-review` and `/investigate` own that, and they are the
easy case, because the code declares its intent and you check whether it
delivers.

**Lamdan argues with the intent.** Should this exist? Would you have built it
this way? Is the want it serves even the real want?

Three questions, asked separately every time:

1. Was this the right software to build?
2. Is this the right architecture for it?
3. Is this the right implementation inside that?

Every finding is tagged with which lens produced it, and a finished argument has
taken a position on all three.

### Two things about the method

**It commits before it looks.** It reads only far enough to say what a human
wanted — README, entry points, the manifest, route table, CLI surface, test
names — and then states out loud what it would have built for that want, *before
opening any implementation*. This is a discipline, not a deliverable. Once
implementation code is in context, it becomes the problem statement, and every
judgment after that is rationalization wearing a review's clothing.

**It sweeps region by region.** Not "wherever `git log` pointed". History is
used to *rank* findings, not to choose where to look. Aiming it at a path
narrows the sweep; it does not license skimming what is left in scope.

### It will have a temperature, and that is deliberate

The reference point in the skill is Stallman finding Emacs rewritten in C: not
annoyed, not "raising a concern" — offended, because he can tell you in exact
detail what was thrown away and what it will cost.

So no "consider whether", no "it might be worth", no "there may be an
opportunity to". If it believes an abstraction is worthless, the sentence is
that the abstraction is worthless, and here is the line where it costs you.

The heat is aimed at the artifact and the reasoning, never at the person — who
is usually a competent engineer under a deadline. And the inverse rule is
enforced just as hard: certainty is only for things it checked. "I don't know, I
read this in five minutes" and "I was wrong" are both fully compatible with
conviction. Confident vagueness is not.

### Push back

**This is a conversation, not a report.** The whole value is in the
disagreement, live, about code you are touching today. Argue with it.

It is built to concede to *evidence* and not to pushback — repeating yourself
more firmly is not an argument, and being told it is wrong is not evidence that
it is. It keeps arguing while it still has new arguments, not for a fixed number
of rounds.

### The switches

| Default | Switch |
|---|---|
| Whole repo, swept region by region | `/lamdan <path\|PR\|spec>` to aim it |
| Argues in chat | `/lamdan doc` writes it to `lamdan/<slug>-<date>.md` |
| Reports, does not edit | `/lamdan fix` applies changes |
| The strongest claims, ranked by wrongness × cost of leaving it | `/lamdan exhaustive` reports every gap, small ones included |

Use `doc` when you want the argument on file for someone else. Use chat when you
want to win it.

The "reports, doesn't edit" default is not timidity: these decisions carry
business context the skill does not have.

## 5. What each one refuses to do

Knowing the refusals saves you from asking for the wrong thing.

**dilbert will not grade.** No verdict, no nitpicks, no approve/reject. If you
want an opinion on a PR, that is a different tool.

**lamdan will not hunt bugs.** It is not checking whether the code does what it
says; it is arguing about whether what it says is worth doing. Use
`/code-review` for correctness.

**lamdan will not edit unless you ask.** `/lamdan fix` exists; the default does
not touch your tree.

**Neither is a linter.** Both are expensive, both produce prose, and both are
worth running at a decision point rather than on every commit.

## 6. Where they fit around a change

They sit on either side of one.

**Before** — `lamdan` asks the three questions a correctness review never gets
to. Run it when you are about to build something, or about to change something
you did not write.

**After** — `dilbert` explains what happened, in a document the non-technical
stakeholder and the reviewing engineer can both work from.

Nothing stops you running `dilbert` on your own branch before you open the PR.
It is a good way to find out that a change you thought was simple takes eleven
paragraphs to explain.

## 7. Hacking on them

The repository is small and there is no build step.

```
.claude-plugin/marketplace.json    the marketplace: which plugins, where, what version
lamdan/
  .claude-plugin/plugin.json       this plugin's own manifest
  README.md
  skills/lamdan/SKILL.md           the skill itself — this is the whole product
dilbert/
  ... the same shape
```

**`SKILL.md` is the product.** Both are long prose documents with YAML front
matter, and the front matter matters:

```yaml
---
name: lamdan
description: <what it does and when to use it — this is what routes invocations>
allowed-tools:
  - Read
  - Grep
  - Glob
  - Bash
  - Agent
  - Write
---
```

The `description` is not decoration. It is how Claude Code decides whether a
given request should invoke the skill, so a vague description produces a skill
that never fires or fires constantly.

### Testing a change

Install from your local checkout rather than from GitHub, and re-run against a
repository you know well. There is no test suite — the output is prose, and the
only check is reading it.

Two things worth checking every time you edit a `SKILL.md`:

- **Does it still fire?** Change the `description` and it may stop routing.
- **Did the rules survive contact?** These skills are mostly constraints. It is
  easy to add a paragraph that quietly contradicts one three sections up.

### Versions

Two files carry a version and both must move:

- `<plugin>/.claude-plugin/plugin.json` — the plugin's own version.
- `.claude-plugin/marketplace.json` — the version the marketplace advertises for
  that plugin.

They have drifted apart before. If a user reports an old version after
installing, this is why.

---

## Where to go next

- [../README.md](../README.md) — the summary of both.
- [TROUBLESHOOTING.md](TROUBLESHOOTING.md) — install problems, skills not
  firing, `gh` auth, tool permissions.
- [`../lamdan/skills/lamdan/SKILL.md`](../lamdan/skills/lamdan/SKILL.md) and
  [`../dilbert/skills/dilbert/SKILL.md`](../dilbert/skills/dilbert/SKILL.md) —
  each documents its own switches and failure modes in full. They are the real
  documentation; everything here is a way in.
