---
title: "Pulse Tonometer"
author: Juan I. Cerrudo
date: 2026-02-05
lang: en
page_id: pulse-tonometer
permalink: /tutorial/pulse-tonometer/
categories:
  - Tutorials
tags:
  - tonometer
  - design
  - prototyping
layout: single
toc: true
toc_sticky: true
excerpt: "Design and prototyping of a non-invasive pulse tonometer integrated with BioAmp for simultaneous ECG and radial blood-pressure recording, aimed at estimating arterial stiffness."
---

## Introduction

A few years ago, a physician came to the Prototyping Laboratory with the idea of measuring blood pressure using a pulse tonometer.
A pulse tonometer measures blood pressure and pulse-wave characteristics directly over a superficial artery, usually at the wrist. The sensor flattens the radial artery against the bone and measures the pressure needed to keep it compressed.
The idea was to add this capability to [BioAmp](https://github.com/prototipado/bioamp), so we could measure blood pressure and the heart's electrical activity at the same time. Simultaneous ECG and pressure measurements make it possible to estimate parameters such as pulse transit time (PTT), which correlates with arterial stiffness and is used as an early indicator of cardiovascular risk.

We initially tried to replicate a system that was already being used, based on an invasive pressure sensor. This is a medical device designed for continuous, accurate monitoring of blood pressure and other physiological parameters. It works by inserting a catheter directly into a blood vessel (usually an artery); the catheter transmits pressure to an external transducer, which converts the mechanical signal into an electrical one. The sensor's tubing was filled with water, and the catheter tip was modified with a flexible membrane (a latex glove finger or a balloon) and placed on the patient's wrist. Blood pressure moved the membrane and, with it, the water column, making it possible to measure arterial pressure. The sensor was connected to one of BioAmp's differential channels; its Wheatstone-bridge output produced a voltage difference proportional to the applied pressure, which could be monitored alongside the ECG signal.
This system was not very effective, at least in our implementation. Filling the tubing was difficult, and it was hard to prevent air bubbles from forming inside it. In addition, from a dynamic perspective, the hydraulic system introduced high compliance and strong damping, degrading the fidelity of the pressure waveform and reducing sensor sensitivity.

## Redesign

We decided to redesign the system from scratch, changing the transducer to make it easier to use. During the literature review we conduct for our projects, patents are one of our sources. One that we found particularly interesting was patent [US20050177047A1](https://patents.google.com/patent/US20050177047A1/en), "Device for, and a method of, transcutaneous pressure waveform sensing of an artery and a related target apparatus."

{% include figure popup=true image_path="/assets/images/tonometro/US20050177047A1-20050811-D00000.png" alt="Patent US20050177047A1" caption="Figure from patent US20050177047A1 describing a tonometry system with a gel cone." class="align-center" %}

The device illustration showed a pressure sensor and a kind of gel cone that transmitted arterial pressure from the skin surface to the sensor. We were already looking for invasive pressure sensors to assess whether they could be reused, and the illustration made us think we recognized the package of the sensor used. We are not completely certain, but it looks very much like NXP's [MPX2300DT1](https://www.nxp.com/docs/en/data-sheet/MPX2300D.pdf).

{% include figure popup=true image_path="/assets/images/tonometro/MPX2300DT1.png" alt="MPX2300DT1 sensor" caption="NXP MPX2300DT1 pressure sensor, with a package compatible with the one used in invasive monitoring systems." class="align-center" %}

This sensor is a silicon pressure transducer rated from 0 to 2.3 MPa (0 to 23 bar), with a temperature-compensated voltage output, used in invasive pressure-monitoring devices.

## CAD Modeling

We began modeling the device in CAD with the goal of reproducing the geometry described in the patent. We chose silicone for the gel cone and first designed and 3D-printed a PLA mold. The mold was held in a vise to keep it stable during casting.

{% include figure popup=true image_path="/assets/images/tonometro/corte_a.png" alt="Cross-section of the device" caption="Longitudinal section of the CAD model. The yellow part holds the sensor, and the green part secures the cable. The front part is printed in TPU, with only a couple of material layers at the contact surface." class="align-center" %}

Instead of conventional liquid silicone (RTV), we used a hot-glue gun. Although commonly called "silicone guns," the glue sticks are actually made of thermoplastic polymers (usually EVA or other copolymers). Because the solidified material has a similar consistency and is easy to process, we decided to evaluate it as a practical, low-cost alternative.

{% include figure popup=true image_path="/assets/images/tonometro/molde_A.png" alt="Mold for the gel cone" caption="PLA-printed mold secured for casting the gel cone. The vent hole at the top helps ensure the cavity fills completely. Aluminum foil was also placed at the mold entrance to protect the PLA from the hot glue." class="align-center" %}

We needed multiple iterations of both the mold and the gel cone to optimize the geometry and achieve a good casting. The rest of the device also went through several iterations as we looked for a shape that could hold the sensor securely while remaining comfortable to use.

{% include figure popup=true image_path="/assets/images/tonometro/prototipos.jpg" alt="Prototype iterations" caption="Different iterations of the molds and device structure, aimed at improving functionality and ease of assembly." class="align-center" %}

## Testing

The video shows the prototype connected to BioAmp, with the pressure signal displayed in BrainBay. This versatile software makes it possible to build a signal-processing flow diagram and obtain blood pressure directly in mmHg.

<p align="center">
  <video controls width="80%">
    <source src="/assets/images/tonometro/video_test.mp4" type="video/mp4">
  </video>
</p>

When ECG capture is added as well, we obtain recordings like this:

{% include figure popup=true image_path="/assets/images/tonometro/ECG_presion.png" alt="ECG and pressure recording" caption="Simultaneous ECG and radial blood-pressure waveform captured with the final prototype." class="align-center" %}

The redesign produced a pressure signal with a morphology consistent with a physiological radial pulse wave, avoiding the dynamic limitations of the initial hydraulic system. Integration with BioAmp demonstrated that simultaneous ECG and blood-pressure recording is feasible using a low-cost, non-invasive transducer.

Although the system still needs formal calibration against a clinical reference method, the preliminary results validate the concept of integrated radial tonometry and open the way to developing tools for estimating arterial stiffness and derived hemodynamic parameters.

---

[![Hits](https://hits.sh/prototipado.github.io/en/tutorial/pulse-tonometer/.svg)](https://hits.sh/prototipado.github.io/en/tutorial/pulse-tonometer/)