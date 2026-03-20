# Forxa 

I've been typing on the same kinds of keyboards for years. Linear switches, tactile switches, clicky switches — they all share one fundamental flaw: they don't know anything about *you*. They don't know if you've been coding for six hours straight. They don't know you're gaming and need sub-millisecond re-actuation. They don't know your wrist is at the wrong angle.

Forxa is my attempt to fix that.

---

## What is Forxa?

Forxa is a fully custom 65% mechanical keyboard built from scratch — PCB, switches, firmware, case, and all. The core innovation is that every single switch uses **Hall Effect magnetic sensing** instead of traditional metal contacts. No click. No contact. Just a magnet moving toward a sensor, measured thousands of times per second.

This means the keyboard knows *exactly* how far down each key is pressed — not just "pressed" or "not pressed" — but the actual depth, in fractions of a millimeter.

Everything else builds on top of that.

---

## Features

### Rapid Trigger + Dead Zone
Forget fixed actuation points. Forxa actuates the moment a key starts moving *down*, and resets the moment it starts moving *up*. The sensitivity (how much movement counts) is configurable per-key down to ~0.1mm. There's also a dead zone at the top of travel so resting your fingers doesn't accidentally fire a keystroke.

If you've ever played CS2 or Valorant and wanted faster direction changes on WASD — this is what that feels like.

### Analog Joystick Mode
WASD aren't buttons on Forxa. They're axes. Press W halfway → half throttle. Press it fully → full throttle. The keyboard presents itself as both a keyboard *and* a gamepad to your OS simultaneously (composite USB HID). 

I built this mainly because I wanted analog throttle/brake in F1 sims without buying a separate controller. It works.

### AI Fatigue Mode
Forxa watches your typing. Dwell time, press depth, inter-key intervals, WPM — it builds a continuous picture of how you're typing. When it detects you're getting tired (slower, shallower presses, longer gaps), it automatically lowers the actuation threshold so each key requires less effort.

The model runs entirely on the STM32 chip — no cloud, no phone, no subscription. It learns from your data, collected over real sessions, trained offline, and flashed directly into firmware.

### Rotary Knob
One physical knob on the top-right. It does whatever you need it to — volume, actuation sensitivity, tilt angle, mode switching. Long-press to change what it controls. It sounds simple. It's genuinely one of my favorite parts of using it.

### 2.5" Live Display
A 240×320 color IPS display sits flush on the top panel. It shows:
- Current WPM (rolling average)
- Active mode (Gaming / Writing / Fatigue detected)
- Tilt angle
- Actuation depth per key (bar graph, toggleable)
- AI fatigue confidence %

No more guessing what mode you're in.

### Smart Tilt — Three Modes

This one took a while to figure out the right UX for.

The keyboard has two SG90 servo motors in the rear feet and an ultrasonic distance sensor at the front edge. Together they control the tilt angle of the keyboard automatically. There are three modes:

- **Calibration Mode** — Run once. The servos sweep through their range while the sensor measures your distance. It finds the angle that matches your posture and saves it to flash memory.
- **Memory Mode** — Default on every boot. Loads your saved profile, sets the angle in ~1 second, then passively monitors in the background. If you shift position significantly, it readjusts.
- **Manual Mode** — The knob directly controls tilt angle in 1° increments. Press to save as new default.

Switch between them with Fn + M or a long-press on the knob.

---

## Hardware

| Component | Part |
|---|---|
| Microcontroller | WeAct STM32F401CCU6 (Blackpill) |
| Hall Effect Sensors | A1302 (one per key) |
| ADC Multiplexer | CD74HC4067 × 5 |
| Magnets | N52 Neodymium 3×2mm disc |
| Display | ST7789 2.5" IPS 240×320 SPI |
| Encoder | EC11 Rotary + aluminum knob |
| Tilt Motors | SG90 Micro Servo × 2 |
| Distance Sensor | HC-SR04 Ultrasonic |
| RGB | WS2812B per-key (70 LEDs) |
| PCB | Custom KiCad design, JLCPCB fab |
| Firmware | QMK + Vial + custom HE layer |
| Switch stems | 3D printed POM |
| Switch housing | 3D printed PA12 Nylon |
| Case | CNC aluminum / acrylic hybrid |
| Connectivity | USB-C wired |

---

## How the Switches Work

Standard mechanical switches register a keypress when two metal contacts touch. That's it — binary, dumb, and wear-prone.

Forxa switches have no contacts at all. Each stem has a small N52 neodymium magnet press-fit into its base. Beneath each switch, soldered directly to the PCB, is a linear Hall Effect sensor (A1302). When you press a key, the magnet moves closer to the sensor. The sensor's output voltage rises proportionally to the magnetic field strength. The STM32's 12-bit ADC reads that voltage ~4000 times per second per key (via analog multiplexing) and converts it to a precise position value between 0 and 4095.

That number is what every feature in Forxa runs on.

The tactile bump is mechanical — a small ridge on the stem wall brushes a leaf spring inside the housing. Completely independent from the sensing. So you can have the bump without it affecting actuation, or tune it away entirely.

---

## Firmware Architecture

```
STM32F401 (Main)
├── ADC scan loop (DMA-based, all 65 keys)
│ ├── MUX select → ADC read → store in key_state[]
│ └── Runs on background DMA, zero CPU blocking
├── Rapid Trigger engine
│ ├── Per-key delta tracking
│ └── Dead zone filter at top of travel
├── Joystick mode (composite USB HID)
│ └── Maps WASD ADC values → axis_report[]
├── AI Fatigue engine
│ ├── Rolling 500-keystroke buffer
│ ├── Feature extraction (dwell, depth, WPM)
│ └── TFLite Micro inference every 60s
├── Tilt controller
│ ├── HC-SR04 distance polling (every 5s)
│ ├── PID servo control loop
│ └── 3-mode state machine (CALIBRATION / MEMORY / MANUAL)
├── Display driver (ST7789 SPI via LVGL)
└── Encoder handler (QMK native)
```

---

## AI Model Pipeline

```
Forxa keyboard (USB serial)
→ typing_sessions.csv (Python logger)
→ feature extraction (dwell, depth, WPM, error rate)
→ TensorFlow / scikit-learn (offline training)
→ TFLite Micro conversion (.h model file)
→ flashed to STM32 with QMK firmware
→ live inference, fatigue % on display
```


The model is a lightweight 1D CNN trained on labeled fatigue sessions. "Fatigue" is defined as: dwell time >1.5× baseline, press depth <60% baseline, WPM drop >25% over 5-minute window. You collect your own data, label it, train it. The model is yours — trained on you, for you.

---

## Build Log

> I'll be documenting every stage here as the build progresses — PCB spins, firmware milestones, mechanical failures, the works.

- [ ] Switch prototype v1 (single key breadboard test)
- [ ] PCB design complete (KiCad)
- [ ] PCB ordered (JLCPCB)
- [ ] PCB assembled
- [ ] QMK base firmware (65-key ADC scan working)
- [ ] Rapid Trigger + Dead Zone
- [ ] Joystick Mode
- [ ] Display + Knob UI
- [ ] Tilt system (all 3 modes)
- [ ] AI fatigue model trained + flashed
- [ ] Case built
- [ ] Full typing test

---

## Why "Forxa"?

Forxa comes from *force* and *flux* — the two physical phenomena at the heart of every keypress. Magnetic flux is what the sensor measures. Force is what your finger applies. The name sits at the intersection of the two things that make this keyboard possible.

It also sounds like a supercar. That wasn't unintentional.

---

## Project Status

**Active build — design phase**

This is being built as part of [Hack Club Stasis](https://stasis.hackclub.com) — a hardware grant program for teenage builders. If you're a teenager building something ambitious in hardware, check it out.

---

## Made by

[Swamstick](https://github.com/Swamstick911) — 9th grade, Kanpur, India.
Building things before I fully understand them, then figuring it out along the way.

---

*If you're building something similar or want to talk Hall Effect keyboards, open an issue or find me on the Hack Club Slack.*
