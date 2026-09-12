---
layout: post
title: "Why PID Controllers Fall Apart Under Network Delay: Lessons from an Inverted Pendulum"
permalink: /projects/pid-controllers-fall-apart-under-network-delay/
---

# Why PID Controllers Fall Apart Under Network Delay: Lessons from an Inverted Pendulum

The inverted pendulum is one of those problems every controls student eventually runs into, and for good reason — it packs almost every core difficulty of dynamic systems into one deceptively simple setup. It's inherently unstable, its dynamics are nonlinear (thanks to trigonometric coupling between the cart and the pendulum), and the cart's motion and the pendulum's angle are tightly, dynamically linked. Unlike a hanging pendulum that happily settles at the bottom, an inverted one requires continuous, active correction just to stay upright. For this project, my partner and I modeled the physics, built a multi-threaded C++ simulator and PID controller, tuned it by hand, and then deliberately broke it with network delay and jitter to see exactly how it would fail.

## Modeling the physics

To keep the math tractable, we simplified the system with a few standard assumptions: frictionless cart motion, a perfectly rigid pendulum rod, point masses for the cart and pendulum, and everything confined to a single vertical plane.

Even with those simplifications, the coupling between cart and pendulum makes the equations of motion genuinely nonlinear. The cart's horizontal acceleration comes out to:

$$
\ddot{x} = -\frac{m_p l}{m_c + m_p}\ddot{\theta}\cos(\theta) - \frac{m_p l}{m_c + m_p}\dot{\theta}^2\sin(\theta) + \frac{F}{m_c + m_p}
$$

The first term accounts for the pendulum's inertia pushing back against the cart, the second captures the centripetal force from the pendulum's angular velocity, and the last term is just the normalized control force applied to the cart.

Using the parallel-axis theorem, the pendulum's angular acceleration about its pivot works out to:

$$
\ddot{\theta} = \frac{m_p g l}{I_p + m_p l^2}\sin(\theta) - \frac{m_p l}{I_p + m_p l^2}\ddot{x}\cos(\theta)
$$

The $m_p l^2$ term shifts the axis of rotation from the center of mass to the pivot, increasing the effective rotational inertia. Physically, gravity is constantly trying to pull the pendulum away from vertical (first term), and the cart's own reactionary acceleration is what fights back against that (second term).

## Closing the loop: PID control

To keep the pendulum upright, we feed the error between the measured and desired state into a standard PID controller:

$$
u(t) = K_p e(t) + K_i \int e(t)\,dt + K_d \frac{de(t)}{dt}
$$

Each term does a different job: proportional gain gives immediate correction relative to the current error, integral gain accumulates past error to kill steady-state offset, and derivative gain looks at the rate of change of the error to anticipate where things are headed and add damping.

Since this runs on a discrete embedded system rather than continuous hardware, the continuous law had to be converted into a discrete recurrence — computed at each sampling interval from current and past error values. The raw derivative term is also extremely sensitive to sensor noise, so we added a derivative filter coefficient (N): a lower N smooths harder and protects against erratic control signals, while a higher N reacts faster but lets more noise through.

## The C++ architecture

We built the controller as a `PIDController` class, splitting the header (`controller.h`) from the implementation (`controller.cpp`) to keep the interface clean. Internally, the class tracks a short history of error and output values and exposes only `output()` and `update_params()` publicly. The actual discrete PID recurrence, computed every timestep, looks like this:

```cpp
// Shift error history one step back
e2 = e1; e1 = e0; e0 = error;

// Discrete PID recurrence equation
u0 = ke0 * e0 + ke1 * e1 + ke2 * e2
   + ku1 * u1 + ku2 * u2;

// Anti-windup output clamp
if (u0 > u_max) u0 = u_max;
else if (u0 < u_min) u0 = u_min;

// Shift output history
u2 = u1; u1 = u0;
```

The anti-windup clamp matters more than it looks — without it, the integral term can accumulate unbounded error during saturation and cause wild overshoot once the system recovers.

To simulate realistic network jitter, we resample a fresh random delay value at every single simulation step rather than applying one fixed offset — jitter, by definition, has to vary iteration to iteration:

```cpp
// Sample a fresh random jitter value each iteration
int actual_jitter = (m_params.jitter > 0)
    ? std::experimental::randint(0, m_params.jitter)
    : 0;

// Use randomised jitter when computing the read index
int delay_index = calculate_delay_index(
    i, buffer_size, m_params.delta_t,
    m_params.delay, actual_jitter);
```

**Concurrency.** Real-time simulation needs physics calculations, sensor reads, and network communication all running simultaneously. Rather than paying the overhead of separate OS processes, we used threads sharing a common memory space for fast communication — which of course opens the door to race conditions if one thread reads a variable while another is mid-write. We handled this with the standard C++ concurrency primitives: `std::mutex` for exclusive access, `std::lock_guard` for automatic scope-based locking, and `std::condition_variable` so threads can sleep until data is actually ready instead of burning CPU in a polling loop.

**Build system.** The whole thing is built with CMake, with Boost handling WebSocket communication and nlohmann/json handling data serialization — keeping the build portable and letting `make` do the heavy lifting.

## Tuning it by hand

Manually tuning a nonlinear, multi-variable system makes the isolated role of each gain term very visible:

- **Proportional gain** acts like the primary restorative spring — pushing it up gives a faster rise time, but push too far and the system goes underdamped, with the cart aggressively overcorrecting into overshoot.
- **Derivative gain** acts as artificial friction. By anticipating where the error is heading, it effectively brakes the cart as the pendulum approaches vertical, smoothing out the oscillations that a high proportional gain introduces on its own.
- **Integral gain** cleans up steady-state droop — without it, the pendulum tends to hang at a slight, persistent offset instead of resting exactly vertical.

*[Image: Figure 1 — undamped oscillation with low gains (Kp=0.5, Ki=0, Kd=2.0), showing growing amplitude from insufficient derivative damping]*

The final tuned values we landed on:

| Simulation Case | Kp | Ki | Kd | Reference (rad) |
| --- | ---: | ---: | ---: | ---: |
| Baseline stabilisation | 100 | 0 | 70 | 0.00 |
| Positive step reference | 180 | 1 | 70 | +0.10 |
| Negative step reference | 180 | 0 | 70 | −0.10 |

The higher Kp needed for the ±0.1 rad cases makes sense physically — driving the pendulum to a non-zero angle means fighting gravity continuously rather than just holding a balance point. And the small integral term (Ki = 1) in the positive-reference case exists specifically to eliminate the residual steady-state offset that shows up without it.

*[Image: Figure 4 — stable baseline simulation at 0 rad reference, converging within ~1s with minimal overshoot]*

*[Image: Figure 5 — positive reference angle simulation showing the integral term correcting steady-state offset]*

*[Image: Figure 6 — negative reference angle simulation, with a note on the inflated "7062% overshoot" figure being a percentage artifact of a small target magnitude, not an actual physical instability]*

## Breaking it on purpose: delay and jitter

Once the controller was working cleanly, we intentionally degraded it with network delay and jitter to see how a real embedded deployment — with actual sensor and communication latency — would behave.

**Delay** acts as a pure phase shift: a lag between when the pendulum's angle is actually measured and when the corrective force gets applied. Once the fixed delay grew past the sampling interval, the controller started reacting to stale information — applying a strong corrective force to counter a tilt that the pendulum had, by the time the force landed, already started swinging back out of. The correction ended up accelerating the pendulum in the wrong direction. The result was aggressive, undamped oscillation, dramatically longer settling times, and eventually total instability.

*[Image: Figure 2 — erratic force spikes and noisy control output under network degradation]*

**Jitter** turned out to be a different, arguably nastier problem. Because the derivative term is fundamentally a rate of change ($\frac{de(t)}{dt}$), non-uniform timing between samples makes the controller perceive massive, instantaneous velocity spikes even when the pendulum is physically moving smoothly. Those phantom spikes translated directly into violent, noisy thrusts from the cart actuator — the kind of behavior that would chew through real actuator hardware fast and makes smooth stabilization nearly impossible.

*[Image: Figure 3(a–c) — comparison of undamped, damped, and jittered (2000µs) setup configurations]*

## What this taught us

The clearest takeaway from the whole exercise: a mathematically correct PID controller isn't enough on its own. A controller tuned perfectly for a zero-delay, zero-jitter simulation will fail — sometimes catastrophically — the moment it meets real network latency. The derivative term's sensitivity to jitter specifically makes a strong case for a low-pass derivative filter in any real deployment, trading a bit of responsiveness for a large gain in stability.

We also leaned on some simplifying assumptions that don't hold in the real world — frictionless cart motion and a perfectly rigid rod chief among them — and manual tuning, while a great way to build intuition, doesn't guarantee mathematically optimal gains. If I were extending this, the obvious next step is a Linear Quadratic Regulator (LQR), which handles multi-variable state-space systems more gracefully than a standard PID, or a Kalman filter to let the system natively predict and filter out jitter rather than just reacting to it after the fact.

## Takeaways

This project was as much about embedded software architecture — threading, synchronization, anti-windup clamping — as it was about control theory itself. The math tells you what the *ideal* controller should do; the delay and jitter analysis is what tells you whether it'll survive contact with real hardware. If you're building any kind of real-time control system, that second question deserves at least as much attention as the first.
