---
name: grid5000
description: How to use the Grid'5000 testbed (incl. the Inria "abaca"/production queue) — OAR scheduler (oarsub), GPU reservations, kadeploy, per-site storage, modules/conda/uv, and the access→frontend→node workflow. Use when the user wants to run, reserve, submit, monitor, or debug jobs on Grid'5000 or abaca; write oarsub jobs; or set up a Python/ML env there.
---

# Grid'5000 (and abaca / production)

Grid'5000 is a large French experimental testbed. Its scheduler is **OAR** (`oarsub`), **not
SLURM**. The Inria **"abaca"** infrastructure is the renamed **`production` queue** on the same
hardware/tooling — reached with `oarsub -q abaca` (or `-q production`), still OAR.

> **SLURM note:** Grid'5000 does *not* offer a managed SLURM. The only "SLURM support" is a legacy
> tutorial for deploying your *own* SLURM onto kadeploy'd nodes — heavyweight and not worth it for
> normal jobs. On Grid'5000, use OAR.

> **Golden rules**
> 1. **Three tiers: access bastion → site frontend → compute node.** `access.grid5000.fr` is only a
>    gateway; the per-site **frontend** (e.g. `nancy`, `rennes`, `sophia`, `grenoble`) is for
>    `oarsub`/editing/data only; **all compute runs on the node OAR gives you.** NEVER train, run
>    heavy Python, or do big installs on the frontend.
> 2. **Long runs (GPU *or* CPU) → `-q production` (a.k.a. `-q abaca`).** The `default` queue
>    forbids crossing `09:00` or `19:00` on weekdays and caps daytime usage; `production` allows
>    multi-hour/day CPU and GPU jobs across weekdays. Production access must be enabled on your
>    group/account.
> 3. **Home is per-site and NOT shared across sites; there are NO backups.** Copy data explicitly
>    between sites; pull results off-cluster yourself.
> 4. **Tell the user what you reserve and where** (job id, site, node, queue) before/after.
> 5. **If one login is shared across projects, never touch another project's jobs.**
>    `oarstat -u` lists *all* of a login's jobs, which may belong to unrelated projects. NEVER
>    `oardel`, `oarwalltime`, or `-C`/connect to a job you did not create for the current project.
>    Before any destructive/modifying OAR action, positively confirm the job is this project's
>    (see **Project isolation** below). When unsure, ask.

## Project isolation (shared / multi-project account)

Grid'5000 logins are often shared across several projects or teammates. To avoid interfering with
unrelated work:

- **Name every job you create** with a project-specific prefix:
  `oarsub -n <project>-<stage> ...` (e.g. `-n myproj-stage0`). The `#OAR -n <name>` directive
  works in submission scripts too.
- **Only ever act on jobs whose name (or `command`/`initial_request`) matches the current
  project.** Identify them precisely before acting:
  ```bash
  oarstat -u --json    # inspect 'name' + 'command' per job; filter to the project's prefix
  oarstat -j <id> -f   # confirm name/command/assigned_hostnames before oardel/connect
  ```
- Treat any job you can't attribute to the current project as **off-limits** — do not cancel,
  extend, connect to, or run commands inside its node/cpuset.

## Access & SSH config

```bash
ssh login@access.grid5000.fr     # bastion (SSH keys only)
ssh sophia                        # hop to a site frontend (run OAR here)
```
Recommended `~/.ssh/config` so `ssh <site>.g5k` and `rsync` work directly:
```sshconfig
Host g5k
  User <login>
  Hostname access.grid5000.fr
Host *.g5k
  User <login>
  ProxyCommand ssh g5k -W "$(basename %h .g5k):%p"
```
Then `ssh sophia.g5k`, `rsync -avP ./data/ sophia.g5k:~/data/`.

## OAR — reserving resources

OAR resource model: `host` (whole node) → `cpu` → `core`, plus `gpu`. Walltime is `H:M:S`
(`walltime=8` = 8h, `walltime=1:30` = 90 min). Default `oarsub` = 1 node, 1 hour.

```bash
# Interactive GPU shell, 1 GPU, 8h, production/abaca queue (typical for ML):
oarsub -q production -l gpu=1,walltime=8 -I

# Whole multi-GPU node:
oarsub -q production -p "gpu_count>0" -l host=1,walltime=8 -I

# Passive/batch (no shell) — runs a command, logs to OAR.<jobid>.std{out,err}:
oarsub -q production -l gpu=1,walltime=24 "bash run.sh"

# Submission script with embedded #OAR directives:
oarsub -q production -S ./job_oar.sh

# Advance reservation (default queue; restricted in production):
oarsub -r "2026-09-15 20:00:00" -l gpu=1,walltime=10
```
Monitor / manage:
```bash
oarstat -u                 # your jobs (oarstat = whole site)
oarstat -j <jobid> -f      # full detail
oardel <jobid>             # cancel
oarwalltime <jobid> +1:00  # extend a running job
```
Property filters (`-p`, SQL-like): `gpu_count`, `gpu_model`, `gpu_mem`,
`gpu_compute_capability_major`, `cluster`, `host`.
```bash
oarsub -q production -l gpu=1 -p "gpu_model='A100'" -I
oarsub -q production -l gpu=1 -p "gpu_compute_capability_major>=8" -I   # bf16-capable
oarsub -q production -p "cluster='grat'" -l gpu=1,walltime=8 -I
```

### `default`-queue Usage Policy (`-t night`, `-t day`, `usagepolicycheck`)

Per `Grid5000:UsagePolicy`, `-q default` splits weekdays (`Mon–Fri`) into **daytime (`09:00–19:00`)**
(per-cluster core-hour caps and short walltimes) and **nighttime (`19:00–09:00`)**, with continuous
weekend windows from **Friday `19:00` to Monday `09:00`**. Jobs in `-q default` **must not cross
`09:00` or `19:00` on weekdays**.

- **FIFO queue-slip trap:** OAR does *not* reject a long `-q default` batch job submitted without
  `-t night`. If target nodes are busy — or if idle nodes belong to `-t exotic` clusters that OAR
  filters out automatically unless `-t exotic` is passed — FIFO scheduling can delay a weekend or
  evening job until Monday/weekday daytime and run it across `09:00`/`19:00`, triggering an
  automated Usage Policy violation notice.
- **Always bound `-q default` batch jobs:** pass **`-t night`** (`oarsub -q default -t night ...`)
  for overnight or weekend runs so OAR admission rules enforce `postpone` + a hard `09:00` (or
  `Monday 09:00`) `deadline`, or **`-t day`** (`deadline=19:00`) for daytime runs; inspect
  `scheduled_start` and `properties` with `oarstat -j <jobid> -f`. For unattended multi-day CPU or
  GPU campaigns, use `-q production` (`-q abaca`) instead.
- **Verify live on any site frontend with `usagepolicycheck`:**
  ```bash
  usagepolicycheck -l --detailed-limits --sites <site>   # per-cluster daytime core-hour caps
  usagepolicycheck                                       # audit past-week + queued jobs (all sites)
  usagepolicycheck --sites <site>                        # audit one site (caps + 09:00/19:00 crossings)
  ```

### `#OAR` submission script template
```bash
#!/bin/bash
#OAR -q production
#OAR -l host=1/gpu=1,walltime=12:00:00
#OAR -p gpu_compute_capability_major>=8
#OAR -O OAR_%jobid%.out
#OAR -E OAR_%jobid%.err
set -euo pipefail
hostname; nvidia-smi
cd ~/project
uv run python train.py
```

## "abaca" specifics

- `abaca` == the `production` queue (both `-q abaca` and `-q production` accepted; if one is
  rejected try the other — aliasing can be site-dependent).
- Spans GPU and CPU clusters across **Nancy, Rennes, Grenoble, Sophia** (e.g. Nancy `grat` = 8×
  A100 80GB; Sophia `musa` = H100 NVL; filter CPU-only nodes with `-p "gpu_count=0"`). **Inventory
  changes** — confirm the live cluster/GPU via the site
  Hardware wiki page or `oarnodes` on the frontend before hardcoding a cluster name.
- Production walltime caps by priority tier: p1 up to 168h, p2 96h, p3 48h, p4 24h. Advance
  reservations in production are restricted (high-priority users only).
- Requires production rights on your "Group Granting Access"; otherwise `-q production` fails.

## GPUs & CUDA

NVIDIA drivers + CUDA are preinstalled on GPU nodes (default environment — **no kadeploy needed for
normal ML jobs**). On the node:
```bash
nvidia-smi; nvcc --version
module av cuda && module load cuda/<ver>
```
`-l gpu=1` gives one GPU + a fraction of a possibly-shared node; for all GPUs of a node use
`host=1` with `-p "gpu_count>0"`. If `nvidia-smi` shows fewer GPUs than expected, you reserved a
fraction — re-reserve the whole host.

## Software env (Python, conda, uv, internet)

- Default OS is Debian with `module`, conda, Guix preinstalled; `sudo-g5k` gives temporary root on
  your node without kadeploy.
- **Outbound internet works from nodes (NATed) — no proxy needed** for pip/uv/git. (The
  `*.proxy.grid5000.fr` reverse proxy is the opposite direction: exposing a node-hosted UI outward.)
- conda: `module load conda` (don't install your own Miniconda; base is read-only — `conda create`
  your own). **uv works** (internet + standard Python available); do installs **on a node, not the
  frontend**:
  ```bash
  curl -LsSf https://astral.sh/uv/install.sh | sh
  cd ~/project && ~/.local/bin/uv venv && ~/.local/bin/uv sync
  ```

## Storage

| Storage | Path | Notes |
|---|---|---|
| Home | `<site>:/home/<login>` | NFS, **per-site (not shared)**, 25GB soft / 100GB hard default quota, **no backup** (`quota -s` to check) |
| Group storage | requested per team | larger persistent shared NFS, mounted under `/srv/storage/<group>@<host>` |
| Node local `/tmp` | on the node | fast scratch, wiped between jobs — stage datasets here for training |

**Home vs group storage — capacity matters.** Homes are **per-site** with their own quota (default
25 GB soft / 100 GB hard); check with `quota -s` per site rather than assuming, since some sites
raise the quota for some users. Group storage is far larger and persistent. Policy: **privilege the
group volume for anything not needed immediately** (checkpoints, datasets, large artifacts) and keep
home free for code/configs/small results and momentary scratch. Pick based on size, how soon you
need it, and the project.

**Group (team) NFS volumes** are auto-mounted (via G5K's global autofs) at a path of the form:
```
/srv/storage/<group>@<storage-host>.grid5000.fr
```
They are typically mounted on **every site frontend and on compute nodes cross-site**, so a job
works regardless of where it lands; cross-site access is plain NFS over the G5K backbone (fine for
checkpoints, just slower than same-site for huge sequential I/O). Volumes usually carry the setgid
bit + team group ownership, so new files inherit the team's Unix group. Find and check yours with:
```bash
ls -ld /srv/storage/*@*.grid5000.fr                                   # discover mounted volumes
GROUP_VOL="/srv/storage/<group>@<storage-host>.grid5000.fr/<login>"
ssh sophia.g5k "df -h /srv/storage/<group>@<storage-host>.grid5000.fr"  # check from any site
```

Move data in / pull results out (no backups!):
```bash
rsync -avP ./data/ sophia.g5k:~/data/          # laptop -> site
rsync -avP sophia.g5k:~/project/out/ ./out/    # pull results OFF (do this!)
```
Between sites (homes differ): `rsync -avP ~/data/ rennes:~/data/`.

## kadeploy (usually NOT needed)

Only if you need a custom OS / full root / specific kernel: reserve with `-t deploy`, then
`kadeploy3 <env>` (e.g. `debian12-nfs`, `ubuntu2204-big`). For routine GPU training the default
environment already has drivers+CUDA+conda+uv — skip kadeploy.

## End-to-end ML workflow

```bash
rsync -avP ./project/ sophia.g5k:~/project/         # ship code+data
ssh sophia.g5k                                      # frontend (no compute!)
oarsub -q production -l gpu=1,walltime=12 -I       # reserve -> lands on node
#  on the node:
nvidia-smi; cd ~/project
~/.local/bin/uv sync && uv run python train.py --out ~/project/out
exit
rsync -avP sophia.g5k:~/project/out/ ./out/         # pull results off-cluster
```
For unattended runs use a passive job (`oarsub -q production -l gpu=1,walltime=24 -S ./job_oar.sh`)
so it survives disconnects; monitor with `oarstat -u` and the `OAR.<jobid>.*` logs.

## Gotchas

- It's **OAR, not SLURM** — `sbatch`/`srun`/`squeue` don't exist; use `oarsub`/`oarstat`/`oardel`.
- **Never compute on the frontend** — even heavy `uv`/`conda` installs go on a reserved node.
- **Home is per-site**; "my files vanished" is usually the wrong site. **No backups** — `/tmp` wiped
  between jobs; copy results out.
- **Multi-hour/multi-day CPU or GPU work belongs in `-q production`/`-q abaca`** (requires group
  access). In `-q default`, OAR does *not* block batch jobs from crossing `09:00`/`19:00` on
  weekdays — always pass **`-t night`** on overnight/weekend `default` jobs so FIFO delays cannot
  push them past `09:00`, and audit queued jobs with `usagepolicycheck`.
- `besteffort` queue is preemptible (killed when needed) — checkpoint often if you use it.
- Between your own nodes in a multi-node job use `oarsh`, not `ssh`, unless you own the whole node.
- Inventory (clusters/GPUs under abaca) rotates — verify before hardcoding cluster names.

## Sources
Grid'5000 wiki (https://www.grid5000.fr/w/): Getting_Started, Production, Production:Getting_Started,
GPUs_on_Grid5000, Storage, Conda, HTTP/HTTPs_access, Grid5000:UsagePolicy, Hardware. SLURM-on-G5000
is only the archived OAR tutorial (oar.imag.fr) — not a managed service. Some `/w/` pages are
login-gated; re-verify abaca cluster/GPU inventory live with `oarnodes`/the site Hardware page.
