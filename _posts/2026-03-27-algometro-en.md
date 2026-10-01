---
title: "Portable Algometer"
author: Juan I. Cerrudo
date: 2026-02-05
lang: en
page_id: portable-algometer
permalink: /tutorial/portable-algometer/
categories:
  - Tutorials
tags:
  - algometer
  - design
  - prototyping
layout: single
toc: true
toc_sticky: true
excerpt: "Development of a handheld portable algometer for measuring pain threshold, featuring a Honeywell force sensor, OLED display, rotary encoder, and 3D-printed structure."
---

## Introduction

Part of the Prototyping Laboratory's work involves providing technological solutions to other laboratories at the UNER Faculty of Engineering. As part of these activities, we began working some time ago with the [Center for Engineering in Rehabilitation and Neuromuscular and Sensory Research (CIRINS)](https://ingenieria.uner.edu.ar/nucleos/cirins/), supporting one of its research areas: the study of pain.
In the first stage, they asked us to develop an algometer, a device used to measure a patient's pain threshold. The complete system involved a robotic setup with a linear actuator that applied pressure to the patient's skin, a force sensor to measure the applied pressure, and a control system for the actuator.

{% include figure popup=true image_path="/assets/images/sensor_dolor/robot.png" alt="Robotic algometer" caption="Left: algometer and stimulator structure (from 'Evaluation of a protocol for a pain model using punctate sensation stimuli with a robotic system: a pilot study'). Right: CAD model of the stimulator; the final version has side supports for added rigidity. CAD model of the control system." class="align-center" %}

In a second stage, they asked us to develop a handheld algometer that could manually measure a patient's pain threshold, significantly simplifying the system.

## Redesign

The original control system had three power supplies: one for the linear actuator, one for a stepper motor, and one for the control system. For the manual version, we wanted to simplify this arrangement; by reducing the number of components, we were able to power the entire device from a single supply. The original version used an ESP32 DevKit, but it was out of stock when we placed the order, so we chose a [FireBeetle 2 ESP32-E](https://wiki.dfrobot.com/dfr0654/#tech_specs). This board includes a LiPo battery charger, which will make it easier to add a battery in the future and make the device fully portable.

The user interface consists of a 128×64-pixel OLED display and a rotary encoder with a push button. We also made the FireBeetle board's buttons available for quick use. The FireBeetle 2 ESP32-E includes LEDs that we used to indicate the device's status.
The force sensor is the same one used in the original system: Honeywell's [FMAMSDXX005WC2C3](https://automation.honeywell.com/content/dam/honeywell-edam/sps/common/en-us/industries/healthcare-and-life-sciences/medical-equipment/documents/sps-medical-fma-series.pdf).
We also added a BNC connector, which can be used as an input with a push button (to indicate external events) or as an output to synchronize with other devices.

## CAD Modeling

We based the physical design on the original system, but simplified the structure to make it significantly more compact and lightweight. The overall structure is a stack of 3D-printed parts held together by three M3×35 mm screws. We chose this approach because it allowed us to iterate quickly on individual sections of the design and made it easier to orient the parts for printing.

We placed the sensor in a kind of cradle built into the plastic part that holds it, and soldered the wires directly to its pads, avoiding the need to add a separate PCB.

{% include figure popup=true image_path="/assets/images/sensor_dolor/sensor.png" alt="Pressure sensor" caption="Pressure sensor, interior view." class="align-center" %}

Force is transmitted from the external contact point to the internal sensor by a 2.5 mm diameter steel rod. It slides smoothly inside a bronze sleeve and is supported by a plastic part. A spring applies force to the rod, keeping it in constant contact with the sensor with a preload of approximately 120 grams.

{% include figure popup=true image_path="/assets/images/sensor_dolor/estiumulador.png" alt="Robotic algometer" caption="Manual stimulator, cross-sectional view." class="align-center" %}

We also designed a small handle on the plastic part that supports the rod. It lets us pull the rod away from the sensor temporarily and lock it in place. This protects the delicate sensor from overload while we change the patient's stimulation tip, which may be pointed, a blunt blade, or another type.

{% include figure popup=true image_path="/assets/images/sensor_dolor/traba_resorte.png" alt="Spring lock" caption="Spring-lock detail: it allows the rod to be separated from the sensor and safely locked while the test tip is changed. The lock is shown in both of its possible positions." class="align-center" %}

We also made a cap that attaches to the front to protect both the stimulator and the internal parts during transport. At the back, we added a cable clamp made from a wedge printed in a soft material (TPU) and a quick-threading nut. This allowed us to secure the cable firmly to the device body and prevent it from being pulled out.

Finally, we designed the control-module enclosure as two large interlocking parts joined by three M3 screws.

{% include figure popup=true image_path="/assets/images/sensor_dolor/gabinete.png" alt="Enclosure" caption="Control-system enclosure, interior view." class="align-center" %}

To make the FireBeetle board's tactile buttons easy to use, we designed and printed small flexible plastic structures that press the on-board buttons when pushed from the outside. We had used similar solutions before, but in this case the FireBeetle buttons were soldered at 90 degrees. This orientation required more careful analysis to ensure the natural rotation of the printed part produced the optimal linear movement to trigger the button's internal mechanical click without damaging it.

<p align="center">
  <video autoplay loop muted playsinline width="80%">
    <source src="/assets/images/sensor_dolor/button_desplazamiento.webm" type="video/webm">
  </video>
</p>

The stimulation tip used in this particular case is a hypodermic needle whose bevel was removed to prevent cuts and skin penetration. We considered printing a Luer-style lock to hold the needle, but the part was too small and complex, so we chose to reuse a plastic lock from a syringe.

{% include figure popup=true image_path="/assets/images/sensor_dolor/luer.jpeg" alt="Luer lock" caption="Reused Luer lock for attaching the needle." class="align-center" %}

## Testing

To test the device, we printed additional mounting parts. These let us firmly clamp the stimulator in a vise and place a standard 100-gram weight on the actuator.

{% include figure popup=true image_path="/assets/images/sensor_dolor/setup_pruebas.jpg" alt="Test setup" caption="Test setup, with the sensor resting on a 100-gram weight." class="align-center" %}

We captured data with no load and with the weight on the sensor, at both 10 Hz (the normal operating frequency with real-time display) and 500 Hz (the maximum frequency during measurements).

{% include figure popup=true image_path="/assets/images/sensor_dolor/analisis_0gr_10Hz.png" alt="No-load analysis at 10 Hz" caption="Signal analysis results with no load (0 g) at 10 Hz." class="align-center" %}

{% include figure popup=true image_path="/assets/images/sensor_dolor/analisis_0gr_500Hz.png" alt="No-load analysis at 500 Hz" caption="Signal analysis results with no load (0 g) at 500 Hz." class="align-center" %}

{% include figure popup=true image_path="/assets/images/sensor_dolor/analisis_100gr_10Hz.png" alt="100-gram load analysis at 10 Hz" caption="Signal analysis results with a 100 g load at 10 Hz." class="align-center" %}

{% include figure popup=true image_path="/assets/images/sensor_dolor/analisis_100gr_500Hz.png" alt="100-gram load analysis at 500 Hz" caption="Signal analysis results with a 100 g load at 500 Hz." class="align-center" %}

When analyzing the sensor measurements, the first thing we see is that the average values are accurate: when a 100 g load is applied, the system measures very close to that value. However, looking at the signal in detail over long periods reveals small variations and slow drifts. These are not random noise, but effects intrinsic to the sensor, such as temperature changes or internal mechanical settling. This is also visible in the histograms, where the distribution is not perfectly clean, and in the frequency analysis, which shows that nearly all the variation happens very slowly, close to zero Hz.

In actual use, however, things are quite different. If the sensor is zeroed immediately before a measurement and used for only a few seconds, these slow effects are no longer a significant problem. Over that short interval, the system is much more stable, and what dominates is simply a little electrical noise in the signal. In practice, this means the sensor works well for quick measurements as long as it is calibrated beforehand. The reading can be smoothed further by averaging a few values. In short, although it is not ideal for prolonged measurements without recalibration, it is perfectly usable and reliable in short, controlled scenarios, where it performs best.

<p align="center">
  <video controls width="80%">
    <source src="/assets/images/sensor_dolor/video.mp4" type="video/mp4">
  </video>
</p>

---

[![Hits](https://hits.sh/prototipado.github.io/en/tutorial/portable-algometer/.svg)](https://hits.sh/prototipado.github.io/en/tutorial/portable-algometer/)