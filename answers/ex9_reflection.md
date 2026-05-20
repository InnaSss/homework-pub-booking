# Ex9 — Reflection

## Q1 — Planner handoff decision

### Your answer

In session sess_9cebe396e4d0 the planner produced 3 subgoals at
16:53:12, all assigned_half: "loop" (visible in session.json under
planner.subgoals). None were assigned to the structured half — the
task description contained no policy/rules language, only open-ended
research instructions ("compile a list", "select the most suitable",
"draft the flyer"). The DefaultPlanner reads the subgoal description
and routes to structured only when it sees constraint language like
"under policy rules" or "confirm within limits".

The handoff_to_structured that did occur (16:53:58) was not a planned
subgoal assignment — it was the executor bailing out after 3 failed
venue_search calls, passing the dead end to the structured half with
reason: "Venue search returned no results despite multiple parameter
adjustments". This is a failure-mode handoff, not a planned one.

The distinction matters: planned handoffs carry a subgoal with a
success criterion; failure handoffs carry a reason and raw search
data. A structured half receiving a failure handoff needs to detect
this difference, or it will try to confirm a booking that was never
found.

### Citation

- sess_9cebe396e4d0/session.json — planner.subgoals[*].assigned_half
- sess_9cebe396e4d0/logs/trace.jsonl — executor.tool_called handoff_to_structured at 16:53:58

---

## Q2 — Dataflow integrity catch

### Your answer

Session sess_9cebe396e4d0 shows a clean failure of dataflow integrity
that the integrity check would have caught. The task in SESSION.md
specified party_size=6 and near='Haymarket'. The executor called
venue_search with party_size=10 and near='Edinburgh City Centre' —
neither parameter matches the task. All three search attempts returned
0 results, and workspace/ remained empty (no flyer.html produced).

The offline run (sess_0958e6435e99) shows the contrast: the
FakeLLMClient followed the required tool sequence exactly, produced
flyer.html (1415 bytes), and the integrity check returned "verified 4
fact(s) against tool outputs". The real LLM hallucinated both the
party size and the location, producing a session that looked
successful (all tool calls returned success: true) but delivered
nothing.

This is why integrity checks must compare against _TOOL_CALL_LOG
ground truth, not against "did the tool return success". Every call
in sess_9cebe396e4d0 returned success: true — the executor had no
signal that anything was wrong until 0 results came back three times.

### Citation

- sess_9cebe396e4d0/logs/trace.jsonl — party_size=10 vs SESSION.md party_size=6
- sess_9cebe396e4d0/workspace/ — empty, no flyer.html
- sess_0958e6435e99 — offline run, dataflow OK, 4 facts verified

---

## Q3 — Removing one framework primitive

### Your answer

I would keep session directories and remove atomic-rename IPC last.
The argument from sess_9cebe396e4d0: that session exists in state
"executing" with no result and an empty workspace. Without the
directory I cannot answer "what did the executor actually call" — the
trace.jsonl is the only record that party_size=10 was used instead of
6. Remove session directories and this failure becomes invisible; the
only observable symptom is "no flyer" with no explanation.

Tickets (tk_0fd24a1c, tk_a8986418, tk_dfc0e5f2 from sess_0958e6435e99)
I could reconstruct as append-only lines in trace.jsonl. Atomic-rename
IPC I could replace with directory polling with a small latency cost.
The forward-only state machine matters but is enforceable in code
without a separate primitive.

Session directories are the audit layer. Everything else can be
rebuilt from them. They cannot be rebuilt from anything else.

### Citation

- sess_9cebe396e4d0/session.json — state: "executing", result: null
- sess_9cebe396e4d0/logs/trace.jsonl — only source of ground truth for hallucinated args
- sess_0958e6435e99 tickets — tk_0fd24a1c, tk_a8986418, tk_dfc0e5f2
