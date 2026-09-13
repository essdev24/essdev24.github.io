---
layout: post
title: "Graphene Field-Effect Transistors: How an Ammonia Rinse Fixed Our Dirac Point"
permalink: /projects/graphene-gfet-ammonia-rinse/
---

## Why graphene, and why it's harder than it sounds

Silicon transistors are running into physical limits as devices keep shrinking, which is part of why graphene has gotten so much attention as an alternative channel material — it's atomically thin, has a symmetric band structure, and in ideal, suspended conditions can theoretically hit mobilities north of 100,000 cm²/Vs. In a back-gated GFET, a voltage on the gate capacitively modulates the carrier concentration in the graphene channel, shifting the Fermi level and letting you tune conduction between electrons and holes — this ambipolar behavior is one of graphene's defining features.

The catch is that once you put graphene on a real substrate instead of suspending it in a vacuum, mobility drops hard — typically into the 1,000 to 24,000 cm²/Vs range — because of substrate-induced scattering and impurities. And because graphene has no intrinsic bandgap, these devices are hard to switch fully off, which limits them for digital logic but makes them genuinely promising for analog and RF applications.


![image 1](/assets/images/projects/graphene_gfet/scheme_gfet.png)
*[Image: Figure 1 — schematic cross-section of the GFET, showing the doped silicon back-gate, SiO₂ dielectric, Cr/Au source/drain, and graphene channel]*

## Building the device

**Source and drain first.** We started with a silicon substrate that already had a back gate applied, cleaned it (ultrasonic acetone bath, isopropanol rinse, nitrogen dry), then patterned the source/drain contacts using a lift-off process with negative photoresist (AZ nLOF 2070). After a 14 s UV exposure and 70 s development, we sputter-deposited the metal contacts — a chromium adhesion layer topped with gold. Dissolving the sample in acetone lifted off the excess metal, leaving clean, precisely defined source and drain electrodes with a microscopic gap between them — which is what eventually becomes the graphene channel.

**Transferring the graphene.** This is where things get delicate. We used an electrochemical bubbling transfer: a PMMA support layer was spin-coated onto copper foil that already had graphene grown on it via CVD. The copper foil then acted as the cathode in an electrolytic cell with a dilute NaOH solution, and applying 2.5 V generated hydrogen bubbles right at the copper-graphene interface — mechanically peeling the PMMA/graphene stack off the foil. That floating stack got rinsed in DI water, and for one of our two samples, we added an extra step: soaking it in a 1 molar ammonia solution for 30 minutes. The goal was to counteract the p-doping that wet transfer processes are notorious for introducing. After baking and removing the PMMA in acetone, the graphene was ready on the substrate.

**Defining the channel.** A second lithography step (positive resist, AZ 5214E) patterned the channel geometry, and reactive ion etching with an oxygen plasma removed the unprotected graphene, leaving a clean channel between source and drain.

![image 2](/assets/images/projects/graphene_gfet/S2_after_rie_2.png)
*[Image: Figure 2 — optical microscope image of the finished device, with the 100 µm scale bar visible]*

**Device parameters**, for reference:

| Parameter | Value |
| --- | --- |
| Oxide thickness | 85 nm |
| Channel length (Sample A / B) | 20 µm / 30 µm |
| Channel width | 50 µm |
| Contact metal | Cr adhesion layer / Au top layer |

## Measuring it

We characterized the devices at room temperature using a probe station connected to a semiconductor parameter analyzer, sweeping the back-gate voltage from −30 V to 70 V while holding the source-drain voltage at a constant 100 mV. Current compliances were set conservatively (10 mA drain, 10 µA gate leakage) to avoid frying the channel.


![image 3](/assets/images/projects/graphene_gfet/sample_b_curve-1.png)
*[Image: Figure 3 — I-V transfer characteristics comparing Sample A and Sample B]*

The resulting curve is where graphene's ambipolar nature actually shows up visually: current is high on both ends of the sweep and dips to a minimum at the Dirac point, with hole conduction on the left side of the curve and electron conduction on the right.

To pull mobility out of this curve, we needed the transconductance — the slope of the linear region on the hole-conduction side. We extracted this over a 20 V window rather than a narrow slice, specifically to avoid the curve's noise skewing the result:

$$
g_m = \frac{dI_{DS}}{dV_{GS}}
$$

For Sample A, taken between 10 V and 30 V:

$$
g_{mA} = \frac{|60,\mu\text{A} - 160,\mu\text{A}|}{30,\text{V} - 10,\text{V}} = 5.0 \times 10^{-6}\ \text{A/V}
$$

For Sample B, taken between 0 V and 20 V:

$$
g_{mB} = \frac{|90,\mu\text{A} - 260,\mu\text{A}|}{20,\text{V} - 0,\text{V}} = 8.5 \times 10^{-6}\ \text{A/V}
$$

From there, field-effect mobility comes from combining the transconductance with the channel geometry and gate oxide capacitance:

$$
\mu = \frac{L}{W \cdot C_{ox} \cdot V_{DS}} \cdot g_m
$$

## What we found

With an 85 nm oxide thickness and a channel width of 50 µm for both samples, Sample A (20 µm channel length, no ammonia treatment) came out to **492 cm²/Vs**. Sample B (30 µm channel length, with the NH₃ soak) came out to **1256 cm²/Vs** — more than double.

The Dirac point told the same story. Sample B's minimum conduction point sat at 26 V, much closer to the ideal 0 V than Sample A's, which was shifted well into positive territory. That shift toward 0 V is a direct signature of reduced p-doping — the ammonia treatment appears to have meaningfully counteracted the contamination introduced during the wet transfer process.

Compared against the literature, these numbers sit toward the lower end: published work on well-controlled graphene transfer has reported mobilities up to 9,000 and even 24,000 cm²/Vs. Our devices landed at roughly 2% and 5% of that top figure, respectively. The gap is consistent with what you'd expect from wet-transferred CVD graphene — during optical inspection, we spotted microscopic cracks in the channel, almost certainly introduced by the surface tension of water during drying and mechanical stress during the bubbling delamination step. Electrically, cracks like these act as scattering centers, which drags mobility down and pushes channel resistance up; in the worst cases, they can break the circuit entirely.

## Takeaways

The headline result here isn't that we matched state-of-the-art mobility — we didn't, and the literature makes clear why that's a hard bar to clear with a wet transfer process. The more interesting result is that a comparatively simple, low-cost step — a 30-minute soak in dilute ammonia — produced a measurable, more-than-2x improvement in mobility and pulled the Dirac point substantially closer to ideal. For anyone working with CVD graphene and a wet transfer process, that's a strong signal that post-transfer chemical cleaning deserves to be a standard step, not an optional one.

If I were pushing this further, the next place I'd look is the transfer process itself — a dry transfer method would sidestep the water-related cracking altogether, which based on our results looks like the dominant factor limiting mobility more than the doping itself.
