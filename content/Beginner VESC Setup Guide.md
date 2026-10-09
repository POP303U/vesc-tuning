---
title: Beginner VESC Setup Guide
date: 2026-10-08
description: A conservative VESC setup workflow for getting a motor running safely and smoothly before touching advanced FOC tuning.
tags:
  - vesc
  - beginner
  - setup
---

# Beginner VESC Setup Guide

> This guide is for getting a normal VESC setup running safely and smoothly. It deliberately avoids most advanced FOC tuning.

If your motor detects correctly, runs smoothly, transitions cleanly from sensors to sensorless operation, stays inside temperature/current limits and reaches the expected speed, **leave the advanced settings alone**.

For observer tuning, saturation compensation, MTPA, switching/sample mode, overmodulation and field weakening, see [[Advanced VESC Tuning Guide]].

## Before you start

- Put the driven wheel(s) safely off the ground for initial setup.
- Make sure the battery voltage, cell count and controller voltage rating are correct.
- Make sure motor and controller temperature sensors are configured correctly if present.
- Know the controller's safe motor-current, battery-current and voltage limits.
- Do not raise several limits at once.
- Change one setting at a time and test after each change.

A VESC can damage the controller, motor or battery very quickly when current, voltage or control-loop settings are wrong.

# 1. Run Motor Detection

Start with a cold motor.

Use the normal VESC Tool motor setup/detection flow and select the correct motor type. If you do not already understand what **Max Power Loss** changes, leave the advanced override alone.

The detection should produce the motor parameters used by FOC, including:

- motor resistance
- motor inductance
- flux linkage
- Hall sensor table if halls are used

## Sanity-check the result

After detection:

1. Spin the motor gently.
2. Make sure direction is correct.
3. Listen for strong vibration, crunching or repeated loss of sync.
4. Check that the detected motor parameters are plausible for the motor.
5. Re-run detection under the same cold conditions if the result looks suspicious.

Detection values should be reasonably repeatable when current, motor temperature and test conditions are kept the same. Large changes are a reason to investigate the test conditions, wiring/connectors or the motor itself.

# 2. Set hardware limits before performance limits

Do this before trying to make the vehicle faster.

## Motor current

Motor current is phase current. It determines low-speed torque and is normally much higher than battery current.

Set this to a value your:

- motor
- controller
- connectors
- phase wiring

can tolerate.

## Battery current

Battery current is the DC current drawn from the pack.

Keep this inside the safe limits of the:

- cells
- BMS
- battery wiring
- connectors

Motor current and battery current are **not the same value**.

## Regeneration

Set battery regen current to a value the pack and BMS can safely accept.

Do not assume the pack can accept the same charge current that it can discharge.

## Temperature limits

Use controller and motor temperature protection when sensors are available.

Temperature limiting is preferable to discovering the thermal limit by damaging hardware.

# 3. Configure wheel and motor information

Set:

- battery series cell count
- wheel diameter
- motor pole pairs
- gearing, if applicable

Then verify that VESC Tool's speed reading is close to the real vehicle speed.

A badly wrong speed reading usually means one of those values is wrong.

# 4. Configure sensors

If the motor has working Hall sensors, use them.

A normal Hall setup should:

- start smoothly from standstill
- accelerate cleanly at low speed
- transition into sensorless operation without a large kick or crunch

Do not lower the Hall/sensorless transition just because a lower number looks better. The observer needs enough back-EMF to track rotor angle reliably.

If the normal detected setup transitions cleanly, there is no reason to aggressively optimize it.

# 5. Configure throttle and brakes

Set up the input method you actually use:

- ADC throttle
- UART
- CAN
- PPM
- other app/package input

Verify:

- neutral is stable
- full throttle reaches the expected command
- brake direction is correct
- throttle and brake do not overlap unexpectedly
- loss-of-signal behavior is safe

Do this at conservative current first.

# 6. First test

Start with a low-power test.

Check:

- smooth startup
- smooth Hall operation
- clean sensorless transition
- no unexpected faults
- sensible motor current
- sensible battery current
- sensible motor/controller temperatures

Then raise load gradually.

Do not jump directly from an unloaded setup test to maximum phase current.

# 7. Before increasing top speed

First work out what is actually limiting the setup.

## Current limited

If duty/modulation still has headroom but the motor cannot accelerate harder, adding more voltage tricks will not create extra low-speed torque.

Find the actual current/power limit first.

## Voltage limited

If the controller reaches the available motor-voltage ceiling at high speed, then advanced options such as overmodulation or field weakening can increase speed.

## Observer limited

If the motor becomes rough, crunches or loses tracking while electrical headroom remains, fix position estimation/model accuracy before adding more speed.

## Thermal limited

If temperature limiting is reducing current, adding more aggressive FOC settings is not the solution.

# Common problems

## Motor cogs during Hall-to-sensorless transition

First verify:

- Hall detection/table
- motor parameters
- motor temperature compensation
- transition ERPM range

Only move into observer tuning if the normal setup is still rough.

## Motor gets rough only at very high speed

This can be a voltage-limit, observer or overmodulation problem.

See [[Advanced VESC Tuning Guide]].

## Motor runs hot

Check:

- phase current
- current duration
- motor resistance
- cooling
- field weakening
- overmodulation
- mechanical load

Do not treat higher switching frequency or observer changes as a substitute for correct thermal limits.

## Speed reading is wrong

Check:

- pole pairs
- wheel diameter
- gearing

# When to use the advanced guide

Move on to [[Advanced VESC Tuning Guide]] when you specifically need to work on:

- observer selection and tracking
- saturation compensation
- Hall-to-sensorless transition optimization
- V0/V7 sampling and switching frequency
- HFI
- MTPA
- overmodulation
- field weakening
- high-current/high-speed diagnostic work

If the setup already works correctly, there is no requirement to use those features.
