---
layout: post
title: "Building a Two-Loop Fan Control System: Bugs, Bilinear Transforms, and a Ghost in the Simulation"
permalink: /projects/building-a-two-loop-fan-control-system/
---

# Building a Two-Loop Fan Control System: Bugs, Bilinear Transforms, and a Ghost in the Simulation

Some projects go exactly to plan. This one didn't — and honestly, that's what made it worth writing up. My team built a distributed fan control system across four Raspberry Pi units, with two independent PID control loops talking to each other over a message queue. Along the way we found three real bugs through documentation review before ever touching the tuning dashboard, then spent a tuning session that included two dead-end iterations, a regression we had to prove *wasn't* a coding defect, and a genuine unsolved mystery that's still sitting there at the end of the report.

## The system

The architecture splits into two structurally identical control loops. Loop 0 runs `control_service` on one Raspberry Pi (rpi2) talking to `fan_service` on another (rpi3); loop 1 is the same pair, duplicated as `control_service1`/`fan_service1` on rpi4/rpi5. A central `server` service handles parameter updates, run/stop control, and status reporting for all four Pis independently. All inter-service traffic — including the two messages that actually close each PID loop — gets routed through a ZeroMQ-backed `msg_queue` middleware rather than going service-to-service directly.

*[Image: Figure 1 — distributed services overview, showing the rpi2↔rpi3 and rpi4↔rpi5 loop pairs plus their separate connections to the central server]*

Unlike a simpler single-service setup, the control service here doesn't compute its own tracking error. Instead, the fan service measures the actual flap angle, computes the error against its target, and sends that error over the queue; the control service receives it, runs the PID law, and sends a PWM command back. Two queue semantics handle this traffic: a FIFO mode for one-shot events like run/stop commands, and a "latest-value" mode for continuously updated values like the angle error and PWM command — which makes sense, since a controller reading a stale queued value instead of the most recent one would be actively harmful.

## The control law

The continuous-time controller is a standard PID with a first-order filtered derivative term:

$$
C(s) = K_p + \frac{K_i}{s} + \frac{K_d N s}{s + N}
$$

where $N$ is a derivative filter coefficient that caps how much high-frequency gain the derivative branch can inject — without it, sensor noise gets amplified straight into the actuator. Discretizing this with the bilinear (Tustin) transform, $s \approx \frac{2}{T_s}\frac{1-z^{-1}}{1+z^{-1}}$, produces a recursive difference equation:

$$
u[k] = \frac{1}{a_0}\Big(b_0,e[k] + b_1,e[k-1] + b_2,e[k-2] - a_1,u[k-1] - a_2,u[k-2]\Big)
$$

In the actual implementation, the coefficients get pre-computed once and normalized by $a_0$, so each call to `output()` only needs a simple recursive sum over the last two error samples and the last two output samples, which are held as controller state:

```cpp
double ControlService::output(double error)
{
    // Shift error history: e(k-2) <- e(k-1) <- e(k) <- new error
    e2 = e1; e1 = e0; e0 = error;

    // Shift output history: u(k-2) <- u(k-1) <- previous output
    u2 = u1; u1 = u0;

    // Discrete PID recursion
    double u = ku1 * u1 + ku2 * u2 + ke0 * e0 + ke1 * e1 + ke2 * e2;

    // Saturate to actuator limits
    if (u > UMAX) u = UMAX;
    else if (u < UMIN) u = UMIN;

    u0 = u;
    return u;
}
```

*[Image: Figure 2 — the live tuning dashboard: service health matrix, run control, PID gain sliders, and telemetry plots]*

## Finding bugs before tuning anything

Before touching a single gain, we systematically checked the finished implementation against the task documentation — message routing map, PID derivation, fan hardware semantics. Most of it checked out cleanly. Three things didn't.

**Bug 1: a silent uninitialized-state defect.** One of the two control service headers declared its state variables like this:

```cpp
double e2, e1, e0, u2, u1, u0 = 0;  // only u0 is actually initialized
```

In C++, an initializer in a multi-variable declaration only applies to the variable it's directly attached to — so `e2`, `e1`, `e0`, `u2`, `u1` were left holding indeterminate garbage values, and only `u0` was genuinely zeroed. This was invisible at runtime because `init()` always calls `reset()`, which explicitly zeroes all six variables before the control loop starts — but it silently depended on that call ordering never changing, and would have produced a garbage-derivative kick on the very first control cycle if `reset()` were ever skipped. We fixed the declaration to explicitly initialize every variable, matching the twin file for the other loop.

**Bug 2: `run()` never actually returning.** This was the big one. `main.cpp` implements its own outer service loop, gated by a start/stop flag:

```cpp
while (true) {
    bool should_run = service_loop_running.load();
    if (should_run) {
        service.run();     // expects run() to do ONE cycle and return
        running = true;
    } else if (running && !should_run) {
        service.reset();   // can only execute if run() returns!
        running = false;
    }
    std::this_thread::sleep_for(std::chrono::milliseconds(50));
}
```

`run()` was supposed to perform exactly one receive-compute-send cycle and hand control back. Instead, the original implementation wrapped that cycle in its own internal `while(true)`, which never returned. This broke two things at once: the stop mechanism became dead code, since the `reset()` branch could literally never execute — meaning the server could never actually stop a running loop — and the unit test suite started hanging indefinitely, since tests expecting `run()` to process one message and return would instead block forever, getting silently killed by the test runner's timeout with zero log output. The fix removed the internal loop (and its redundant sleep, since `handleServiceLoop` already sleeps 50ms between calls) from all four service files.

**Bug 3: a PWM payload field name mismatch.** An earlier revision had the classic distributed-systems failure mode — sender and receiver disagreeing on a payload field name. This was already caught and fixed by the time we did the final review, but we verified it directly: the sender writes the PWM value under the key `"pwm_value"`, and the receiver reads that same key back out. Worth noting: this field name is intentionally distinct from the message-routing port name (`"pwm_cmd"`) used elsewhere — the two serve different purposes and were never required to match each other, only to be internally consistent on each side, which they now are.

After fixing all three, the full test suite — 132 cases — passed in its entirety.

## Tuning: the part that didn't go smoothly

Tuning was done interactively through the live dashboard, and it's worth documenting the failed attempts alongside the working result, since the failures are what actually explain the final gain choice.

**Iteration 1 — code defaults ($K_p{=}0.5$, $K_i{=}0$, $K_d{=}2$).** Both loops crawled toward their targets over roughly 100 seconds, in an almost straight-line ramp rather than a typical PID response curve.

*[Image: Figure 3 — Iteration 1 telemetry showing the extremely slow ramp]*

With $K_d$ more than 2.5x larger than $K_p$, the derivative term was suppressing fast movement before the proportional term had any authority to act. This told us the default gains weren't a usable starting point on their own — $K_d$ needed to come down hard.

**Iteration 2 — proportional-only ($K_p{=}1.5$ / $K_p{=}3$, $K_i{=}K_d{=}0$).** With derivative removed and proportional gain pushed up, both loops reached their target region in under a second — a massive improvement over Iteration 1. But with no damping and no integral action, both settled into a persistent noisy oscillation band around (not exactly on) the target — exactly what you'd expect from a pure-P controller on a noisy sensor.

*[Image: Figure 4 — Iteration 2 telemetry showing fast rise time but a noisy steady-state band on both loops]*

**Iteration 3 — adding small $K_i$ and $K_d$ ($K_p{=}1.5$, $K_i{=}0.05$, $K_d{=}0.10$).** This was meant to clean up Iteration 2's noise. Instead it regressed badly: both loops settled into a large, sustained flap-error offset (roughly −50 to −65) that never shrank over a full 60-second run.

*[Image: Figure 5 — Iteration 3 telemetry showing the sustained offset that never converges]*

Before assuming this was a tuning problem, we checked whether it might actually be an integral-windup coding defect — verifying algebraically that the discrete transfer function's numerator correctly cancels the integrator pole when $K_i{=}0$, confirming Iteration 2 genuinely had no integral action and that this wasn't a runaway bug. The real explanation turned out to be a physical actuator limitation: with $U_{MIN}{=}0$, the PWM signal can only coast toward zero, never go negative. Once the added gains caused enough overshoot to pass the target, the controller had no way to actively correct back down — it could only wait for the fan to coast, and got stuck with the PWM command pinned near its floor. Lesson: overshoot needs to be brought under control via $K_p$/$K_d$ *before* introducing $K_i$, not alongside it.

**Iteration 4 — reverting to Iteration 2 gains, and discovering a freeze.** Back to the known-good proportional-only gains, but this time watching the raw log stream alongside the chart. Loop 1 converged normally, exactly as before. Loop 0 went completely flat after its initial step and stayed that way for the entire run.

*[Image: Figure 6 — Iteration 4 telemetry: loop 0 flat-lined, loop 1 converging normally]*

Digging into the logs told the real story. Loop 0's control service log showed the received error pinned at exactly `100.000000` — the raw, unmodified target — for the entire run, with the PID math computing correctly from that stale input ($1.5 \times 100 = 150$). The fan service log confirmed it was receiving PWM commands and actually setting them, but the angle error it reported back never changed from the raw target.

*[Image: Figure 7 and Figure 8 — loop 0's control_service and fan_service logs, both showing the error frozen at the raw target value]*

Loop 1, running identical code with the same class of gains in the same session, was genuinely converging — its logged error dropped from 200 down to 111 over the run, and its PWM/angle values were changing run to run exactly like a live, responding system should.

*[Image: Figure 9 and Figure 10 — loop 1's logs showing a real, decreasing error and changing telemetry]*

Since loop 0 had worked correctly under the exact same gains in Iteration 2, this pointed to an intermittent simulation or session-state issue specific to loop 0 — not a defect in `control_service.cpp` itself, which was demonstrably computing correctly from whatever error it was fed. The problem was upstream: the error value it received from the fan simulation had simply stopped updating.

**Iteration 5 — a recovery attempt.** We reset loop 0's fan target and re-verified loop 1 at $K_p{=}1.5$ as an additional sanity check. Loop 1 confirmed a clean, fast response with a small residual error (about 3.5% of target). Loop 0 came back with the exact same frozen signature as Iteration 4 — same flat line, same stuck error value.

*[Image: Figure 11 — Iteration 5 telemetry: loop 1 converging cleanly, loop 0 still frozen after the reset attempt]*

That persistence across a reset attempt is what rules out a one-off glitch — it points at something isolated to loop 0's simulation or session state, most likely on the dashboard/backend side rather than anything reachable from the tuning interface or fixable in the service code itself.

## Where tuning landed

| Control Loop | Kp | Ki | Kd | Status |
| --- | ---: | ---: | ---: | --- |
| loop 0 (`control_service`) | 1.5 | 0 | 0 | Baseline works (Iteration 2); untuned further due to the freeze |
| loop 1 (`control_service1`) | 1.5 | 0 | 0 | Validated: sub-second convergence, ~3.5% residual error |

Loop 1's $K_p{=}1.5$ result is the strongest validated outcome from the whole session — fast, consistent, and a clean proportional-only response. It's still missing derivative damping and integral offset correction, so some residual noise and steady-state offset is expected; the natural next step (small $K_d$ around 0.1–0.3 to damp oscillation, then a very small $K_i$ around 0.05, each re-tested in isolation) was identified based on the Iteration 3 failure mode but never completed, since the loop 0 freeze consumed the rest of the available tuning time.

Flap target 0 was never tested at all in this session — the time went to fixing the Iteration 3 regression and then chasing the loop 0 freeze. It's flagged explicitly as outstanding work rather than filled in with unverified data, and is expected to be the hardest of the three targets anyway, since driving the flap down to its lowest position runs into the same $U_{MIN}{=}0$ asymmetric-actuation limitation implicated in Iteration 3.

## What this taught us

The biggest lesson here isn't really about PID gains — it's about how a defect can hide in plain sight. The `run()` infinite-loop bug didn't show up in any single function's logic; the PID math, the message routing, and the fan actuation were all individually correct. It only became visible once we traced the actual control-flow *contract* between `run()` and its caller — the assumption that "run() performs one cycle and returns" is easy to silently violate without anything looking obviously wrong in isolation. Checking that kind of caller/callee contract explicitly, rather than assuming a self-contained loop behaves the way its caller expects, turned out to matter more than any amount of staring at the PID recursion itself.

The tuning side reinforced a similar idea: not every bad result is a tuning problem, and not every good-looking result rules out a hidden fault. The Iteration 3 regression looked like it could easily have been an integral windup bug — it took an actual algebraic check against the discrete transfer function to rule that out and correctly attribute it to actuator saturation instead. And the loop 0 freeze is the flip side of the same coin: a completely valid, correctly-computing control service producing useless output because something upstream, outside the code we were responsible for, quietly stopped feeding it real data.

## Where it stands

Two things remain open: getting loop 0's simulation/session freeze resolved so its gains can actually be refined past the Iteration 2 baseline and flap target 0 can finally be tested, and applying the small-$K_d$-then-small-$K_i$ refinement plan to loop 1's already-working baseline. Neither is a code defect in the sense the three bugs above were — they're the next steps in an otherwise successful build, not corrections to a broken one.
