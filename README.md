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

When `PI_RESEARCH_SLURM_SSH_HOST` is set, `sbatch`, `squeue`, `sacct`, `sinfo`,
`scontrol`, `scancel`, and `sacctmgr` run over SSH, and the `--chdir` and
`--output` paths are rewritten from `_FROM` to `_TO`. The local project
directory must have a matching checkout on the cluster (for example via git),
and the SSH connection must be passwordless (`BatchMode=yes`). Job logs are
written to the cluster path, so read them back with `scp` or `ssh tail`.

## Optional integrations

The extension emits `slurm:finished` on Pi's event bus when a tracked job reaches
a terminal state. Its payload is `{ id, name, logPath, status, detail }`.
`pi-sleep` uses this event to interrupt a sleep watching that Slurm job.

`slurm_submit` accepts an optional `qos`. When it is omitted, the extension checks the current account's Slurm associations and selects a QoS matching the requested partition (for example, `cpu` for the `cpu` partition). Use `PI_RESEARCH_SLURM_QOS` to set a site-specific default when accounting queries are unavailable.
