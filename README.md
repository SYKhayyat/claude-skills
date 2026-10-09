# claude-skills

Skills for [Claude Code](https://claude.com/claude-code), distributed as a plugin
marketplace.

Each is opinionated about one thing. None is a bug finder.

| Plugin | What it does |
|---|---|
| **[lamdan](./lamdan)** | Argues about whether the code should exist and whether this is the way to build it. Design critique, not correctness review. |
| **[dilbert](./dilbert)** | Explains a pull request as a document a non-technical reader can follow end to end — while still being complete enough to review the change from. |
| **[knobs](./knobs)** | Finds the policy the code has already decided but never wrote down, and extracts the values that genuinely vary into typed config. Behavior-preserving. |

## Install

In Claude Code:

```
/plugin marketplace add SYKhayyat/claude-skills
/plugin install lamdan@claude-skills
/plugin install dilbert@claude-skills
/plugin install knobs@claude-skills
```

Install any one on its own; they are independent. The first command registers
this repository as a marketplace — it does not install anything by itself.

### Requirements

- Claude Code with plugin support (`/plugin` must be a command it knows).
- `git`, for all skills.
- The `gh` CLI, authenticated — **dilbert only**, and only for `/dilbert <PR>`
  or a GitHub URL. Branch, commit-range and working-tree modes need only `git`.

## Use

```
/lamdan                    # sweep the whole repo, argue about the design
/lamdan src/ingest         # aim it at a path, a PR, or a spec
/lamdan doc                # write the argument to a file instead of chatting
/lamdan fix                # apply changes instead of reporting
/lamdan exhaustive         # every gap, small ones included

/dilbert 60                # explain PR #60
/dilbert some/branch       # explain a branch against its merge base
/dilbert abc123..def456    # explain a commit range
/dilbert                   # explain the uncommitted working tree

/knobs                    # sweep the repo, ask once, extract the config surface
/knobs services/billing   # aim it at a subtree
/knobs --eu               # target an upcoming environment; reframes the variance test
/knobs --no-deep          # config only; find the deep findings as notes, change none
```

Each skill's `SKILL.md` documents its own switches and failure modes in full.

## Documentation

| | |
|---|---|
| [docs/ONBOARDING.md](./docs/ONBOARDING.md) | Install, first run of each skill, how to read what they produce, what each one refuses to do, and how to hack on them. |
| [docs/TROUBLESHOOTING.md](./docs/TROUBLESHOOTING.md) | Install failures, skills that will not fire, tool-permission problems, `gh` auth, and the per-skill gotchas. |
| [`lamdan/skills/lamdan/SKILL.md`](./lamdan/skills/lamdan/SKILL.md) | The lamdan skill itself. This is the real documentation. |
| [`dilbert/skills/dilbert/SKILL.md`](./dilbert/skills/dilbert/SKILL.md) | The dilbert skill itself. Likewise. |
| [`knobs/skills/knobs/SKILL.md`](./knobs/skills/knobs/SKILL.md) | The knobs skill itself. Likewise. |

## Why these

`lamdan` and `dilbert` sit on either side of a code change.

Before the change, `lamdan` asks the three questions a correctness review never gets to:
was this the right software to build, is this the right architecture for it, and is this
the right implementation inside that. It commits to its own design before reading yours,
then sweeps the codebase region by region rather than following whatever `git log`
happened to surface.

After the change, `dilbert` explains it. Plain words, every term defined before first use,
real code quoted generously — two complete layers in one document, so the non-technical
reader and the reviewer can both work from it.

`knobs` runs whenever the change is about **policy** rather than logic — running this
somewhere new, per-tenant anything, settings scattered across modules. Its hardest part
isn't extracting values, it's refusing to: most hardcoded literals never vary, and each one
promoted to config costs type safety, validation, and a file that can silently disagree
with the code. So it classifies every candidate by what axis it varies along before it asks
anything, asks once instead of per finding, defers to whatever config system the repo already
has, and defaults structural findings to notes rather than refactors. Then it sweeps its own
work.

## Repository layout

```
.claude-plugin/marketplace.json    which plugins, where, and at what version
lamdan/
  .claude-plugin/plugin.json       the plugin's own manifest
  skills/lamdan/SKILL.md           the skill — this is the product
dilbert/
  ...                              the same shape
knobs/
  ...                              the same shape
```

`SKILL.md` is a prose document with YAML front matter. The `description` field
is not decoration: it is what routes an invocation, so editing it changes
whether the skill fires at all.

Both the plugin manifest and the marketplace entry carry a version, and both have
to move together on a release.

## License

MIT — see [LICENSE](./LICENSE).
