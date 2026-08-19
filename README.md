# claude-skills

Two skills for [Claude Code](https://claude.com/claude-code), distributed as a plugin
marketplace.

Both are opinionated about one thing each. Neither is a bug finder.

| Plugin | What it does |
|---|---|
| **[lamdan](./lamdan)** | Argues about whether the code should exist and whether this is the way to build it. Design critique, not correctness review. |
| **[dilbert](./dilbert)** | Explains a pull request as a document a non-technical reader can follow end to end — while still being complete enough to review the change from. |

## Install

In Claude Code:

```
/plugin marketplace add SYKhayyat/claude-skills
/plugin install lamdan@claude-skills
/plugin install dilbert@claude-skills
```

Install either one on its own; they are independent.

## Use

```
/lamdan                    # sweep the whole repo, argue about the design
/lamdan src/ingest         # aim it at a path, a PR, or a spec
/lamdan doc                # write the argument to a file instead of chatting

/dilbert 60                # explain PR #60
/dilbert some/branch       # explain a branch against its merge base
/dilbert abc123..def456    # explain a commit range
/dilbert                   # explain the uncommitted working tree
```

Each skill's `SKILL.md` documents its own switches and failure modes in full.

## Why these two

`lamdan` and `dilbert` sit on either side of a code change.

Before the change, `lamdan` asks the three questions a correctness review never gets to:
was this the right software to build, is this the right architecture for it, and is this
the right implementation inside that. It commits to its own design before reading yours,
then sweeps the codebase region by region rather than following whatever `git log`
happened to surface.

After the change, `dilbert` explains it. Plain words, every term defined before first use,
real code quoted generously — two complete layers in one document, so the non-technical
reader and the reviewer can both work from it.

## License

MIT — see [LICENSE](./LICENSE).
