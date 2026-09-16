# MaxLab infrastructure — agent instructions

This repo holds MaxLab infrastructure guidance. Babel is the current surface and is
CMU's shared cluster, not MaxLab's. Its full guidance is in `skills/babel/SKILL.md`.

## Hard rules

- If Max's approval for a `maxlab` workload is unknown, ask once and trust the
  user's answer. Do not request proof or reconfirm resubmissions or array tasks.
- `sbatch --test-only` is safe without approval.
- Never run `bin/install.sh` unless the user explicitly asks for installation.

## Babel nuances

| Need | Partition |
|---|---|
| Restartable GPU work | `preempt` |
| MaxLab GPUs, with Max's approval | `maxlab` |
| CPU shell with `/data` | `maxlab-cpu` |
| Non-preemptible GPU work, if the user has `normal` QOS | `general` |

- `/data` and `/scratch` exist only on compute nodes.
- Default `maxlab-cpu` requests to 2 CPUs and 4 GB unless clearly insufficient.
- For `preempt`, require durable checkpoints, atomic writes, automatic resume,
  stable run paths, and append logs. Requeue is already on (`JobRequeue=1`); never
  set `--no-requeue`.
- Never request more than a partition's live `MaxTime`; excessive requests may pend
  forever because Babel uses `EnforcePartLimits=NO`.
- Personal persistent data belongs in `/data/user_data/$USER`.
  `/data/group_data/maxlab/common_datasets` is only for team-shared artifacts or an
  exigent fallback when user storage is unusable.
- Hugging Face jobs use the shared, evictable `/data/hf_cache/{hub,datasets}`.

Commands and details: `skills/babel/references/`.
`skills/babel/examples/preempt.sbatch` is illustrative, not scaffolding to copy —
match the user's existing layout and tooling.

## Authoring conventions

Applies when editing this repo. `CLAUDE.md` is a symlink to this file, so every
agent gets the same instructions.

- One skill per infrastructure surface (`babel`, later `gcp-tpu`), not per task.
- Put only non-obvious policy and decisions in `SKILL.md`; put task-specific detail
  in `references/`, loaded on demand.
- Put changing limits behind a short inspection command rather than hardcoding them.
- Verify cluster claims against the live system before writing them down, and keep
  the command that proves it nearby. On the documented partitions, explicit
  `--requeue` and the partition's matching `--qos=` are redundant.
- Assume capable agents; omit generic Slurm, shell, and debugging tutorials.
- Name a third-party API only when its default or constraint is a trap; give the
  symbol, never versions or full call signatures, and prefer a short local
  implementation to a new dependency.
- Keep `examples/` runnable and comments limited to Babel-specific traps. They are
  illustrative, not scaffolding; never write guidance that tells an agent to start
  from one, and do not reintroduce the name "templates".
- Use plain Markdown; add automation only when deterministic enforcement clearly
  justifies the review surface.
- Keep this file short; it is always loaded by agents working in this repo.
- `README.md` carries the token budget per loading tier; refresh it with the
  command there whenever a `SKILL.md`, reference, or example changes size.
- Put Codex metadata in `skills/<name>/agents/openai.yaml`; `bin/install.sh` links
  skills into `~/.agents/skills/`.

Validate:

```bash
python3 -m unittest discover -s tests -v
python3 ~/.codex/skills/.system/skill-creator/scripts/quick_validate.py skills/<name>
claude plugin validate .
```
