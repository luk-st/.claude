---
name: wcss-lem-slurm
description: Write and submit Slurm jobs on the WCSS LEM supercomputer (ui.wcss.pl) - node specs, lem-gpu/hopper H100 96GB partitions, sbatch boilerplate, job-array chunking around AssocMaxSubmitJobLimit, and the read-only *-mount inspection workflow. Use when writing an sbatch script, sizing a job for LEM, planning an array sweep, or inspecting cluster artifacts through a mount.
---

# WCSS LEM

Polish HPC cluster at Wrocław Centre for Networking and Supercomputing.
Login node `ui.wcss.pl`, AlmaLinux, Slurm, 21 PFLOPS aggregate.

## Hardware

| | GPU nodes | CPU nodes |
|---|---|---|
| Count | 76x Dell PowerEdge XE9640 | 188x HPE Cray XD225v |
| CPU | 2x Intel Xeon Platinum 8462Y (32 cores, 2.8 GHz) | 2x AMD EPYC 9554 (64 cores, 3.1 GHz) |
| GPU | **4x NVIDIA H100 96 GB** (16896 CUDA cores) | — |
| RAM | 1006 GB DDR5-4800 | 1536 GB DDR5-4800 |
| Scratch | — | 3.5 TB NVMe |
| Network | Infiniband NDR200 4x 200 Gbps | Infiniband NDR200 2x 200 Gbps |


## sbatch boilerplate

```bash
#!/bin/bash -l
# ABOUTME: one-line description of what this job does
# ABOUTME: second line if needed
#SBATCH --nodes=1
#SBATCH --ntasks-per-node=1
#SBATCH --cpus-per-task=12
#SBATCH --mem=200G
#SBATCH --time=04:00:00
#SBATCH --partition=lem-gpu
#SBATCH --gres=gpu:hopper:1      # 1-4 per node

module load Python/3.10.4-GCCcore-11.3.0
cd "${SLURM_SUBMIT_DIR:-$PWD}"
export PYTHONPATH=$PWD
source .venv/bin/activate
```

- Partition is `lem-gpu`; GPU type string is `hopper`.
- Request only the GPUs you use — 4 per node is the ceiling.
- `--mem` is per node. 200G is a reasonable default; nodes have ~1 TB.
- Always `cd "${SLURM_SUBMIT_DIR:-$PWD}"` so relative paths resolve.

## Job arrays

`AssocMaxSubmitJobLimit` trips above roughly **108 tasks per submit**. Chunk
larger sweeps:

```bash
sbatch --array=0-107   run_sweep.sh
sbatch --array=108-215 run_sweep.sh
sbatch --array=216-323 run_sweep.sh
```

Parameterize tasks with a manifest TSV rather than encoding parameters in the
script. Generate the manifest inside the job, then select the row:

```bash
META="outputs/sweeps/${SHORT}_meta/task_${SLURM_ARRAY_TASK_ID}"
mkdir -p "$META"
python scripts/make_manifest.py --manifest "$META/jobs.tsv" --grid "$GRID" || exit 1

IFS=$'\t' read -r CONCEPT TAG ALPHAS KWARGS SAVE_DIR \
  <<< "$(sed -n "$((SLURM_ARRAY_TASK_ID + 1))p" "$META/jobs.tsv")"
[ -n "${SAVE_DIR:-}" ] || { echo "no manifest line for task $SLURM_ARRAY_TASK_ID" >&2; exit 1; }
```

Pass the grid selector by environment variable so one script serves many sweeps:

```bash
GRID=scripts.sweeps.my_sweep.sweep_grid sbatch --array=0-107 run_sweep.sh
```

## Scripts must be right on first submit

The usual workflow gives no fast debug loop on the cluster: code is developed
and smoke-tested elsewhere, pushed, then pulled and submitted by the user. Every
script therefore needs:

**Idempotence** — skip work already done, so a resubmit after partial failure is
free:

```bash
if [ -f "$SAVE_DIR/protocol_results/lpaps.csv" ]; then
  echo "[skip] already scored: $SAVE_DIR"; exit 0
fi
rm -rf "${SAVE_DIR:?}"/alpha_* "${SAVE_DIR:?}"/protocol_results
```

The `${VAR:?}` form aborts on an empty variable instead of `rm -rf`-ing the
current directory.

**Output-path guards** — refuse to write outside the expected root:

```bash
case "$SAVE_DIR" in
  outputs/sweeps/my_experiment/*) : ;;
  *) echo "refusing to write outside the sweep root: $SAVE_DIR" >&2; exit 1 ;;
esac
```

**Retries** for transient failures (node hiccups, transient OOM, flaky I/O):

```bash
ok=0
for attempt in 1 2 3; do
  python src/run.py --config "$CFG" && { ok=1; break; }
  echo "attempt $attempt failed; retrying after 90s..."; sleep 90
done
[ "$ok" = 1 ] || { echo "failed after 3 attempts" >&2; exit 1; }
```

**Loud early logging** — echo the resolved parameters and a timestamp before the
work starts, so a failure is diagnosable from the log alone:

```bash
echo "GRID=$SHORT CONCEPT=$CONCEPT TAG=$TAG SAVE_DIR=$SAVE_DIR $(date)"
```

## Inspecting results: `*-mount/` is READ-ONLY

`pwr-mount/` and siblings are sshfs mounts of a repo clone on the cluster,
present so cluster artifacts can be read locally: checkpoints, Slurm logs,
generated samples, eval JSONs, dataset state.

**Never write code into a mount.** The flow is one-directional:

1. Develop and smoke-test locally.
2. Commit and push.
3. The user pulls on the cluster and submits.
4. Read results back through the mount.

Editing in the mount silently diverges from the pushed commit, which breaks the
guarantee that every result traces to a known SHA.

## Checklist before handing over a script

- [ ] Smoke-tested on local GPUs, exact same code path
- [ ] `--gres` matches GPUs actually used
- [ ] `--time` has headroom; job is idempotent if it hits the wall
- [ ] Array chunked under ~108 tasks per submit
- [ ] Output paths guarded and greppable (timestamp + variant)
- [ ] stdout/stderr tee'd to a timestamped log
- [ ] Resolved parameters echoed before work starts
