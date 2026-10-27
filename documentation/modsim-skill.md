# Create your own ModSim skill

Most domain scientists already have a modeling and simulation (ModSim) code that runs on a
cluster: a solver, a training script, an analysis pipeline, with a batch script that
works. This tutorial shows how to turn that code into a **VISTA skill**, so that you, or
a colleague, can run it from the chat panel: "run my melt simulation on Frontier at
1200 K", and get the results back in the conversation.

There are two ways to do it:

| | You start with | Who writes the skill | Worked example |
|---|---|---|---|
| **Path 1. Bring your own skill** | your code, plus a coding agent (Claude Code, Cursor, …) | you and your agent, outside VISTA | AMD **Primus** LLM training on **Lux** |
| **Path 2. Let VISTA write it** | a repository on GitHub or GitLab, or a zip of a folder | VISTA, in chat | **mpi4py** "hello world" on **Frontier** |

Both paths end the same way: you import the skill into a project, and VISTA submits its
job to the cluster. Neither needs any change to VISTA itself.

## Before you start

You need:

- **A VISTA project** to add the skill to. Any project works; you can also make a new
  one, as in [Quick Start](quick-start.md).
- **Access to the cluster your code runs on**, connected in VISTA:
    - **Lux** (Path 1's example): an OLCF account on Lux, your RSA SecurID token, and your
      Lux account set in **Settings → Compute**. Jobs are submitted over SSH, so you log in
      when VISTA first contacts Lux.
    - **Frontier** (Path 2's example): an S3M token, and Globus connected for Frontier,
      both in **Settings → Compute**.

!!! note "Where these features are available"
    Skills that carry their own HPC jobs, and the `hpc-skill-author` skill, are new. If
    **Import…** on the **Skills** page has no **From project output** tab, or the Skill Hub
    has no `hpc-skill-author`, your VISTA installation is older. Ask your administrator to
    update it.

## What a ModSim skill is

A skill is a folder. Its `SKILL.md` tells VISTA's agent what the skill is for and how to
use it. A ModSim skill also carries one or more **jobs** under `hpc_jobs/`, each one what
VISTA needs to run your code on a cluster:

```
my-skill/
├── SKILL.md                   # when to use it, the job's name, its arguments, how to read the results
└── hpc_jobs/
    └── my-job/
        ├── README.md          # starts with "# my-job"; what the job does, its arguments and outputs
        ├── cluster_defaults.json   # nodes, walltime, queue, environment, per cluster
        ├── job.frontier.slurm      # the batch script body, one per cluster: frontier, lux, odo, perlmutter
        ├── setup_frontier.sh       # optional setup before the job
        └── ...                     # your small scripts and configs, uploaded with the job
```

When a project has the skill loaded, the agent runs the job as `<skill>/<job>`, for
example `my-skill/my-job`. Five rules cover almost everything:

- **VISTA writes the `#SBATCH` header.** Nodes, walltime and queue go in
  `cluster_defaults.json`, not in the script. Leave out `-A`, `-o`, `-e` and `-J`: VISTA
  fills in your account and its own log paths.
- **Write results to `$VISTA_OUT`.** It is the run's own output folder, already created,
  and the only place VISTA fetches files from.
- **Your files arrive in `$RUN_DIR_<Cluster>`** (`$RUN_DIR_Frontier`, `$RUN_DIR_Lux`, …).
  Everything in the job folder is uploaded there, except the README, `cluster_defaults.json`
  and the job and setup scripts. Ship sources and configs; download models and build
  environments on the cluster.
- **Shared state goes in `$VISTA_JOB_DIR`** (Lux and Frontier): a checkout, a build, an
  environment that every run of the job reuses.
- **Arguments are `KEY=VALUE` pairs from a fixed list.** What you want to vary per run
  (a temperature, a config file, an iteration count) becomes a key the job script accepts.
  The job refuses any other key, so a mistake fails at once instead of being silently
  ignored.

The full contract is the **Self-contained HPC skills** section of VISTA's
[HPC jobs guide](reference/hpc-jobs.md#self-contained-hpc-skills).

## Path 1. Bring your own skill

Use this path when you want full control over the skill, or your code needs care that
is easier to give in your own editor: a large launcher, a container, cluster-specific
networking.

### 1. Reformat your code as a skill

Open your code's repository with a coding agent and point it at the contract. A prompt
like this works well:

```
Reformat this repository as a VISTA skill that ships its own HPC job, following the
"Self-contained HPC skills" section of
https://github.com/Genesis-VISTA/vista/blob/main/docs/hpc-jobs.md
The job runs <what it runs> on <cluster>. Keep my existing scripts working by hand,
and make them use the job folder instead of duplicating it.
The values I vary per run are <list>; make them KEY=VALUE arguments and refuse others.
```

Before importing, check the result yourself:

- The skill folder's name equals `name:` in `SKILL.md`.
- Each job's `README.md` starts with `# <job>`, and lists the arguments and outputs.
- `cluster_defaults.json` has a section for every cluster that has a `job.<cluster>.slurm`.
- The job script has no `#SBATCH` lines that matter, and writes its results under `$VISTA_OUT`.
- `SKILL.md` names the job as `<skill>/<job>` and shows an example submission.
- No credentials, `.env` files or keys are in the folder.

Push the skill to GitHub (or keep it as a local folder).

### 2. Import it into VISTA

1. Open the project, then select **Skills** in the sidebar.
2. Select **Import…**.
3. Choose **GitHub URL** and paste the repository address. For a skill inside a larger
   repository, paste the address of its folder (`…/tree/<branch>/<path>`). Or choose
   **Local folder** and pick the folder on your computer.
4. Select **Import**.

The skill is imported as private, loaded into the project, and its card shows a **Runs HPC
jobs: …** badge naming its jobs. It is available from your next message.

### Example: Primus LLM training on Lux

[`lux-primus-skill`](https://github.com/jqyin/lux-primus-skill) runs AMD's **Primus**
training framework (with a Megatron backend) in an apptainer container on Lux's MI355X
GPUs. Its job, `primus-smoke`, trains Mistral-7B-v0.3 on mock data for 20 iterations:
a short run that measures throughput and confirms training works before a longer one.

It started as hand-run scripts and was reformatted with a coding agent, as in step 1.
Everything that runs on Lux now lives in one job folder:

```
lux-primus-skill/                 (name: primus-on-lux)
├── SKILL.md
├── env.lux.sh, scripts/          # running by hand still works, from the same files
└── hpc_jobs/primus-smoke/
    ├── README.md
    ├── cluster_defaults.json     # 1 node, 8 GPUs, exclusive, partition batch, 30 min; the image path
    ├── setup_lux.sh              # login node: clone and pin Primus, build its helpers, once
    ├── job.lux.slurm             # torchrun in the container; KEY=VALUE arguments; summary to $VISTA_OUT
    ├── build_image.sh, build_helpers.sh, summarize.sh
    └── *.yaml                    # model and experiment configs
```

**Import it.** On the **Skills** page, select **Import…** → **GitHub URL**, paste
`https://github.com/jqyin/lux-primus-skill`, and select **Import**. The card for
`primus-on-lux` shows **Runs HPC jobs: primus-on-lux/primus-smoke**.

**Run it.** In the chat panel:

```
run the Primus smoke test on Lux on 1 node. Confirm with me before submitting.
```

VISTA reads the skill and proposes the submission: job `primus-on-lux/primus-smoke`,
cluster `lux`, one node. Reply **yes**, or change it first, for example "use 2 nodes with
GLOBAL_BATCH_SIZE=512". The arguments the job accepts are:

| Argument | Default | Notes |
|---|---|---|
| `CONFIG_NAME` | `smoke-mistral_7B-bf16-mock.yaml` | also `smoke-qwen2.5_3B-bf16-mock.yaml` |
| `GLOBAL_BATCH_SIZE` | 256 | keep 256 × nodes to compare node counts fairly |
| `TRAIN_ITERS` | 20 | |

Two login prompts follow, as in the [LLM modeling](llm-modeling.md#logging-in-to-lux)
tutorial: first `hub.ccs.ornl.gov`, then the Lux login node, each with a **new**
passcode. VISTA reuses the login for the rest of the conversation.

Before `sbatch`, the job's `setup_lux.sh` runs on the Lux login node. The first time,
it clones Primus and builds a helper library inside the container, which takes a few
minutes; later submissions find both already there. If the setup fails, VISTA shows its
error instead of submitting. Then VISTA reports the job id and where its log and outputs
are on the cluster.

!!! note "The container image"
    The job uses a shared Primus image on Lustre and only reads it. If you cannot read it,
    the job builds your own copy in your own job folder on its compute node first. That
    takes about 30 minutes, so give the first run a walltime of at least an hour ("use a
    duration of 1:00:00"). A job with too little time left stops within seconds and says
    so. Later runs reuse your copy.

**Read the result.** When the job has left the queue:

```
the job is done. Show me summary.txt and tell me if the run was healthy.
```

VISTA fetches `summary.txt` from the job's output folder: the first, third and last
iteration lines, with loss, step time and TFLOP/s per GPU, and the exit status. The full
log is in `train.log` beside it.

- **A healthy run exits 0, with 20 of 20 iterations and a falling loss.** On mock data the
  model memorizes its input, so a loss near 0 says nothing about quality.
- **Ignore iterations 1 and 2.** They include one-time compilation.
- **Expect about 900 TFLOP/s per GPU on 1 node**, and about 800 on 2 nodes at
  `GLOBAL_BATCH_SIZE=512`. Compare node counts only at the same per-GPU batch: halving it
  costs about a quarter of the throughput on its own.
- **No new iteration for 5 minutes after the first means a hang.** It has been seen once
  and the identical resubmission passed. Ask VISTA to cancel the job and resubmit it.

!!! tip "From smoke test to continual training"
    The smoke test is the base for longer runs. To continue training a model on your own
    data, add an experiment config that loads a checkpoint and points at your dataset, and
    accept its settings as new arguments in `job.lux.slurm`. The skill's `SKILL.md`
    explains how to add a model or experiment, and which settings reach the container.

## Path 2. Let VISTA write the skill

Use this path when your code already runs on a cluster and you would rather describe it
than write the job yourself. VISTA's `hpc-skill-author` skill reads your code, asks what
it cannot work out, writes the skill, checks it, and hands it to you to import.

### 1. Load the hpc-skill-author skill

Select **Skill Hub** in the sidebar, search for `hpc-skill-author`, and select **Load**.
It joins the project's skills.

### 2. Point VISTA at your code

In the chat panel, give it the code and the cluster:

```
make an HPC skill that runs <your code> from <repository URL> on <cluster>
```

The code can be:

- **A repository on github.com or gitlab.com**, or a folder in one
  (`…/tree/<branch>/<path>`). VISTA downloads it and pins it to the commit it got, so the
  skill keeps running the code you checked. Private repositories work only when your VISTA
  deployment has a token for that host.
- **A zip of a local folder**, uploaded to the chat. Use this for code on any other host,
  or code that is not in git.

VISTA then:

1. **Takes an inventory** of the code: READMEs, build files, existing batch scripts and
   their `#SBATCH` lines, entry points and their flags.
2. **Proposes the job**: the one command it runs, which values become arguments, what
   software it needs, how many nodes and how long, which files it writes. It asks you to
   confirm, and asks only what it could not tell from the code.
3. **Asks about the software environment.** If one is already built on the cluster (a
   conda environment, a container, a module), the job just activates it. Building one
   inside every job wastes queue time, so say so if you have one.
4. **Writes the skill** into the project's output folder, translating your batch script
   rather than calling it: resources move to `cluster_defaults.json`, and hard-coded output
   paths become `$VISTA_OUT`.
5. **Checks it** with a validator that applies the same rules VISTA applies when you import
   the skill and submit the job, and fixes what it finds.
6. **Tells you how to import it.** VISTA cannot import or load a skill by itself: loading a
   skill lets it run jobs under your credentials, so that step is yours.

### 3. Import it and run it

1. On the **Skills** page, select **Import…** → **From project output**.
2. Choose the skill VISTA wrote. Its **Runs HPC jobs: …** badge names its jobs.
3. Select **Import**.

From your next message, ask VISTA to run the job.

To change the skill later, ask VISTA in the same chat ("make the rank count an argument").
It edits the files and checks them again; import again, choosing the same skill, and select
**Replace**.

### Example: mpi4py on Frontier

OLCF's [New User Quick Start](https://github.com/olcf/NewUserQuickStart) includes a Python
exercise for Frontier: `hello_mpi.py`, which prints a line from each MPI rank, and
`submit_hello.sbatch`, which runs it on 42 ranks of one node in a conda environment with
`mpi4py`.

**Ask for the skill.** With `hpc-skill-author` loaded:

```
make an HPC skill that runs the mpi4py hello world from
https://github.com/olcf/NewUserQuickStart/tree/master/Frontier/python_hands-on
on Frontier
```

VISTA downloads the folder, reads both files, and proposes a job along these lines:

> Run `srun -n42 python3 -u hello_mpi.py` on 1 Frontier node for 10 minutes, in the batch
> queue, with `PrgEnv-gnu` and `miniforge3` loaded. The batch script activates a shared
> training environment, `/lustre/orion/stf007/world-shared/new-user-training/mpi4py_env`.
> Should the job use that environment, or one of your own? Should the number of ranks be
> an argument?

For a first run, use the shared environment, and make the rank count an argument:

```
use the shared environment, and make the number of ranks an argument with default 42
```

VISTA writes a skill such as `mpi4py-hello-frontier` with one job. The names it picks may
differ:

```
mpi4py-hello-frontier/
├── SKILL.md                  # the job, its NRANKS argument, how to read the output
└── hpc_jobs/hello/
    ├── README.md
    ├── cluster_defaults.json # frontier: 1 node, 600 s, batch; CONDA_ENV = the shared environment
    ├── job.frontier.slurm    # load modules, activate CONDA_ENV, srun -n "$NRANKS", output to $VISTA_OUT
    └── hello_mpi.py          # uploaded with the job
```

It runs the validator, then tells you to import the skill.

**Import and run it.** Select **Skills** → **Import…** → **From project output**, choose
`mpi4py-hello-frontier`, and select **Import**. Then:

```
run mpi4py-hello-frontier/hello on Frontier. Confirm with me before submitting.
```

After you confirm, VISTA uploads `hello_mpi.py` through Globus, submits the job and
reports its id. When it has finished:

```
the job is done. How many ranks reported, and were any missing?
```

VISTA fetches the output and counts the `Hello from MPI rank N !` lines.

- **Expect one line per rank, 0 to 41, in no particular order.** The ranks print
  independently, so the lines are interleaved. Check the count, not the order.
- **A missing rank, or fewer lines than ranks, means something failed.** Ask VISTA to show
  the error output: a Python traceback there usually means the environment did not
  activate, or `mpi4py` was built against a different MPI.
- **Use your own environment for real work.** The shared environment is for training. The
  Quick Start explains how to build your own; tell VISTA its path, and it puts it in
  `cluster_defaults.json`.

!!! note "Frontier jobs do not run as you"
    On Frontier, VISTA's jobs run as your project's automation user, which can read the
    uploaded files but not change them. That is why the job writes everything under
    `$VISTA_OUT`, and why an environment it uses must be readable by your project.

## What to be careful about

- **Review a skill before loading it.** A loaded skill's jobs run on the cluster under
  your credentials and are charged to your allocation. The **Runs HPC jobs** badge tells
  you a skill has them; read its `hpc_jobs/` folder before loading a skill you did not
  write.
- **Ask VISTA to confirm before it submits.** Check the job name, cluster, nodes, walltime
  and arguments it proposes before saying yes.
- **Name the cluster in your request.** A job runs only on the clusters its skill has a
  script for.
- **Keep the job small and the skill honest.** One job that works end to end is better than
  a general one. Make `SKILL.md` say what a healthy result looks like, so VISTA can tell you
  when a run went wrong.
- **Pin what the job runs.** A job that clones the latest code on every run can change
  under you. Pin a commit or tag, and change it on purpose.

## Getting help

The full job contract, including every cluster's settings, is in VISTA's
[HPC jobs guide](reference/hpc-jobs.md#self-contained-hpc-skills),
and the import paths are in its
[skill onboarding guide](reference/skill-onboarding.md).

[Return to the VISTA homepage](https://genesis-vista.github.io/){ .md-button }
