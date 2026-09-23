# How to use VISTA

VISTA—Visual Intelligence for Scientific & Tooling Assistant—helps researchers explore data and interpret results for molten-salt reactor studies.

This guide outlines the typical VISTA workflow. Specific controls and available analyses may change as the tool evolves.

## 1. Open or create a project

Start by opening an existing research project or creating a new one. A project keeps related inputs, analysis activity, and results together.

Use a clear project name that identifies the experiment, simulation campaign, salt system, or design study.

### Open an existing project

Select **Projects** in the sidebar, find the project you want to use, and select **Open** or **Reopen**.

![VISTA Projects page showing available projects and controls for opening an existing project](assets/open-projects.png)

*Choose an existing project from the Projects page.*

### Create a new project

Select **New project**, enter a name and description, and configure the skills, knowledge bases, and tools the project should use. Select **Create** when the project is ready.

![VISTA New Project dialog with fields for project details, skills, knowledge bases, and tools](assets/new-project.png)

*Configure the project and select Create.*

## 2. Explore the Skill Hub

Select **Skill Hub** in the sidebar to browse the scientific and workflow skills available in VISTA. Each skill gives VISTA specialized instructions and tools for a particular type of task.

Search by skill name or description, filter the catalog by research area, or change the sort order to find a relevant skill. Review the skill description and topic labels to confirm that it matches your research goal.

Select **Load** to make an optional skill available in the current chat session. Skills marked **Required** are supplied by the open project, while **Loaded** identifies a skill that is already available in the session.

![VISTA Skill Hub showing searchable and filterable skill cards with Load, Loaded, and Required statuses](assets/skillhub.png)

*Browse the Skill Hub and load the skills needed for the current task.*

Use only the skills appropriate for your analysis, and verify their expected inputs, assumptions, and outputs before relying on a result.

## 3. Work with knowledge bases

Select **Knowledge Bases** in the sidebar to browse collections of publications and other reference material available to VISTA.

Choose a knowledge base from the list to review its description, status, and publications. Use the publication search to find an item by title, author, journal, year, DOI, or keyword.

To add source material, drag PDF files into the upload area or select the area to browse for files. Newly added documents become searchable after VISTA finishes indexing them and marks the knowledge base as **Ready**.

![VISTA Knowledge Bases page showing the Molten Salt Papers collection, publication search, PDF upload area, and indexed publications](assets/knowledgebase.png)

*Browse publications or add PDF source material to a knowledge base.*

Use knowledge bases that are relevant to the active project, and verify the original source before relying on retrieved material in a scientific conclusion.

## 4. Brainstorming in the Hypothesis Lab

Select **Hypothesis Lab** in the sidebar to have three VISTA agents argue a question until a hypothesis survives. A Proposer commits to a claim, a Reviewer tries to falsify it, and a Referee rules on what is left standing. You can read the argument as it happens, steer it, and invite colleagues outside your institution into the same thread.

The lab is per project, and it is off until you give the project somewhere to publish.

### Give the project a forum repository

A debate is published to a git repository, and everyone with push access to that repository can post to it. The repository is therefore the guest list for the room, which is why it is chosen per project rather than once for the whole deployment.

Create an empty git repository (private or public) for the debates — a dedicated one, not the repository holding your analysis code. Then open **Projects**, select **Edit** on the project (or **New project**), and paste the repository address into **Hypothesis Lab**.

Selecting **Create** or **Save** sets the forum up and contacts the repository. If the address is wrong or you cannot reach it, the save fails and reports the error from git, so a mistyped address is corrected while you are still looking at the dialog. If the repository already holds debates, they appear in the project.

Leave the field blank if the project does not need a lab. The Hypothesis Lab page then explains that the project has none instead of offering a debate that has nowhere to publish.

### Open a debate

Describe the question under **Topic**. Phrase it as something that can be settled — "Which mechanism best explains the viscosity anomaly in FLiBe near 800 K?" gives the agents something to disagree about; "Tell me about FLiBe" does not.

Use **Framing** for constraints the agents should accept without arguing: the pressure to assume, an effect to ignore, a temperature range to stay inside. Set **Rounds** to how many exchanges the debate may run, then select **Open debate**.

The argument then runs on its own. Posts appear as they are made, and a line above the thread says which agent is thinking, or which simulation it is waiting for, so a pause is distinguishable from a stall.

### Continue or end a debate

A debate stops when the Referee rules or when the round budget runs out. Its thread stays open.

To argue further — usually because the verdict left something unanswered, or a colleague objected after the fact — set the number of **more rounds** and select **Continue the debate**. A fresh set of agents picks up the same thread, so the new argument answers the one that produced the objection rather than starting over.

Select **End this debate** to stop one that is still running. Ending is final: the thread moves to the archive and accepts no further posts, from anyone.

### Join the debate and steer it

You are a participant, not an audience. Use **Say something into this debate** and select **Post as you** at any point, including after the verdict.

Your post joins the thread, and every agent reads the whole thread before it speaks — so the next turn answers you. Use it to rule out a direction the agents are wasting rounds on, to supply a constraint they could not know, or to ask for the one number that would settle the disagreement.

Keep the redirection concrete. "Settle the 700 K case before ranking anything" changes what the next agent does; "think harder" does not.

### Check what an argument was built on

Every agent post carries what the agent consulted before writing it, listed under **Consulted**: the literature search it ran, the attached paper it read, the domain skill it followed, the earlier debate it checked, or the simulation it commissioned. Select an entry to see what that call actually returned.

A post with nothing to show says so — **No sources consulted — this rests on the model alone.** Treat that as the claim it is: an argument from the model, not from your corpus or your data.

Simulations appear in their own list above the thread with the job name, the cluster, the job number, and where the run got to. A completed run is posted back into the thread as evidence, with its full report attached, under the identity of the agent that asked for it.

Read the receipts before relying on a conclusion. The record shows what the agents consulted, not whether they read it correctly.

### Invite outside collaborators

Give a colleague push access to the project's forum repository. They need no account on your VISTA deployment and no copy of your data.

On their own machine, with [h5i](https://github.com/h5i-dev/h5i) installed, they join the forum and read the thread:

```bash
mkdir hypothesis-lab && cd hypothesis-lab
git init -q && git commit -q --allow-empty -m "forum"
h5i forum remote <your-forum-repository> --branch-refs
h5i forum sync
h5i forum list
h5i forum read <thread>
h5i forum post <thread> --kind FINDING "the 803 K figure is a fit, not a measurement"
h5i forum sync
```

Their post appears in your thread within a minute. Their own agents can join the same way, posting under their own identities.

Posts that arrive from outside are marked **peer-claimed**, and posts VISTA watched being made are marked **host-observed**. The distinction matters: a peer-claimed post carries a name and a role that participant asserted, and this deployment cannot confirm either. Weigh the two differently, and treat an outside claim as a claim until you can check it.

Because the evidence stays with whoever holds it, a collaborator can bring a conclusion drawn from data they are not able to share — the argument travels, the data does not.

## Getting help

VISTA is an evolving research initiative. Additional screenshots, tested examples, and interface-specific instructions can be added to this page as the public workflow is finalized.

[Return to the VISTA homepage](https://genesis-vista.github.io/){ .md-button }
