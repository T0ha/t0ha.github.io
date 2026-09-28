---
layout: post
locale: en_US
title: "$63 in One Night: When Your Elixir Polling Loop Costs Money"
description: "An AshOban trigger re-dispatched an AI agent every two minutes for eight hours: 243 runs, 31M tokens, $63 wasted. Claude Code wrote it, I approved it."
image: "/assets/images/posts/camelotai_error_loop.png"
tags:
  - elixir
  - ash
  - oban
  - ai
  - llm
  - camelot
status: published
categories:
  - devops
published: true
---

{% picture "{{page.image}}" --alt Camelot task page with a Sessions panel listing one agent run after another, each marked failed with exit 255 and no JSON object found in output %}

I opened the board one morning and one card looked wrong. Not broken — busy.
It had a wall of activity behind it from the previous evening, and I had not
touched it since lunch.

My stomach did the thing it does before I even know why. I went digging to find
out how much that evening had cost me.

**243 agent runs in eight hours. About 31 million input tokens. $64.25 spent, of
which $63.10 was pure waste. Zero commits produced.**

I read that number a few times. Sixty-three dollars, for nothing, while I slept.

Then the second part landed, and that one was worse. I did not write the code
that did this. Claude Code wrote it, inside Camelot, on a task I created myself.
I read the diff. I approved it. I merged it. It looked fine to me.

The loop did not stop because I noticed it, or because a guard caught it. It
stopped because the runner eventually crashed on malformed output and pushed the
task into an error state. Even the crash felt, in hindsight, like the most
responsible member of the team.

So this is the story of a three-line cron trigger, of what happens when you put
something expensive behind it, and of a code review where I was looking at all
the wrong things.

## A two-minute GitHub poller in Ash and Oban

 In [Camelot AI](https://camelotai.tech?utm_source=t0ha.dev&utm_medium=post&utm_campaign=2026-09-18-63-dollars-in-one-night)
a task moves through stages on a kanban
board. When the agent opens a pull request, the task lands in the `pr` stage and
a poller takes over: every two minutes it checks GitHub and reacts. Merged →
done. CI failing → send the agent back in. Reviewer requested changes → send the
agent back in.

That poller was itself a Camelot task. I described what I wanted, the agent went
away, opened a PR, and I read it on a Tuesday evening. With Ash and Oban the
trigger is almost nothing to write, and what came back was this:

```elixir
triggers do
  trigger :check_pr_status do
    action(:check_pr_status)
    scheduler_cron("*/2 * * * *")
    queue(:github)
    max_attempts(3)

    where(
      expr(
        not is_nil(pr_number) and
          stage == :pr and
          project.status == :active
      )
    )
  end
end
```

That is the whole thing. [AshOban](https://hexdocs.pm/ash_oban) takes the `where`
expression, finds matching
rows every two minutes, and runs the action on each of them. I have written
dozens of these by hand. They are boring and reliable and I like them very much.

Which is exactly why I read this one so fast. I recognised the shape, and the
moment I recognised it I stopped reading and started scrolling.

The problem is that this one is boring and reliable in front of something that
starts a container and buys LLM tokens.

## What actually happened?

Further down the same diff — well past the point where I was still reading
carefully — there was a branch that read, in spirit:

```elixir
cond do
  # ...
  has_review_state?(reviews, "CHANGES_REQUESTED") ->
    request_changes(task, comments, 0)
  # ...
end
```

`reviews` is what [GitHub's reviews endpoint](https://docs.github.com/en/rest/pulls/reviews)
returns. And here is the thing
neither of us thought about carefully enough — though only one of us was supposed
to:

**GitHub keeps every review ever submitted, forever.**

A review is not a flag you clear. It is an immutable historical record. If a
reviewer requested changes once, that entry is in the list on every subsequent
call, for the rest of the pull request's life. Dismissing it adds a *new* review
with state `DISMISSED` — the old one stays.

So `has_review_state?(reviews, "CHANGES_REQUESTED")` was not asking "does this PR
need work?". It was asking "has anyone ever asked for changes on this PR?". Once
true, it was true forever.

I read that sentence three times before it landed. The function name is honest,
by the way. `has_review_state?` says precisely what it does. I read it as "does
this PR need work?" because that is what I wanted it to mean, and because the
line under it did the thing I expected the line to do.

Every two minutes, the trigger matched the row. Every two minutes, the condition
was true. Every two minutes, Camelot dispatched an agent to go fix feedback that
had been addressed hours earlier.

And the agent, to its credit, kept looking, finding nothing to do, and exiting
cleanly. 243 times. More discipline than I showed in that review, honestly.

## Level-triggered, with a price tag

The name for this is a level-triggered check. The code was testing a *state*
("changes are requested") instead of an *event* ("changes were requested, and I
have not acted on it yet").

Level-triggered is usually the right default in a reconciliation loop. It is what
makes the loop self-healing: if you miss an event, the next pass still sees the
state and fixes it. [The first article in this series]({% post_url 2026-09-04-supervising-what-you-dont-own %})
is an argument in favour of exactly that style.

The difference is the actuator on the other end.

In an ordinary Ash application, a level-triggered check that fires redundantly
costs you a database query every two minutes, and you will never notice. Here the
actuator provisions a container, clones a repository, and hands a language model
about 130 thousand tokens of context.

A redundant poll in an ordinary app is a rounding error. A redundant poll here is
an invoice. That sentence is still a bit hard for me to write.

There is a second-order effect that makes it worse. The prompt includes the PR
conversation. Every useless run added its own noise to that conversation. So the
cost per run climbed steadily through the night — from about $0.12 early on to
$0.45 by the end. The loop was not just wasteful, it was *accelerating*.

And I did not know any of this while it was happening. Nobody was watching. That
is probably the loneliest part of this kind of failure: it happens at 3am, at its
own quiet pace, and it sends you the bill at breakfast.

## Why none of the guards helped

This thing was not built without limits. There were three mechanisms that should
have stopped it, and the loop walked straight past all three.

**The freshness check was dead code on that path.** There is a
`pr_comments_seen_at` timestamp precisely so feedback is not acted on twice. But
it was only consulted by the *comments* branch, which sits below the review
branch in the same `cond`. The review branch never reached it. The guard existed,
was correct, and was unreachable.

**The branch reset the counter that was supposed to bound it.** Look again at the
call:

```elixir
request_changes(task, comments, 0)
```

That third argument is the auto-fix attempt counter. There is a cap on
consecutive automatic re-dispatches — default 2 — so a PR the agent cannot fix
gets left for a human. But the cap only counts merge-conflict and CI-failure
fixes, and this branch passed `0`, resetting the counter on every single pass.

The guard was not bypassed by accident. The code actively disarmed it, twice an
hour, all night. And that `0` sat in the diff, in plain sight, three characters
wide.

**The order of the branches made recovery impossible.** `CHANGES_REQUESTED` was
checked before `APPROVED`. So even if the reviewer had come back and approved the
PR, the loop would have kept firing — the earlier branch matched first and the
approval was never reached.

I want to be clear that none of these are exotic mistakes. Each one is the kind
of thing that reads fine in review — I know that for a fact, because each one was
on my screen and I read it fine. For about half an hour that morning I tried to
make this the agent's failure. It is not. The agent wrote plausible code, which
is what it does, and it wrote the pattern I have written myself dozens of times.
That is exactly why it got through: it looked like my code. I am the approval
gate, and the gate opened.

## The fix: compare the review against the last commit

Two changes, and the important one is not the obvious one. This time I wrote them
by hand — not because that is the lesson, but because that is where my confidence
was that morning.

First, reduce the review list to each reviewer's *current* verdict before asking
anything about it:

```elixir
@review_states ~w(APPROVED CHANGES_REQUESTED DISMISSED)

def latest_reviews(reviews) do
  reviews
  |> Enum.filter(&(&1["state"] in @review_states))
  |> Enum.group_by(&reviewer_login/1)
  |> Enum.map(fn {_login, submitted} ->
    Enum.max_by(submitted, &review_date/1)
  end)
end
```

Group by reviewer, take the latest state-bearing review from each. `COMMENTED`
and `PENDING` carry no verdict and never supersede one. Now a reviewer who
requested changes and later approved reads as approved, which is what a human
would say if you asked them.

Second — and this is the part that actually stops the loop — apply the same
staleness guards to reviews that the comment path already used:

```elixir
def changes_requested?(reviews, last_commit_date, seen_at) do
  reviews
  |> latest_reviews()
  |> Enum.any?(fn review ->
    review["state"] == "CHANGES_REQUESTED" and
      unaddressed?(review_date(review), last_commit_date, seen_at)
  end)
end
```

A verdict has to be **newer than the last commit** — pushing a fix addresses it —
**and unseen** since the last dispatch. Two independent reasons to consider a
review handled.

On the actual incident this alone would have ended it in one pass. The review was
submitted at 11:46:46Z. The head commit was 11:52:34Z. The review was already
stale before the loop even started. One timestamp comparison. That is the whole
tragedy.

## The other bug: an optional call inside a `with` chain

While I was in there I found a second failure with the same flavour — same PR,
same review, also mine to catch and also missed. It is worth its own paragraph
because it is a pure Elixir idiom mistake, and the kind that is very easy to
approve.

The poller makes five GitHub calls, chained in one `with`:

```elixir
with {:ok, pr_data} <- Client.get_pull_request(...),
     {:ok, reviews} <- Client.list_pull_request_reviews(...),
     {:ok, comments} <- Client.list_pull_request_comments(...),
     {:ok, commits} <- Client.list_pull_request_commits(...),
     {:ok, check_runs} <- fetch_check_runs(...) do
  # ...
else
  {:error, reason} ->
    Logger.warning("Failed to check PR for task #{task.id}: #{inspect(reason)}")
end
```

This looks like good Elixir. It mostly is. That is the trap — it is idiomatic
enough that your eye slides over it. But the last call needs a **Checks: Read**
permission on the GitHub App installation, and without it GitHub returns 403.

One 403, and the whole `with` collapses into a log line. Merge-conflict handling,
review handling, comment handling, merged detection, closed detection — all of it
stopped, for every task in the `pr` stage, silently, because of an optional
enrichment call.

The fix is to be explicit that this one call is allowed to fail:

```elixir
@spec best_effort_check_runs({:ok, [map()]} | {:error, term()}) :: {:ok, [map()]}
def best_effort_check_runs({:ok, runs}), do: {:ok, runs}

def best_effort_check_runs({:error, reason}) do
  Logger.warning("check-runs unavailable (#{inspect(reason)}); treating as no checks")
  {:ok, []}
end
```

Degrade to "no checks" and keep going. CI failures go undetected until the
permission is granted, which is annoying but survivable. Everything else keeps
working, which is the point.

`with` gives you one error channel for a whole chain. That is the feature. It
becomes a bug the moment one link in the chain is not actually essential, and it
fails quietly because the `else` clause is usually where you put the logging and
stop thinking.

## What I changed in how I work

**I was reviewing agent diffs for the wrong things.** I read for naming, for
structure, for whether it fits the codebase, for whether I would have written it
that way. All of that was fine here, which is precisely why I merged it. Now I
try to ask three questions of any diff that touches a loop: what makes it fire,
what makes it stop, and what does one pass cost. If the diff does not answer the
second and the third, it is not ready, however good it looks.

**A level-triggered check needs a termination condition, not just a trigger
condition.** "Is this state true?" is not enough. "Is this state true *and have I
not already handled it*?" is the real question, and the second half has to be
durable — a timestamp or a counter in the database, not something in memory.

**Never reset a guard inside the branch the guard protects.** If I had to name the
single line that cost me $63, it is the `0` in that function call. One character,
which I did not write and did not question. I keep coming back to that.

**Best-effort calls do not belong in a `with` chain.** If a step is optional,
write a function that makes it always succeed and say so in the docs. The type
signature becomes the documentation: `{:ok, term} | {:error, term}` going in,
`{:ok, term}` coming out.

**Cost is a first-class signal.** I had logs. I had session records. What I did
not have was anything that would say "this task has run 243 times today" or "this
task has spent $60". A counter and a threshold would have caught this in twenty
minutes instead of eight hours. The
[OpenTelemetry collector I put on those Swarm nodes]({% post_url 2026-09-17-opentelemetry-collector-on-docker-swarm %})
ships logs and traces already — it just does not know anything about dollars yet.
That is now on the list, near the top.

## Conclusion

The trigger was three lines. The branch that looped was four. Nothing in this
incident was complicated, and nothing in it would have mattered at all in a
normal CRUD application — a redundant poll would have cost me a few hundred
cheap queries and I would never have known.

What changed is what sits on the other end of the loop. When the actuator costs
money, "harmless redundant work" stops being a category. Every pass has to earn
itself.

And there is the part I find harder to write. Camelot exists because I think an
agent's work should pass a human gate before it becomes real. The gate was there.
It worked exactly as designed. The gate is me, on a Tuesday evening, looking at a
diff that resembled something I would have written myself, and clicking approve.
A human approval loop is not a guarantee. It is a place where a human has to
actually be present, and that is a harder promise than the one I thought I was
making.

My practice says: notice the thought that repeats, and stop feeding it. It seems
to be my poller needed that teaching. So did my reading.

Next time I want to write about the layer underneath all of this: what happened
when I tried to run long-lived, stateful agent containers on an orchestrator that
is designed around the opposite assumption.

And if an agent wrote a loop for you recently and you approved it because it
looked right — go check what it did last night. I really wish somebody had told
me that a week earlier.
