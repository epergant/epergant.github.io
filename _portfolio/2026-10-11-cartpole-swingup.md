---
title: "Cart-pole swing-up: classical control vs. MPC vs. RL"
collection: portfolio
permalink: /portfolio/cartpole-swingup/
date: 2026-10-25
excerpt: "One under-actuated benchmark, six controllers, five axes of comparison: sample complexity, robustness to model mismatch, constraint handling, compute, and engineering effort. <em>In progress, updated as results come in.</em><br/><img src='/images/cartpole/teaser.png' width='500' alt='Time-lapse of the cart-pole swing-up'>"
---

<p style="font-size: 0.9em;"><strong>Status:</strong> in progress &middot; <strong>Last updated:</strong> October 25, 2026 &middot; <strong>Code:</strong> <a href="https://github.com/epergant/cartpole-swingup">github.com/epergant/cartpole-swingup</a></p>

In my day job and in my research, I keep coming back to the same question: when should you use model predictive control (MPC), and when should you use reinforcement learning (RL)? In buildings, every data point for that question costs a month of field testing. This project asks the same question on a problem where a trial takes a fraction of a second: swinging up a pole on a cart and balancing it upright.

The cart-pole is small, but it has the features that make the real question hard. It is nonlinear and under-actuated (one force, four states), and it has hard limits on both the actuator and the state. Because I know the exact model, I can also do something I can never do in a house: make the model *wrong* on purpose, by exactly as much as I choose, and watch what each controller does.

This page is a running log. I'll post results as they come in and record each change in the [update log](#update-log) at the bottom.

## The problem

Swing the pole up from hanging and hold it upright, subject to two hard limits:

- **Track:** the cart must stay within $$\lvert x \rvert \le 1.2$$ m. Hitting the end stop ends the episode.
- **Actuator:** the force on the cart must stay within $$\lvert F \rvert \le 10$$ N. Larger requests get clipped, and the clip is counted as a violation.

The plant is a uniform rod (0.25 kg, 0.8 m) on a 1 kg cart, with viscous friction at the cart and at the pivot. With $$\theta = 0$$ upright, the equations of motion are

$$
(M+m)\,\ddot{x} + m l \cos\theta\,\ddot{\theta} - m l \sin\theta\,\dot{\theta}^2 = F - b_c \dot{x}, \qquad
m l \cos\theta\,\ddot{x} + J\,\ddot{\theta} - m g l \sin\theta = -b_p \dot{\theta},
$$

where $$l = L/2$$ and $$J = mL^2/3$$. The simulator integrates them with RK4 at 2 ms, and the controllers run at 50 Hz (every 20 ms).

An episode lasts 10 s. It counts as a **success** if the cart never hits the end stop and $$\lvert \theta \rvert < 0.1$$ rad for the final 2 s.

**The ground rule:** every controller is designed or trained on the *nominal* plant. Model mismatch (pole mass and length off by up to ±50%) and actuator latency are applied only to the simulated "true" plant. The controllers never get told about them.

## The contenders

| Method | Model it uses | Track limit | Force limit |
|---|---|---|---|
| LQR | linearized at upright | none | none (clipped by the actuator) |
| Energy shaping + LQR catch | exact | cart-centering PD term | saturated |
| iLQR MPC (box-DDP) | exact | soft penalty | hard |
| Direct-collocation MPC (CasADi + IPOPT) | exact | hard | hard |
| SAC (model-free RL) | none | reward + termination | tanh squashing |
| PETS (model-based RL) | learned neural ensemble | soft planner penalty | hard (CEM bounds) |

LQR is there as a reference point: it's optimal near upright and useless far from it. Energy shaping (Åström & Furuta, 2000) is the textbook classical answer. The two MPCs plan on the exact nonlinear model, and the two RL methods learn from interaction.

## Five axes of comparison

1. **Sample complexity:** how many environment steps it takes to reach 80% success. The model-based designs are counted as zero; their cost shows up under engineering effort instead.
2. **Robustness:** the success rate when the true pole mass and length differ from the design model.
3. **Constraints:** end-stop hits, and how often the controller asked for more force than the actuator has.
4. **Compute:** microseconds per control step, single-threaded.
5. **Engineering effort:** hours spent, restarts, and configurations tried. Nobody reports this axis, and it's the one practitioners care about most. Every work session gets logged, and every training run records a hash of its config automatically.

A separate study adds actuator latency from 0 to 50 ms.

## First results: the model-based controllers

Everything below is a **single episode with seed 0**, so treat it as a first look, not as statistics. The multi-seed sweeps are next.

<video src="/images/cartpole/nominal.mp4" autoplay loop muted playsinline controls style="width: 100%; max-width: 900px;"></video>
<p style="font-size: 0.85em;"><em>The nominal plant. Each column is one controller. Top row: the cart on the track, with the hatched regions past the end stops; the pole turns green inside the success band; for the MPCs, faint "ghost" poles show the current plan. Below that: the phase portrait (the dashed curve is the energy of the upright pole), the pole energy (−1 is hanging, 0 is upright), and the force requested vs. applied.</em></p>

| Controller | Success | Upright at | Max \|x\| | Force-limit violations | Median compute |
|---|---|---|---|---|---|
| LQR (from hanging) | no | — (end stop at 7.9 s) | 1.22 m | 397 of 500 steps | ~9 µs |
| Energy shaping + LQR | yes | 3.4 s | 0.50 m | 0 | ~9 µs |
| iLQR MPC | yes | 1.6 s | 0.65 m | 0 | ~16 ms |
| Collocation MPC | yes | 1.6 s | 0.63 m | 0 | ~18 ms |

The four controllers behave very differently:

- **LQR** only makes sense near upright. From hanging, it demands far more force than exists on almost every step, and the cart drifts into the end stop.
- **Energy shaping** takes about three pumping swings. In the phase portrait you can watch the trajectory spiral outward until it lands on the dashed upright-energy orbit and rides it to the top, where LQR catches it.
- **Both MPCs** do it in one large, deliberate swing that uses more of the track. The ghost poles show the whole swing-up planned 1.6–2 s ahead.

<img src="/images/cartpole/strobe.png" alt="Time-lapse of each controller's swing-up, pole drawn every 0.1 s" style="width: 100%; max-width: 520px;">
<p style="font-size: 0.85em;"><em>Time-lapse of the LQR, energy-shaping and iLQR runs: the pole is drawn every 0.1 s and colored by time.</em></p>

## When the model is wrong

Here are the same controllers, still designed on the nominal model, run on a different "true" plant:

| True plant | Energy + LQR | iLQR MPC | Collocation MPC |
|---|---|---|---|
| Pole mass +50% | ✅ 3.3 s | ✅ 5.3 s | ✅ 2.6 s |
| Pole length −25% | ❌ | ❌ | ✅ 3.0 s |
| 40 ms actuator latency | ✅ 4.1 s | ❌ | ✅ 2.1 s |

<video src="/images/cartpole/length075.mp4" autoplay loop muted playsinline controls style="width: 100%; max-width: 900px;"></video>
<p style="font-size: 0.85em;"><em>The true pole is 25% shorter than the design model. Watch the energy trace: energy shaping pumps toward the upright energy of the <strong>nominal</strong> pole, so the real pole never quite reaches the catch region.</em></p>

Why does a 25% length error hurt so much more than a 50% mass error? For a uniform rod, the pole's own dynamics are

$$
\ddot{\theta} = \frac{3g}{2L}\sin\theta - \frac{3}{2L}\cos\theta\,\ddot{x},
$$

and the mass drops out entirely. Mass only enters through the coupling to the cart, and here that turns out to matter much less. Length sets the pendulum's natural frequency and the target energy. Get it wrong, and every controller aims at the wrong thing.

<video src="/images/cartpole/latency40.mp4" autoplay loop muted playsinline controls style="width: 100%; max-width: 900px;"></video>
<p style="font-size: 0.85em;"><em>A 40 ms delay between command and actuation. iLQR gets the pole to the top but can't stop it there: its corrections arrive late, and the pole keeps going over the top in full rotations (the repeated bumps in its energy trace). Energy shaping's LQR catch and the collocation MPC both hold.</em></p>

Collocation surviving both cases is interesting, but it's one episode. I won't draw conclusions until the 5-seed sweeps are in.

## The compute elephant in the room

The control period is 20 ms. In single-threaded Python, the iLQR MPC takes about 16 ms per step at the median, and its 99th-percentile solve ranged from about 27 ms to 120 ms across the runs above. Collocation sits at about 18 ms median, with 99th-percentile solves of 40–70 ms. Energy shaping takes about 9 *microseconds*.

In this simulation the solver is free: the plant waits while the solver works. On real hardware it wouldn't, and the slow solves would show up as exactly the kind of latency that already breaks iLQR above. A C/C++ implementation (e.g. acados) would be far faster than my Python, so this isn't a verdict on MPC. It is a reminder that "compute" and "latency" aren't separate axes on a real controller.

## What's next

- Train SAC and PETS on 5 seeds and plot learning curves (axis 1). The pipeline runs end to end; the full training runs haven't been done yet.
- Run the full robustness sweep (5 seeds × 4 episodes per cell, plus the mass × length heatmaps) and the latency sweep.
- Charge each MPC for its own solve time, by feeding measured compute back in as latency.
- Build latency-aware variants: state prediction for the MPCs, and SAC trained with randomized latency.
- Try domain randomization for SAC and PETS over the mass/length grid. This is the fair RL answer to the robustness axis.

## Update log

- **2026-10-25:** First post. Problem setup, the four model-based controllers, and first single-episode results under mismatch and latency.
