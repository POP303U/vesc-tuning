---
title: Custom VESC QML Dashboard
date: 2026-10-08
description: Custom VESC Tool QML dashboard with configurable gauges, presets and an FOC diagnostic view.
tags:
  - vesc
  - qml
  - dashboard
  - diagnostics
---

# Custom VESC QML Dashboard

Repository: [POP303U/vesc-dashboard](https://github.com/POP303U/vesc-dashboard)

This is a custom QML dashboard for VESC Tool built around reusable gauge definitions and presets instead of a fixed telemetry layout.

The goal is to have a normal riding dashboard and a useful FOC diagnostic dashboard in the same UI.

# Gauge presets

The dashboard uses a gauge catalog plus named presets.

Typical layouts include:

- **Full** — speed, phase current, line current, field weakening, duty and temperatures
- **Ride** — speed, power, range, battery, line current and temperatures
- **Tuning** — current, field weakening, duty, power and controller temperature
- **Efficiency** — consumption, speed, power, range, battery and voltage
- **FOC** — modulation/voltage-vector view with raw d/q telemetry

The FOC preset is intended for diagnosing motor-control behavior rather than normal riding.

# FOC diagnostic preset

The main gauge is modulation/voltage-vector utilization.

The six smaller gauges are:

- $I_d$
- $I_q$
- $V_d$
- $V_q$
- phase current
- input/line current

This makes it possible to watch current-vector behavior, voltage allocation and DC-side current at the same time.

## d/q voltage magnitude

The dashboard can calculate:

$$
V_{dq} = \sqrt{V_d^2 + V_q^2}
$$

A useful physical normalization is the linear-SVM voltage limit:

$$
m_{lin} = \frac{V_{dq}}{V_{dc}/\sqrt{3}}
$$

With this definition:

- $m_{lin}=1.0$ is the linear-SVM boundary.
- Overmodulation operates beyond that boundary.
- Ideal six-step provides about $2\sqrt{3}/\pi \approx 1.103$ times the linear-SVM fundamental-voltage limit.

This is **not the same normalization as VESC's `foc_overmod_factor`**. A VESC overmodulation factor such as `1.15` should not be read as 15% more fundamental motor voltage.

## Current vector

The d/q current-vector magnitude is:

$$
I_s = \sqrt{I_d^2 + I_q^2}
$$

The dashboard keeps VESC's normal phase-current telemetry as its phase-current gauge rather than silently replacing it with a derived value.

# What the FOC view can show

The diagnostic view is useful for spotting patterns such as:

## Hall/sensorless handoff

If the electrical-angle reference changes at the transition, $I_d$ and $I_q$ can shift even when total phase-current magnitude changes much less.

A large d/q disturbance exactly at the handoff points toward an angle/model problem.

## Field weakening

Negative $I_d$ can be watched directly instead of displaying only a positive "field weakening amps" value.

The combination of $V_d$, $V_q$ and modulation can show how much voltage-vector headroom remains while field weakening is active.

## Overmodulation

The dashboard can show whether rough behavior begins specifically when the voltage-vector demand reaches the linear modulation boundary.

## Current/voltage-loop instability

If $V_d/V_q$ move strongly before $I_d/I_q$, the controller output may be driving the disturbance.

If d/q current changes first, the controller may instead be reacting to an observer, sensing or load disturbance.

# Important telemetry limitation

The normal VESC `COMM_GET_VALUES` path is useful for a dashboard, but it is **not a high-speed oscilloscope**.

Several motor/FOC quantities returned through the normal values stream are averaged, and a typical dashboard refresh is much slower than the FOC control loop.

That means fast oscillations can be hidden or aliased.

Use the dashboard to:

- identify repeatable operating regions
- see slow envelopes/trends
- correlate behavior with speed, duty and modulation
- decide what deserves a higher-rate recording

Do not assume a visually steady 10 Hz gauge proves that the underlying FOC variable is steady at control-loop timescales.

# Custom gauge architecture

The QML UI is built around:

- `gaugeIds` / `gaugeNames`
- preset objects
- live telemetry properties
- `gDef()` gauge definitions
- the normal `valuesReceived` telemetry handler

That keeps new gauges from requiring a separate page implementation.

For example, the FOC preset can be represented conceptually as:

```qml
{
    name: "FOC",
    main: "mod",
    slots: ["id", "iq", "vd", "vq", "phase", "line"]
}
```

Signed FOC quantities should use symmetric scales around zero:

- $I_d$: `-Imax ... +Imax`
- $I_q$: `-Imax ... +Imax`
- $V_d$: negative to positive voltage scale
- $V_q$: negative to positive voltage scale

The normal user-facing field-weakening gauge can still display `-Id` as a positive number; the diagnostic `Id` gauge should retain the real sign.

# Related guide

For the motor-control concepts behind these gauges, see [[Advanced VESC Tuning Guide]].
