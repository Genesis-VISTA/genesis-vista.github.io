# Connect VISTA with your personal agent

This tutorial connects VISTA to Slack, so you can work with OLCF from your phone, or
through a personal agent that posts to Slack for you. Here that agent is **Muse**, a
voice assistant on an Android phone. You ask Muse something out loud, Muse asks VISTA in
Slack, and VISTA does the work on the clusters and replies in the same thread.

<video controls preload="metadata" playsinline
       poster="../assets/vista-muse-demo.jpg" style="width:100%">
  <source src="../assets/vista-muse-demo.mp4" type="video/mp4">
  Your browser can't play this video. <a href="../assets/vista-muse-demo.mp4">Download it</a>.
</video>

*The session in part 3, in under three minutes: a question about the Lux queue, a job
started after a Confirm in Slack, Muse following up on its own, and VISTA's record of
the job. The idle stretches are sped up 4×.*

There are two agents in this picture, and they do different jobs:

| Agent | Where it runs | What it does |
|---|---|---|
| **Your personal agent** (Muse) | your phone | turns what you say into a Slack message, follows the thread, and tells you the answer |
| **VISTA** (the facility agent) | your own computer, the one you use VISTA on | answers in Slack with the same projects, skills and HPC logins as the desktop app, and runs the jobs |

Nothing new runs at OLCF. VISTA connects out to Slack from your machine, and reaches the
clusters with the logins you already made in the desktop app. Slack never sees a
cluster credential: VISTA never asks for a password, passcode, MFA code or token in
Slack.

The three parts build on one another. Part 1 connects VISTA to Slack, part 2 lets your
personal agent use it, and part 3 runs a job from your phone. Every example below comes
from the real session on Lux in the video, recorded in October 2026.

## Before you start

You need:

- **VISTA installed on a computer that stays on** while you are away. VISTA only
  answers while it is running; part 1.4 explains how to keep it running.
- **A Slack workspace where you can create apps**, or an admin who can do part 1.1 for
  you.
- **Logins for the clusters you want to use, made in the desktop app.** For Lux, that is
  your Lux account in **Settings → Compute** and a login with your RSA SecurID token. A
  Lux login lasts an hour, and Slack can use it while it lasts.
- **A personal agent that can post to Slack as you** and read the replies, such as Muse.
  It must post as *you*, not as a bot of its own: VISTA ignores messages from bots.

!!! note "VISTA answers only while it runs"
    VISTA uses Slack's Socket Mode: your machine opens a connection out to Slack, so
    there is no public address and no port to open. The cost is that a message sent
    while VISTA is off is not queued. VISTA posts a notice in your DM when it goes
    offline and again when it is back.

## 1. Connect VISTA to Slack

### 1.1 Create the Slack app

1. Go to <https://api.slack.com/apps> and choose **Create New App → From an app
   manifest**.
2. Pick your workspace, and paste VISTA's manifest,
   [`backend/src/vista_backend/slack/manifest.yaml`](https://github.com/Genesis-VISTA/vista/blob/main/backend/src/vista_backend/slack/manifest.yaml).
   It asks only for DMs, @mentions, the Home tab and the scopes VISTA uses. It cannot read
   channels VISTA is not mentioned in.
3. Select **Install App** and install it to the workspace. Copy the **Bot User OAuth
   Token** (`xoxb-…`).
4. Under **Basic Information → App-Level Tokens**, generate a token with the
   `connections:write` scope. Copy it (`xapp-…`).

### 1.2 Turn on Slack in VISTA

Open **Settings → Slack** in the desktop app and fill in:

| Setting | What to enter |
|---|---|
| **Bot token**, **App-level token** | the two tokens from part 1.1. They go to the system keychain and are never shown again. |
| **Allowed workspace IDs** | your workspace's ID (`T…`, from **About this workspace**). Events from any other workspace are ignored. |
| **Allowed channel IDs** | optional. Leave empty to allow any channel VISTA is invited to. DMs are always allowed. |
| **Slack**, **Handle Slack on this machine** | turn both on |

The status line should say **Connected**.

!!! tip "If it does not connect"
    The text under the status says what Slack answered. `invalid_auth` means a wrong or
    revoked bot token, and `not_allowed_token_type` means the two tokens are swapped.
    **Standing by** means another of your machines is handling Slack; select **Handle
    Slack here instead** to take over.

### 1.3 Link your Slack account

Only linked Slack accounts can use VISTA. In Slack, DM VISTA:

```
link
```

VISTA replies with a 6-digit code. Enter it under **Your linked Slack accounts** in
**Settings → Slack** within 10 minutes. There you can also choose which project new
Slack conversations start in (**First project**), and whether job notices come in the
job's thread or in a DM.

Then invite VISTA to the channel you will use, for example `#dev`:

```
/invite @VISTA
```

### 1.4 Keep VISTA reachable

Under **Notifications and power**:

- **Run VISTA in the background** starts VISTA's services when you log in, with no
  window, and keeps them running after you close the window. It takes effect at your
  next login.
- **Keep this machine awake** stops idle sleep while Slack is connected.

A laptop with its lid closed sleeps anyway, unless it is on power with an external
display.

### 1.5 Try it from Slack

Before adding your personal agent, check VISTA on its own. In `#dev`:

```
@VISTA how busy is the job queue on Lux right now?
```

VISTA posts **Working on it…** within a few seconds, keeps it up to date with what it is
doing, and then replaces it with the answer. The conversation also appears in the
desktop app's chat list with a Slack badge.

## 2. Let your personal agent talk to VISTA

Your personal agent needs only two things from Slack: to post in `#dev` as you, and to
read the thread VISTA answers in. Give it those in its own Slack settings; how depends
on the agent. Messages it posts show **Sent using Muse** (or your agent's name) under
them.

To VISTA, such a message is simply you @-mentioning it, from your linked account, in an
allowed workspace and channel. So the same rules apply to it as to anything you type
yourself, and the agent never holds anything that lets it act on a cluster alone.

Tell your agent how to reach VISTA. With Muse, for example:

> When I ask you to do something on an OLCF cluster, ask VISTA in #dev: start the message
> with @VISTA, wait for its reply in the thread, and tell me the answer. If VISTA asks for
> a Confirm, tell me; never press it yourself.

## 3. Work with OLCF from your phone

### 3.1 Ask a question

Say to Muse:

> Ask VISTA how busy the job queue on Lux is.

Muse posts `@VISTA how busy is the job queue on Lux right now?` in `#dev`, follows the
thread, and reads you the answer. **Example answer** (about a minute after asking):

```text
Lux is busy. 6 jobs are running on 261 nodes, and 33 jobs are pending that ask for
9,561 nodes in total. That is far more than is running now, so new jobs will probably
wait a while. You have no jobs queued or running.
```

Reading runs straight away, with no confirmation: questions, queue status, job status,
outputs and logs.

### 3.2 Start a job, with your confirmation

Say to Muse:

> It's not that busy. Let's start a model serving on the Inkling model.

Muse asks VISTA to `start a model service on Lux for inkling model`. Starting a job
changes something, so VISTA does not run it. It posts a card in the thread instead,
showing what would actually be submitted: **System** (Lux), **Nodes**, **Walltime** and
**Script**, with **Confirm** and **Cancel**. Muse tells you:

> VISTA's ready to submit it, with the default cluster, nodes and walltime. It needs your
> Confirm in #dev, and expires in 10 minutes. Only you can hit it.

Open Slack, check the card, and tap **Confirm**. The card then shows **Confirmed by
@you**. Go back to Muse and say:

> I have confirmed, ask it to start.

VISTA submits the job and reports back in the thread:

```text
Inkling-MXFP4 is submitted on Lux as job 349976. It's still starting up, so there's no
endpoint yet.
```

The job is the `vllm-serve` job from the [Lux skills](lux-llm.md#26-run-it-from-vista)
tutorial. It takes 11 to 14 minutes to start serving.

!!! important "Confirm is always yours"
    - **Only the person who asked can confirm.** A tap from anyone else is refused.
    - **A card expires after 10 minutes,** and is then treated as Cancel.
    - **Check the card before you tap.** It lists the cluster, job, nodes and walltime
      VISTA worked out itself, not what the model described.
    - **Don't let your personal agent confirm for you.** The card is the one place a
      human decides.

### 3.3 Follow up

Your personal agent can keep track of the job for you. In this session Muse asked on
its own, a minute later:

```
@VISTA what is the status of job 349976 on Lux? Is the endpoint up?
```

VISTA's answer:

```text
Job 349976 is RUNNING on Lux, but I can't confirm the endpoint is up yet.
```

When a job started from Slack ends (completed, failed, cancelled or timed out), VISTA
tells you once, in its thread or a DM. You can also ask in plain words, such as "my last
job" or "show me its log". Logs are posted only when you ask.

A status check that needs a tool which can change something, such as a shell command on
the login node, asks for a Confirm like a submission does.

### 3.4 See VISTA's record of the job

Every Slack conversation is also in the desktop app, under the project you chose when
linking, marked with a Slack badge. Open one to see each step VISTA took, the tools it
called and their results: the same record as a conversation started in the app.

## What to be careful about

- **Set limits for Slack.** Under **Limits for Slack requests**, set the most nodes per
  job, the longest walltime and the number of jobs per day Slack may submit. A request
  over a limit is refused before any card. The limits are set only in the desktop app,
  so a lost phone can't raise them.
- **Lock it if your phone is lost.** Say `lock` to VISTA in Slack to block every action
  that changes something. Questions still work. You can unlock only in the desktop app,
  with the **Locked** switch.
- **Keep your cluster logins fresh.** VISTA DMs you 24 hours and 2 hours before a
  credential expires. Renew it in the desktop app; after it expires, VISTA tells you what
  it can't do on that cluster. Without a Lux login, it tells you to log in from the app.
- **Never type a credential into Slack.** VISTA never needs one there. Anything that does,
  such as a Lux login or connecting Globus, happens in the desktop app.
- **Expect no answer while VISTA is off.** Check the **Home** tab of the VISTA app in
  Slack: it shows whether VISTA is connected and since when, whether it is locked, your
  recent jobs and expiring credentials.
- **Everything is audited.** Every request, refusal, card, confirm, cancel and tool
  result is recorded, with credentials removed, in `<data dir>/logs/slack-audit.jsonl`.

## Getting help

- **Nothing happens on an @mention:** invite VISTA to the channel (`/invite @VISTA`) and
  check **Allowed channel IDs**.
- **"Your account isn't linked":** DM `link` and enter the code in the desktop app.
- **Your personal agent gets no reply:** check that it posts as you, not as its own bot,
  and that VISTA is connected (the **Home** tab).

The full reference is
[`docs/slack.md`](https://github.com/Genesis-VISTA/vista/blob/main/docs/slack.md) in the
VISTA repository.

[Return to the VISTA homepage](https://genesis-vista.github.io/){ .md-button }
