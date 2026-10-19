---
title: "Reverse Engineering Specialized's ANGi Sensor with AI Tools: A Case Study"
author: Juan I. Cerrudo
date: 2026-10-01
lang: en
page_id: angi-inv
permalink: /developments/specialized-angi-reverse-engineering/
categories:
  - Developments
tags:
  - Hardware
  - BLE
  - Android
  - Kotlin
  - nRF52
  - Artificial Intelligence
layout: single
toc: true
toc_sticky: true
excerpt: "Reverse engineering the Specialized ANGi sensor: reconstructing its BLE protocol, building Android apps, analyzing nRF52 firmware, and characterizing power consumption with AI tools."
---

Cycling is one of my favorite activities outside work. Near Oro Verde, there are roads leading to small towns and villages, with little traffic and plenty of people who use them for riding. Helmets are mandatory, and when I bought mine in early 2024, [Specialized](https://www.specialized.com) offered the [ANGi](https://www.dcrainmaker.com/2018/11/specialized-helmet-sensor.html) sensor: a crash detector that connects to a phone and, after an impact or sudden movement, sends text messages with an alert and location to the contacts you configured.

This year, Specialized discontinued the app that supported it and moved to a new one that is incompatible. The servers also stopped responding: it is no longer possible to sign in, reset a password, or create an account, so I can no longer use the old app. I was left with a useful device I could not use, and an uncomfortable question: did I really own it?

In the laboratory, we are working on an R&D project focused on the use of AI tools in the design and development of medical products. My own situation led me to explore a different application: not for design, but rather to understand how a closed-system device functions, so that I could continue using it.

### Document Structure

This case study documents the complete reverse engineering process of the ANGi sensor and the development of an open, independent alternative (*OpenAngi*) using AI tools. The work is organized into 5 main parts:

- **[Part 1: Claude and BLE Exploration](#part-1-claude-and-ble-exploration)** — Initial hardware exploration, BLE service enumeration, and static decompilation of the original APK.
- **[Part 2: OpenCode and Android App Development](#part-2-opencode-and-android-app-development)** — Building the *AngiTester* test bench, developing the final independent app *OpenAngi* (100% server-independent), and code comparison.
- **[Part 3: Hardware and Firmware – Comparative Analysis](#part-3-hardware-and-firmware--comparative-analysis-antigravity-vs-opencode)** — Extracting nRF52832 firmware, Ghidra analysis, finding the production UART console, and electrical power characterization.
- **[Part 4: DFU Firmware Update](#part-4-dfu-firmware-update)** — Locating the official firmware inside the APK, updating via Bluetooth (DFU), and integrating the process into the app.
- **[Part 5: Assessment of AI Tool Usage](#part-5-assessment-of-ai-tool-usage)** — Comparing strengths and limitations of each model, project scale/costs, and legal considerations.

A note up front: all the tools used are available for free, which is why several different ones appear throughout the article. In some cases, we were trying different tools; in others, we ran into model limitations (for example, many models in OpenCode do not support images) or reached the available token limits.

---

## Part 1: Claude and BLE Exploration

We carried out the first phase entirely in Claude (Sonnet 5) through the claude.ai web interface. One continuous session, one model, no interruptions.

Our first question was whether the project we wanted to tackle was feasible. Claude searched the web and confirmed that ANGi runs on a Nordic Semiconductor [nRF52832](https://www.nordicsemi.com/Products/nRF52832) (Cortex-M4F, 512 KB flash)—an extremely well-documented BLE chip with a public SDK, identified from the device's [FCC filing](https://fccid.io/OXXANGI001).

Since the official app was effectively dead (no login, no password recovery, no new account creation), we ruled out capturing authenticated traffic and reorganized the plan around two parallel tracks:
1. **Direct hardware exploration** with nRF Connect for Mobile (Nordic's free tool), enumerating BLE services and characteristics without going through Specialized's app.
2. **Static APK decompilation** of the app downloaded from a mirror (apkmirror.com, since Google Play would not let us download it either) using [jadx](https://github.com/skylot/jadx)—the decompiled code does not depend on the backend being online.

We connected to the sensor using nRF Connect and shared the first screenshots for analysis. That gave us an initial map of its services: the custom service `afd0ffa0…afd0ffa8`, Battery Service, Device Information, and Nordic's "Buttonless" DFU service.

### a. Incremental Development of Exploration Scripts

With Claude, we incrementally built a series of Python scripts using [bleak](https://bleak.readthedocs.io/en/latest/) (a cross-platform BLE client). Each script answered a specific question left open by the previous one:

- `angi_sniffer.py`: connects to the sensor, subscribes to NOTIFY characteristics, and logs all raw traffic with timestamps to the screen and a CSV file.
- `angi_command_sweep.py`: automatically sweeps all 256 possible values (`0x00–0xFF`) on WRITE characteristics, comparing the STATUS register before and after each write to detect reactions.
- `angi_heartbeat_test.py`: listens passively without writing anything, specifically to rule out the hypothesis that certain observed packets were command responses. The result: they were a periodic heartbeat (an uptime counter increasing ~1/second).

The workflow was always the same: run the script against the real sensor → paste the log or traceback into Claude → let it adjust the code → repeat. Simple, but effective.

We made progress this way, but the process was relatively slow. The big leap came with decompilation. Claude put together a specific, detailed prompt for [OpenCode](https://opencode.ai/) (another AI tool, with direct filesystem access), with step-by-step instructions: decompile with jadx, perform a targeted search (grep) for UUIDs and constants already identified over BLE, identify classes and methods for each UUID, and generate a protocol document.

The result was a complete protocol map extracted from the decompiled Java classes. Using that information, we built the final `angi_client.py`: a complete interactive BLE client implementing the sensor's real extended-command pattern, parsers for the three response packet formats (ASCII, 15-byte IMU report, and 20-byte impact report), named commands, a raw `<hex>` mode for free-form experimentation, and an autotest that sweeps the 17 known opcodes × 5 parameter variants with automatic reconnection.

### b. Identifying the Hardware and On-Board Sensors

Another line of investigation focused on the hardware itself: we opened the sensor and took pictures of the board.

{% include figure popup=true image_path="/assets/images/sensor_angi/top_angi.JPG" alt="Top view of the ANGi sensor board" caption="Top view of the ANGi sensor PCB, showing the nRF52832 SoC in a BGA package, sensors, and the CR2032 coin-cell holder. The PCB is glued to the enclosure; removing it may have flexed it enough to break some traces. A jumper is visible bridging two VCC points, which was needed to get it working again." class="align-center" %}

{% include figure popup=true image_path="/assets/images/sensor_angi/bottom_angi.JPG" alt="Bottom view of the ANGi sensor board" caption="Bottom view of the PCB showing the test points." class="align-center" %}

Claude identified the chips from their markings and cross-checked them against public manufacturer documentation:

- **ST LIS3DH accelerometer**: identified by the marking "C3H," confirmed against a post by an STMicroelectronics engineer on the company's forum.
- **ST L3GD20H gyroscope**: identified by the marking "3GDH," verified against the same source.
- **nRF52832 BLE SoC**: in a BGA package beneath a protective layer of silicone.

{% include figure popup=true image_path="/assets/images/sensor_angi/foto_accel_gyro.JPG" alt="Close-up of the sensors on the board" caption="Close-up of the STMicroelectronics sensors identified on the board: LIS3DH accelerometer ('C3H') and L3GD20H gyroscope ('3GDH')." class="align-center" %}

### c. Claude Session Numbers

| Metric | Value |
| :--- | :--- |
| **Model** | Claude Sonnet 5 (one model for the entire session) |
| **Interface** | claude.ai (web chat) |
| **Conversation turns** | 26 |
| **Tool calls** | ~54 (manual count) |
| **Images analyzed** | 11 (nRF Connect screenshots, PCB photos, logos) |
| **Documents uploaded** | 4 |
| **Python files created** | 4 |
| **Cost** | Claude subscription plan |

At this point, with the original APK decompiled, we moved to OpenCode to continue developing the test apps and the final solution.

---

## Part 2: OpenCode and Android App Development

We built the functional Android apps in the second phase using OpenCode, an AI-assisted IDE that runs in a terminal with direct filesystem access, command execution (bash/PowerShell), and code-editing tools. Everything that follows comes from real data in OpenCode's local SQLite database, not estimates.

OpenCode used 3 different models during the session, all on free tiers:

| Model | Period | Messages | Input tokens | Output tokens | Reasoning |
| :--- | :--- | :--- | :--- | :--- | :--- |
| ling-3.0-flash-fin | 26/09 (kickoff) | 231 | 1,859,556 | 85,643 | 39,546 |
| longcat-2.5-preview | 28/09 (exploration) | 24 | 312,099 | 14,417 | 55,808 |
| mimo-v2.6-flash | 28/09–29/09 (most of the development) | 483 | 2,171,889 | 218,998 | 256,083 |

738 assistant messages, 4.3 million input tokens, 319K output tokens, and 351K reasoning tokens. We switched between models based on the results: sometimes a model started taking longer or failed to solve problems, possibly because the session context had grown too large, so we tried another model.

---

### a. Development with AngiTester

Before building the final app (OpenAngi), we needed an Android "lab test bench." That is how **AngiTester** came about: a lightweight app made of 6 [Kotlin](https://kotlinlang.org/) files, whose sole purpose was to interact directly with the real SHS-D33F sensor (all sensors present as SHS-“last 4 digits of MAC”) and validate every hypothesis byte by byte.

AngiTester's main screen has three key areas for inspecting BLE traffic in real time:

- **Live command terminal (`log [HH:mm:ss.SSS]`)**:
  - Console-style view with smart auto-scroll (it pauses when the user scrolls up to read the history and resumes when they return to the bottom).
  - Logs all bidirectional traffic (TX/RX), ASCII notifications, status responses, and battery readings in real time every five minutes.
- **Custom real-time IMU graph (`ImuGraphView.kt`)**:
  - Plots the accelerometer's three axes ($g$) and angular velocity ($	ext{°/s}$) at 5 Hz in TEST mode.
  - Indexed by sample number (`sampleNumber`), avoiding horizontal distortion or "jumps" when BLE notifications arrive in bursts.
  - Smoothed Y-axis auto-scaling prevents a sudden peak from flattening the continuous curve.
- **Direct and extended command buttons**:
  - Direct modes (`afd0ffa1`): Activate (syncs RTC + MODE `0x01`), Normal (`0x00`), Test (`0x40` IMU streaming).
  - Extended commands (`BLOCK_DATA afd0ffa4` + trigger `0x20`): firmware version (`0xC3`), accelerometer/gyroscope test (`0xC0/0xC1`), LED control (`0xFE/0xC2`), impact simulation (`0xFB`), read/report recorded events (`0x80/0x81`), among others.

AngiTester was not merely a test app: it was the empirical tool that allowed us to decipher the hardware's real behavior and uncover aspects that were unclear in the decompiled code:

i. **Decoding and validating STATUS (`afd0ffa2`)**:
   We empirically verified the structure of the 1-byte bit field returned by the sensor:
   - **Bit 7 (`0x80`)**: ERROR bit (if 1, the remaining 7 bits are the error code).
   - **Bits 6-5 (`0x60`)**: Mode (`0`=NORMAL, `1`=EXT_CMD, `2`=TEST).
   - **Bit 4 (`0x10`)**: Active session.
   - **Bit 3 (`0x08`)**: Connection heartbeat active.
   - **Bit 0 (`0x01`)**: Last command completed.

   We catalogued the raw status values observed during operation: `0x00` (freshly connected), `0x01` (NORMAL + active session), `0x03` (active session with pending events), `0x21` (EXT_CMD mode completed), and `0x82` (ACK with error/transient status following impact commands).

ii. **Discovering and calibrating the accelerometer scale**:
   The original APK divided all values in the IMU report (`IcedotTestReport`) by 100. However, when capturing stationary data with AngiTester and processing the evidence (`angi_static.csv`), we discovered that axes X, Y, and Z produced a resultant vector of ~1.73 g, which is physically impossible at rest.

   Through cross-testing, we derived the true divisor for the accelerometer axes:
   $$	ext{Divisor } K = 173.2 pprox 100 \cdot \sqrt{3}$$
   With this correction, the axes reported exactly ~1.00 g at rest (a mean error of only 9 mg), while `accelVector` and `rotation` retained a divisor of 100.

iii. **Confirming the gyroscope's single channel**:
   We confirmed that Specialized's firmware does not transmit the gyroscope's 3 independent axes over BLE streaming. Instead, it reduces angular velocity to a single scalar channel (`rotation`) in °/s (bytes 11-12) and its angular acceleration (`rotationalAccel`, bytes 13-14). We determined that this corresponds to a tilting rotation axis (close to the helmet's X axis).

iv. **Validating the session arming and impact flow**:
   We verified that if the 7-byte UTC RTC (`afd0ffa7`) is not sent before enabling `MODE 0x01`, impact reports are recorded with a timestamp of 0.
   We captured and validated the real impact notification on `afd0ffa6` (a 20-byte packet: type, battery, version, timestamp, peakG, peakR, peakRA, and deviceID).

<figure class="align-center">
  <video controls width="100%">
    <source src="/assets/images/sensor_angi/angi_tester.mp4" type="video/mp4">
    <source src="/assets/images/sensor_angi/angi_tester.MOV" type="video/quicktime">
    Your browser does not support playing this video.
  </video>
  <figcaption>AngiTester: application execution.</figcaption>
</figure>

---

### b. OpenAngi: the final independent app

**OpenAngi** is the final application (30 Kotlin files, a 16.5 MB APK): a complete rewrite from scratch using a modern architecture with Kotlin + Jetpack Compose Material 3 + MVVM + Room + DataStore + Foreground Service.

The app works **without a server, accounts, or a backend**: the phone itself sends SMS messages directly through `SmsManager`.

**OpenAngi screens:**

The app has four main tabs in a bottom bar, plus two overlay screens (Device and Alert):

| Screen | What it does |
| :--- | :--- |
| **Scan** | Lists nearby ANGi sensors (filtered by BLE service, with a name-based SHS- fallback), and shows signal RSSI. Tap a sensor to connect. If a session is already active, a "Scanning paused" banner appears to avoid interfering with the radio. |
| **Device** | The main dashboard: animated "Sensor active and listening" pulse, sensor battery, firmware, hardware, raw status, active session, sensor heartbeat, and received impact alerts (diagnostic counter). Buttons: Disconnect, Simulate impact, and Declare emergency. |
| **Alert** | Emergency screen: a large countdown (configurable from 5 to 120 seconds), impact G peak, and a large "I'M OK — CANCEL" button. If not cancelled, it shows confirmation with the SMS messages sent and the location. It appears as a full-screen notification if the app is in the background. |
| **Contacts** | Add emergency contacts manually or from the phone's contact picker, edit them, or remove them. When a contact is added, the app automatically sends them an SMS: "You've been added as an emergency contact in OpenAngi." |
| **History** | All recorded events: date, status chip (cancelled/notified/failed), G peaks, and location (tap to open Google Maps). Clear button (cancelled/failed only, or all). |
| **Settings** | Countdown duration, sound/vibration, your name (to personalize the SMS), simulate impact, declare emergency, battery-optimization exemption status, and the full legal disclaimer. |

{% include figure popup=true image_path="/assets/images/sensor_angi/pantallas.png" alt="OpenAngi App Screens" caption="Screenshots of the OpenAngi Android app, server-free with direct SMS delivery." class="align-center" %}

**BLE connection flow**

{% include figure popup=true image_path="/assets/images/sensor_angi/flujo_conexion_ble.png" alt="BLE connection flow" caption="Diagram of the BLE connection and synchronization flow implemented in OpenAngi." class="align-center" %}

Every GATT operation goes through a Mutex (one at a time, with an 8-second timeout). If any step fails, automatic reconnection repeats the entire setup from scratch, because the sensor loses its state when disconnected. The backoff grows from 2 to 60 seconds with jitter.

**Impact flow: from detection to SMS**

This is the app's central component—the `ImpactFlowController` state machine:

{% include figure popup=true image_path="/assets/images/sensor_angi/flujo_impact_controller.png" alt="Impact state machine" caption="State machine of the ImpactFlowController for impact detection and emergency messaging." class="align-center" %}

The flow runs at the process level, not the UI level: even if the app is closed, as long as the Foreground Service is running, detection works and the full-screen notification opens the alarm directly.

**Declaring an emergency:** there is a dedicated button on the Device and Settings screens that triggers exactly the same flow with a synthetic event (type DECLARED, peak 0 g). The SMS says *"declared an emergency (manual alert)"* instead of *"suffered an impact"*. Useful if you need help without having had a crash.

**Reliability: the Foreground Service**

The `AngiMonitorService` owns the BLE connection and runs with a persistent notification showing the actual status:
- 🟢 **Green:** connected and monitoring (shows sensor battery)
- 🟡 **Amber:** reconnecting (with a countdown)
- 🔴 **Red:** disconnected

If there is no connection for 5 minutes, the app displays a high-priority notification: *"OpenAngi lost connection to the sensor"*. The guiding philosophy is never to fail silently—if something goes wrong, the user should know.

The `BootReceiver` reconnects automatically when the phone starts, and if Android blocks background startup, it sends a notification prompting the user to open the app manually.

<figure class="align-center">
  <video controls width="100%">
    <source src="/assets/images/sensor_angi/open_angi.mp4" type="video/mp4">
    Your browser does not support playing this video.
  </video>
  <figcaption>OpenAngi: application execution.</figcaption>
</figure>

---

### c. How It Differs from the Original App: Protocol Facts vs. Code

One of the most important aspects of this project is understanding what was taken from Specialized's original app and what was not—how much the tools copied from the original APK and what was created from scratch. We ran automated scripts comparing the 262,495 lines of the decompiled APK against all the new code:

| Check | AngiTester | OpenAngi |
| :--- | :--- | :--- |
| Lines identical to decompiled code (≥25 chars) | 2 | 1 |
| Shared UI strings | 0 | 0 |
| Copied resources (out of 879 drawables) | 0 | 0 |

The 3 "identical" lines are idiomatic Android boilerplate (`if (bluetoothManager != null)`, `if (cursor.moveToFirst())`). The only shared literals are BLE UUIDs—functional data essential for communicating with the sensor.

From the decompilation, we extracted technical protocol facts—the minimum information any app needs to communicate with the sensor:
- BLE characteristic UUIDs (`afd0ffa0…ffa8`).
- Packet offsets and lengths: the impact report is 20 bytes, the RTC is 7 bytes, and the IMU report is 15 bytes.
- Byte values: modes (`0x00` NORMAL, `0x01` ACTIVE, `0x20` EXT_CMD, `0x40` TEST), report types (1=LINEAR, 2=ROTATIONAL, 3=LINEARROTATIONAL), and STATUS bits.
- Numeric error codes (33=SUCCESS, 64=INVALID_PARAM).
- Activation sequence: write RTC + MODE `0x01`.

All of this is documented and cited in the code comments. These are functional facts—like documenting that "the HTTP port is 80"—not copied creative expression.

| Aspect | Original app (Specialized Ride) | OpenAngi |
| :--- | :--- | :--- |
| **Namespace** | `com.specialized.ride` | `com.openangi.app` |
| **Class names** | `IcedotReport`, `IcedotBLE` | `ImpactReport`, `AngiProtocol` |
| **Packet parsing** | `==20` with an exception (`IcedotException`) | Tolerant `≥16` with `parseOrNull` |
| **Architecture** | Legacy Java code | Kotlin + Compose M3 + MVVM + Room |
| **Emergency alerts** | Own backend → `POST /events` → server sends SMS | Direct SMS via `SmsManager` (serverless) |
| **UI / resources** | XML layouts, Specialized drawables | Pure Compose, custom dark theme, original icon |
| **Backend / URLs** | `api.prod.s-digital.co` | None — zero external dependencies |

The most significant architectural change is how alerts work: the original app depended on a Specialized backend that received the event through an API and sent the SMS from its server. When Specialized discontinued the service, that stopped working. OpenAngi removes that dependency entirely: the phone sends the SMS directly, without going through any server. It is simpler, more reliable, and does not depend on anyone keeping a backend alive.

In summary: from the original APK we took the protocol data (UUIDs, packet formats, activation sequence)—technical facts required to interoperate with the hardware—and built an entirely new app with a different architecture, different UI, and a fundamentally different alert mechanism.

---

### d. Overall Software Development Summary

The numbers for the entire project reflect the scale of the AI-assisted work:

| Metric | Claude | OpenCode | Total |
| :--- | :--- | :--- | :--- |
| **Models used** | 1 | 3 | 4 |
| **User turns / prompts** | 26 | 75 | 101 |
| **Assistant messages** | 26 | 738 | 764 |
| **Tool calls** | ~54 | 1,067 | ~1,121 |
| **Code files created** | 4 (Python) | 36+ (Kotlin) | 40+ |
| **Cost** | Claude subscription | $0 (free tier) | — |
| **Period** | 1 session | 3 days (26–29/09) | ~3 days |

The table below breaks down the tools used in OpenCode:

| Tool | Calls | Description |
| :--- | :--- | :--- |
| bash | 378 | Gradle, Python, git, apksigner, system commands |
| edit | 268 | Surgical code edits (search & replace) |
| read | 241 | Reading project files and decompiled code |
| write | 103 | Creating new files |
| grep | 54 | Searching for patterns in code |
| glob | 11 | Searching for files by name |
| todowrite | 6 | Task list updates |
| MCP codebase-memory | 4 | Codebase knowledge graph |
| task (subagent) | 1 | Delegating to a child agent |

In addition: 720 reasoning traces, 256 text responses, and 8 context compactions (when the context window fills up, the model compresses the history and continues).

**What did we learn from the process?**

AI did not replace the work—we were still the ones connecting the sensor, running scripts, taking photos, opening the device, decompiling the APK, and making design decisions. But it dramatically accelerated every intermediate step: instead of searching datasheets, reading forums, and writing byte parsers by hand, conversations with the models took us from "what is this chip?" to "we have a signed, working APK" in 3 days.

It is even one of the lines of research we are incorporating into the PID: closing the loop—granting LLMs access to hardware and enabling interaction with physical tools—to further accelerate the experimentation process.

The combination of tools worked well:
- **Claude** (web chat with vision, search, and image analysis) was ideal for the exploratory phase: identifying chips from photos, putting together an attack plan, and building incremental scripts.
- **OpenCode** (filesystem access, bash/PowerShell command execution, and code editing) was ideal for the construction phase: compiling, fixing Gradle errors, and iterating over 30 Kotlin files until we had a functional APK.

All Android app development—738 messages, 4.3 million input tokens, and more than a thousand tool calls—cost exactly zero dollars using free models.

---

## Part 3: Hardware and Firmware – Comparative Analysis (Antigravity vs. OpenCode)

With the app running, we turned to the hardware itself: the sensor PCB has six test pads arranged in the standard [Tag-Connect TC2030](https://www.tag-connect.com/product/tc2030-idc-6-pin-plug-of-nails-spring-pin-cable-with-ribbon-connector) footprint.

{% include figure popup=true image_path="/assets/images/sensor_angi/diy_tag_connect.png" alt="DIY Tag-Connect connector" caption="Handmade Tag-Connect programming cable built to dump the ANGi sensor's nRF52832 flash." class="align-center" %}

We connected a J-Link via SWD, and since the manufacturer had not enabled read protection on the nRF52832 (`APPROTECT`), we were able to dump the entire 512 KB flash memory over SWD without restrictions.

This stage presented an interesting opportunity: having already completed the reverse engineering of the protocol from the APK, we used the extracted firmware to compare two AI coding environments working in parallel on the same problem: **Antigravity** (Google's agentic IDE) and **OpenCode**.

---

### a. Initial Extraction and Decompilation with Claude and Ghidra

We dumped the firmware using OpenOCD and a Segger J-Link probe over SWD:

{% include figure popup=true image_path="/assets/images/sensor_angi/tag_connect.jpg" alt="Tag-Connect pads on the PCB" caption="Tag-Connect programming pads on the ANGi sensor PCB: SWDIO, SWDCLK, VCC, GND, and RESET." class="align-center" %}

The dump yielded a 512 KB `.bin` file that we analyzed with [Ghidra](https://ghidra-sre.org/) (the open-source reverse engineering framework).

We turned to Claude (Sonnet 5.5) to assist in the initial Ghidra analysis. We provided the memory map and string references, and Claude guided the setup of memory segments, identify the interrupt vector table, and distinguish Nordic SoftDevice calls (SVCs) from application code:

{% include figure popup=true image_path="/assets/images/sensor_angi/nrf_dump.png" alt="Ghidra memory map of the nRF52832" caption="Memory structure of the nRF52832 in Ghidra: SoftDevice S132, Application space, and Bootloader." class="align-center" %}

Claude also generated a Ghidra Java script (`AngiAutoAnalyzer.java`) to automatically label known Nordic SDK 14 functions and annotate BLE event handlers.

With the disassembly baseline established, we provided the raw Ghidra output to both Antigravity and OpenCode to assess their decompilation, analysis, and hardware-reconstruction capabilities.

---

### b. Comparative Summary: Antigravity vs. OpenCode

We set both tools the task of analyzing the decompiled firmware: identify sensor initialization, locate the impact-detection logic, extract BLE handlers, and identify any debug/test interfaces.

| Capability | Antigravity (Gemini 3.6 Flash) | OpenCode (big-pickle) |
| :--- | :--- | :--- |
| **Model** | Gemini 3.6 Flash | big-pickle (free tier) |
| **Approach** | Hardware-first: test points, UART, pins | Code-first: event architecture, flash map |
| **UART discovery** | Identified `P0.30` as test trigger and 38,400 baud | Identified UART functions, missed trigger pin |
| **Hardware validation** | High: guided oscilloscope and multimeter testing | Medium: focused on static code analysis |
| **C code quality** | Prototypes, register maps, and comments (no function bodies) | 77,793 lines of C; duplicated function bodies |
| **Sensor bus** | Hallucinated hardware SPI (actually software bit-banging) | Correctly identified register-level sensor access |
| **Impact algorithm** | Described high-level thresholds from strings | Analyzed `impact_detector_config` (0x2F60C) in detail |

---

### c. Main Achievements of Each Tool

#### Antigravity: Hardware Insight and UART Console

Antigravity's biggest contribution was bridging the gap between firmware and physical hardware. It not only identified the presence of a factory test console in the code, but deduced the exact GPIO pin (`P0.30`) used to trigger it, the required baud rate (38,400), and guided the physical search for those signals among the PCB test points using an oscilloscope.

#### OpenCode: Firmware Decompilation and Architecture

OpenCode excelled at deep static analysis of the firmware binary. It produced a comprehensive 77,793-line annotated C file, mapped the event-driven architecture centered around `evt_set`, identified the 24-byte telemetry buffer structure, and performed the most detailed decompilation of the fall-detection algorithm's configuration and processing routines.

---

### d. Finding the UART Console and Validating It on the Board

Identifying the PCB test points was a much more hands-on process: we used a multimeter and oscilloscope to identify the physical signals. GND and VCC were easy to locate from the battery holder.

Once the UART and console pin were confirmed, the task was to find them on the PCB. We observed activity on all test points while powering the device and found six with pulse signals (sensor communication), another six held low, and a single test point held high (3 V).

{% include figure popup=true image_path="/assets/images/sensor_angi/test_points.png" alt="Test points on the ANGi PCB" caption="Test points identified on the ANGi PCB. TM is pin P0.30 (Test Mode)." class="align-center" %}

At first, we expected to find at least two test points high, corresponding to the UART's idle state. When we asked Antigravity, it explained that the UART is not configured in normal mode to save power, and is only enabled at startup when `P0.30` is held low. This led us to think that the only high test point should be the test-mode pin (`P0.30`) with a pull-up.

We described the situation to the model:

**👤 User (step 258):**  
> *"Another question about the test points: one is at VCC, six show some kind of signal/pulses, and six show a low signal to GND. Could it be that the UART isn't configured unless it is in test mode? Why would I expect to see two more UART pins idle at 3 V?"*

**🤖 Antigravity (step 259):**  
> *"Exactly! Your deduction is 100% correct.
> 
> **1. Why don't you see the UART at 3 V (idle) in normal mode?**  
> On Nordic nRF52 microcontrollers, the UART is not permanently assigned to fixed hardware pins....
> 
> **2. Explanation of the signals you are currently measuring (Normal Mode)**  
> - **1 test point at VCC (3 V):** Main battery / VDD supply.  
> - **6 test points with pulses/signals:** Bit-banged communication bus for the MEMS sensors (LIS3DH + gyroscope): clock, MOSI, MISO, chip selects (CS). Impact/movement interrupts (INT1 / INT2).  
> - **6 test points at GND:** Disabled pins, unused inputs, or PCB ground pads (GND)."*

**👤 User (step 260):**  
> *"I can't access the pins; the nRF52 is a BGA. Would it be plausible for the test point at VCC to be that pin with a pull-up?"*

**🤖 Antigravity (step 261):**  
> *"Yes, that is HIGHLY likely! It is the standard factory-test architecture for boards with a BGA package."*

This response caught us somewhat off guard, as we could not identify what technical reasoning connected the nRF52's BGA package to the conclusion that this specific test point corresponded to a pull-up. While we had provided several hints to see if the model could follow our own reasoning, its conclusion appeared to reach beyond what the available context supported at that moment.

To test the theory, we forced `P0.30` low during power-up: the LEDs flashed in a different sequence, confirming test mode. When we measured again with the oscilloscope, two test points that had previously been low transitioned to high (UART idle level).

**👤 User (step 262):**  
> *"I grounded it, and on power-up the red and green LEDs flashed, and TP10 and TP12 transitioned to VCC. Those must be TX and RX, right?"*

**🤖 Antigravity (step 263):**  
> *"GREAT NEWS! 🎉 You successfully entered Factory Test Mode!"*

Next, we connected a USB-to-serial converter to both pins. Through trial and error, we determined which test point was RX and which was TX, and verified the actual baud rate of **38,400 baud**:

{% include figure popup=true image_path="/assets/images/sensor_angi/uart.jpg" alt="UART serial connection on the PCB" caption="Physical connection of the USB-to-serial converter to the UART test points on the ANGi sensor board." class="align-center" %}

With Realterm configured for 38,400 baud and the device in test mode, we interacted with the production console and verified its command set:

{% include figure popup=true image_path="/assets/images/sensor_angi/terminal.png" alt="Serial terminal interacting with ANGi" caption="Serial terminal interacting at 38,400 baud with the firmware's production commands." class="align-center" %}

Test mode allows the device's operating mode to be changed from an RF perspective for certification testing. This aligns exactly with the photographic documentation on the FCC website: the device powered by an external supply, with three wires exiting through one side (UART TX/RX and GND):

{% include figure popup=true image_path="/assets/images/sensor_angi/angi_fcc_test_setup.png" alt="Test setup reported to the FCC" caption="Test setup documented with the FCC: external power and a serial connection for ANGi sensor RF certification tests." class="align-center" %}

---

### e. BLE Operating Modes

The firmware analysis and UART validation allowed us to reconstruct the ANGi sensor's complete state machine across all operating modes:

{% include figure popup=true image_path="/assets/images/sensor_angi/flujo_estados_dispositivo.png" alt="ANGi sensor operating modes" caption="Reconstructed operating modes of the Specialized ANGi sensor." class="align-center" %}

1. **Low Power / Sleep:** Default state on the shelf or between rides. The nRF52 enters System ON sleep; the LIS3DH accelerometer remains configured for threshold-based wake-on-motion. Current draw drops to ~21 µA.
2. **Factory / Production Test:** Entered by pulling `P0.30` low during power-on. Enables the 38,400-baud UART console, provides direct register access, and supports RF test modes for regulatory certification.
3. **Normal Connected (Ride Mode):** The sensor pairs with the phone, writes RTC UTC, enables `MODE 0x01`, and enters continuous crash-monitoring mode. Periodic heartbeats maintain connection state. Average current draw is ~62 µA.
4. **Test / IMU Streaming:** Enabled via `MODE 0x40`. Streams raw accelerometer and gyroscope data at 5 Hz over BLE characteristic `afd0ffa8`. Used by AngiTester for sensor calibration.

---

### f. Fall Detection Algorithm

To understand how ANGi decides that a fall has occurred, we consulted several models and compared their answers with the firmware. Gemini (Gemini 3.6 Flash), ChatGPT (GPT 5.6 Luna), and Copilot all pointed to two relevant parts: impact configuration for the **LIS3DH** accelerometer and triaxial analysis by the **L3GD20H** gyroscope. Their explanations, however, did not show the same degree of caution. ChatGPT acknowledged several unknowns; Gemini proposed a more complete sequence but presented some hypotheses as facts that the available code did not prove. For example, it described a buffer as circular and pre-impact, and interpreted the **8** and **2** counts as confirmation samples.

Reviewing the C code later showed that these interpretations went beyond the evidence: the code writes samples, but that does not prove they are later read to decide whether a fall occurred; the values **8** and **2** appear as parameters, but we could not find who consumes them or what their timing means. OpenCode provided a more detailed reading of both branches. In its analysis, `impact_detector_config` (`0x2F60C`) sets a default threshold of **16000** and **8** counts for the accelerometer. When an interrupt occurs, it calculates the magnitude of the vector:

$$	ext{magnitud} = \sqrt{X^2 + Y^2 + Z^2}$$

It checks whether it exceeds that threshold and whether the difference with the previous reading is greater than **1000**. The routine records maximums, counts impacts, and signals an event. The gyroscope branch (`0x2FAF8`) sets a threshold of **1000** and **20** counts. It evaluates the difference between consecutive values:

$$	ext{dif}_i = 	ext{val}_i - 	ext{prev}_i$$

and accumulates:

$$	ext{acc}_i = 	ext{dif}_i + (	ext{acc}_i - 	ext{acc}_i/4)$$

It checks whether $	ext{dif}_i > 1000$ and whether $	ext{acc}_i$ exceeds **20**. It also updates counters and minima, can store `X, Y, Z, 0xAAAA`, and signals an event. The sensor's identity is confirmed; the units and physical meaning of these values are not. “Energy” and *jerk* are analogies, not verified conclusions.

A LIS3DH task publishes telemetry while the L3GD20H gyroscope routine analyzes samples. This confirms parallel activity, but not that the two branches converge on a single decision. We have not yet reconstructed the criterion that turns these signals into a confirmed fall or the path that generates an alarm.

{% include figure popup=true image_path="/assets/images/sensor_angi/algoritmo_deteccion.png" alt="Fall detection algorithm diagram" caption="Firmware operations observed in the sensor-analysis branches; relationships that have not been confirmed are shown as unknowns." class="align-center" %}

We could continue with a more exhaustive analysis, but considered the work completed sufficient for the project's goals and activities. What is certain—and confirmed by the analysis of the APK—is that the device reports three types of events or alarms: Linear, Rotational, and Linear+Rotational.

---

### g. Critical Review of the C Files and Detected Hallucinations

A close audit of the delivered `.c` files revealed strengths, but also inconsistencies and typical LLM hallucinations:

**`app_annotated.c` (Antigravity)**
Although its Markdown report was very accurate about the UART console, the delivered C file (630 lines) was incomplete:
- **Missing decompiled code:** It contained only function prototypes, register tables, and comments; it did not include any function bodies.
- **Technical inconsistencies:**
  - It claimed the sensors used hardware SPI0/TWI0 and SPI1/TWI1 peripheral buses, while the firmware actually implements software bit-banging.
  - It identified the gyroscope as an L3GD20 with `WHO_AM_I` register value `0xD4` or `0xD7` (in fact, `0xD7` corresponds to the L3GD20H).

**`app_decompiled_annotated.c` (OpenCode)**
At 1.9 MB and 77,793 lines of Ghidra decompiled code, this was a massive deliverable, but suffered from structural flaws:
- **Duplicated function bodies:** Of the 134 renamed functions, approximately 79 contained the new annotated body followed immediately by the original Ghidra body with a stray `{` bracket, preventing direct compilation.
- **UART trigger confusion:** The `main()` code checks whether `P0.30` is held low to flash 3 times and enter `rf_test_console_poll()`. OpenCode misinterpreted this check as "button detection / charging contact."
- **Register map errors:** It assumed the nRF52832 had a "P1" pin port (it only has port P0 with 32 GPIOs), confused the GPIO register base `0x50000500`, and discarded the sensor's `0xD7` response because it did not recognize it as the L3GD20H.

**Final Verdict and Optimal Workflow**

| Aspect | Antigravity (Gemini 3.6 Flash) | OpenCode (big-pickle) |
| :--- | :--- | :--- |
| **Delivered C code** | Prototypes, structures, and comments only (26 KB) | Full named C decompilation (1.9 MB) |
| **Internal coherence** | Low (.c contradicts findings in the .md report) | Medium (sound architecture analysis, duplicated C functions) |
| **High-impact findings** | UART console, pin P0.30, baud rate, and TPs (verified on chip) | Event architecture (`evt_set`), 24 B bus, flash map |
| **Hallucinations / Errors** | False hardware SPI, omitted function bodies | nRF52 GPIO register confusion, duplicated brackets in C |

Neither tool delivered a perfect result on its own, but together they complemented each other: Antigravity excelled at physical and hardware validation; OpenCode excelled at decompilation breadth and architecture. The optimal workflow consisted of taking OpenCode's decompilation base, stripping duplicates via script, and incorporating the hardware parameters discovered by Antigravity.

---

### h. Power Measurements and Electrical Characterization

To conclude the hardware investigation, we measured power consumption with the device in different states using Nordic Semiconductor's [Power Profiler Kit II (PPK2)](https://www.nordicsemi.com/Products/Development-hardware/Power-Profiler-Kit-2).

The tool measures currents from 200 nA to 1 A with a 100 ksps sampling rate. We connected the ANGi to the PPK2, powering it at 3.0 V (simulating a CR2032 coin cell), and tested three representative operating modes from the device's life cycle:

{% include figure popup=true image_path="/assets/images/sensor_angi/ppk2.jpg" alt="PPK2 measurement setup" caption="Measurement setup with the Nordic Power Profiler Kit II (PPK2) connected to the ANGi sensor to characterize real-time power consumption." class="align-center" %}

---

#### i. Normal Connected Activity (Monitoring During a Ride)

Once the sensor wakes up (from movement) and successfully connects to the phone, it enters its normal operating regime: the processor monitors the accelerometer/gyroscope and periodically sends a heartbeat over BLE:

{% include figure popup=true image_path="/assets/images/sensor_angi/se_despierta_y_conecta_anotado.png" alt="Power consumption while waking and connecting" caption="PPK2 measurement: Cyan, low power/sleep; Yellow, wake-on-motion and connection search; Pink, Connected, BLE activity and normal consumption." class="align-center" %}

Analyzing the stable interval (from second 20 of the capture onward, duration 41.86 s):
- **Average current:** 0.062 mA (62.1 µA)
- **Maximum current:** 21.64 mA (BLE radio spikes)
- **Minimum current:** 0.004 mA (3.9 µA in inter-event sleep)
- **Charge consumed:** 0.000722 mAh
- **Estimated lifetime (continuous CR2032):** 3,706.6 hours (~154.4 days)

---

#### ii. Connection Search and Low Power (Sleep / Standby)

When the sensor powers on or detects motion, it attempts to connect during an initial window (15 to 38 seconds). If it finds no authorized phone, it terminates transmission and enters deep low-power sleep after second 40:

{% include figure popup=true image_path="/assets/images/sensor_angi/se_enciende_no_se_conecta_anotado.png" alt="Power consumption during search and standby" caption="PPK2 measurement: Green, wake-on-motion; Yellow, connection search without finding a host; Cyan, low power/sleep." class="align-center" %}

Analyzing the low-power period (seconds 40 to 90.8, duration 50.85 s):
- **Average current:** 0.021 mA (21.0 µA)
- **Maximum current:** 0.024 mA (24.1 µA)
- **Minimum current:** 0.019 mA (18.8 µA)
- **Charge consumed:** 0.000297 mAh
- **Estimated lifetime (continuous CR2032):** 10,949.6 hours (~456.2 days / 1.25 years)

---

#### iii. Intensive Search Sequence After Disconnection

If the sensor loses BLE connection with the phone during a ride (for example, by moving out of range or signal drop), it initiates an intensive sequence of repeated retries (~15 attempts across a 180 to 900-second window) to re-establish the session:

{% include figure popup=true image_path="/assets/images/sensor_angi/buscaba_se_conecta_luego_se_desconecta_anotado.png" alt="Search, connection, and disconnection" caption="PPK2 measurement: Yellow, search and connection attempt; Pink, Connected, BLE activity and normal consumption; Red, reconnection attempts following disconnection." class="align-center" %}

{% include figure popup=true image_path="/assets/images/sensor_angi/prende_busca_no_conecta_se_mueve_busca_conecta_desconecta_no_reconecta_anotado.png" alt="Failed reconnection cycle" caption="PPK2 measurement: Cyan, low power; Green, wake-on-motion; Yellow, search without finding host; Pink, Connected, BLE activity and normal consumption; Red, reconnection attempts following disconnection." class="align-center" %}

Analyzing the search burst period (seconds 180 to 900, duration 720 s):
- **Average current:** 1.878 mA (1877.8 µA)
- **Maximum current:** 23.73 mA (continuous advertising and RF scanning peaks)
- **Minimum current:** 0.004 mA (3.7 µA)
- **Charge consumed:** 0.375561 mAh
- **Estimated lifetime (if searching 100% of the time):** 122.5 hours (~5.1 days)

---

#### Summary of Power Consumption and Estimated Battery Life

| State / Mode | Average Current | Max / Min Current | Estimated CR2032 lifetime (225 mAh) |
| :--- | :--- | :--- | :--- |
| **Connected during a ride (monitoring)** | 62.1 µA (0.062 mA) | 21.64 mA / 3.9 µA | ~154.4 days (continuous) |
| **Low power (standby / sleep)** | 21.0 µA (0.021 mA) | 24.1 µA / 18.8 µA | ~456.2 days (~1.25 years) |
| **Active search (reconnection)** | 1.878 mA (1877.8 µA) | 23.73 mA / 3.7 µA | ~5.1 days (if continuous) |

Specialized's manual specifies a typical battery life of 6 months with rides of 6 to 8 hours per week. Using the empirical PPK2 measurements, we can technically validate this estimate:

- For 7 hours per week connected (pedaling at 62.1 µA):  
  `7 h × 0.0621 mA = 0.435 mAh/week`
- For the remaining time (161 hours per week) in standby (at 21.0 µA):  
  `161 h × 0.0210 mA = 3.381 mAh/week`
- **Estimated total consumption:** ~3.816 mAh/week.

For a CR2032 coin cell with a nominal usable capacity of 225 mAh:  
**Theoretical lifetime** = 225 mAh / 3.816 mAh/week ≈ **58.9 weeks** (~13.5 months)

Considering natural CR2032 self-discharge, voltage drops caused by repeated 21–23 mA BLE radio spikes, outdoor temperature variation, and occasional searches after disconnection, the manufacturer's 6-month estimate is highly realistic and conservative from a hardware-engineering perspective.

---

## Part 4: DFU Firmware Update

The disassembly process of the original sensor was not particularly harmless; the PCB ended up with issues, and opening the enclosure was far from delicate, damaging the sealing gasket (IP67). So I bought a couple of new sensors. It turned out that these sensors came with an older firmware and were overly sensitive to impacts, and with the official app discontinued, there was no way to update them. The initial idea was to try using the firmware extracted with the J-Link and upload it over Bluetooth—this time without damaging the sensor.

### a. Building the Package and First Attempt with Claude

We provided the dump to Claude (Sonnet 5.5 on claude.ai), and in a few iterations we organized the memory map: MBR, SoftDevice S132 v3.0.0, an application of approximately 79 KB, a Nordic Secure DFU bootloader, and a settings page whose CRC matched that of the app. We built a DFU package using [nRF Util](https://www.nordicsemi.com/Products/Development-tools/nRF-Util) and a Python client with bleak that reboots the sensor into bootloader mode, uploads the package, and logs every step. Claude tested it against a simulated bootloader, but real hardware steered the corrections: it looked for the wrong DFU UUID (the sensor uses the experimental Nordic SDK 12.x service, `8E400001`), and the log revealed the correct one.

With that correction, the bootloader received the init packet properly (the CRC matched), but rejected it during validation with a generic error (`INVALID_OBJECT`), which in this bootloader covers signature, version, or format without specifying which.

### b. Inspecting the APK with Antigravity and Finding the Embedded Firmware

That is when we switched tools. With Antigravity (Claude Sonnet 4.6 in Thinking mode and, briefly, Gemini 3.1 Pro in Low), we re-examined the decompiled APK. We found the code the official app uses to assemble the DFU and, even better, the official firmware embedded within its resources: `shs_v2_6_13.zip`, the exact same version 2.6.13 that had appeared in the dump strings. Analyzing previous logs pointed to a firmware version validation check. With that insight, the script was rewritten and the procedure documented in a `README_DFU.md`. The update finally worked: the sensor accepted the package, completed the transfer, and booted with the new version.

A detail that stung to discover: dumping the firmware with the J-Link was completely unnecessary. The binary had been there all along, inside the APK, packaged as just another app resource. We had expected the opposite: that firmware would be downloaded on demand from manufacturer servers as new releases appeared, rather than bundled inside the APK. Under that assumption, it never crossed our minds to inspect the APK's raw resources before cracking open the sensor and wiring up the debugger. Furthermore, the APK's `.zip` contains something our dump lacked: the init packet already signed and prepared by the manufacturer, which is precisely what the bootloader validates. The dump was valuable for understanding how the firmware was laid out, but updating the sensor only required inspecting the APK first.

{% include figure popup=true image_path="/assets/images/sensor_angi/diagrama_flujo_dfu.png" alt="DFU process flowchart" caption="Flowchart of the ANGi sensor firmware DFU update process." class="align-center" %}

### c. Integrating DFU into OpenAngi

With the update already working from the Python script, the next step was bringing it directly into the app. We asked OpenCode (using mimo-v2.6-flash-free, a free-tier model) to review the DFU folder (the README, the `.zip`, the script, and the successful update log) and add an alert and DFU update flow to OpenAngi whenever an outdated firmware is detected. The result was version detection against 2.6.13, a "Firmware (Outdated)" row on the device screen, and a series of dialogs: update required, low battery warning, progress bar, rebooting, and error with a retry button. Under the hood, it uses the Nordic DFU library, with the official embedded `.zip` and the parameters previously used by the original app.

The final test, verified on real hardware, was a sensor (SHS-E6F5) updated from version 2.5.22 to 2.6.13: the command was sent, the sensor confirmed, appeared as `DfuTarg`, the transfer took 47 seconds, and upon reconnection, the app read the new version and reactivated the session. That ~2 h 10 min window ran entirely on a single model without subagents: 213 assistant messages vs. 13 from us, 291 tool calls (125 bash, 61 edits, and 59 reads), ~1.05 million input tokens, 65K output tokens, and 243K reasoning tokens, at zero cost.

<figure class="align-center">
  <video controls width="100%">
    <source src="/assets/images/sensor_angi/dfu_process.mp4" type="video/mp4">
    Your browser does not support playing this video.
  </video>
  <figcaption>DFU update of ANGi sensor SHS-E6F5 from version 2.5.22 to 2.6.13.</figcaption>
</figure>

---

## Part 5: Assessment of AI Tool Usage

Each phase of this project utilized AI tools in a different way. To conclude, it is worth looking at the broader picture and drawing insights into how tasks were distributed among models and what worked best in each context.

### a. What Each Tool Did

| Tool / Model | Primary role in the project |
| :--- | :--- |
| **Claude Sonnet 5** *(claude.ai)* | Initial BLE exploration, Python scripts, chip identification from PCB photos |
| **Claude Sonnet 5.5** *(claude.ai)* | Hex dump analysis, nRF52 memory map, Ghidra Java scripts |
| **OpenCode** (`mimo`, `ling`, `big-pickle`) | Android development (AngiTester and OpenAngi), decompilation and annotation of Ghidra C code |
| **Antigravity** (`Claude Sonnet 4.6 Thinking`) | Reading APK to understand DFU, extracting embedded firmware, rewriting the update script |
| **ChatGPT** (GPT-5.6 Luna) | Legal framework consultation on reverse engineering in Argentina |

No tool replaced real fieldwork: every hypothesis had to be measured or executed directly on the sensor. What models accelerated was triaging raw binaries, writing and correcting tools, and parsing thousands of lines of decompiled Java. But they cannot touch hardware, and some of their interpretations were plain wrong.

### b. Project Scale and Cost

The volume of work processed by the models was considerable. The exact numbers for OpenCode come from its SQLite database; those for Claude and Antigravity are estimates based on session logs:

| Session | Assistant messages | Tool calls | Input tokens (uncached) | Cost |
| :--- | :--- | :--- | :--- | :--- |
| Claude Web (BLE exploration + firmware) | ~32 | ~68 | N/A (Web UI) | Subscription |
| OpenCode (Android apps + C decompilation) | 738 + 85 | 1,067 + 209 | ~4.94 M tokens | $0 |
| Antigravity (DFU) | 213 | 291 | ~1.05 M tokens | $0 |
| **Total** | **~1,070** | **~1,635** | **~6 M tokens** | **Minimal** |

The bulk of development—Android apps, firmware decompilation, DFU updates—ran entirely on free tiers at zero cost.

### c. Observed Specialization and Limitations

The choice of tool was not random; each demonstrated clear advantages depending on the task.

- **Claude** (web interface with vision) was optimal for the exploratory phase: identifying chips from photos, analyzing nRF Connect screenshots, and iteratively constructing scripts without filesystem access.
- **OpenCode** (terminal with bash, file editing, and build capabilities) was the natural choice for building and debugging real applications: compiling with Gradle, fixing errors, and maintaining coherence across more than 30 Kotlin files simultaneously.
- **Antigravity** (agent with filesystem access and Thinking mode) proved effective when tasks required reasoning over complex decompiled Java code and debugging undocumented protocol quirks.

Every tool exhibited hallucinations or errors at some stage—detailed in [section g of Part 3](#g-critical-review-of-the-c-files-and-detected-hallucinations) and in [section f on the fall detection algorithm](#f-fall-detection-algorithm). The guiding rule was consistent across the board: verify every result against real hardware or the original binary before making decisions.

As previously mentioned, this is also one of the lines of research we are incorporating into the Laboratory’s R&D project: closing the loop—granting LLMs access to hardware and the ability to interact with physical tools—thereby eliminating the need for a user to be present during the majority of the development process.

---

## Conclusions and Legal Considerations

The entire process was influenced by the level of protection present on the device and its software. The analyzed APK did not employ obfuscation or encryption that prevented examining its logic, and the communication protocol with the ANGi could be observed and characterized without breaking cryptographic mechanisms. Furthermore, during hardware analysis we found that the nRF52 firmware was accessible under the device's conditions, whereas it technically could have been locked down using the microcontroller's hardware protection mechanisms (APPROTECT). All of this significantly facilitated our work and allowed us to carry out reverse engineering using relatively standard software, communication, and firmware analysis tools. It is important to emphasize that these observations describe the conditions found on the analyzed device and do not imply that the manufacturer deliberately designed the product to facilitate reverse engineering.

### How Legal Is Reverse Engineering?

At this point, another question arose: if the device belongs to us, what are the legal ramifications of reverse engineering it to put it back into service? For this inquiry, we consulted ChatGPT (OpenAI, GPT-5.6 Luna) regarding the legal aspects of the process, particularly within the Argentine legal framework. The general consensus was that there is an important distinction between analyzing software to extract the functional information necessary to interoperate with one's own hardware versus copying, modifying, or redistributing the original proprietary software. In this project, the Ride APK was used strictly as an object of analysis to uncover the communication protocol, but the applications developed subsequently were written entirely from scratch and incorporate no code, resources, or layouts from the original app. Similarly, the device's original firmware is not part of OpenAngi's codebase; it was used solely as analysis material and, subsequently, to experimentally validate the DFU update mechanism. In fact, analyzing the firmware was not even necessary to implement OpenAngi: it served primarily as an opportunity to explore using AI tools in firmware reverse engineering.

This question becomes especially relevant when considering potential publication of the project. One possibility—the one presented on this page—is to publish solely the process and findings of the reverse engineering effort; another is to additionally document the discovered protocol specifications; and a third is to potentially publish the source code or an APK of an independent implementation like OpenAngi. In contrast, publishing the original APK, Ride's decompiled code, or proprietary firmware raises distinct intellectual property concerns, and for that reason, those materials are not part of what is considered for publication. In all cases, this represents a preliminary assessment rather than formal legal advice: if the project were to move toward public or commercial distribution, specific reviews of licenses, intellectual property, and trademarks would be required.

There is an interesting exception within the development itself. During testing, we also implemented the ability to update the ANGi firmware via DFU. To conduct this test, we used the firmware binary file bundled inside the original APK. This allowed us to experimentally confirm the update process and verify that an independent DFU mechanism could be implemented. However, the binary used is not code we developed, but a proprietary component originating from the manufacturer's app. For that reason, we do not consider it appropriate to redistribute it as part of an open application without prior analysis of its rights and licensing terms. The approach we are evaluating is maintaining an experimental build that includes that file solely for internal DFU testing, alongside a separate public version of OpenAngi devoid of proprietary firmware. The latter would be the candidate for any potential public distribution.

[![Hits](https://hits.sh/prototipado.github.io/en/developments/specialized-angi-reverse-engineering/.svg)](https://hits.sh/prototipado.github.io/en/developments/specialized-angi-reverse-engineering/)
