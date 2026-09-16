# Writing code that survives preemption

`preemption.md` is the contract and owns the full-state list; this is how to satisfy
it in code. Resume must be the default path, not a flag someone remembers to pass.
Where your stack has no resumable primitive, a short writer you own often beats a new
dependency.

## State that gets left out

- EMA and averaged weights are learned state; losing them degrades eval silently.
- Running normalization statistics and BN buffers are state, not config.
- Save counters rather than values derived from them: tokens seen, curriculum stage,
  early-stop patience, best-so-far metric.
- RNG is per rank and per device; saving one generator restores one stream.

## Data position

- Restoring the sampler seed is not restoring position. A map-style resume must skip
  the epoch's consumed indices, not reshuffle it.
- `torchdata`'s `StatefulDataLoader` is a drop-in `DataLoader` with `state_dict` and
  `load_state_dict`; it needs the same `num_workers` and holds no cross-rank state.
- HF `datasets` `IterableDataset.state_dict` stores shard plus example index and
  replays from the shard start, so shard count bounds the replay cost.
- `accelerate.skip_first_batches` pays a real pass over the data; it is a fallback.
- Best is a batch order that is a pure function of seed and step over a randomly
  indexable store, as Levanter does; resume is then a seek at any world size.
- Log the first example id after a resume and check it is the one that was next.

## Publishing a checkpoint

- Write a temporary name in the destination directory, fsync, then `os.replace`.
  Rename is atomic only within one filesystem, so never stage through `/scratch`.
- Publish by swapping a pointer so a reader never opens a half-written file.
- A sharded write is not atomic as a set; swap the pointer only once all shards land.
- Keep at least two checkpoints and prune only once the replacement is durable. The
  newest may be mid-write, which is why `torchtitan`'s `keep_latest_k` refuses 1.
- On load failure fall back to the previous checkpoint as one collective decision
  re-broadcast by rank 0, and say so loudly in the log.

## Cadence

- Budget lost work in minutes against the live distribution in `preemption.md`.
- Measure the save. If a synchronous save costs more than a few percent of step time,
  overlap it (`torch.distributed.checkpoint.async_save`).
- An asynchronous save must be awaited before exit, and two attempts must never write
  the same run path at once.

## Resume-first control flow

- Discover the checkpoint from the stable run path instead of requiring a human to
  pass `--resume`; Composer calls this `autoresume`.
- Do that discovery before anything else writes into the run directory.
- Requeue re-runs the batch script from line 1, so everything above the training
  command must be idempotent.
- The next attempt may land on another node. Rebuild scratch, caches, and temp dirs
  every attempt and assume they are empty.
- `${SLURM_RESTART_COUNT:-0}` is the attempt number, unset by Slurm on the first
  attempt; use it to spot a job that keeps restarting without progressing.

## Reinforcement learning

- The replay buffer is training state; an off-policy run that drops it restarts cold.
  SB3's `save_replay_buffer` is off unless you ask for it.
- Save the buffer separately from the policy and pick its cadence against refill
  time, not the policy's.
- On-policy, checkpoint at rollout boundaries; losing one rollout usually beats
  serializing a partial buffer.
- `VecNormalize`-style running observation and reward statistics are learned state.
- Save env RNG and episode step, or checkpoint only at episode boundaries when env
  state will not serialize. Env steps and gradient steps diverge; save both.
- Restore data-consumption counters alongside RNG so rollouts after a resume continue
  the stream instead of resampling it.
- With split actors and learners, persist learner weights continuously and treat
  in-flight trajectories as droppable.

## Generation, eval, and sweeps

- Make the item the unit of recovery: append one JSON line per item, and flush.
- Key each line by an id derived from the input, never by enumeration order.
- On start, build the skip set by reading the output file, discarding a trailing
  unparseable line; a saved counter or index lies after a hard kill.
- One appender per rank or task (`rank_3.jsonl`), merged at the end; concurrent
  appends to one file interleave.
- For arrays, write a per-task done marker after the final flush.

## Multi-rank

- All ranks must resume at the same step; one fresh rank breaks the first collective.
- Rank 0 resolves the checkpoint path and broadcasts it. Ranks globbing independently
  can disagree on which checkpoint is latest mid-write.
- Use a resharding-tolerant format such as PyTorch DCP when a later attempt may get a
  different GPU count.
- Barrier after the write and before the pointer swap.
- Initialize the tracker on rank 0 only, or each attempt multiplies the run count.

## Trackers and logs

- Derive a stable tracker run id from the run path and pass it to `wandb.init` as an
  explicit `id` with `resume="allow"`, which creates on attempt 1 and resumes after.
- `resume="auto"` is the trap: it needs the killed process's working directory, so a
  requeue onto another node silently starts a fresh run instead.
- `WANDB_RUN_ID` and `WANDB_RESUME` set both from the launcher, editing no code.
- A resumed run's step is ahead of the checkpoint, and re-logging that span is
  silently dropped as non-monotonic — expect the gap, and never reset to zero.
- Log the resumed step and `${SLURM_RESTART_COUNT:-0}` at startup. A silent restart
  from zero is the most common preemption bug and the easiest to miss.

## Prove it works

- `scontrol requeue` your own job mid-run; that is the rehearsal preemption runs.
  `scancel` plus resubmit tests less, because a new job id writes a new log.
- Compare 200 uninterrupted steps against 100 plus resume plus 100; the curves should
  match, and match bitwise if RNG and data position were restored.

## Shutdown signals are a bonus

- Checkpoint on a cadence regardless. A handler only shrinks the loss window, and
  only when there is any warning at all; see `preemption.md`.
- Set a flag and save at the next step boundary rather than checkpointing inside the
  handler with a collective in flight, and never start a save you cannot finish.
- `--signal` is measured from the job's end time, so it covers hitting `--time`
  rather than preemption, and may fire up to a minute early.
- `--signal` reaches only job steps unless prefixed `B:`; without it a handler in the
  batch script never runs.
- Do not reuse a signal your stack claims; submitit left SIGUSR1 over an NCCL clash.
