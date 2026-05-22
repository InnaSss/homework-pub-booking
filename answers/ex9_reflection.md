# Ex9 — Reflection

## Q1 — Planner handoff decision

### Your answer

In session sess_6bfdd94fa8fd (Ex7 handoff bridge), the planner's
handoff decision is visible at the executor level rather than the
planner level. The trace shows that at 16:11:46 the planner produced
1 subgoal: "find venue near haymarket for 12". The executor then
called venue_search(near="Haymarket", party_size=12,
budget_max_gbp=2000) which returned 0 results, and immediately
called handoff_to_structured with reason: "loop half identified a
candidate venue; passing to structured half for confirmation under
policy rules". The signal was the subgoal description containing
the phrase "confirmation under policy rules" — the FakeLLMClient
scripted response routes to structured whenever the task involves
policy confirmation rather than open-ended research.

The trace then shows session.state_changed from loop to structured
(round 1), then back from structured to loop with rejection_reason:
"rasa unreachable: Connection refused". This triggered round 2:
the planner produced a new subgoal "retry with larger venue after
rejection", the executor called venue_search(near="Old Town",
party_size=6) which returned 1 result (The Royal Oak, 16 seats),
and handed off again. The architectural lesson: the handoff
decision is encoded in the executor's scripted tool call sequence,
not in a separate routing layer. The bridge reads next_action from
the loop result and dispatches accordingly.

### Citation

- sessions/ex7-handoff-bridge/sess_6bfdd94fa8fd/logs/trace.jsonl — bridge.round_start round 1,
  executor.tool_called handoff_to_structured at 16:11:46
- sessions/ex7-handoff-bridge/sess_6bfdd94fa8fd/logs/trace.jsonl — session.state_changed
  loop→structured and structured→loop, round 1 and round 2

---

## Q2 — Dataflow integrity catch

### Your answer

In session sess_9cebe396e4d0 (Ex5 Edinburgh research), the integrity
check would have caught a fabrication that manual review missed.
The task in SESSION.md specified party_size=6 and
near="Haymarket". The executor called venue_search with
party_size=10 and near="Edinburgh City Centre" — neither parameter
matches the task. All three search attempts returned 0 results and
workspace/ remained empty: no flyer.html was produced.

A human reviewer looking at the trace would see success: true on
every tool call and might conclude the agent ran correctly. The
integrity check catches this because it compares facts in the
flyer against _TOOL_CALL_LOG ground truth — but here there was no
flyer to check, which is itself a catch: the scenario requires
generate_flyer to be called, and it never was.

The contrast is visible in the same session directory: SESSION.md
specifies the required tool sequence explicitly (venue_search →
get_weather → calculate_cost → generate_flyer → complete_task),
and the trace shows only venue_search was called three times before
handoff_to_structured. The integrity check is the only layer that
would surface this — every call returned success: true, so the
executor had no signal that anything was wrong until 0 results
came back three times.

### Citation

- sessions/ex5-edinburgh-research/sess_9cebe396e4d0/logs/trace.jsonl — party_size=10 vs
  SESSION.md party_size=6; near="Edinburgh City Centre" vs near="Haymarket"
- sessions/ex5-edinburgh-research/sess_9cebe396e4d0/SESSION.md — required tool sequence
- sessions/ex5-edinburgh-research/sess_9cebe396e4d0/session.json — state: "executing", result: null

---

## Q3 — First production failure

### Your answer

The first production failure I would expect is the Rasa structured
half becoming unreachable mid-session — exactly what happened in
sess_6bfdd94fa8fd, where both rounds returned rejection_reason:
"rasa unreachable: Connection refused". In a real pub-booking
business, Rasa restarts for a config push at 2am, and the bridge
gets a refused connection on the forward handoff. The session is
now in state "executing" with a handoff_to_structured.json sitting
in ipc/ — the forward handoff was written but never consumed.

The primitive that surfaces this is the atomic-rename IPC: the
handoff file is written atomically to ipc/handoff_to_structured.json
before the HTTP call is made. If the structured half is
unreachable, the file stays in ipc/. On the next bridge run
(retry or restart), the file's presence signals an in-flight
handoff that was never acknowledged. Without atomic-rename, a
partial write could leave a corrupt handoff that looks valid but
contains half a JSON payload — the bridge would parse it, get
garbage booking data, and confirm a nonexistent reservation.

The atomic-rename guarantee means the file either exists and is
complete, or does not exist. The bridge can therefore treat
presence as a reliable "structured half has not yet confirmed"
signal and retry safely, without risking a double-booking or a
silent drop.

### Citation

- sessions/ex7-handoff-bridge/sess_6bfdd94fa8fd/logs/trace.jsonl — session.state_changed
  structured→loop, rejection_reason "rasa unreachable: Connection refused", rounds 1 and 2
- sessions/ex7-handoff-bridge/sess_6bfdd94fa8fd/ipc/handoff_to_structured.json — handoff file
  written before HTTP call
