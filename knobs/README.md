# knobs

Policy-as-metadata for Claude Code. Finds the decisions a codebase has already
made but never written down — hardcoded limits, endpoints, rates, budgets,
structural assumptions that vary by environment — and extracts the ones the owner
picks into config the code reads at runtime.

See [`skills/knobs/SKILL.md`](./skills/knobs/SKILL.md) for the full skill.

```
/plugin marketplace add SYKhayyat/claude-skills
/plugin install knobs@claude-skills
```

Two passes: extract, then verify. The first pass inventories candidates with
`file:line` evidence and asks about all of them **once**; the second pass re-reads
what was changed and checks it against what was claimed.

For install problems, the switches, or what it refuses to do, see
[docs/ONBOARDING.md](../docs/ONBOARDING.md) and
[docs/TROUBLESHOOTING.md](../docs/TROUBLESHOOTING.md).