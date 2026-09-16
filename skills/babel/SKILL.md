---
name: babel
description: CMU's shared Babel Slurm cluster. Use for Babel, sbatch, srun, squeue, GPU partitions, or /data paths. Covers MaxLab policy, partition choice, preemption, and storage.
---

# Babel

Babel is CMU's shared cluster, used by many labs; it is not MaxLab's cluster.

## Partition choice

| Need | Partition |
|---|---|
| Restartable GPU work | `preempt` |
| MaxLab's scarce, non-preemptible GPUs | `maxlab` — Max must approve the workload |
| CPU shell or `/data` access | `maxlab-cpu` |
| Non-preemptible GPU work | `general`, only with `normal` QOS |

Read `references/partitions.md` for the `maxlab-cpu` command, the `general` access
check, and the live `MaxTime` check. GPU partitions require at least one GPU.

## MaxLab approval

If approval is unknown, ask once whether Max approved the workload. Trust the
user's answer; do not request proof or reconfirm resubmissions or array tasks.
Remember the answer for that workload. `sbatch --test-only` is safe without
approval.

Approval governs scarce GPUs used non-preemptibly. Separately, `maxlab_qos` may
preempt eligible jobs from any cluster user; that is expected and allowed.

## Preempt jobs

Require durable checkpoints, automatic full-state resume, atomic writes, stable run
paths, and append-only logs. Requeue is already on (`JobRequeue=1`); never set
`--no-requeue`. Preemption is routine and usually early — most jobs go within the
hour — so `preempt` suits only work that genuinely restarts.

Read `references/preemption.md` for the requirements and how preemption arrives, and
`references/checkpointing.md` when writing or changing code that must survive a
restart.

`examples/preempt.sbatch` is a worked example of those requirements, not a
structure to adopt. Fit the user's existing layout, tooling, and checkpoint format;
copy from it only where it genuinely suits.

## Storage

- Personal persistent work: `/data/user_data/$USER`
- Team-shared artifacts: `/data/group_data/maxlab/common_datasets`
- Exigent fallback when user storage is unusable: the group path
- Shared, evictable HF caches: `/data/hf_cache/hub` and `/data/hf_cache/datasets`
- Disposable node-local work: `/scratch/$USER/...`

Read `references/storage.md` for the Hugging Face environment. `/data` and
`/scratch` exist only on compute nodes.

Never run `bin/install.sh` unless the user explicitly asks to install the repo.
