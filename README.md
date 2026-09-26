# pi-slurm

Pi extension providing session-persistent Slurm submission, monitoring, inspection, and cancellation tools.

## Tools

- `slurm_submit`
- `slurm_jobs`
- `slurm_cancel`

Jobs are persisted in the Pi session and state changes generate context notifications. By default the extension runs Slurm commands on the local host and stores logs under `.pi-research-engineer/slurm`; set `PI_RESEARCH_SLURM_LOG_DIR` to override the log directory.

## Remote submission over SSH

When Pi runs on a machine without a Slurm client (for example a laptop that
must not run agents on a cluster login node), the extension can submit and
monitor jobs on a remote cluster over SSH. Set:

| Variable | Meaning |
|---|---|
| `PI_RESEARCH_SLURM_SSH_HOST` | SSH host (from `~/.ssh/config`) that has the Slurm client commands |
| `PI_RESEARCH_SLURM_REMOTE_PREFIX_FROM` | Local filesystem prefix, e.g. `/Users/alice` |
| `PI_RESEARCH_SLURM_REMOTE_PREFIX_TO` | Cluster filesystem prefix, e.g. `/nfs_home/alice` |
| `PI_RESEARCH_SLURM_REQUEUE` | Optional: `1` forces `--requeue`, `0` forces `--no-requeue`; unset inherits the cluster `JobRequeue` default |
| `PI_RESEARCH_SLURM_LOG_DIR` | Override the log directory (see Tools) |

When `PI_RESEARCH_SLURM_SSH_HOST` is set, `sbatch`, `squeue`, `sacct`, `sinfo`,
`scontrol`, `scancel`, and `sacctmgr` run over SSH, and the `--chdir` and
`--output` paths are rewritten from `_FROM` to `_TO`. The local project
directory must have a matching checkout on the cluster (for example via git),
and the SSH connection must be passwordless (`BatchMode=yes`). Job logs are
written to the cluster path, so read them back with `scp` or `ssh tail`.

### Example: run Pi on a laptop, submit to the cluster

On the laptop, `~/.ssh/config` needs a passwordless entry (an SSH key already
installed in the cluster account's `authorized_keys`):

```
Host cluster
    HostName <login-node>
    User zou
    IdentityFile ~/.ssh/id_ed25519
    BatchMode yes
```

Then launch Pi with the mapping exported (or put them in the shell profile):

```bash
export PI_RESEARCH_SLURM_SSH_HOST=cluster
export PI_RESEARCH_SLURM_REMOTE_PREFIX_FROM="$HOME"        # e.g. /Users/alice
export PI_RESEARCH_SLURM_REMOTE_PREFIX_TO=/nfs_home/alice   # the same tree on the cluster
pi
```

`slurm_submit` then runs `sbatch` over SSH, and the monitor polls `squeue`/
`sacct` over SSH every 5 s.

## Node failure, requeue, and reconnect

- When a compute node dies mid-job, Slurm requeues the job by default when the
  cluster sets `JobRequeue=1` (restarting the batch script **from scratch**, up
  to the cluster's `MaxBatchRequeue`). Use `PI_RESEARCH_SLURM_REQUEUE=1`/`0` to
  force the per-job choice on clusters whose default differs.
- The monitor detects a `RUNNING -> PENDING` transition (a requeue) and emits a
  notification naming the reason, so the agent does not mistake a restarted job
  for one that kept its progress.
- Requeued jobs restart from the beginning. **Progress is only preserved if the
  job itself checkpoints** (save state/weights periodically and resume from the
  latest checkpoint on start). Pi cannot recover a job that did not save state.
- If the SSH link (or the scheduler) drops, `squeue`/`sacct` fail for ~30 s, the
  monitor emits a "lost contact" warning, keeps polling, and emits a
  "reconnected" notice when the link returns. Terminal jobs already recorded in
  the session are re-announced after a Pi restart.

## Optional integrations

The extension emits `slurm:finished` on Pi's event bus when a tracked job reaches
a terminal state. Its payload is `{ id, name, logPath, status, detail }`.
`pi-sleep` uses this event to interrupt a sleep watching that Slurm job.

`slurm_submit` accepts an optional `qos`. When it is omitted, the extension checks the current account's Slurm associations and selects a QoS matching the requested partition (for example, `cpu` for the `cpu` partition). Use `PI_RESEARCH_SLURM_QOS` to set a site-specific default when accounting queries are unavailable.
