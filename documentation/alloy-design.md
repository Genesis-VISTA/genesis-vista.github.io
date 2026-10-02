# Alloy design with VISTA

This tutorial walks through a complete alloy-design study in VISTA, using the refractory
high-entropy alloy **MoNbTaW** as the worked example. You will estimate a transition
temperature, train a machine-learned order parameter from simulation data, use it to
accelerate sampling, and finish by running an autonomous search for the best composition.

Each step is driven by a prompt in the chat panel. VISTA selects the appropriate skill,
submits the work to an HPC cluster, and reports the result back into the conversation.

All five steps run on the **Odo** cluster at OLCF and follow on from one another: steps 2
through 4 form a three-stage pipeline that shares a single workspace.

## Before you start

Open the **alloy-design** project from the **Projects** page. It supplies the four skills
this tutorial uses:

| Skill | What it does |
|---|---|
| `alloy-thermo-mc` | parallel-tempering Monte Carlo for one composition |
| `deepthermo-wl` | Wang-Landau sampling, and collecting configurations for training |
| `vae-orderparam` | trains the VAE that serves as a learned order parameter |
| `alloy-tc-planner` | drives a multi-cycle composition search |

You also need HPC credentials for Odo on your user record. Without them VISTA will pause
mid-task to ask for them. Confirm they are set before starting a long step.

!!! note "These jobs take real time"
    A screening simulation runs for minutes; a Wang-Landau run can take hours. VISTA
    returns as soon as a job is queued and emails you when it completes, so you do not
    have to keep the conversation open. Ask for a status update at any point.

## 1. Estimate Tc for an equimolar alloy

Start with a single composition to confirm the pipeline works end to end.

```
estimate the Tc for equal-molar MoNbTaW alloy
```

VISTA uses the `alloy-thermo-mc` skill. It converts the composition into a job on Odo that
runs a replica-exchange (parallel tempering) Monte Carlo simulation, with one temperature
replica per MPI rank, on a DFT-derived pair-interaction model of the alloy.

Equimolar means each element is at an atom fraction of 0.25. **The four fractions must sum
to exactly 1.0** — they are atom fractions, and anything else is physically meaningless.
VISTA checks this before submitting.

!!! tip "Asking for one composition, not a search"
    This step evaluates a single composition. If VISTA offers to start a multi-cycle
    search instead, say that you want one simulation and no campaign — the composition
    search is step 5.

When the job finishes, ask for the results:

```
the job is done — report the Tc and show the thermodynamics plots
```

VISTA fetches the job output and reports:

- **`Tc_cv_K`** — the transition temperature from the specific-heat peak. This is the
  headline number.
- **`Tc_chi_K`** — the same quantity from the susceptibility peak, as an independent check.
- **`sro_alpha1`** — the Warren-Cowley short-range-order parameter.
- two figures: specific heat, energy, susceptibility and Binder cumulant against
  temperature, and the short-range-order parameters against temperature.

### Reading the answer critically

A single number is not the whole story, and the skill is written to surface the caveats
rather than hide them:

- **If `peak_bracketed` is false, the Tc is not a measurement.** It means the temperature
  ladder did not span the transition, so the reported peak sits at the edge of the
  sampled range. Ask VISTA to rerun with a wider temperature range.
- **Check whether the two estimators agree.** A large gap between the specific-heat and
  susceptibility estimates means the transition is poorly resolved.
- **A specific-heat bump with no short-range order is not an ordering transition.** If
  `sro_alpha1` is near zero, the alloy is behaving as a random solid solution.
- **One lattice size cannot pin Tc precisely.** The peak shifts and sharpens as the system
  grows. Treat this as a screening estimate.

### Driving this step from the API

The same simulation can be run without the chat panel. This is the step to automate: one
composition in, one transition temperature out.

The flow is **two agent turns with a wait between them**. The job sits in the cluster
queue for minutes, so it cannot finish inside a single turn — turn 1 submits and returns
a job id, turn 2 collects once it has completed. Poll in between through the tool
endpoint, which runs no model and therefore costs no tokens.

??? example "Full request sequence"

    Set your HPC credentials once. Without them the agent stops mid-turn to ask for
    them interactively, which a non-streaming client cannot answer:

    ```bash
    curl -X PUT http://localhost:8001/users/me \
         -H 'Content-Type: application/json' \
         -d '{"s3m_token": "<your OLCF S3M token>"}'
    ```

    Create a chat session, so the second turn remembers the first:

    ```bash
    SESSION=$(curl -s -X POST http://localhost:8001/projects/alloy-design/chat-sessions \
      -H 'Content-Type: application/json' -d '{"title":"Tc of equimolar MoNbTaW"}' \
      | python3 -c 'import json,sys; print(json.load(sys.stdin)["id"])')
    ```

    **Turn 1 — submit.** Tell the agent to stop after submitting; otherwise it may poll
    the queue itself and spend requests waiting:

    ```bash
    curl -s -X POST http://localhost:8001/projects/alloy-design/agent/run \
      -H 'Content-Type: application/json' \
      -d "{\"user_prompt\": \"Estimate the Tc for equimolar MoNbTaW (Mo=0.25, Nb=0.25, Ta=0.25, W=0.25). Submit a single alloy-thermo-mc job on odo with default screening settings - do not start a campaign. Report the job id and stop.\",
           \"stream\": false, \"chat_session_id\": \"$SESSION\"}"
    ```

    The response carries `new_messages` (the model requests, tool calls, tool returns and
    final text of the turn), `usage`, and `logs`. The cluster job id is in the tool-return
    part for `submit_hpc_job`:

    ```
    job_id: 44521
    cluster: odo
    ```

    **Poll — no model in the loop:**

    ```bash
    curl -s -X POST http://localhost:8001/projects/alloy-design/mcp/call \
      -H 'Content-Type: application/json' \
      -d '{"name":"get_hpc_job_status","arguments":{"job_id":"44521","cluster":"odo"}}'
    ```

    Two things to know about this endpoint. It returns the raw tool-result envelope
    rather than a plain string, so read the text parts and check the error flag rather
    than pattern-matching the JSON:

    ```json
    {"content": [{"type": "text", "text": "... STATE=RUNNING ..."}], "isError": false}
    ```

    And when more than one cluster is configured, **`cluster` is required** — omitting it
    returns an error asking which one you meant. Wait for `STATE=COMPLETED`.

    **Turn 2 — collect and interpret:**

    ```bash
    curl -s -X POST http://localhost:8001/projects/alloy-design/agent/run \
      -H 'Content-Type: application/json' \
      -d "{\"user_prompt\": \"The job finished. Fetch its results.json and report the Tc from the specific-heat peak, the susceptibility cross-check, whether they agree, whether the temperature ladder bracketed the peak, and the short-range-order parameter.\",
           \"stream\": false, \"chat_session_id\": \"$SESSION\"}"
    ```

Authentication is the development-mode `X-Vista-User-Email` header; omit it to use the
default development user. Deployments with single sign-on enabled will reject unauthenticated
requests.

Set `"stream": true` to receive server-sent events instead, which is what you want if the
agent may need to ask you something mid-turn — including for credentials you have not
pre-set.

A runnable end-to-end version of this sequence ships with VISTA as
`backend/scripts/example_alloy_tc.py`, with the full walkthrough in
`docs/api-example-alloy-tc.md`.

## 2. Collect configurations for training

The remaining steps build a *learned* order parameter: instead of choosing a symmetry to
measure in advance, you train a model to discover the coordinates that separate ordered
from disordered configurations. That model needs training data, which comes from a short
simulation.

```
perform a short parallel-tempering simulation to collect alloy configurations
that can be used to train a VAE model as order parameter
```

VISTA uses the `deepthermo-wl` skill in **collect** mode. It runs a parallel-tempering
warm-up with configuration capture turned on, writing snapshots spanning the temperature
ladder — ordered configurations from the cold replicas, disordered ones from the hot end.

!!! important "Name the workspace and reuse it"
    Steps 2, 3 and 4 are separate HPC jobs that hand artifacts to one another through a
    shared **workspace**. Tell VISTA which name to use, for example `tc-n10`, and use the
    same name in all three steps. Snapshots from this step must still be there when
    training starts, and the trained model must still be there when sampling starts.

Keep this run short. You need enough frames to train on, not a production dataset.

When it finishes, ask what it produced:

```
how many configurations did that produce, and what was the swap acceptance?
```

Two things are worth checking before moving on:

- **Parallel tempering may not mix.** If the replica swap acceptance is near zero, the
  temperature ladder is not exchanging configurations, and the low-energy ordered states —
  exactly what the order parameter most needs to resolve — will be the thinnest part of
  your training set. The fix is more replicas or longer decorrelation between samples.
- **`used_bootstrap_model: true` is expected here.** The simulation engine loads an
  encoder even when only gathering data, so VISTA supplies a random-weight placeholder.
  The configurations come from the physics and are unaffected by it.

This step also records the energy range the simulation actually reached. Step 4 uses that
automatically, so you never have to supply it by hand.

## 3. Train the VAE order parameter

```
train a VAE model on the collected data
```

VISTA uses the `vae-orderparam` skill, pointed at the same workspace. The job converts the
snapshots into a one-hot lattice representation, removes duplicate frames, splits training
and validation data, trains a variational autoencoder, and exports it in the form the
simulation engine can load.

Ask to see how training went:

```
plot the training and validation loss
```

What to look for:

- **Watch the gap between training and validation loss, not the absolute values.** The
  first epoch's training figure is an initialisation transient and recovers immediately.
  A gap that closes to a few percent means the model is not overfitting.
- **Check the duplicate fraction.** Cold replicas repeat configurations because their
  dynamics freeze. A high duplicate fraction means the ordered states are underrepresented
  in training, which points back at step 2 rather than at the model.

!!! warning "Training loss alone does not prove the model is useful"
    A VAE can train cleanly and still propose moves the sampler rejects. The real test is
    the acceptance rate in step 4.

## 4. Run VAE-accelerated Wang-Landau sampling

```
Run a VAE accelerated Wang-Landau sampling
```

VISTA uses `deepthermo-wl` in **sample** mode, on the same workspace. Wang-Landau sampling
builds up the **density of states**, from which the full temperature dependence of the
thermodynamics follows in a single run — unlike step 1, which samples one temperature
ladder. The trained VAE proposes large configuration changes that a local move could not
reach.

This is the long step. Monitor it while it runs:

```
how is the sampling going?
```

VISTA reports on the files the job publishes as it works:

- **The density of states, one file per Wang-Landau iteration.** The iteration number is
  the progress indicator: the algorithm repeatedly refines its estimate, and roughly
  twenty iterations complete a run. Ask VISTA to plot successive iterations to watch the
  histogram flatten — that flattening is the convergence criterion.
- **The VAE move acceptance.** This is the number that tells you whether step 3 paid off.
  A well-trained model gives a high acceptance on moves that change a meaningful fraction
  of the lattice. An acceptance near one percent means the sampler is effectively running
  on an untrained model.

!!! note "Acceptance decays as the run proceeds — that is expected"
    Wang-Landau tightens its acceptance criterion as the density of states flattens, so a
    falling acceptance rate late in a run is the algorithm working, not a failure.

When it completes, ask for the physics:

```
the sampling finished — show the thermodynamics derived from the density of states
```

## 5. Search for the highest Tc

The first four steps evaluate compositions you chose. The last step hands the choosing to
VISTA.

```
search for the highest Tc for MoNbTaW
```

This starts a **campaign** using the `alloy-tc-planner` skill. VISTA will first ask what
you want; answer with the budget for this tutorial:

- **Target Tc:** about 1000 K
- **Maximum cycles:** 3
- **Candidates per cycle:** 4

VISTA then drafts a numbered plan and asks you to approve it. **Nothing is submitted to
the cluster until you approve**, and your edits to the plan take precedence over its
defaults. Twelve jobs is the full budget here — four compositions per cycle, three cycles.

On approval, VISTA dispatches the first cycle: one simulation per candidate composition,
running in parallel. You are emailed as each completes.

### Between cycles

When a cycle's jobs finish, VISTA scores them and reports back. Ask for the picture:

```
show the specific heat curves and short-range order for this cycle's candidates
```

Candidates are ranked by transition temperature, but only after the ones that cannot be
trusted are filtered out — a composition whose ladder missed the transition, or one whose
specific-heat feature is not accompanied by real short-range order. VISTA reports those
rejections and the reason for each rather than quietly dropping them.

It then proposes the next cycle, concentrating on the promising region of composition
space. Approve, adjust the proposed compositions, or stop the campaign at any point.

### Finishing

The campaign ends when the target is met, the cycle budget is exhausted, or you end it.
Ask for the summary:

```
summarize the campaign: best composition, how it compares to equimolar, and what you would try next
```

## What to be careful about

- **Every composition must sum to 1.0.** This is the constraint most easily broken when
  suggesting compositions by hand.
- **Only MoNbTaW is parameterized.** The machinery is chemistry-agnostic, but running
  another alloy requires DFT-derived interaction parameters for that system. VISTA will
  tell you when they are missing instead of substituting another alloy's.
- **Reuse the workspace name across steps 2 to 4**, and start a new one for an unrelated
  study. Each run clears its own stage's previous output, so results never mix within a
  stage.
- **A result is only as good as its diagnostics.** The bracketing check, the estimator
  agreement, the short-range-order signal and the move acceptance are all reported for a
  reason. Ask about them when a number looks surprising.

## Getting help

This tutorial reflects the alloy-design workflow as currently implemented. Screenshots and
worked numerical examples can be added to this page as the workflow is finalized.

[Return to the VISTA homepage](https://genesis-vista.github.io/){ .md-button }
