# Preemption on `preempt`

A valid `preempt` job must:

- never set `--no-requeue` (Babel sets `JobRequeue=1`, so requeue is already on and
  an explicit `--requeue` changes nothing; `--no-requeue` forfeits it);
- checkpoint often enough that the worst-case lost work is acceptable;
- discover and resume its latest durable checkpoint at startup;
- write checkpoints atomically;
- use a stable run path across manual resubmissions; and
- append rather than truncate its Slurm log.

Keep run state in `/data/user_data/$USER`, not `/scratch`. Restore the full training
state: model, optimizer, scheduler/scaler, RNG, sampler/dataloader position, and
global step. For arrays, include `$SLURM_ARRAY_TASK_ID` in the stable run identity
when each task is a distinct run.

## How preemption arrives

Babel runs `PreemptType=preempt/qos` with `PreemptMode=REQUEUE`, and nearly every lab
QOS outranks `preempt_qos`. A `preempt` job is therefore requeued from the top of the
batch script, with the same job id, whenever someone with priority wants the node.

Assume no warning. The partition reports `GraceTime=0` while `preempt_qos` reports
`GraceTime=00:02:00`, and Slurm does not document which applies under `preempt/qos`.
A SIGTERM handler may get roughly two minutes or nothing at all; treat it as a bonus,
never as the checkpoint strategy.

```bash
scontrol show partition preempt --oneliner | tr ' ' '\n' | grep GraceTime
sacctmgr -nP show qos format=Name,Priority,GraceTime | sort -t'|' -k2 -nr
```

Preemption is routine and usually early, so set cadence from the live distribution
rather than a guess. `-D` is required: requeued attempts reuse the job id, and
without it `sacct` reports only each job's final record. Over the seven days to
2026-09-15 that is roughly 9,800 preemptions, p50 17 min, p90 227 min:

```bash
sacct -aXnPD --partition=preempt -S now-7days -o State,ElapsedRaw \
  | awk -F'|' '$1 == "PREEMPTED" {print $2}' | sort -n \
  | awk '{a[NR] = $1}
         END {printf "n=%d p50=%dmin p90=%dmin\n",
                     NR, a[int(NR * .5)] / 60, a[int(NR * .9)] / 60}'
```

Read `checkpointing.md` for how to satisfy the list above in code.

`examples/preempt.sbatch` is one worked example of the list above, not a structure
to adopt. Its cache layout, `last.pt` naming, and `srun python train.py` are
illustrative. Satisfy the requirements in whatever shape fits the user's code.
