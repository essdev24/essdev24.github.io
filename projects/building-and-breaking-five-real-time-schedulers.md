---
layout: post
title: "Building and Breaking Five Real-Time Schedulers: EDF, LDF, and LLF Compared"
permalink: /projects/building-and-breaking-five-real-time-schedulers/
---

# Building and Breaking Five Real-Time Schedulers: EDF, LDF, and LLF Compared

Real-time embedded systems live or die by their scheduling. Get it wrong in something like an automotive safety system or a healthcare monitor, and a missed deadline isn't just a performance hiccup — it's a failure. For this lab, my team implemented and tested five real-time scheduling algorithms — EDF and LDF for single-node systems, and EDF, LDF, and LLF for distributed multi-node systems — and along the way, one of them missed a deadline we genuinely didn't expect, while a "less intuitive" algorithm sailed right past it.

## Modeling tasks as a graph

The foundation for all five algorithms is a Directed Acyclic Graph (DAG): each task is a node, and directed edges represent precedence — task A must finish before task B can start. The "acyclic" part matters a lot here, since a circular dependency would make scheduling logically impossible. We used Python's NetworkX library to build and traverse these graphs. Root nodes (no predecessors) are immediately schedulable; everything else becomes eligible only once all its predecessors have completed.

*[Image: Figure 1 — the test application's dependency graph: Task 1 as the sole entry point, fanning out to Tasks 2 and 3, which lead to Tasks 4, 5, and 6]*

Each task carries an id, a Worst-Case Execution Time (WCET), a Mean-Case Execution Time (MCET), and a deadline. The platform side models the actual hardware: compute nodes, routers, sensors, and actuators, connected by links with delay and bandwidth. Tasks can only be assigned to compute-type nodes — a distinction that, as it turns out, becomes important later. For this exercise, communication delay between nodes was fixed at zero, isolating pure scheduling behavior from network effects.

*[Image: Figure 2 — the platform model: six compute nodes (green) and four routers (yellow) with labeled link delays]*

## Three strategies, three different philosophies

**Earliest Deadline First (EDF)** is the intuitive one: at every scheduling step, look at whatever tasks are currently eligible and pick the one with the closest deadline. Ties go to the lowest task ID. For multi-node scheduling, the same priority ordering builds the schedule, but tasks then get dispatched to whichever compute node frees up first, letting independent tasks run in parallel.

**Latest Deadline First (LDF)** works backwards. Starting from the leaf tasks (no successors), it repeatedly picks the task with the *latest* deadline and prepends it to the schedule. A predecessor only becomes eligible for this backward selection once all of its successors have already been placed. It sounds counterintuitive — why would picking the latest deadline first help? — but building the schedule in reverse tends to push tasks with tight early deadlines toward the front of the final execution order.

**Least Laxity First (LLF)** prioritizes based on laxity: how much slack a task has before it's at risk of missing its deadline. Laxity for task $i$ at current time $t_c$ is:

$$
L_i = d_i - (t_c + C_i)
$$

where $d_i$ is the task's deadline and $C_i$ is its WCET. Low laxity means a task is in danger — little room left before it has to run — and gets prioritized accordingly. At each step, the schedulable task with the smallest $L_i$ goes to whichever compute node becomes available first.

## Building it out

The backend runs on Python 3 and FastAPI, exposing a single `POST /schedule_jobs` endpoint that takes the combined application and platform JSON and returns all five schedules. It's split into three modules: `backend.py` for the API layer and CORS handling, `algorithms.py` holding the actual scheduling logic (`edf_single_node`, `ldf_single_node`, `edf_multinode_no_delay`, `ldf_multinode_no_delay`, and `ll_multinode_no_delay`), and `config.py` for environment settings like host and port. All input gets validated against a strict `input_schema.json`, and every returned schedule conforms to a fixed output shape — job id, node id, start/end time, deadline, and missed deadlines — so the frontend visualization always knows exactly what it's getting.

## Testing against a real task set

We validated all five implementations with a 51-test pytest suite, organized into five algorithm-specific classes plus one edge-case class. Every test class checks the same correctness categories: the right return structure, correct algorithm naming, every task accounted for in exactly one of `schedule` or `missed_deadlines`, no overlapping execution windows on any node, precedence constraints honored, deadlines respected, and node assignments actually pointing at valid platform nodes. The edge cases specifically probed things like an empty task set, a single task with no dependencies, and a task whose WCET exceeds its own deadline — a guaranteed, forced miss.

All 51 tests passed. The real story, though, came from running all five algorithms against the professor's example task set: six tasks, five dependency edges, deadlines ranging from a tight 40 time units (Task 1) up to a relaxed 120 (Task 6), with every task sharing a WCET of 20 time units.

*[Image: Table 1 — task parameters: WCET, MCET, and deadline for each of the six tasks]*

## The EDF surprise

*[Image: Figure 3 — single-node Gantt charts comparing EDF and LDF]*

On a single node, EDF produced the order 1 → 3 → 2 → 5 → 6 — and Task 4 is conspicuously missing. Here's why: once Task 1 finishes, both Task 2 (deadline 100) and Task 3 (deadline 80) become eligible, and EDF — doing exactly what it's supposed to — picks Task 3 first since its deadline is closer. That decision delays Task 2 until t=60. But Task 4 depends on Task 2, and by the time Task 2 finishes, Task 4 wouldn't complete until t=80 — one time unit past its deadline of 77. EDF correctly detects this and excludes Task 4 (and would exclude its successors, if it had any) from the schedule.

LDF, on the same task set, produced the order 1 → 2 → 4 → 3 → 5 → 6 — and hit every single deadline. Building the schedule backwards from the leaves naturally placed Task 4 right after Task 2, letting it start at t=40 and finish comfortably at t=60, well inside its 77-unit deadline. It's a genuinely useful reminder that "prioritize by nearest deadline" isn't automatically the globally optimal strategy once tasks have dependencies — a locally greedy choice (EDF picking Task 3 over Task 2) can create a cascading failure further down the dependency chain.

## Parallelism fixes what greediness broke

*[Image: Figure 4 — multi-node Gantt charts for EDF and LDF, showing parallel dispatch across compute nodes]*

Move to a multi-node platform, and the picture changes completely. Both EDF and LDF multi-node achieve a makespan of just 60 time units — a 40% reduction from single-node EDF's 100 — by dispatching independent tasks to separate compute nodes. Once Task 1 finishes at t=20, Tasks 2 and 3 start immediately in parallel on separate nodes, and Tasks 4, 5, and 6 follow from t=40. With Task 2 no longer stuck waiting behind Task 3 on a shared node, Task 4 finishes at t=60 — comfortably inside its deadline. The exact same task set, the exact same dependency structure, and a deadline miss simply disappears once real parallelism enters the picture.

## Finding a real bug

*[Image: Figure 5 — the Least Laxity multi-node Gantt chart, including Node 0 (a router) as an execution target]*

The LLF results were where things got interesting for a different reason. The schedule assigned Tasks 1, 3, and 4 to Node 0 — which, according to the platform model, is a router, not a compute node. All deadlines were still technically met, since Node 0's ID exists somewhere in the platform model and the scheduler didn't error out. But it's semantically wrong: routers can't execute tasks on a real distributed system. Digging into why revealed that the EDF and LDF multi-node implementations both pre-filter the node list down to compute-type nodes before scheduling, but the LLF implementation iterates over *all* platform nodes without that filter — meaning it happily treats a router as a valid execution target. It's a clean example of a bug that a "does it pass" test wouldn't necessarily catch (the deadlines were all met!) but a semantic correctness check absolutely should.

## What we took away

A few things stood out clearly by the end of this lab. First, no single scheduling algorithm is universally best — EDF's greedy, locally-optimal choices can create failures further down a dependency chain that a backward-construction strategy like LDF simply avoids for certain task sets. Second, parallelism is an extremely powerful lever: distributing independent tasks across nodes eliminated a deadline miss entirely, without changing a single task parameter. And third, comprehensive testing needs to go beyond "did it produce a valid-looking schedule" — the LLF router bug passed every deadline check while still being fundamentally wrong, which is exactly the kind of defect that only surfaces when you check the *semantics* of the output, not just whether the numbers add up.
