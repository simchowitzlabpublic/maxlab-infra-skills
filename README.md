# maxlab-infra-skills

Agent guidance for MaxLab infrastructure. The current skill covers Babel, CMU's
shared cluster; future GCP/TPU and other infrastructure guidance can live alongside
it.

## Install

```bash
git clone <this repo> ~/maxlab-infra-skills
cd ~/maxlab-infra-skills
./bin/install.sh
```

When explicitly run, the installer links:

- the Babel skill into `~/.agents/skills/` for Codex; and
- the repo into `~/.claude/skills/` for Claude Code.

Symlinks keep both agents on the current working tree after `git pull`.

## Token budget

Skills load in tiers, so what matters is what an agent pays before it needs the
detail. Measured with `tiktoken` `o200k_base` as a portable proxy; Claude's own
tokenizer differs slightly, but not enough to change a budgeting decision.

| Tier | Loads | Tokens |
|---|---|---|
| Always on | the skill index entry, from `SKILL.md` frontmatter | 47 |
| On invocation | `SKILL.md` | 585 |
| On demand | `references/storage.md` | 247 |
| On demand | `references/partitions.md` | 301 |
| On demand | `examples/preempt.sbatch` | 611 |
| On demand | `references/preemption.md` | 670 |
| On demand | `references/checkpointing.md` | 1578 |
| Worst case | index + `SKILL.md` + every reference + every example | 4039 |

Totals include the always-on index entry: a partition question costs 933, and
writing a preempt job that pulls the checkpointing catalog costs 2880. Codex builds
its index entry from `agents/openai.yaml` instead, at comparable size. `AGENTS.md`
(830) is charged only to agents editing this repo, not to agents using the skill.

Regenerate whenever a `SKILL.md`, reference, or example changes size. Needs
`tiktoken` (`pip install tiktoken`) and must run from the repo root; it discovers
every skill, reference, and example, so new ones are counted automatically.

```bash
python3 - <<'PY'
import tiktoken, pathlib, re
e = tiktoken.get_encoding("o200k_base"); n = lambda s: len(e.encode(s))
P = pathlib.Path
for sk in sorted(P("skills").glob("*/SKILL.md")):
    s = sk.read_text()
    fm = re.match(r"---\n(.*?)\n---\n", s, re.S).group(1)
    idx = n("- " + re.search(r"name: (.*)", fm).group(1) + ": "
            + re.search(r"description: (.*)", fm).group(1))
    got = (sorted((sk.parent / "references").glob("*.md"))
           + sorted(q for q in (sk.parent / "examples").glob("*") if q.is_file()))
    print(f"{idx:>5}  {sk.parent.name}: index entry (always on)")
    print(f"{n(s):>5}  {sk}")
    for q in got:
        print(f"{n(q.read_text()):>5}  {q}")
    tot = idx + n(s) + sum(n(q.read_text()) for q in got)
    print(f"{tot:>5}  {sk.parent.name}: WORST CASE")
print(f"{n(P('AGENTS.md').read_text()):>5}  AGENTS.md (repo authoring only)")
PY
```

## Layout

```text
skills/                infrastructure skills and their on-demand resources
AGENTS.md              portable policy + authoring conventions
CLAUDE.md              symlink to AGENTS.md, so every agent reads the same file
bin/install.sh         cross-agent installation
tests/                 regression tests
```
