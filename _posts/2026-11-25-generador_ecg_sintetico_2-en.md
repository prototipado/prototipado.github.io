---
title: "From ECG to ECG-PPG: An Event-Driven Coupled Model"
author: Juan I. Cerrudo
date: 2026-11-25
lang: en
page_id: ecg-ppg-event-model
permalink: /developments/ecg-ppg-event-model/
categories:
  - Developments
tags:
  - ECG
  - PPG
  - Raspberry Pi Pico
  - Python
layout: single
toc: true
toc_sticky: true
excerpt: "An event-driven model generates synchronized ECG and PPG signals: atrial activity, AV conduction, and each ventricular beat also determine the peripheral pulse."
---

## A New Implementation of Synthetic Generation

This series started with a generator that replayed stored ECG samples. In the [first post](/developments/ecg-signal-generator/) we covered that initial development and its hardware iterations.

Later, in [From Record to Model: Real-Time Synthetic ECG Generation](/developments/synthetic-ecg-generator-1/), we took the step from simple replay to synthetic generation: we implemented a mathematical model of cardiac conduction based on coupled nonlinear oscillators on the Raspberry Pi Pico. That model allowed us to modify heart rate and explore different morphologies without relying on pre-recorded signals.

This post presents a new implementation of that synthetic generation, based on a different approach. As a primary reference, we took the paper [“An ECG–PPG Physiological Signal Emulator for Calibration and Validation of Cardiovascular Monitoring Devices”](https://doi.org/10.3390/s26175578), published in *Sensors* in 2026. In addition to proposing an alternative approach to generating the ECG, the article introduces something absent in the previous implementation: a photoplethysmography (PPG) waveform temporally coupled with each ventricular beat.

## An Event-Driven Model

The model divides the cardiac cycle into functional blocks: atrial pacemaker → atrioventricular (AV) conduction → ventricular activity → peripheral pulse generation. Instead of synthesizing every waveform around a rigid sequence of R-peaks, it maintains separate atrial and ventricular event sequences.

Each atrial event can either be conducted or blocked. If conducted, the corresponding ventricular event appears after the PR interval. This separation makes it straightforward to represent conditions such as first-degree AV block (prolonged PR), Mobitz I (progressive PR prolongation culminating in a blocked impulse), or complete AV block, where ventricular activity follows its own independent rate.

ECG waves are synthesized using Gaussian functions: the P wave is linked to atrial events, while Q, R, S, and T waves are tied to ventricular events. A three-dimensional cardiac vector is then projected onto the lead axes to yield leads with anatomically coherent spatial relationships. The model also supports distinct beat classes: a premature ventricular contraction (PVC), for instance, can produce a wider, delayed QRS complex by applying a different set of parameters.

## Synchronized PPG

The most compelling addition is how the PPG signal is generated. Each peripheral pulse originates from a ventricular event and emerges after an electromechanical delay:

**pulse foot time = ventricular event + PAT**  
**PAT = PEP + vascular transit distance / pulse wave velocity**

In these expressions, PEP represents the pre-ejection period and PAT the pulse arrival time from electrical activation to peripheral arrival. The contour of each pulse is approximated using a combination of five Gaussian components. This allows independent tuning of its amplitude and morphology while preserving strict temporal alignment with the initiating ventricular beat.

As a consequence, both signals respond in tandem to model dynamics. If an atrial impulse is blocked, no conducted ventricular beat occurs, and no peripheral pulse is generated. If a ventricular beat is ectopic, the model can simultaneously produce a widened QRS and a delayed, attenuated pulse. There is no need to engineer a separate pulse sequence or manually align it post hoc.

It is also important to clarify what "PPG output" entails in this context: the paper models the pulsatile component as an analog electrical voltage intended to connect to a measurement input. It does not emit light, replace an optical sensor, or model absolute optical intensity or DC baseline levels.

## What Changes Compared to the Coupled-Oscillator Model

In the previous post, the heart was represented using coupled oscillators: the model captured conduction dynamics and generated the ECG from components tied to its waves. The 2026 paper organizes the process differently: it tracks discrete atrial and ventricular events, models AV conduction between them, and derives both the ECG and PPG from those events.

| Aspect | Previous Model (Oscillators) | New Implementation (Paper) |
|---|---|---|
| Model Architecture | Coupled oscillators modeling conduction dynamics | Separate atrial events, AV conduction, and ventricular events |
| ECG Generation | Wave-associated components within the oscillator system | Gaussian functions mapped to atrial and ventricular events |
| Signal Outputs | ECG only | Synchronized ECG and PPG |
| Beat-to-Pulse Coupling | No coupled PPG output | Each ventricular event triggers a peripheral pulse with electromechanical delay |

This is not simply a matter of one implementation replacing a flawed predecessor. The coupled-oscillator model already generated synthetic ECG in real time and permitted exploring frequency and morphology variations. The new approach alters how rhythm is modeled and, crucially, extends the system to incorporate peripheral physiological responses: PPG is not an arbitrary signal appended after the fact, but an emergent consequence of ventricular events and their properties.

The 2026 paper also addresses replaying clinical 12-lead ECGs by transforming them into electrode potentials to feed the same output stage. While that reproduction feature complements the synthetic framework, our primary focus here is the coupled generation of ECG and PPG.

The model also allows modulating event intervals with a sinusoidal respiratory component. However, in the paper's experimental validation, this respiratory modulation was disabled to isolate timing accuracy and analog stage performance. This is a nuance worth keeping in mind: the fact that the model supports respiratory sinus arrhythmia does not imply all benchmarked signals included it.

## In-Browser Interactive Simulator

As in the previous post, we provide an interactive implementation of the model running directly in the browser, built in JavaScript. It lets you tweak atrial pacemaker parameters, test various rhythms and AV conduction blocks, and observe in real time how each ventricular event produces both the QRS complex and the peripheral pulse wave (PPG) with its corresponding PAT delay:

<a href="{{ '/assets/sims/ecg_ppg_model_sim.html' | relative_url }}"
   target="_blank" rel="noopener"
   style="display:block; max-width:720px; margin:1.5em auto;">
  <img src="{{ '/assets/images/generador_ecg_sintetico_2/sim.png' | relative_url }}"
       alt="Event-driven ECG–PPG simulator — open in new tab"
       style="width:100%; border-radius:10px; display:block;">
</a>

## From Model to Hardware

The team behind the paper implemented their emulator using an STM32F407 microcontroller alongside dedicated DACs to deliver ten synchronized analog outputs: nine ECG electrode potentials and one PPG channel, running at 500 samples per second. This architecture guarantees phase alignment across all channels, including the latency between electrical activation and the arrival of the peripheral pulse.

In our earlier implementation, the Raspberry Pi Pico was configured around eight ECG outputs: I, II, and V1 through V6, with the remaining limb leads derived mathematically from the active signals. Incorporating PPG is therefore not merely a matter of plotting an extra trace on an interface: it requires integrating and calibrating an additional analog output channel, scaling its voltage range, and ensuring its update cycle remains tightly synchronized with the ECG output DACs.

We conducted tests repurposing one of the existing analog channels to output the PPG signal. Moving forward, the ideal path involves expanding the generator's analog channel count to accommodate additional physiological waveforms in the future, such as respiration, arterial blood pressure, or temperature.

## Bonus: Finger Phantom

During our literature review, we encountered the study [An Optical Signal Simulator for the Characterization of Photoplethysmographic Devices](https://www.mdpi.com/1424-8220/24/3/1008), which showed that PPG waveforms could be optical simulated by modulating LED emission directly into commercial sensors. Taking that concept as inspiration, we set out to see if we could replicate the underlying principle with a more streamlined implementation, initially omitting the sensor optical synchronization stage and the dedicated controlled current sources employed in the paper.

To achieve this, we had to account for the fact that the relationship between LED forward voltage, drive current, and emitted light intensity is nonlinear, heavily influenced by diode threshold voltage, series resistance, and output stage compliance. Rather than building a dedicated analog current-driver stage, we opted to pre-compensate the driving waveform directly in software, tailoring the input to counteract these optical and electrical nonlinearities. Through an iterative tuning process using an arbitrary waveform generator while observing the resulting pulse contour on the oximeter, we shaped the driving signal until it closely approximated a physiological PPG pulse.

To evaluate this concept under realistic conditions, the next step was fabricating a physical phantom that could clip straight into a commercial finger-clip pulse oximeter and testing whether the device could successfully lock onto and track the synthetic optical signal.

### Mold and Silicone Casting

The casting mold was 3D printed in ABS based on a STEP model of a human finger. Inside, we positioned three LEDs: green, red, and infrared. Once the LEDs and wiring were set, silicone was cast into the cavity to provide mechanical compliance and optical scattering properties resembling human tissue, ensuring the emitted light diffused realistically. The result was a solid finger phantom ready to be inserted into the pulse oximeter. For our initial trials, only the red LED was driven, leaving the green and IR emitters unconnected.

{% include figure popup=true image_path="/assets/images/generador_ecg_sintetico_2/molde.png" alt="Finger phantom fabrication" caption="Phantom fabrication process. From left to right, top to bottom: the three LEDs (green, red, and IR) before assembly; the 3D-printed ABS mold; LEDs positioned inside the mold; the completed silicone finger phantom." class="align-center" %}

### The Challenge of Optical Synchronization

A commercial pulse oximeter is far from a passive photodetector: it time-multiplexes red and infrared emission and samples ambient background light in interleaved phases to cancel ambient interference. Emitting light continuously and asynchronously from our phantom means portions of our signal risk falling into background baseline correction intervals and being rejected. Nonetheless, preliminary tests revealed that the finger clip did respond reliably to external modulation, paving the way for testing tailored waveforms.

{% include figure popup=true image_path="/assets/images/generador_ecg_sintetico_2/setup.jpg" alt="Experimental setup with the phantom and clip oximeter" caption="Full experimental bench: the silicone finger phantom inserted into the commercial fingertip oximeter, driven by the ECG–PPG generator." class="align-center" %}

### From Raw Signal to Empirical Pre-Compensation

When injecting the raw PPG waveform directly from the generator into the LED, the trace rendered on the oximeter's screen was unrecognizable. While the pulse rate was detected accurately, the morphology suffered severe distortion.

{% include figure popup=true image_path="/assets/images/generador_ecg_sintetico_2/dedo_ppg_generador.png" alt="Four panels: real finger, phantom with PPG, function generator, and settings" caption="Top left: oximeter capture with a real human finger (baseline reference). Top right: signal obtained with the phantom driven by the generator's uncompensated PPG. Bottom left: waveform achieved using an arbitrary function generator (asymmetric ramp). Bottom right: Siglent SDG1050 function generator setup used during the experiment." class="align-center" %}

To diagnose how the instrument's front-end processed incoming light, we turned to a Siglent SDG1050 arbitrary function generator and evaluated several test patterns. Two findings proved critical:

1. **Polarity Inversion:** In pulse oximetry, increased arterial blood volume during systole corresponds to greater light absorption and less light reaching the photodiode. Therefore, the optical emitter must invert signal polarity relative to the conventional PPG display curve so that the trace deflects upward on screen during systole.
2. **Differentiating Front-End Behavior:** The oximeter's analog and digital conditioning acts effectively as a high-pass filter, stripping DC offsets and emphasizing rapid transitions. For example, applying an asymmetric ramp (slow rise, sharp fall) produced a pulse with a pronounced leading edge followed by a plateau.

### Firmware Compensation

Guided by these observations, we adapted the generator's firmware to digitally pre-compensate the PPG signal prior to feeding it to the LED PWM stage:

- Inverted the polarity of the synthetic curve.
- Applied a digital pre-emphasis filter (a leaky integrator to counteract the differentiation introduced by the oximeter's AC-coupling).
- Mapped the compensated range into the active conduction window of the red LED, taking into account its forward turn-on voltage and operational amplifier headroom through its 120 Ω series resistor.

{% include figure popup=true image_path="/assets/images/generador_ecg_sintetico_2/ppg_preemph_plot.png" alt="Original PPG waveform vs pre-emphasized signal" caption="Original PPG waveform (Gaussian, inverted) compared against the pre-emphasized curve sent to the PWM driving the red LED." class="align-center" %}

The modification yielded an immediate improvement: the oximeter locked stably onto the simulated heart rate and displayed a plethysmographic contour much closer to a genuine pulse.

<figure class="align-center">
  <video controls width="100%">
    <source src="/assets/images/generador_ecg_sintetico_2/comparacion.mp4" type="video/mp4">
    Your browser does not support playing this video.
  </video>
  <figcaption>Side-by-side comparison: real finger (top) vs. phantom running pre-emphasis firmware (bottom). Pulse rate tracking is stable; morphology is noticeably improved though not yet identical.</figcaption>
</figure>

### Current Status and Limitations

The finger phantom serves as an effective proof of concept for validating the coupled generator non-invasively, though several limitations remain:

- **Asynchronous Operation:** Lacking phase-lock synchronization with the oximeter's internal optical multiplexing clock, portions of the emitted light overlap with background dark phases.
- **Single-Channel Optical Driving:** By driving only the red LED, the ratio-of-ratios ($R = \frac{AC_{red}/DC_{red}}{AC_{IR}/DC_{IR}}$) is uncontrolled; consequently, the SpO₂ percentage shown on the display lacks physical validity.
- **Uncalibrated Transfer Function:** The pre-compensation curve was derived empirically; a rigorous linearization would require formal system identification of the optical sensor and receiver circuitry.

Even with these constraints, verifying that a synthetic signal coupled to the ECG can be optically injected and interpreted by standard medical monitoring hardware confirmed the viability of the approach and established a clear foundation for subsequent revisions.

The firmware source code, hardware design files, and simulator assets are available in the [Laboratory repository](https://github.com/prototipado/ECG_Phantom).

---

[![Hits](https://hits.sh/prototipado.github.io/developments/ecg-ppg-event-model/.svg)](https://hits.sh/prototipado.github.io/developments/ecg-ppg-event-model/)
