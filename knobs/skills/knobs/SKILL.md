---
name: knobs
description: Finds the policy the code has already decided but never wrote down — hardcoded values, limits, endpoints, rates, and structural assumptions that vary by environment, tenant, or time — and turns the chosen ones into metadata the code reads at runtime. Asks scope, then asks once (not per finding), then proposes a format that defers to whatever the repo already uses. Use when you want a config surface extracted, before deploying somewhere new, when settings are scattered across the codebase, or when you want to know what would have to change to run this in prod/staging/a second customer. Two passes: extract, then verify.
allowed-tools:
  - Read
  - Grep
  - Glob
  - Bash
  - Agent
  - Edit
  - Write
---

# Knobs

A codebase is a machine that has already made every decision it will ever make. Some of
those decisions are policy — *the retry budget is 3*, *one tenant may hold 500 seats*, *the
ingest endpoint lives here* — and policy that lives in code is invisible to operators,
unchangeable without a deploy, and indistinguishable from a bug once it's wrong.

**Policy as metadata:** the decision moves out of the statement and into data the code reads
at startup. The behavior doesn't change. The *authority* to change it does.

This skill finds those decisions, sorts the ones that genuinely vary from the ones that
don't, asks the user about them **once**, extracts the selected ones behind a typed
contract with code-side defaults, then runs a second pass over the result to check its own
work.

**This is not "replace magic numbers with constants."** A named constant is still a hardcoded
policy. A knob is a value the deployer can move without a code change. If it can't be moved
without a deploy, it isn't a knob, and the skill that promotes it is just spreading config
across files.

---

## Invocation

```
/knobs                    # whole repo
/knobs services/billing   # a subtree or glob
/knobs --eu               # target an upcoming environment
/knobs --no-deep          # config only, no structural findings
```

Flags map onto the questions in §2 and §5. **They answer a question, they never skip the
test** — `--eu` sets the environment target the variance test runs against, and every finding
still has to survive it on its own evidence.

| Flag | Effect |
|---|---|
| a path or glob | Scopes the sweep. Unscoped means the whole tracked tree. |
| `--env <name>` / `--eu` / `--prod` | Names the target environment. Reframes the variance test — a value that never varies *today* may be environment variance *tomorrow*, and this is the flag that says so. |
| `--no-deep` | Skips lens B entirely (§5's "don't search"). |
| `--notes-only` | Finds and reports; changes nothing. The variance test and `KNOBS.md` still get written. |
| `--verify` | Runs only the second pass (§9), against an extraction that's already committed. |

Anything not given as a flag, ask. `--no-deep` and a path are the two most useful.

---

## 0. A knob is not free

Every knob is a trade: you give up a compile-time constant and get a runtime value that can
be wrong. The bill arrives later, and it's always the same bill —

- **type safety.** `retries` was an `int`; now it's a string someone has to parse.
- **the disagreement file.** The config says `30`, the code was written assuming `60`, and
  nothing crashes. The file is not wrong, it's *consistent*, which is worse.
- **silent fallback.** A parse that swallows bad input turns a typo into a mysterious
  production incident three weeks later.
- **the sprawl.** Two hundred keys, no owner, no way to tell which are live.
- **the lie of documentation.** A comment saying "default 3" that drifts from the real default.

**You pay all of that only when the variance is real.** So the variance test gates
everything, before any question is asked. A hardcoded `60` that is `60` in every environment
and every tenant has zero variance, and zero variance means the config buys the deployer
nothing while the codebase pays full price. That's the trade this skill exists to refuse.

---

## 1. The variance test

Every candidate gets classified on **what axis it varies along**. This classification, not
the finding's severity or how ugly the literal looks, decides its fate.

| Axis                | Examples                                              | Verdict                                             |
| ------------------- | ----------------------------------------------------- | --------------------------------------------------- |
| **Environment**     | ports, hosts, bucket names, endpoints, DSNs, API keys, per-env feature flags, log levels | **Knob.** Highest confidence. Deploy-shaped policy that operators change without you. |
| **Tenant/customer** | seat caps, rate limits, quotas, price tiers, overage thresholds, feature entitlements | **Knob.** Where policy-as-metadata earns its keep — a value that must differ per account can't live in code at all. |
| **Mode / time**     | debug, dry-run, verbosity, sampling rate, timeout, cache TTL | **Usually not the file.** CLI flag or env var. These change per invocation, not per deploy, and a properties file is the wrong lifetime. Argue it if you think one belongs in the file. |
| **Never varies**    | page size, days-in-week, buffer size, algorithmic tuning, domain rules written as bare numbers | **Not a knob.** Named constant, or leave it alone. Zero variance, full cost. |

The last row is the one that keeps this skill from being vandalism, and it needs defending,
not just asserting: *"page size is a policy decision."* It isn't — it's a value that's `25`
everywhere forever, so it's a **constant with a bad name**, and the fix is `const PAGE_SIZE = 25`
at the top of the file. Turning it into config would mean a deploy to change something that
has never changed, in exchange for a file that can now disagree with the code about it.

**The load-bearing question is not "could this vary?" — every value could. It's "does it
already vary, or demonstrably will?"** Evidence for real variance:

- multiple config files, `.env.example`s, or deployment manifests already setting the same concept differently per environment
- the same literal implemented twice with different values, in two services or two branches
- comments, tickets, TODOs, or commit messages that say "this is different in prod" / "hardcoded for now" / "move to config"
- CI/CD matrices, Dockerfile `ARG`s, k8s manifests, Terraform variables
- `if (env === 'production')` branches near the literal

**Loudness is not evidence.** A big ugly magic number that appears once, reads clearly, and
has one obvious correct value is a fine constant. A quiet `3` in an env branch is the real
find. Scan for variance signals; don't grade on how much the literal offends you.

**Secrets are their own category.** A hardcoded credential gets flagged **loudly** and routed
to whatever secret mechanism the repo already uses (env, vault, secret manager). It never
gets written into a committed config file as a knob, and it never gets a default in code.
It is reported, not extracted. See §7.

---

## 2. Preflight: scope and conventions

Before reading a line of implementation, settle these — and settle them by asking, not by
guessing.

**Ask for scope, first question of the run:**

- whole repo, or a subtree/glob (`services/billing/**`, the ingest path)
- which environment or customer is the target, if there is one — that reframes the variance
  test sharply. "We need to run this in the EU next month" turns a lot of never-varied
  values into environment variance, and that's a different skill run than "general audit"

Then check the ground:

```bash
git status --porcelain          # must be clean, or stop and say why
git log --oneline -5
git ls-files | wc -l            # size of the promise
tokei || cloc --by-file .       # language mix, if present
```

**And capture a green baseline before touching anything** (see §8) — the test and lint
commands from the repo's README / manifest / `AGENTS.md`, run now. You need to know which
failures are yours, and the only way to know is to have run them before the run starts.
Record the exact command and its result; a pre-existing failure reported after your diff
looks like your diff.

**Detect existing config machinery, and report it before asking anything.** This is the
single highest-leverage step in the skill:

```bash
git ls-files | grep -iE '\.(ya?ml|json|toml|ini|properties|conf|env|config)$|config|settings|\.env'
grep -rlEi 'os\.environ|process\.env|viper|dotenv|pydantic.*settings|@ConfigurationProperties|@Value|getenv|env\(|configparser|figment|koanf' --include='*' . | head -40
```

Whatever comes back is the answer to §6's question. **If the repo already has a settings
module, a `.env` loader, a `Config` dataclass, or an `application.yml` — that is the format.
Not a better one.** Adding a parallel properties file next to a working config system
produces two sources of truth, and the new one will be the one nobody opens.

Note what you found, explicitly, in the run's output:

> Existing config: `app/settings.py` with a pydantic `Settings` class, `BaseSettings`,
> 14 fields, loaded from env with a `.env` file in dev. No YAML.

That sentence saves the user from making a decision they didn't know they were facing.

---

## 3. Sweep: shallow and deep, by region

Coverage first, signals second. Partition the tree into **5–12 regions** describable in one
sentence each — "everything that talks to Stripe", "the ingest path", "the CLI surface". One
subagent per region, fresh context, so no region's conventions make another's look normal.
Exclude vendored deps, lockfiles, generated code, build output, fixtures — and **say out loud
what you excluded** so "I scanned the codebase" stays a true sentence.

Every region reports both lenses. A region that finds nothing says so *and names what it
rejected and why*.

### Lens A — shallow

Policy values reachable by a local edit: literals, constants, inline defaults, env branches,
per-tenant `if`s. For each candidate report:

- `file:line`, the current value, its type
- the **variance axis** from §1, with the evidence for it
- proposed key name (domain-derived — see §7)
- one-line blast radius: what breaks if this value is wrong
- secret? (flag, don't extract)

### Lens B — deep

Policy welded into **structure**: a hardcoded tenant list, a `switch` that is really a policy
table, a rate limit enforced in three unrelated places, a provider chosen at import time, an
algorithm selected by a literal. These need code structure to move, so the cost of being
wrong is higher.

For each: `file:line`, the policy being encoded, which of four shapes it is —

| Shape                     | The knob would be                                             |
| ------------------------- | ------------------------------------------------------------- |
| **parameter threading**   | a value passed down through N call layers                        |
| **table / dispatch**      | a `switch` or `if/elif` chain replaced by a data lookup         |
| **strategy / injection**  | a dependency chosen at import time or hardcoded in a constructor |
| **extraction**            | a block of policy logic lifted into its own named unit           |

— plus how many files the change touches, and what you'd have to verify afterward.

**Depth is the expensive direction.** A wrong shallow knob is a bad default. A wrong deep
refactor is a broken system. That's why §5 defaults deep to *note*.

---

## 4. Present once, ranked

**One question. Not one per finding.** Asking the user to adjudicate 200 items one at a time
is how this skill dies on a real repo — the user abandons it at item nine and the work is
worthless. Interaction cost has to be sub-linear in findings, or the answer is always "no."

Lay out the full inventory as a table, sorted by **variance confidence × blast radius**, with
your recommendation per row already filled in. Then ask once:

> **Shallow knobs.** I found 47 candidates. My recommendation:
> - **change (31)** — high-confidence environment/tenant variance
> - **note (11)** — variance is real but the value is still moving in code; here's the file
> - **skip (5)** — never varies, listed so you can overrule me
>
> Reply with any changes — "all", "none", "only the tenant ones", "skip 23" — or name
> individual rows.

Rank so the top of the table is what actually matters. The user should be able to read the
first ten rows and make a correct decision; everything below is available if they want it.

The **note** option is a real deliverable, not a consolation prize: a `KNOBS.md` (or
`docs/knobs.md`) with `file:line`, current value, variance evidence, and proposed key. Many
users will prefer the notes to the extraction, and that run is still worth having.

**This one question decides all shallow candidates.** The rules for whichever ones get
changed are non-negotiable and enforced per knob, not per file — see §7.

---

## 5. Deep: three ways, default is the safe one

Asked once, as its own question, because the answer isn't per-item:

| Option         | What happens                                                                    |
| -------------- | ------------------------------------------------------------------------------- |
| **don't search** | Skip lens B entirely. Right answer when the codebase is small, when the user knows the one region, or when they've said "no refactors, I just want config." |
| **note** *(default)* | Report deep findings as plans in `KNOBS.md`: the policy, the shape from §3, files touched, how to verify. **Zero code risk, and it preserves the analysis** for whoever does the work in a headspace where they can test it properly. |
| **change**       | Execute the refactor, **per finding, with explicit approval each time**. Never in bulk. |

Two guards on the change option, because an agent doing open-ended structural refactoring on
an unfamiliar codebase is how systems break:

- **Depth budget.** Cap files touched per finding (default: 5). When the cap is hit, stop and
  report — the refactor is bigger than it looked, which is itself the finding.
- **Name the abstraction before writing it.** The plan names the shape (thread / table /
  strategy / extract) and waits for a yes. "Do you want this to be a lookup table or a
  strategy object?" is a question a human answers in a second. An agent choosing silently and
  inventing a fourth shape is how you get an abstraction nobody asked for and can't delete.

Deep changes land in their own commit, separate from shallow extractions — they carry
different risk and different revert decisions.

---

## 6. Format: existing convention first

Only asked if something actually changed. Three tiers, in strict priority order:

**1. The repo's existing config system.** Extend it. The correct answer to "what format?"
is very often "the one already there," and this tier should be the default even when a
prettier option exists. Adding `config.yaml` to a repo with a working `Settings` class
creates two sources of truth, and the new one becomes the one nobody opens.

**2. A layered model, not a single file.** Real config is a stack, and a properties file is
only one layer:

```
code defaults  <  config file  <  environment variables  <  CLI flags
```

Lowest priority first. The layer that gets overridden most often should be the most
ergonomic one — which is why env and CLI usually sit on top, and why "mode / time" values
from §1 go there rather than in the file.

**3. Only then, propose — and name the tradeoff, don't just offer a menu.** Language defaults
worth knowing:

| Language / stack        | Default                                    | Notes                                                                    |
| ----------------------- | ------------------------------------------ | ------------------------------------------------------------------------ |
| Python                  | TOML (`tomllib`), or pydantic-settings + env | `tomllib` is stdlib on 3.11+; pydantic-settings gives you validation and env fallback for free |
| Rust                    | TOML via `figment` / `config` + `serde`     | Compile-time types, `#[serde(default)]` — excellent defaults story      |
| Go                      | `struct` + `env` (via `envconfig`/`viper`), optional TOML overlay | Stdlib-ish, strict, fast; env-only is the 12-factor default |
| Node / TypeScript       | Zod schema + env; `zodenv` / `envalid`      | One schema, validated types, good DX; `dotenv` only for dev            |
| Java / Kotlin           | `.properties` or YAML w/ `@ConfigurationProperties` | Typed binding is the native pattern; YAML if the repo already has it |
| C# / .NET               | `appsettings.json` + `IOptions<T>`          | Built-in, layered, reloadable                                            |
| k8s / infra-adjacent    | Kustomize / Helm values                    | Environment axis already has a home; don't duplicate it in app config   |

Rank the proposals, recommend one, and say what you'd give up. "TOML, because `tomllib` is
stdlib on the Python version this repo targets and it gives typed sections without a new
dependency" is a recommendation. A menu with no argument is homework for the user.

---

## 7. The knob contract

Every extracted knob, no exceptions. These are what separate a config surface that survives
six months from one that rots in six weeks.

- **The default lives in code, not the file.** Missing file, empty file, partial file — same
  behavior as before the run. This is the property that makes the change safe to review and
  safe to revert.
- **The file overrides the code; it never replaces it.** No "required" knobs with no default.
  If a value genuinely has no sane default, that's a different finding — a required argument,
  not a knob. Say so.
- **Commit the file empty, or all-defaults — never partial.** A checked-in file with a value
  in it is a second home for a default that already lives in code, and the two will drift.
  An empty file (or one containing exactly the defaults, generated and labeled as such) keeps
  the discoverable listing without creating a second truth.
- **Type once, at startup.** Parse into a typed settings object in one place. Never
  `int(config["retries"])` scattered through call sites — that trades a compile error for a
  runtime one and calls the trade even.
- **Unknown key = hard error.** Otherwise `max_retry` silently does nothing forever. Fail at
  startup, not in production.
- **Invalid value = loud failure**, with the key, the file, and what was expected. Silent
  fallback to the default is how a config file kills a service.
- **Keys derive from the domain, not the path.** `retry.max_attempts`, never
  `services/http.py:42`. Path-derived keys are correct exactly until someone moves the file,
  at which point every deployment breaks and nobody knows why. Namespace by owning domain,
  reuse the repo's existing naming style if it has one, and don't collide with existing keys.
- **Secrets never land here.** Flag, report, route to the existing mechanism. No default in
  code, no committed value.
- **Documented and checkable.** Every knob in the listing, and — the thing that actually
  keeps it alive — **a test or generated list that fails when code has a knob the docs
  don't mention.** Drift is guaranteed without it; this is the only cheap defense.

---

## 8. Behavior-preserving, and proven

The invariant for the whole run: **new default == old literal, exactly.** Nothing about
behavior changes. That's what makes a large mechanical diff reviewable — a reviewer checks
that values match, not that logic works.

Prove it, don't assume it — re-run the baseline commands captured in §2:

```bash
<the repo's test command>       # captured in preflight
<the repo's typecheck / lint>
```

Compare against the preflight result, don't just look at the new one. Anything that fails now
and passed before is yours. Anything that failed before **must** be reported as pre-existing
and was not fixed by this run — silently inheriting someone else's red suite is how a knob
change gets blamed for a break it didn't cause.

Then commit hygiene, which is the difference between "revert the lot" and "revert that one":
one commit per knob (or per tight logical group), shallow extractions separate from deep
refactors, message naming the knob and its domain. On a 31-knob run, per-knob revert is the
entire reason the user agreed to it.

---

## 9. Second pass: verify

**Always run a second sweep after the extraction pass, as a separate act.** Fresh subagents,
fresh context — a reviewer that just watched you write the code will confirm it. Anything else
is a rubber stamp.

Verify four things, and report failures as failures:

1. **Defaults preserved.** For every knob: the value in the config file, if present, must
   equal the literal it replaced. Any drift is a production incident waiting to happen.
2. **No config reads outside the settings object.** `grep` for direct access to the config
   map from business logic. Escape hatches at the edge are fine; a scattered read means the
   typed boundary didn't hold.
3. **Contract present.** Each knob has a default in code, a type, a doc entry, and a failing-
   on-drift check.
4. **No second layer.** The run is idempotent — re-running finds the existing knobs as knobs,
   not as new candidates to wrap. If pass two sees fresh hardcoded policy where it just
   extracted policy, pass one missed something or built indirection it shouldn't have.

Then re-run the variance test over the result and report honestly:

> 31 knobs extracted. Of those, 4 have no observed variance evidence beyond a comment —
> flagged as `[weak]` in `KNOBS.md`. I think 2 of them should probably have stayed
> constants; say the word and I'll pull them.

**Self-indictment is the point of the pass.** A verification pass that can only return
"looks good" isn't verification, it's ceremony. If nothing is wrong, the honest report names
what it checked and found clean — not nothing.