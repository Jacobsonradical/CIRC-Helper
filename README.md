# CIRC-Helper

The Center for Integrated Research Computing (CIRC) at the University of Rochester is genuinely hard to start using, and almost all of that difficulty is documentation, not the machines. **Nobody has time to sit through a three-hour training session for weeks on end, least of all the PhD students who actually need the cluster. Just write down the commands and give people a working example, so they can get on with their research.**

So this is my attempt at that. Everything here is stuff I had to work out the hard way, mostly by breaking things first.

> My older notes, from when I was using the previous BlueHive environment, are in **[README_old.md](README_old.md)**. Some of it still applies. Some of it is now wrong in ways that will cost you a day, and I say which bits below.

---

### Register an account

https://registration.circ.rochester.edu/account

---

<div align="center">

## Which machine do you connect to?

</div>

There are two environments, and this matters more than it looks.

```bash
ssh YourNetIDHere@bluehive.circ.rochester.edu     # the old one, RHEL 7.9
ssh YourNetIDHere@bluehive3.circ.rochester.edu    # the new one, RHEL 9.4
```

**Use BlueHive3 unless you have a specific reason not to.** It runs a much newer operating system, a newer SLURM, and it is where compute nodes are being migrated. The rest of this page assumes BlueHive3.

The reason it matters is not just "newer is nicer". The old environment is RHEL 7.9, which ships glibc 2.17, and a lot of modern Python packages have simply stopped publishing builds that old. You can hit a wall where a package flatly cannot be installed and no amount of fiddling helps. BlueHive3 is RHEL 9.4 with glibc 2.34, and that whole category of problem goes away.

Log in and check for yourself:
```bash
cat /etc/os-release | head -3
ldd --version | head -1
```

**Be warned that the two environments have different module trees.** The Python versions available on BlueHive are not the ones available on BlueHive3. Do not assume anything carries over — check.

---

<div align="center">

## The single most important thing: login node vs compute node

</div>

When you ssh in, you land on the **login node**. It is one machine shared by everybody on the cluster. It exists for editing files, moving data around, and submitting jobs.

**It is not where your work runs.** If you run something heavy there you slow it down for every other person logged in, and CIRC will notice.

Your actual computing happens on **compute nodes**, which are separate machines you have to ask SLURM for. There are two ways to ask:

```bash
# 1. Interactive: give me a shell on a compute node right now
srun -p interactive -t 2:00:00 -c 4 --mem=16G --pty bash

# 2. Batch: queue this script, run it whenever, I will come back later
sbatch myjob.slurm
```

Use **interactive** when you want to type commands and watch what happens — installing packages, testing that something imports, poking at data. Use **batch** for the real thing.

Always know where you are:
```bash
hostname
```
- `bluehive3` → you are on the login node. Do not run anything heavy.
- `bhd0099`, `bhgrb4x0085`, anything like that → you are on a compute node. Go ahead.

`exit` puts you back on the login node.

Two things that confuse everybody at first:

**Running `srun` on the login node is correct.** That is its entire job — it is how you leave. Asking SLURM for a machine is a light operation.

**Your interactive session and your batch jobs are completely unrelated.** If you `srun` a shell with `--mem=16G` and then `sbatch` a job asking for 900G, the job gets 900G. It runs on a different machine entirely. Your interactive session is just where you happened to be standing when you typed `sbatch`. You can log out and the batch job carries on.

---

<div align="center">

## Check your environment before you plan anything

</div>

This section is the one I wish had existed. Every one of these commands answers a question that will otherwise cost you hours.

### What Python is available, and does it actually work?

```bash
module avail python3
```

Now the important part. **A lot of the Python modules on CIRC are broken in a specific way: they have no `ssl` module.** No `ssl` means no `pip`. You cannot fix it, because installing it needs root and you do not have root.

So test before you commit to a version:
```bash
module load python3/3.9.18
python3 -V
python3 -c 'import ssl; print("ssl OK:", ssl.OPENSSL_VERSION)'
```

If you get `ModuleNotFoundError: No module named '_ssl'`, that version is useless for installing anything. Try another.

To test every version at once (this is worth running once and keeping the output):
```bash
for M in $(module -t avail python3 2>&1 | grep -v '^/' | grep -v ':$' | sort -u); do
  R=$( module load "$M" >/dev/null 2>&1 && python3 -c 'import ssl,sys;print(".".join(map(str,sys.version_info[:3])),"ssl OK",ssl.OPENSSL_VERSION)' 2>&1 | tail -1 )
  printf '%-26s %s\n' "$M" "$R"
done
```

On BlueHive3 when I ran this, only some had `ssl`:
```
python3/3.9.18    3.9.18  ssl OK OpenSSL 3.0.7
python3/3.11.0    ModuleNotFoundError: No module named '_ssl'
python3/3.14.2    3.14.2  ssl OK OpenSSL 3.0.7
```

**My old notes say "3.12.4 is the only version with ssl". Ignore that.** It was true for the old BlueHive module tree and it is not true here. Run the loop and find out for yourself, because it changes between environments and probably over time too.

### Pick your Python version *after* checking what your packages need

This is the bit that actually bit me. A working `ssl` is necessary but not sufficient — the version also has to be one your packages publish builds for. Pinned scientific packages are often only built for a narrow range of Python versions, and if you pick one outside it, `pip` either fails outright or quietly compiles from source for twenty minutes and then fails.

So the order is: find which versions have `ssl`, then pick the one your project's requirements actually support. Not the newest.

You can check what a package supports before installing anything:
```bash
pip index versions torch
pip download --no-deps --only-binary=:all: --python-version 3.9 torch==2.1.2 -d /tmp/x
```
If the second one fails, there is no prebuilt package for that combination and you should change Python version rather than fight it.

### Other things worth checking

```bash
module avail gcc                       # compilers, if you need to build anything
module avail | grep -i conda           # miniforge/conda, if you prefer it to venv
module list                            # what is loaded right now
```

**Do not run `module purge` casually.** SLURM's own commands come from a module that is loaded for you automatically. Purge it and `sbatch`, `squeue` and `sinfo` all vanish with a confusing "command not found", and you will think the cluster is broken. I did exactly this.

### How much disk do I have?

```bash
mmlsquota --block-size auto gpfs2:scratch
```

**Do not use `df` for this.** It reports the entire shared university filesystem, so it will tell you something like `4.2P, 94% full` no matter what you personally are using. It is not about you. Your real numbers are printed in the banner when you log in:

```
Filesystem    GB used  GB soft limit  GB hard limit  status
     /home       0.57          20.00          25.00  under soft limit
  /scratch      11.01         200.00        1000.00  under soft limit
```

### Two storage areas, and what goes where

`/home/YourNetIDHere` — about 20 GB. **Put your virtual environments here.** It is backed up.

`/scratch/YourNetIDHere` — 200 GB soft limit, 1000 GB hard. **Put your data and results here.** It is *not backed up*, so download anything you care about once the job finishes.

Home is shared across every node, so a virtual environment you install from one compute node works from any other. You are not installing "into" the machine you are sitting on.

### What hardware exists, and what am I allowed to use?

```bash
# partitions, with memory in GB instead of SLURM's MB
sinfo -h -o "%P|%D|%l|%c|%m" | awk -F'|' '
  BEGIN { printf "%-18s %5s %14s %6s %10s\n", "PARTITION","NODES","TIMELIMIT","CPUS","MEMORY" }
  { m=$5; p=""; if (m ~ /\+$/) { p="+"; sub(/\+$/,"",m) }
    printf "%-18s %5s %14s %6s %9.0f G%s\n", $1,$2,$3,$4, m/1024, p }' | sort -u
sacctmgr -np show assoc user=$USER format=Account,Partition,QOS
```

The first shows what exists. The second shows what *you* can submit to, which is a much shorter list. Run the second one first — there is no point planning around a partition you cannot use.

---

<div align="center">

## Getting your files onto the cluster

</div>

CIRC has *rsync*, which beats *scp* for anything big because it can resume. Install rsync on your own machine too.

Upload a whole folder:
```bash
rsync -av --info=progress2 --partial /local/folder YourNetIDHere@bluehive3.circ.rochester.edu:/scratch/YourNetIDHere/
```

Upload everything *inside* a folder — note the trailing slash on the source, this catches everyone:
```bash
rsync -av --info=progress2 --partial /local/folder/ YourNetIDHere@bluehive3.circ.rochester.edu:/scratch/YourNetIDHere/destination/
```

Download your results — same thing, arguments reversed:
```bash
rsync -av --info=progress2 --partial YourNetIDHere@bluehive3.circ.rochester.edu:/scratch/YourNetIDHere/results/ /local/destination/
```

What the flags do:
- `--info=progress2` gives **one overall progress line** with percentage, speed and ETA. Without it you get a wall of filenames and no sense of how far along you are.
- `--partial` keeps what has already transferred if the connection drops. Run the same command again and it picks up where it stopped. On a 10 GB upload over VPN you will need this.

If you prefer *scp*, it works, it just cannot resume:
```bash
scp /local/file YourNetIDHere@bluehive3.circ.rochester.edu:/scratch/YourNetIDHere/
scp -r /local/folder YourNetIDHere@bluehive3.circ.rochester.edu:/scratch/YourNetIDHere/
```

### About `-z`

You will see `rsync -avz` recommended everywhere, including in my own older notes. `-z` compresses during transfer, and whether you want it depends entirely on what you are sending:

- Sending `.tsv`, `.csv`, `.json`, code? **Use `-z`.** Text compresses hugely and it is a real speedup.
- Sending `.zip`, `.gz`, `.parquet`, images, video? **Leave `-z` off.** That data is already compressed, so `-z` burns CPU on both ends for nothing and can end up *slower* than not using it.

If you have the choice, zip your data locally, send the zips without `-z`, and unzip on the cluster. For one of my datasets that turned a 44 GB upload into a 12 GB one.

### Stop Duo from asking every single time

Every `ssh`, `scp` and `rsync` fires its own Duo push to your phone. Move a few folders and you are tapping your phone constantly.

Put this in `~/.ssh/config` **on your own computer** (this is a config file, not commands to type at the prompt):

```
Host bluehive3
    HostName bluehive3.circ.rochester.edu
    User YourNetIDHere
    ControlMaster auto
    ControlPath ~/.ssh/cm-%r@%h:%p
    ControlPersist 8h
```

Connect once and approve one push:
```bash
ssh bluehive3
```

Now every command reuses that one connection with no authentication, and it is faster too:
```bash
rsync -av --info=progress2 --partial /local/folder bluehive3:/scratch/YourNetIDHere/
ssh bluehive3 'squeue -u YourNetIDHere'
```

The `8h` is an idle timeout, not a fixed window — every command resets it, so if you keep working it stays alive. Rebooting, sleeping, or connecting/disconnecting the VPN will break it and cost you one more push.

If you want to write any script that talks to the cluster more than once, you basically need this.

---

<div align="center">

## Python environment

</div>

Do this **in an interactive session, not on the login node** — pip downloads can be large and unpack thousands of files.

```bash
srun -p interactive -t 2:00:00 -c 4 --mem=16G --pty bash
hostname          # confirm you moved
```

Then:
```bash
module load python3/3.9.18                    # a version you confirmed has ssl
python3 -m venv ~/myvenv/myproject
source ~/myvenv/myproject/bin/activate
pip install --upgrade pip wheel
pip install -r requirements.txt
```

You cannot use `virtualenv` or `pipx` because installing them needs root. The built-in `venv` module works fine and needs nothing.

**Always check what actually installed:**
```bash
pip list | head -20
python -c "import numpy, pandas; print(numpy.__version__, pandas.__version__)"
```

If a version is not what you asked for, or the install took surprisingly long, pip probably compiled from source instead of using a prebuilt package — which usually means your Python version is wrong for those packages. Fix that now rather than discovering it six hours into a job.

Every new shell needs both lines again:
```bash
module load python3/3.9.18
source ~/myvenv/myproject/bin/activate
```
Put them in your job scripts too. A batch job starts in a fresh shell and knows nothing about what you loaded earlier.

### Using it in JupyterLab

If you want this environment as a Jupyter kernel:
```bash
source ~/myvenv/myproject/bin/activate
pip install ipykernel
python -m ipykernel install --user --name=myproject --display-name="Python 3.9 (myproject)"
```
Keep `--name` the same as the folder before `/bin/activate`. `--display-name` can be anything. To remove it later: `jupyter kernelspec uninstall myproject`.

---

<div align="center">

## SLURM: what a partition actually is

</div>

Every example online tells you to put `-p standard` or `-p gpu` in your script and never explains what that means.

A **partition** is a named group of compute nodes with its own rules: how long you may run, who is allowed in, and whether somebody can take the machine back off you mid-job. It is **not** a hardware description. Two partitions can hold identical machines and behave completely differently.

```bash
# partitions, with memory in GB instead of SLURM's MB
sinfo -h -o "%P|%D|%l|%c|%m" | awk -F'|' '
  BEGIN { printf "%-18s %5s %14s %6s %10s\n", "PARTITION","NODES","TIMELIMIT","CPUS","MEMORY" }
  { m=$5; p=""; if (m ~ /\+$/) { p="+"; sub(/\+$/,"",m) }
    printf "%-18s %5s %14s %6s %9.0f G%s\n", $1,$2,$3,$4, m/1024, p }' | sort -u
sacctmgr -np show assoc user=$USER format=Account,Partition,QOS    # what you can use
```

### standard vs preempt

**standard** is CIRC's own hardware. You queue, you wait your turn, and once your job starts **nobody can take the node away** until you finish or hit the time limit. On BlueHive3 that is 15 nodes and a 5-day limit.

**preempt** is the interesting one. Look at the partition list and you will see names like `colala`, `femto`, `luna`, `atmos`, `polariton`, `ising`. Those belong to **individual labs who bought their own machines** — and most of the time those machines sit idle.

`preempt` lets anyone use that idle hardware. That is why it is 57 nodes and 3708 cores, more than three times the size of `standard`.

The catch is in the name. **If the lab that owns the node submits a job, yours gets kicked off.** Could be ten minutes in, could be nineteen hours in.

|                    | standard   | preempt       |
| ------------------ | ---------- | ------------- |
| nodes / cores      | 15 / 1136  | **57 / 3708** |
| time limit         | **5 days** | 2 days        |
| typical queue wait | longer     | shorter       |
| can you be kicked  | no         | **yes**       |

### The thing that will actually catch you out: memory

**The big memory is not in `standard`.** On BlueHive3 `standard` tops out around **493 GB** per node. Ask for more and your job does not queue slowly and eventually run — it never runs at all. `reserved` is the same.

Every node with 1 TB or more belongs to some lab, which means **the only way to reach one is through `preempt`**.

And critically: **memory is per node, and is never pooled.**

```bash
#SBATCH --nodes=4
#SBATCH --mem=250G
```
That is 250 GB on *each* of four machines, not 1 TB spread across them. RAM lives inside one physical box and a process on one node cannot touch memory in another. Splitting work across nodes only helps if your program was written for it (MPI, Dask, Spark). Ordinary Python using `multiprocessing` **cannot span nodes at all**, so whatever it needs must fit in a single machine.

So check before you plan:
```bash
# per node, in GB, with a FREE column -- this is the number that decides
# whether your job can start. Change the partition name to whichever you want.
sinfo -h -p standard -N -O "NodeHost:20,Memory:12,AllocMem:12,CPUs:8,StateLong:14" | awk '
  BEGIN { printf "%-16s %9s %9s %9s %6s %s\n","NODE","TOTAL","ALLOC","FREE","CPUS","STATE" }
  { printf "%-16s %8.0fG %8.0fG %8.0fG %6s %s\n", $1, $2/1024, $3/1024, ($2-$3)/1024, $4, $5 }' \
  | sort -rn -k4 | head
```

`TOTAL` is the node's memory, `ALLOC` is what SLURM has already promised to other jobs, and **`FREE` is what you could actually get right now** — that is the column to compare your `--mem` against. Sorted so the emptiest node is at the top.

The `awk` is only there to divide by 1024 and work out `FREE`, because `sinfo` reports everything in MB and has no option to do either.

If you drop the `awk` and read the raw `sinfo` output, ignore its `FreeMem` column — that is OS-level free memory including cache, and it will happily claim a node has 900 GB free when SLURM has already committed all of it. `Memory - AllocMem` is the real answer.

Also watch the state: a node marked `allocated` has **no free CPUs**, so it cannot take your job even if its memory looks free.

### If you use preempt, find out what happens when you get kicked

```bash
scontrol show partition preempt | grep -iE "PreemptMode|GraceTime"
```

On BlueHive3 this currently says `PreemptMode=REQUEUE` and `GraceTime=0`.

**REQUEUE is good** — SLURM does not kill your job, it puts it back in the queue automatically and runs it again later.

**GraceTime=0 is the nasty part** — you get *no warning*. There is no "you have five minutes to save your work" signal, so trapping a signal to checkpoint on the way out does not work here.

Together those give you one rule: **a requeued job restarts from line one, so it must be able to work out what it already finished by looking at the disk.** If your script just runs everything top to bottom, being kicked means starting from scratch — and the evil part is that this looks exactly like a job that is merely slow. You can lose a day to it without ever seeing an error.

The fix is to break the work into steps, write a small marker file after each step succeeds, and skip any step whose marker already exists. There is an example below.

And do not forget:
```bash
#SBATCH --open-mode=append
```
By default a requeued job **overwrites its own log**, so everything from before you were kicked disappears.

---

<div align="center">

## How to request resources

</div>

A job script is a normal bash script where the lines starting `#SBATCH` are your resource request. SLURM reads them, finds a machine that fits, and runs the rest.

```bash
#SBATCH --job-name=myjob           # what shows up in squeue
#SBATCH --partition=standard       # which pool of nodes
#SBATCH --time=1-12:00:00          # D-HH:MM:SS. Killed at this point, no mercy
#SBATCH --nodes=1                  # how many machines
#SBATCH --cpus-per-task=16         # cores on that machine
#SBATCH --mem=64G                  # memory PER NODE
#SBATCH --output=logs/%x_%j.log    # %x = job name, %j = job id
```

The ones people get wrong:

**`--time`** — your job is killed the second it expires, mid-write, no warning. Overestimate. But not wildly, because shorter jobs are easier to schedule and start sooner.

**`--mem`** — per node, not total. Asking for more than any single machine has means the job never runs. Asking too little means the kernel kills you partway through with an unhelpful message.

**`--cpus-per-task`** — asking for more cores than you use does not speed anything up, it just makes you queue longer.

**`--output`** — by default stdout and stderr go to different files, which is maddening because Python's logging writes to stderr. Setting only `--output` (and *not* `--error`) sends both to one file in the right order. Use `%j` so a second run does not overwrite the first.

**Requesting memory is also how you choose hardware.** `--mem=900G` means only nodes with at least that much can take you, so SLURM automatically picks a big one. This is much better than hardcoding `--nodelist=bhg[0012-0018]`, which is what my old notes did — node lists go stale every time CIRC changes hardware, and you end up queueing forever for machines that no longer exist.

---

<div align="center">

## Example job scripts

</div>

### 1. The simplest thing that works

Save as `hello.slurm`, submit with `sbatch hello.slurm`.

```bash
#!/bin/bash
#SBATCH --job-name=hello
#SBATCH --partition=standard
#SBATCH --time=00:05:00
#SBATCH --nodes=1
#SBATCH --cpus-per-task=1
#SBATCH --mem=1G
#SBATCH --output=logs/hello_%j.log

hostname
date
echo "I am job $SLURM_JOB_ID with $SLURM_CPUS_PER_TASK cores"
```

Run this first. If it works, your account, your partition and your paths are all fine, and anything that breaks later is your own code.

`logs/` must already exist — SLURM will not create it, and the job fails silently with no log to tell you why. `mkdir -p logs` first.

### 2. A Python job

```bash
#!/bin/bash
#SBATCH --job-name=myanalysis
#SBATCH --partition=standard
#SBATCH --time=08:00:00
#SBATCH --nodes=1
#SBATCH --cpus-per-task=8
#SBATCH --mem=32G
#SBATCH --output=logs/myanalysis_%j.log

module load python3/3.9.18
source ~/myvenv/myproject/bin/activate

# One BLAS thread per process. Without this, numpy/scipy each spawn a full
# thread pool and you use far more cores than SLURM gave you, which is both
# slower and rude to everyone else on the machine.
export OMP_NUM_THREADS=1
export OPENBLAS_NUM_THREADS=1
export MKL_NUM_THREADS=1
export NUMEXPR_NUM_THREADS=1

# -u = unbuffered, so `tail -f` shows progress instead of nothing for an hour
python -u myscript.py --input /scratch/$USER/data --output /scratch/$USER/results
```

Those thread variables are not optional decoration. They are the difference between using the 8 cores you asked for and trying to use 64.

### 3. A big-memory job on preempt

```bash
#!/bin/bash
#SBATCH --job-name=bigmem
#SBATCH --partition=preempt
#SBATCH --time=2-00:00:00
#SBATCH --nodes=1
#SBATCH --cpus-per-task=32
#SBATCH --mem=900G
#SBATCH --requeue
#SBATCH --open-mode=append
#SBATCH --output=logs/bigmem_%j.log
#SBATCH --mail-type=BEGIN,END,FAIL,REQUEUE
#SBATCH --mail-user=YourNetIDHere@u.rochester.edu

module load python3/3.9.18
source ~/myvenv/myproject/bin/activate
export OMP_NUM_THREADS=1 OPENBLAS_NUM_THREADS=1 MKL_NUM_THREADS=1 NUMEXPR_NUM_THREADS=1

python -u bigjob.py
```

`--mem=900G` is doing two things: guaranteeing the memory, and restricting you to the handful of nodes that large. Expect to wait. `--mail-type=BEGIN` is worth it precisely because that wait is unpredictable — you get an email the moment it starts.

### 4. Surviving preemption

If you use `preempt` for anything long, structure it so a restart can skip finished work:

```bash
#!/bin/bash
#SBATCH --job-name=resumable
#SBATCH --partition=preempt
#SBATCH --time=2-00:00:00
#SBATCH --nodes=1
#SBATCH --cpus-per-task=16
#SBATCH --mem=200G
#SBATCH --requeue
#SBATCH --open-mode=append
#SBATCH --output=logs/resumable_%j.log

module load python3/3.9.18
source ~/myvenv/myproject/bin/activate

OUT=/scratch/$USER/results
mkdir -p $OUT/_done
echo "restart number: ${SLURM_RESTART_COUNT:-0}"

for STEP in load clean model report; do
  if [ -f "$OUT/_done/$STEP.ok" ]; then
    echo "$STEP already done, skipping"
    continue
  fi
  python -u pipeline.py --step $STEP --outdir $OUT || exit 1
  touch "$OUT/_done/$STEP.ok"      # only written if the step succeeded
done
```

Get kicked during `model` and you lose `model`, not the whole run. The marker is written **only on success**, so a failed step runs again next time.

### 5. Many jobs from one script (job arrays)

Running the same code over different inputs is what arrays are for:

```bash
#!/bin/bash
#SBATCH --job-name=sweep
#SBATCH --partition=standard
#SBATCH --time=04:00:00
#SBATCH --nodes=1
#SBATCH --cpus-per-task=4
#SBATCH --mem=16G
#SBATCH --output=logs/sweep_%A_%a.log

INPUTS=(dataset_a dataset_b dataset_c dataset_d)
THIS=${INPUTS[$SLURM_ARRAY_TASK_ID]}

module load python3/3.9.18
source ~/myvenv/myproject/bin/activate
python -u myscript.py --input $THIS
```

```bash
sbatch --array=0-3 sweep.slurm       # all four at once
sbatch --array=0-3%2 sweep.slurm     # four total, at most 2 running at a time
```

`%A` is the array job id, `%a` is the task number, so each task gets its own log.

**The `%2` throttle matters more than you expect.** Every task is a full independent job with its own memory and its own disk usage. Twenty tasks each writing 50 GB will blow through your scratch quota long before you run out of cores. Disk is usually the real limit, not CPU.

---

<div align="center">

## Watching your jobs

</div>

```bash
squeue -u $USER -o "%.10i %.11P %.12j %.2t %.11M %.11L %R"
```
`%.2t` is state: `PD` pending, `R` running, `CG` finishing. `%.11M` time used, `%.11L` time left, `%R` the node, or the reason it is still waiting.

**Why is it still pending:**
```bash
squeue -u $USER -t PD -o "%.10i %r %S"
```
- `Resources` — SLURM accepted your request and is waiting for a machine big enough. Normal.
- `Priority` — other jobs are ahead of you.
- `QOSMaxJobsPerUserLimit` — you have hit a limit on simultaneous jobs.

```bash
squeue --start -j JOBID       # SLURM's guess at when it will start
scontrol show job JOBID       # everything about one job
```

**Watch the log as it runs:**
```bash
tail -f logs/myjob_12345.log
```
If that shows nothing for a long time, check you used `python -u`. Python buffers its output when writing to a file, so a perfectly healthy job can look completely frozen.

**Cancel:**
```bash
scancel JOBID
scancel -u $USER              # everything of yours
scancel --name=myjob
```

**After it finishes — how long did it take and how much memory did it really use:**
```bash
sacct -j JOBID --format=JobID%16,JobName%14,State%12,Elapsed%11,MaxRSS%11,ReqMem%9,ExitCode
seff JOBID
```

**`MaxRSS` is the most useful number on this page.** It is what your job actually used. Run something once, look at `MaxRSS`, then set `--mem` to that plus some headroom. Guessing means either being killed halfway through or queueing for memory you never touch.

`seff` also shows CPU efficiency. If you asked for 32 cores and it reports 4%, you are queueing for cores you do not use and everyone loses.

**Wait for a job to start, without staring at the terminal:**
```bash
JOB=12345
while squeue -h -j $JOB -t PD | grep -q .; do sleep 300; done; echo -e "\a JOB $JOB STARTED"
```
Run it inside `tmux` (`tmux new -s wait`, detach with `Ctrl-b` then `d`) so a dropped SSH connection does not kill it.

---

<div align="center">

## When things go wrong

</div>

**`ModuleNotFoundError: No module named '_ssl'`** — that Python module has no ssl and cannot pip. Use one that passed the test above.

**pip spends twenty minutes compiling, then fails** — no prebuilt package exists for your Python version. Change Python version, do not fight the compiler.

**`sbatch: command not found`** — you ran `module purge`. Log out and back in.

**Job disappears immediately, no log file** — the `logs/` directory does not exist. `mkdir -p logs`.

**Job pending forever on `Resources`** — nothing free is big enough. Compare your request against `Memory - AllocMem` from the `sinfo` command above. If nothing ever comes close, lower the request.

**`sbatch: error: Requested node configuration is not available`** — you asked for more than any node in that partition physically has. Memory does not pool across nodes.

**Job killed partway, `oom-kill` or exit code 137** — out of memory. Raise `--mem`, or use fewer parallel workers.

**Job restarted from the beginning by itself** — that is `preempt` working as designed. Check `SLURM_RESTART_COUNT` and see the resumable example above.

**Log file lost its beginning** — a requeued job overwrote it. Add `--open-mode=append`.

**`df` says the disk is 94% full** — it is talking about the whole university filesystem, not you. Use `mmlsquota` or read the login banner.

---

If something here is wrong or out of date, tell me. Most of it came from getting it wrong first.
