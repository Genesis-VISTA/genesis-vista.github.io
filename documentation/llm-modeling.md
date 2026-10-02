# LLM modeling with VISTA

This tutorial works through training scientific language models in VISTA using
**FORGE**, a family of GPT-NeoX models pre-trained on scientific literature. You will set
up a project for the work, pre-train a FORGE model from scratch on the **Lux** cluster,
and then fine-tune a FORGE model to predict the **melting point of a molten salt** from
its composition.

Each step is driven by a prompt in the chat panel. VISTA selects the appropriate skill,
submits the work to an HPC cluster, and reports the result back into the conversation.

The three parts build on one another. Part 1 creates the project that parts 2 and 3 run
in. Part 2 runs on Lux and part 3 on Frontier, both at OLCF.

## Before you start

You need:

- **An OLCF account on Lux**,  and your RSA SecurID token. Lux
  jobs are submitted over SSH, so you log in when VISTA first contacts Lux (part 2
  explains the prompts).
- **An S3M token for Frontier**, on your user record, and Globus
  connected for Frontier (**Settings → File transfer**). Part 3 needs both. Without them
  VISTA stops mid-task to ask. Confirm they are set before starting a long step.

!!! note "These jobs take real time"
    A pre-training run holds 16 nodes for up to 30 minutes, after however long it waits
    in the queue; a fine-tuning run holds one node for up to 30 minutes. VISTA returns
    as soon as a job is queued, so you do not have to keep the conversation open. Ask
    for a status update at any point.

## 1. Create the "LLM modeling" project

A project keeps the skills, conversations and results for one line of work together.
This one uses two skills:

| Skill | What it does |
|---|---|
| `llm-pretraining` | pre-trains a FORGE model (forge-s, forge-m or forge-l) on Lux or Frontier, and plots loss and throughput |
| `model-fine-tuning` | fine-tunes a FORGE model to predict molten-salt properties, on Odo, Frontier or Perlmutter |

1. Select **Projects** in the sidebar, then **New project**.
2. Fill in the dialog:
    - **Name:** `LLM modeling`
    - **Description:** for example, *Pre-train FORGE language models and fine-tune them
      for molten-salt property prediction.*
    - **System prompt:** leave blank.
    - **Skills:** check **llm-pretraining** and **model-fine-tuning**.
    - **Knowledge Bases**, **Hypothesis Lab** and **Tools:** leave at their defaults.
3. Select **Create**, then open the project.

Both skills now load automatically in every chat in this project, and appear as
**Required** in the **Skill Hub**.

!!! tip "If a skill is missing from the list"
    `model-fine-tuning` ships with the molten-salt property database it trains on, and
    only appears in deployments set up with VISTA's data package. If it is not in the
    **Skills** list, ask your VISTA administrator. If `llm-pretraining` is missing from an
    older installation, restarting VISTA registers it.

## 2. Pre-train a FORGE model on Lux

Start with the default: the largest model, forge-l, for a short run.

```
pre-train a forge model on Lux
```

VISTA uses the `llm-pretraining` skill. It fills in the defaults and asks you to confirm
before submitting anything:

> Submit forge-l pre-training on Lux: 16 nodes, 0:30:00, 50 iterations, no checkpoint?

Reply **yes**, or change what you want first. For example, "use forge-m and 100
iterations" or "use 32 nodes". The choices are:

| Setting | Default | Notes |
|---|---|---|
| Model | forge-l on Lux (forge-s on Frontier) | forge-s ≈ 1.2B, forge-m ≈ 13B, forge-l ≈ 22B parameters |
| Nodes | 16 (8 GPUs each) | at most 64 |
| Walltime | 30 minutes | at most 4 hours |
| Iterations | 50 | |
| Checkpoints | off | a forge-l checkpoint with optimizer state is hundreds of GB |

!!! note "A short run, not a full pre-training campaign"
    Fifty iterations measure throughput and confirm the model trains: the loss should
    fall steadily. Pre-training to a useful model takes thousands of
    iterations. Use this step to size a longer run.

### Logging in to Lux

Lux's login node is not yet public, so VISTA reaches it through a hub. Two login prompts
appear in turn:

1. **`hub.ccs.ornl.gov`**: your OLCF username, and your PIN followed by the current RSA
   passcode.
2. **The Lux login node**: your username again, and your PIN with a **new** passcode.
   Wait for the token to change; the passcode used for the hub will not work twice.

VISTA reuses this login for every Lux request in the conversation, including status
checks, fetching output and cancelling, until it has been idle for about an hour.

Before submitting, VISTA updates the FORGE code on Lux to its latest version. If that
fails, VISTA shows the error instead of submitting. Then it reports the job id,
and where the job's log, error output and results are written on the cluster:

```
job_id: 1234567
cluster: lux
nodes: 16
duration: 0:30:00 (1800s)
log_path: /lustre/orion/stf218/proj-shared/vista/<session>/out/log-1234567.out
err_path: /lustre/orion/stf218/proj-shared/vista/<session>/out/log-1234567.err
output_dir: /lustre/orion/stf218/proj-shared/vista/<session>/out/1234567
```

### Following the run

```
how is the training going?
```

VISTA fetches the job log and draws two plots: **loss** against iteration, and
**throughput**, in TFLOPS per GPU and samples per second. It reports the iteration
reached, the latest loss, the median time per iteration, and an estimate of the time
remaining.

For updates that keep coming without you asking, say "watch the job". VISTA then refreshes
the plot every 45 seconds until the job ends.

!!! warning "Watching uses up the conversation's request budget"
    Each refresh costs several requests. A new project allows 50 per message, which
    covers roughly ten refreshes. For a long run, ask for status now and then instead.

### Reading the answer critically

- **No iteration lines yet does not mean the job is stuck.** The first run of a given size
  spends several minutes indexing the training corpus before its first iteration. Later
  runs of the same size reuse that index.
- **Leave the first iteration out of the throughput.** It includes one-time setup, which
  is why VISTA's medians skip it.
- **Watch for skipped or NaN iterations.** VISTA reports both counts. A few skipped
  iterations early in mixed-precision training are normal; NaN losses are not.
- **More nodes means a bigger batch, not a faster iteration.** Each node adds to the
  global batch (forge-l on 16 nodes trains on about 2.1 million tokens per iteration).
  Compare losses only between runs of the same size.

If the job fails, VISTA explains why from the job state and the tails of the log and the
error output. For example, a timeout means the walltime was too short for the iterations
requested; a Python traceback appears in the error output.

## 3. Fine-tune FORGE to predict melting points

Pre-training teaches a model the language of science; fine-tuning teaches it one task.
Here, the task is to predict a molten salt's **melting point** from its composition,
using the thermophysical-properties database that ships with VISTA.

```
fine-tune the forge model to predict the melting point of molten salts on Frontier
```

VISTA switches to the `model-fine-tuning` skill. **Name Frontier in the prompt.**
Fine-tuning does not run on Lux, and VISTA asks which cluster to use when you do not say.
It confirms before submitting:

> Submit this as a Frontier job now?

Reply **yes**. The job trains a regression head on top of FORGE, adjusting FORGE's own
weights at a lower learning rate. It runs for 25 epochs on one node, and splits the
database into training, validation and test sets.

!!! important "Which FORGE model this fine-tunes"
    This step starts from the released, fully pre-trained FORGE weights, not from the
    model you trained in part 2. That run was 50 iterations with checkpointing off. Its
    checkpoints would also need converting to HuggingFace format before the fine-tuning
    job could load them.

### Following the run

```
show the training progress
```

VISTA tabulates each epoch's learning rate, training RMSE and validation RMSE, and plots
the two RMSEs against epoch. As with pre-training, "watch the job" refreshes the plot until
the job ends.

When it finishes, ask for the result:

```
the job is done. What is the final validation and test RMSE?
```

### Reading the answer critically

- **The RMSE is in the units of the melting point in the database**, so compare it to
  the spread of melting points, not to zero.
- **Watch the gap between training and validation RMSE.** Training error that keeps
  falling while validation error flattens or rises means the model is overfitting.
- **Report the test RMSE, not the best validation RMSE.** The best checkpoint was chosen
  on the validation set. Only the test set measures how well the model generalizes.

The job saves `checkpoint_best.pt` and `checkpoint_latest.pt` in its output directory,
plus a checkpoint every five epochs. They stay on the cluster; ask VISTA to fetch the
training logs if you want them locally.

## What to be careful about

- **Name the cluster in every request.** Pre-training runs on Lux or Frontier, and
  fine-tuning on Odo, Frontier or Perlmutter. VISTA never guesses. Without a cluster
  it asks, and it never substitutes one for the one you named.
- **Nothing is submitted until you confirm.** Check the model, nodes and walltime in the
  confirmation before saying yes. On Lux, jobs are charged to stf218; on Frontier, to
  chm243.
- **Large models need memory.** forge-l is the default on Lux. On Frontier, whose GPUs
  have less memory, the default is forge-s. Ask for forge-m or forge-l there and VISTA
  warns that the job may run out of GPU memory.
- **Checkpoints are off by default for pre-training.** Turn them on only when you need
  them, for example "save a checkpoint at the end", and expect them to be large.

## Getting help

This tutorial reflects the LLM modeling workflow as currently implemented. Screenshots and
worked numerical examples can be added to this page as the workflow is finalized.

[Return to the VISTA homepage](https://genesis-vista.github.io/){ .md-button }
