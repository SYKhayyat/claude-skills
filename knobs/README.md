# knobs

Config extraction for Claude Code. Finds the policy your code has already decided but never
wrote down, and moves the values that genuinely vary into typed metadata the code reads at
runtime — without changing behavior.

See [`skills/knobs/SKILL.md`](./skills/knobs/SKILL.md) for the full skill.

```
/plugin marketplace add SYKhayyat/claude-skills
/plugin install knobs@claude-skills
```

For install problems, the switches, or why it refuses to turn most literals into knobs, see
[docs/ONBOARDING.md](../docs/ONBOARDING.md) and
[docs/TROUBLESHOOTING.md](../docs/TROUBLESHOOTING.md).