---
title: "Redesigning a DStat Potentiostat: Multi-Material Enclosure, Native GUI, and Characterization"
author: Juan I. Cerrudo
date: 2026-11-13
lang: en
page_id: potentiostat
permalink: /tutorial/potentiostat/
categories:
  - Tutorial
tags:
  - potentiostat
  - design
  - prototyping
layout: single
toc: true
toc_sticky: true
excerpt: "Redesign and commissioning of a DStat potentiostat: multi-material 3D-printed enclosure with interlocking, a new native Python 3 / PyQt6 GUI, and experimental characterization of noise, shielding and calibration."
---

## Introduction

Part of our work at the Prototyping Laboratory is providing solutions to other labs within the institution, mainly from the Engineering Faculty, although we occasionally work with the Agriculture Faculty as well, given its proximity.
This time they approached us with the need for a potentiostat to perform inorganic determinations in water.



## Implementation

This is not the first time someone has come to the Lab with this request. A few years ago we built a potentiostat for the Biosensors Course at FIUNER.

{% include figure popup=true image_path="/assets/images/potensiostato/viejo.jpg" alt="Potentiostat PCB" caption="First potentiostat, entirely manufactured in the Lab." class="align-center" %}


The design was not our own; instead, we chose to replicate an open-source project, the [DStat](https://microfluidics.utoronto.ca/gitlab/dstat). At that time we fabricated the PCB in the Lab, performed the assembly, and designed a 3D-printed enclosure for the device.

{% include figure popup=true image_path="/assets/images/potensiostato/pcb.jpg" alt="PCB" caption="PCB" class="align-center" %}

This time we had the board fabricated by [JLCPCB](https://jlcpcb.com/), performed the assembly in the Lab, and slightly modified the 3D enclosure model.

{% include figure popup=true image_path="/assets/images/potensiostato/partes_pcb.jpg" alt="PCB parts" caption="Enclosure parts and PCB" class="align-center" %}


## CAD Modelling

We had recently added a [Prusa XL](https://www.prusa3d.com/product/original-prusa-xl-single-toolhead/) to the Lab — a large-format printer that, more importantly, supports multi-material printing thanks to its independent extruders.

{% include figure popup=true image_path="/assets/images/potensiostato/render_ensamblaje.png" alt="Render" caption="Assembly render" class="align-center" %}

To take advantage of it we added a full TPU border on the front and rear panels to absorb impacts and vibrations while also providing a nicer finish. The rest of the enclosure was printed in PLA, and since these materials do not adhere well to each other, we used the "interlocking" method available in PrusaSlicer.

{% include figure popup=true image_path="/assets/images/potensiostato/interlocking.png" alt="Interlocking" caption="Interlocking" class="align-center" %}


## From Docker and Python 2 to a Native .exe: Redesigning DStat GUI 2.1

The original DStat software was written with very outdated tools (Python 2.7 and GTK2) that are no longer maintained, so today it is nearly impossible to run. Using it required a long chain of technical steps: installing Docker (a program that creates an isolated "mini system" to run legacy software), enabling a component that forwards Linux graphical windows, and — on Windows — manually connecting the USB device to a Linux virtual machine. And then, with some luck, it worked.

In short: measuring a single sample meant setting up a complicated environment and hoping the USB connection would not fail.

On top of that, the old version was quite limited: it could not reopen saved measurements or analyse the data. It only saved results as text or data files and took screenshots of the plots.

{% include figure popup=true image_path="/assets/images/potensiostato/gui.png" alt="DStat GUI" caption="DStat GUI 2.1" class="align-center" %}

DStat GUI 2.1 leaves all of that behind. The stack was migrated to Python 3.10+ and PyQt6, which made it possible to compile a native Windows executable (onedir, no Python installation required) that opens with a double-click. Serial communication goes directly through the COM port, without containers or X11. The interface was completely redesigned: light/dark theme with a persistent selector, a consistent colour palette (teal accent), card-style panels with rounded borders, semantic buttons, a styled matplotlib chart with accent line and subtle grid, modern toolbar icons, a monospace protocol panel colour-coded by direction, and better high-DPI support. The experiment logic, serial protocol and file format remain untouched, so old data is fully compatible. The result is a tool that anyone can download, plug into the DStat, and start using in seconds — no Docker, no command line.

## Tests

To validate the potentiostat we built a dummy cell with 1 MΩ (0.1 % tolerance) and 4 MΩ (1 % tolerance) resistors and with an open circuit, applying 100 mV with a 3 MΩ transimpedance gain. Each test was a 60 s chronoamperometry sampled at 50 and 60 Hz. In the first round, with the board uncleaned and unshielded, the measured current showed an excess of ~17.5 nA, the same for both resistors: +17.5 % with 1 MΩ and nearly +70 % with 4 MΩ. The fact that the absolute excess was constant pointed to a leakage resistance of ~5.7 MΩ in parallel with the test resistor, and with an open circuit the offset was only ~17 pA.

After cleaning the PCB in an ultrasonic bath and mounting it inside a grounded shielded enclosure, the excess disappeared. The error dropped to −0.7 to −0.9 %, a scale error common to both resistors that is consistent with component tolerances.


| Test | Expected | Before: measured (error) | Before: noise rms | After: measured (error) | After: noise rms |
|---|---|---|---|---|---|
| R = 1 MΩ, 50 Hz | 100 nA | 117.5 nA (+17.5 %) | 245 pA | 99.16 nA (−0.84 %) | 44 pA |
| R = 1 MΩ, 60 Hz | 100 nA | 117.6 nA (+17.6 %) | 3.05 nA | 99.21 nA (−0.79 %) | 1.29 nA |
| R = 4 MΩ, 50 Hz | 25 nA | 42.7 nA (+71 %) | 260 pA | 24.83 nA (−0.66 %) | 50 pA |
| R = 4 MΩ, 60 Hz | 25 nA | 42.3 nA (+69 %) | 2.38 nA | 24.76 nA (−0.94 %) | 1.87 nA |
| Open, 50 Hz | 0 | 0.017 nA | 54 pA | −0.13 nA | 68 pA |
| Open, 60 Hz | 0 | 0.016 nA | 1.95 nA | −0.10 nA | 1.65 nA |


The noise at 50 Hz dropped from ~250 pA to ~45 pA rms, and with an open circuit it is virtually the same, so we are already at the acquisition electronics' noise floor. At 60 Hz a ~9.75 Hz tone of 1.3–1.9 nA rms persists: it is mains interference (50 Hz) aliased by sampling at 60 Hz, which shifts it to 60 − 50 = 10 Hz. The fact that it disappears when the sampling rate is changed to 50 Hz — where mains interference falls exactly on DC and is rejected — confirms this.


{% include figure popup=true image_path="/assets/images/potensiostato/fft_shield_vs_no_shield.png" alt="FFT Shield vs No Shield" caption="FFT Shield vs No Shield" class="align-center" %}

A residual interference component persists, which we currently attribute to mains coupling. We suspect the shield does not have good electrical continuity: the aluminium tape may not make proper contact between its sections (the adhesive can be insulating) or surface oxidation may prevent it. The previous version had been coated with a copper layer, achieving a much more noticeable reduction in line noise.


## Effect of Air Calibration

We repeated the tests with the device's calibration procedure, which measures the current with the electrodes in air and subtracts it. Compared to the same measurements without calibration (clean PCB with shield), accuracy improves: the error on the test resistors went from 0.66–0.94 % down to 0.18–0.43 %, and the open-circuit offset dropped from about −100 to −134 pA to ~+11 pA. A scale error of ~−0.4 % remains (equivalent to ~99.6 mV applied), which air calibration cannot correct and which is consistent with resistor tolerances. The noise does not change: subtracting a constant does not alter the standard deviation, and the differences seen between runs (for example at 60 Hz, from 1.3 to 2.0 nA with 1 MΩ) correspond to variation in the ~9.8 Hz interference tone, not to the calibration.

| Test | Expected | Uncalibrated: measured (error) | Uncalibrated: noise rms | Calibrated: measured (error) | Calibrated: noise rms |
|---|---|---|---|---|---|
| R = 1 MΩ, 50 Hz | 100 nA | 99.16 nA (−0.84 %) | 44 pA | 99.82 nA (−0.18 %) | 35 pA |
| R = 1 MΩ, 60 Hz | 100 nA | 99.21 nA (−0.79 %) | 1.29 nA | 99.82 nA (−0.18 %) | 2.00 nA |
| R = 4 MΩ, 50 Hz | 25 nA | 24.83 nA (−0.66 %) | 50 pA | 25.11 nA (+0.43 %) | 45 pA |
| R = 4 MΩ, 60 Hz | 25 nA | 24.76 nA (−0.94 %) | 1.87 nA | 25.10 nA (+0.42 %) | 2.36 nA |
| Open, 50 Hz | 0 | −0.13 nA | 68 pA | +0.011 nA | 55 pA |
| Open, 60 Hz | 0 | −0.10 nA | 1.65 nA | +0.012 nA | 2.04 nA |


## Repository

All the software and mechanical design work for this project is available as open source at [DStat-RD](https://github.com/prototipado/DStat-RD) under the GPL-3.0 license. The repository includes:

- **GUI (Python 3 / PyQt6):** full source code for DStat GUI v2.1, with instructions to run from source or build a native Windows executable with PyInstaller.
- **Mechanical design:** STEP assembly and individual part models (base, lid, BNC panel, USB panel) and print-ready STL meshes.
- **Documentation:** detailed user manual, step-by-step assembly guide, calibration procedure, electrode wiring, troubleshooting, and the raw characterisation data with analysis scripts.

---

[![Hits](https://hits.sh/prototipado.github.io/tutorial/potentiostat/.svg)](https://hits.sh/prototipado.github.io/tutorial/potentiostat/)

