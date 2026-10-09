# Troubleshooting

Symptom first.

---

## Contents

- [Installation](#installation)
- [The skill does not fire](#the-skill-does-not-fire)
- [Tool permissions](#tool-permissions)
- [dilbert](#dilbert)
- [lamdan](#lamdan)
- [knobs](#knobs)
- [Updating and versions](#updating-and-versions)
- [Developing a change](#developing-a-change)

---

## Installation

### `/plugin` is not a command

Your Claude Code build does not have plugin support, or it is behind a flag.
Update Claude Code first. Nothing else on this page will work until `/plugin`
does.

### `/plugin marketplace add SYKhayyat/claude-skills` fails

Check, in this order:

1. **Network and GitHub reachability.** The marketplace is fetched from GitHub.
   ```sh
   git ls-remote https://github.com/SYKhayyat/claude-skills
   ```
2. **The spelling.** It is `SYKhayyat/claude-skills` — owner then repository, no
   `https://`, no `.git`.
3. **Whether it is already added.** Adding a marketplace twice is not always a
   no-op in every build. `/plugin` lists what is registered.

### The marketplace added but no plugins appear

Adding a marketplace registers it; it does not install anything. One more
command per skill:

```
/plugin install lamdan@claude-skills
/plugin install dilbert@claude-skills
/plugin install knobs@claude-skills
```

The `@claude-skills` suffix names the marketplace. Without it, Claude Code does
not know where to look.

### `/plugin install` says the plugin is not found

The names are `lamdan`, `dilbert` and `knobs`, lowercase. They are the `name` fields in
`.claude-plugin/marketplace.json`, not the directory names — which happen to
match, but the manifest is what is read.

If the marketplace was added at an older commit, refresh it. Removing and
re-adding it is the blunt fix.

### It installed but nothing changed

Restart Claude Code, or start a new session. Skills are read at session start in
most builds.

## The skill does not fire

### `/lamdan`, `/dilbert` or `/knobs` is not recognised

```
/plugin
```

If the plugin is not listed as installed, go back to
[Installation](#installation). If it is listed, restart the session.

### It is installed and the slash command works, but nothing happens

All skills open by reading the repository, and all can spend a while before
producing text — lamdan and knobs sweep region by region, dilbert reads a whole
diff. "Nothing happening" for a minute on a large repository is expected. knobs
is the slowest of the three, and it should ask you for scope before it starts.

If it returns immediately with a short generic answer, the skill did not
actually load. That is the same symptom as the next entry.

### Claude answers in its own voice instead of the skill's

The skill did not route. Two causes:

1. **You described the task instead of invoking it.** Type `/lamdan`,
   `/dilbert` or `/knobs` explicitly rather than asking for "a design review".
2. **The `description` front matter was edited.** In a local checkout, the
   `description` field in `SKILL.md` is what Claude Code matches against to
   decide whether a request should invoke the skill. Vague descriptions produce
   skills that never fire, or that fire constantly.

### More than one skill fires when I only wanted one

They have deliberately different jobs — one argues about design, one explains a
change, one extracts config — so overlap usually means the request was ambiguous.
Invoke the one you want by name.

## Tool permissions

All three declare `Read`, `Grep`, `Glob`, `Bash`, `Agent` and `Write` (knobs also
declares `Edit`, because it writes code changes):

```yaml
allowed-tools:
  - Read
  - Grep
  - Glob
  - Bash
  - Agent
  - Edit   # knobs only
  - Write
```

### The skill stops and asks for permission repeatedly

Expected in a restrictive permission mode — all of them read widely and shell out
to `git`. Allow the read-only tools (`Read`, `Grep`, `Glob`) and `Bash` for
`git`, or run in a mode that does not prompt per call.

### The skill cannot run `git` or `gh`

`Bash` is denied, or the binary is not on the `PATH` Claude Code inherits.

```sh
git --version
gh --version
```

Run those in the same environment Claude Code is running in — a GUI-launched
session often has a different `PATH` than your shell.

### `Write` is denied and `/lamdan doc` produced nothing

`/lamdan doc` writes to `lamdan/<slug>-<date>.md`. Without `Write` it cannot.
Use the default chat mode instead, or allow `Write`.

### `Agent` is denied

All three skills can delegate. With `Agent` denied they will do the work inline,
which is slower and, on a large repository, may run out of room before the sweep
finishes. If lamdan's coverage looks thin on a big codebase, check this first.

## dilbert

### `/dilbert 60` fails to find the pull request

PR mode needs the `gh` CLI, authenticated, with access to that repository.

```sh
gh auth status
gh pr view 60
```

If `gh pr view` fails, so will the skill. Branch, commit-range and working-tree
modes need only `git` — use one of those if `gh` is not available:

```
/dilbert some/branch-name
/dilbert abc123..def456
/dilbert
```

### The same files appear several times in the output

That is the failure mode of `gh pr diff --patch`, which emits **one patch per
commit** — so a file touched in three commits appears three times, with
different context each time, and a reader following along sees the same code
repeatedly.

The skill uses a combined diff (`git diff base...head`) specifically to avoid
this. If you are seeing it, either an adapted copy of the skill reintroduced
`--patch`, or the change genuinely touches the same file in ways that all need
showing.

### `/dilbert some/branch` diffs against the wrong base

Branch mode diffs against the merge base with the repository's **integration
branch**. If your repository's integration branch is not what the skill inferred
— an unusual default branch name, or a fork whose upstream differs — use an
explicit commit range instead:

```
/dilbert $(git merge-base main my-branch)..my-branch
```

### `/dilbert` with no arguments says there is nothing to explain

No-argument mode explains the **uncommitted working tree diff**. A clean tree
has nothing to explain. Stage or make a change, or name a PR, branch or range.

### The document is too long

That is usually the change, not the document. dilbert quotes real code
generously and in order, and it carries two full layers — the plain explanation
and the reviewable diff — deliberately.

If the glossary has run past about ten words, that is the skill's own signal
that the explanation is wrong rather than the change: there is a smaller set of
ideas underneath that has not been found yet. Say so and ask it to find them.

### It graded the change and I did not want that

It should not have — dilbert explains, it does not grade. No findings, no
nitpicks, no verdict, no approve/reject. If it produced a recommendation, say so
and point at the rule; the skill is explicit about it.

If you *want* a verdict, that is a different tool.

## lamdan

### It found nothing, or everything is minor

Two likely causes:

- **It is aimed at code nobody is about to change.** The value is an argument
  about a decision that is still live. Aim it at what you are about to touch.
- **`Agent` is denied and the sweep ran out of room** on a large repository. See
  [Tool permissions](#agent-is-denied).

`/lamdan exhaustive` reports every gap including small ones, if that is what you
want. The default deliberately ranks and truncates.

### It only looked at recently-changed files

It should not. The sweep is region by region across the codebase; git history is
used to **rank** findings, not to choose where to look. If the findings all
cluster around recent commits on a repository where that is not where the
problems are, that is worth reporting.

Aiming it (`/lamdan src/ingest`) narrows the sweep — but narrowing is not
licence to skim what is left in scope.

### It is too harsh

The heat is deliberate and it is aimed at the artifact and the reasoning, never
at the person. If it has aimed at a person, that is a bug in the output and
worth saying so directly — the skill has an explicit rule about it.

If the heat is unearned — adjectives doing the work reasons should be doing —
that is also a stated failure mode. Ask for the argument without the
intensifiers. The skill's own test is that cutting every intensifier should
leave the paragraph *stronger*; if it gets weaker, there was no argument.

### It will not back down

By design. It concedes to **evidence**, not to pushback: repeating a position
more firmly is not an argument, and being told it is wrong is not evidence that
it is. It keeps going while it still has new arguments rather than for a fixed
number of rounds.

If you have given it evidence and it still will not move, quote the evidence
back and ask which specific claim it contradicts.

### I wanted it to fix things

```
/lamdan fix
```

The default reports without editing, deliberately — these decisions carry
business context the skill does not have.

### `/lamdan doc` wrote to an unexpected path

It writes to `lamdan/<slug>-<date>.md`, relative to the repository root, taking
the date from `date +%F`. If your repository already has a `lamdan/` directory
for something else, that is a collision worth knowing about.

### It is hunting bugs instead of arguing about design

Wrong skill for that, and lamdan says so itself. `/code-review` and
`/investigate` own correctness. Lamdan argues with the *intent*: should this
exist, would you have built it this way, is the want it serves the real want.

## knobs

### It found a lot of knobs and I expected fewer

Working as designed. The largest category is **never varies** — constants that
are the same everywhere forever — and the skill argues against extracting those.
If it still promoted values you consider structural, say which, and pull them.

### It wants to refactor and I only wanted config

Use `/knobs --no-deep`, which skips the structural sweep entirely. Deep findings
need approval one at a time by design; the flag means you never see them.

### It added a config file to a repo that already had one

It should not have — the skill is instructed to extend the existing config
system and to report what it found before asking. If it added a parallel file
anyway, that is a bug worth reporting with the repo name.

### It extracted values I did not ask for

The default is one bulk question with a recommendation per row, not a prompt per
finding. Re-read the table it showed you: it changed what it recommended as
"change". You can answer "none" or name individual rows next time. Use
`/knobs --notes-only` when you want the analysis with no writes at all.

### The diff is enormous

Each knob is its own commit, so revert them individually rather than resetting.
If a single knob's change spans many files, the finding was probably classified
deep and should have waited for approval.

### It changed behavior

It should not have. The invariant is that the new default equals the old literal
exactly, and the skill verifies that. If behavior differs, that is a bug — report
it with the key and the two values.

## Updating and versions

### I installed but I have an old version

Two files carry a version, and they have drifted apart before:

- `<plugin>/.claude-plugin/plugin.json` — the plugin's own version.
- `.claude-plugin/marketplace.json` — the version the marketplace advertises.

If the number you see does not match the number in the repository, that is the
likely reason.

### How do I update?

Refresh the marketplace and reinstall:

```
/plugin marketplace add SYKhayyat/claude-skills
/plugin install lamdan@claude-skills
```

If your build has an explicit update or remove command, prefer it.

## Developing a change

### I edited SKILL.md and nothing changed

Reinstall from your local checkout and start a new session. Skills are read at
session start.

### After editing, the skill stopped firing

You changed the `description` in the front matter. That field is what routes
invocations — it is not a summary for humans. Restore something that names both
what the skill does and when to use it.

### There is no test suite

Correct. The output is prose and the only check is reading it. Two things worth
doing on every `SKILL.md` edit:

1. **Confirm it still fires** on a request it used to handle.
2. **Re-read for self-contradiction.** These skills are mostly constraints, and
   it is easy to add a paragraph that quietly contradicts a rule three sections
   above it. That is the most common defect here and nothing automated will
   catch it.

---

## Reporting something not on this page

Include which skill, the exact invocation, your Claude Code version, and what
`/plugin` lists. For dilbert PR mode, add `gh auth status`.
