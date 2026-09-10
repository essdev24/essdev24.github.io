---
layout: post
title: "No Arduino, No Code: Building an Analog Ultrasonic Distance Sensor from Scratch"
permalink: /projects/non-invasive-analog-distance-sensor/
---

When most people think about building a distance sensor, they reach for an Arduino, plug in an ultrasonic module, and let a microcontroller handle all the timing. I wanted to see if I could skip that step entirely — no code, no MCU, just discrete analog ICs doing the work of generating, sending, receiving, and timing an ultrasonic pulse. This post walks through how I built it, what worked, what fought back, and what I’d change next time.

## The idea

Ultrasonic ranging is a well-established technique: send a high-frequency sound pulse, measure how long it takes to bounce back off a surface, and multiply that time by the speed of sound to get distance. It’s the same principle behind sonar and most of the cheap ultrasonic modules you’ll find online — the difference here is that every one of those steps (signal generation, mixing, amplification, filtering, edge-detection, and counting) had to be built out of raw ICs and passive components instead of a few lines of firmware.

The core components I settled on were the XR8038 (function generator), NE555 (timing/oscillator), CD4053B (multiplexer), TL072CP (op-amp), and a handful of CMOS logic ICs — CD4013BE, CD4069CN, and CD4018BE — to handle the digital side of edge detection and gating.

*Figure 1 — functional block diagram of the whole system, transmitter through display.*

## How the system is laid out

At a high level, the signal path looks like this:

1. A function generator produces a 40 kHz carrier — the actual ultrasonic tone that gets transmitted.
2. A slower oscillator produces a low-frequency tone that gets mixed in, which the receiver later uses as a reference for timing.
3. The mixed signal drives the ultrasonic transmitter.
4. The receiver picks up the reflected signal, amplifies it, rectifies it, filters it, and cleans it up with a Schmitt trigger.
5. A flip-flop turns that cleaned signal into a sharp pulse.
6. A clock/counter circuit uses that pulse to time the round trip and display the result as a distance.

None of this happens in software — it’s all timing relationships built into the resistor and capacitor values feeding each IC.

## Generating the signal

The XR8038 handled the 40 kHz square wave that actually gets sent out by the transmitter. I went with a square wave rather than a sine specifically because of how the signal attenuates over distance — square waves held up better for this setup. Getting a clean, symmetrical wave out of the XR8038 comes down to hitting a 50% duty cycle, which is set with a single timing resistor using:

```text
f = 0.15 / (R · C)
```

The NE555 supplied a separate, much slower oscillation (10 Hz) that gets combined with the 40 kHz carrier. Getting the NE555 into the right duty cycle meant working through the standard 555 astable timing relationship:

```text
f = 1 / T
T ≈ (R_A + 2·R_B) · C
```

Both of these signals feed into a CD4053B multiplexer, which mixes the fast carrier with the slow tone. That combined signal is what actually drives the transmitter — and it’s also what makes the receiver’s job possible, since the receiver is really looking for that slower envelope riding on top of the carrier.

*Figure 2 — full schematic, ideally a slightly zoomed-in crop of the transmitter block.*

## Picking the signal back out of the noise

This is where most of the real design work happened. A reflected ultrasonic signal is weak and noisy by the time it gets back to the receiver, so the receive chain has to do a lot of cleanup before you can extract a usable timing edge from it.

The chain looks like this:

- TL072CP — first-stage amplifier, boosting the incoming signal with roughly 10x gain.
- Rectifier — a fast switching diode in parallel with a resistor, which strips out the negative half of the waveform.
- Low-pass filter — tuned to pass only the slower modulating envelope (the 10 Hz tone), filtering out the 40 kHz carrier once it’s done its job.
- Schmitt trigger — built from a comparator with a pull-up resistor to add hysteresis, which is what keeps noise from causing false triggering. Without this stage, the spikes in the raw signal would get misread as valid edges.

*Figure 3 — assembled boards with wiring color code (red = VCC, blue = VEE, black = ground, yellow = signal).* 

Once the signal is clean, it gets fed into a CD4013BE flip-flop, which converts the leading edge into a short, sharp pulse — essentially a synthetic “start” marker for the timing circuit. That narrow pulse only works because of how sharp it is; a slow edge here would make the whole timing measurement fuzzy. A second, independently generated 32 kHz signal from another NE555 stage gets combined with this pulse through CD4018BE (acting as an AND gate) and CD4069CN (acting as an inverter) to produce the final signal that drives the counter.

*Figure 4(d) — oscilloscope capture of the narrow pulse.*

## Testing it

I used LTSpice to simulate the circuit before ever touching a breadboard, and documented the layout in KiCad. Once it was physically built, testing came down to an oscilloscope with two probes on CH1 and CH2, tracing the signal at each stage:

- The raw multiplexed signal was noticeably noisy — visible spikes that, if left alone, would throw off the timing measurement.
- After the amplifier, rectifier, and low-pass filter, the waveform cleaned up into a proper square pulse with the modulation riding on top.
- The Schmitt trigger stage produced a clean, shifted square pulse — the “leading signal” the rest of the system relies on.
- The final narrow pulse output showed the sharp charge/discharge behavior expected from the hysteresis stage.

*Figure 4(a–c) — oscilloscope traces at each cleanup stage, shown side by side.*

With everything wired up, the display panel read out 76 cm for the test distance — confirming the whole chain, from carrier generation to final count, actually worked end to end.

*Figure 5 — photo of the display panel showing the 76 cm reading.*

## What I’d fix next

Pure analog signal chains are unforgiving when it comes to attenuation. The TL072’s gain helped, but noise-driven spikes were still a persistent problem — the circuit would occasionally read a higher amplitude than the true signal, which I partially solved by improving the grounding scheme (multiple ground points rather than a single shared one). Even with that fix, I think there’s more headroom here: an additional amplification stage after the initial receive gain would likely stretch the usable range further and make the whole measurement more robust to attenuation over longer distances.

## Takeaways

This project was as much about learning to read manufacturer datasheets and translate them into working timing behavior as it was about building a rangefinder. Every stage — the XR8038’s duty cycle equation, the NE555’s charge/discharge timing, the Schmitt trigger’s hysteresis band — came directly out of interpreting datasheet specs and turning them into real resistor and capacitor values. If you’re looking for a way to really understand why a microcontroller-based ultrasonic sensor works the way it does, building the analog version from scratch is a surprisingly good way to find out.
