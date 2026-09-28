---
layout: post
locale: en_US
title: "Supervising What You Don't Own: A Controller Loop in OTP"
description: "A deploy killed my OTP processes but not the containers running my AI agents. The fix: a Reconciler GenServer and a pure, testable decision function."
date: 2026-09-04
tags:
  - elixir
  - otp
  - docker
  - swarm
  - ai-agents
  - camelot
status: published
categories:
  - devops
published: true
---

One morning I opened Camelot and saw a card stuck in the `executing` column.
The agent had been "working" on it since the previous day.

The funny part is that the work was already done. The agent had written the
code, pushed the branch, opened a pull request, and I had merged it myself the
evening before. The board just didn't know about any of that.

This post is about why that happened, and about the piece of code I had to
write to make it stop. It seems to be a piece of code that many Elixir projects
eventually need and few of us plan for: a controller loop that reconciles what
the database believes with what the infrastructure is actually doing.

## How Camelot runs agents

[Camelot AI](https://camelotai.tech?utm_source=t0ha.dev&utm_medium=post&utm_campaign=2026-09-04-supervising-what-you-dont-own) runs coding agents on a kanban board. You
create a task, an agent plans it, you approve the plan, the agent implements it
and opens a PR.

Each agent runs inside its own container. In production that means a [Docker
Swarm](https://docs.docker.com/engine/swarm/) service per task, and the BEAM talks to the Swarm manager over an HTTP
socket proxy. A single run can take forty minutes.

Inside the application, each task gets a `TaskRunner` [GenServer](https://hexdocs.pm/elixir/GenServer.html). It builds the
spec, starts the runner, streams output over PubSub to the LiveView, and
finalises the session when the process exits.

It looks like a normal OTP design. For a while it behaved like one.

## Where OTP stops

OTP supervises processes inside your VM. If a `TaskRunner` crashes, its
supervisor restarts it. If a process it monitors dies, it gets a `:DOWN`
message. This is the part everybody knows and loves.

None of it applies to the agent.

The agent is not a BEAM process. It is a CLI inside a container, possibly on
another machine. My `TaskRunner` does not supervise it directly. It *watches* the container, over
a network, through a proxy, with no delivery guarantees.

And here is the thing that broke me: **a deploy kills my processes but not my
containers.**

When I push to `develop`, the app container is replaced. Every `TaskRunner`
dies. Meanwhile the agent containers are still out there, happily compiling
somebody's Elixir project, with nobody left in the world listening to them.

The database still says `status: :running`. The world says something else.

## The row outlives the process

Once I accepted that, the design got a bit simpler to reason about.

The durable thing is a `Session` row in Postgres. The `TaskRunner` GenServer is
just the thing that happens to be attending to it right now. Processes are
disposable. Rows are not.

So after a restart, somebody has to walk the rows and ask, for each session that
claims to be running: is this still true?

That somebody is `Camelot.Runtime.Reconciler`. A `GenServer` that runs once at
boot and then every sixty seconds. Probably the most valuable module in the
whole runtime.

```elixir
def handle_info(:tick, state) do
  state = do_reconcile(state)
  Process.send_after(self(), :tick, @tick_ms)
  {:noreply, state}
end
```

If you have written a Kubernetes controller, this looks familiar. Observe the
desired state, observe the actual state, take the smallest action that moves
one towards the other. Then do it again in a minute, because you were probably
wrong about something.

## Adopting a run that lost its owner

The first thing the reconciler does is find `:running` sessions whose owning
`TaskRunner` no longer exists.

The naive move is to mark them failed. Correct, and also terrible: you just
threw away thirty minutes of an agent's work and real money because you
deployed a CSS fix.

So instead Camelot tries to **adopt** them. If the container is still alive, we
start a fresh `TaskRunner` and re-attach it to the run already in progress.

The problem is that after a restart we no longer have the original `docker exec`
id, so we cannot ask Docker for its exit status. The exec is gone as far as we
are concerned, even though the process inside the container is fine.

The solution lives in the container. Every agent invocation goes through a small
wrapper script:

```bash
out="/tmp/camelot-output-${CAMELOT_SESSION_ID:-session}.log"
set +e
"$@" 2>&1 | tee "$out"
code=${PIPESTATUS[0]}
set -e

echo "$code" > "/tmp/camelot-exit-${CAMELOT_SESSION_ID:-session}"
exit "$code"
```

Two files, and the order matters:

- The output is tee'd to a per-session log, so the complete result can be
  fetched with a short `docker exec cat` after the fact, instead of depending on
  a long-lived, mostly idle exec stream.
- The exit code is written to a marker file **last**, so its presence strictly
  implies the output file is already complete.

An adopting session polls for that marker. When it appears, we know the run
finished and with what code, and we read the tee'd output next to it.

I was quite pleased with this. It survived deploys. It recovered work. It felt
robust.

## The poll that never ends

Here is the poll loop, roughly as I first wrote it: check for the marker, and if
it is not there, sleep and check again.

Read that sentence again and see if you can spot what is missing.

There is no way out.

If the marker never appears, the loop runs forever. And there is a very ordinary
situation where the marker can never appear: **the container was replaced.**

Swarm reschedules a task. A node runs out of memory and kills the container.
The boot sweep rolls the service onto a newer runner image. In all three cases a
brand new container comes up with the same service name, perfectly healthy, and
completely useless to us — the marker and the tee'd output lived in the previous
container's `/tmp`.

My test cluster is two very small nodes. Reschedules are not a rare event there.
They are Tuesday.

So the sequence that produced my eighteen-hour card:

1. A session started executing.
2. I deployed. The `TaskRunner` died, the container kept running.
3. The container got rescheduled onto another node. The marker died with it.
4. The reconciler adopted the session and started polling for a file that no
   longer existed anywhere in the universe.
5. It polled. And polled.

The session stayed `:running`, so the task stayed in `executing`. And every
other recovery path politely stepped around it: the stale-session sweep saw a
live owning process, so it skipped it. The stale-handle sweep saw a live
service, so it skipped it. Everything was working exactly as designed.

Meanwhile the agent had finished long ago and opened the PR. I merged it by
hand. The board never noticed.

## The fix: a decision with no I/O

The fix itself is not complicated. What matters is *where* I put it.

I pulled the decision out of the polling loop entirely, into its own module with
no I/O in it at all:

```elixir
def decide(container_started_at, session_started_at, elapsed_ms, budget_ms) do
  cond do
    container_replaced?(container_started_at, session_started_at) ->
      {:give_up, :container_replaced}

    elapsed_ms >= budget_ms ->
      {:give_up, :timeout}

    true ->
      :poll
  end
end
```

And the check underneath it:

```elixir
def container_replaced?(%DateTime{} = container_started, %DateTime{} = session_started) do
  DateTime.after?(container_started, session_started)
end

def container_replaced?(_container_started, _session_started), do: false
```

If the live container started *after* the session's exec did, the marker is
provably gone. Give up now, fail the session so it is user-recoverable, and
re-queue the task.

Two details I got wrong on the first attempt, and would like to save you:

**The fallback clause returns `false`, not `true`.** If either timestamp is
unknown, we keep polling and let the wall-clock budget decide. A missing
timestamp is not evidence of a dead container, and I would rather wait fifteen
minutes than abandon a healthy run because Docker hiccuped on one request.

**The container start time is re-read on every iteration**, not sampled once at
the beginning. A deploy can replace the container *while* the adoption is
already running. A sample taken a moment too early would keep the poll going for
the whole budget.

The wall-clock budget stays as a backstop. Two independent give-up conditions,
because the precise one depends on timestamps I might not have.

## Why the pure function matters

The polling loop needs Docker, a node proxy, a running Swarm, and forty minutes
of patience. The decision needs four values.

Once they are separated, the test for the incident that cost me a day is this:

```elixir
test "gives up when the container was replaced after the session exec" do
  container = dt("2026-07-14T12:47:00Z")
  session = dt("2026-07-14T10:44:00Z")

  assert AdoptPolicy.decide(container, session, 0, 900_000) ==
           {:give_up, :container_replaced}
end
```

Real timestamps from the real incident. No mocks, no Docker, milliseconds to
run. And because the module is pure, the moduledoc examples are doctests and the
documentation cannot drift away from the behaviour.

This is the part I would most like you to take from the article. A controller
loop is mostly *policy*, and policy has no business being tangled up with the
I/O that feeds it. The reconciler does the same trick for recovery:

```elixir
def recovery_action(LocalPort, _kind, _handle, _presence), do: :fail
def recovery_action(_backend, :bootstrap, _handle, _presence), do: :fail
def recovery_action(_backend, _kind, nil, _presence), do: :fail
def recovery_action(_backend, _kind, _handle, :present), do: :adopt
def recovery_action(_backend, _kind, _handle, _presence), do: :fail
```

Five clauses. No side effects. Exactly the kind of code Elixir is good at, and
exactly the kind that ends up buried in the middle of a `handle_info` if you are
not deliberate about it.

## The order of the steps matters

One more trap, because it took me a second incident to see it.

The boot sweep re-pins runner images to the newest digest. That *replaces
containers* — which means the boot sweep itself creates the very condition that
dooms an adoption.

So the recovery of rolled tasks has to run **before** the stale-session sweep.
Otherwise the sweep hands a session we already know is doomed to an adoption
that will poll for fifteen minutes before reaching the same conclusion.

In a controller loop, the order of your reconciliation steps is not an
implementation detail. It is part of the contract, and it deserves a comment
saying so.

## What I'd tell myself six months ago

- **Every wait needs a deadline.** Not most waits. Every one. If you write a
  poll loop with no exit condition, you have written a hang and just not met it
  yet.
- **The database row is the process.** Anything you want to survive a deploy has
  to be a row, not GenServer state. The GenServer is a fast index over the rows,
  rebuilt on boot.
- **Polling is your `Process.monitor`.** You do not get `:DOWN` messages from
  another machine. Adopt, fail, retry — you are writing a restart strategy by
  hand, so write it like one.
- **Never assume you finish in the container you started in.** Sounds obvious.
  Responsible for most of the bugs in this post.

None of these are new ideas. Kubernetes controllers have worked this way for
years. What surprised me is how naturally the whole thing sits in OTP once you
stop expecting supervision to reach across the network for you. A GenServer, a
sixty-second tick, some pure decision functions and a couple of Ash queries make
a genuinely small controller.

## Conclusion

The stuck card was not a Docker bug or a Swarm bug. It was me assuming that a
file I wrote would still be there when I came back for it.

OTP is very good at supervising things it owns. The moment your real workload
lives somewhere else — a container, another node, somebody else's API — you have
to rebuild the guarantees yourself, and it seems better to rebuild them
explicitly: durable state, a bounded wait, a policy you can test, and a loop
that runs again in a minute because you were probably wrong about something.

Next in this series: [what happens when a polling trigger sits in front of something that costs money on every tick]({% post_url 2026-09-18-63-dollars-in-one-night %}).
That one cost me $63 in a single night.

The process is going on...
