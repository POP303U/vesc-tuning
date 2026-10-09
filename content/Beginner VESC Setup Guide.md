---
title: Beginner VESC Setup Guide
date: 2026-10-08
description: Simple first setup for a VESC. Detection, limits, wheel/motor info, sensors, throttle and a first test, with a common problems table for when something is off.
tags:
  - vesc
  - beginner
  - setup
---

# Beginner VESC setup guide

> "If it works, don't fix it"

If your motor detects correctly, runs smoothly, doesn't cog, switches from halls to sensorless smoothly, stays inside temperature and current limits and reaches the speed you need, leave the advanced settings alone. The [[Advanced VESC Tuning Guide]] is only needed when you got a specific problem or want to push the absolute limits of your setup.

## Rules for the whole guide

- Wheels off the ground for the first setup and the first test.
- Change one setting at a time and test after each change.
- Never raise several limits at once.
- Know your controller's safe motor current, battery current and voltage limits before you start.

A VESC can damage the controller, motor or battery very quickly when current, voltage or control loop settings are wrong.

## Before you start

Check these first:

- Battery voltage, cell count and the controller's voltage rating all match.
- Motor and controller temperature sensors are configured, if you have them.
- Wheel(s) are safely off the ground.

## Something is wrong, where do I look?

| Symptom | Check first | Section |
|---|---|---|
| Motor vibrates, crunches or loses sync after detection | Detection result, wiring, motor direction | [Step 1](#step-1-run-motor-detection) |
| Detection values change between runs | Motor temperature, connectors, wiring | [Step 1](#step-1-run-motor-detection) |
| Speed reading doesn't match real speed | Pole pairs, wheel diameter, gearing | [Step 3](#step-3-set-wheel-and-motor-info) |
| Motor cogs when switching from halls to sensorless | Hall table, motor parameters, temp comp | [Step 4](#step-4-set-up-sensors) |
| Throttle or brake behaves wrong | Neutral, direction, overlap | [Step 5](#step-5-set-up-throttle-and-brakes) |
| Faults or hot parts on first test | Current limits, cooling, load | [Step 6](#step-6-first-test) |
| Rough only at very high speed | Voltage limit or observer | [Advanced guide](#when-to-use-the-advanced-guide) |
| Want more top speed | What actually limits you | [Step 7](#step-7-before-chasing-top-speed) |

# Step 1: run motor detection

Start with a cold motor. Use the normal VESC Tool motor setup flow and select the correct motor type. Leave the advanced *Max Power Loss* override alone unless you already know what it changes.

Detection gives you the motor parameters FOC uses:

- motor resistance
- motor inductance
- flux linkage
- Hall sensor table, if you use halls

## Check the result

1. Spin the motor gently.
2. Make sure the direction is correct.
3. Listen for strong vibration, crunching or repeated loss of sync.
4. Check that the detected values are plausible for the motor.
5. If anything looks off, rerun detection cold.

Values should repeat reasonably well when current, motor temperature and test conditions stay the same. Large changes mean something is wrong with the test conditions, the wiring or connectors, or the motor.

# Step 2: set hardware limits first

Set limits before you try to make anything faster.

## Motor current

This is phase current. It sets low-speed torque and is normally much higher than battery current. Set it to what your motor, controller, connectors and phase wiring can all tolerate.

## Battery current

This is the DC current drawn from the pack. Keep it inside the safe limits of the cells, BMS, battery wiring and connectors.

Motor current and battery current are not the same value.

## Regen

Set battery regen current to what the pack and BMS can safely accept. Don't assume the pack takes the same charge current it can discharge.

## Temperature limits

Turn on controller and motor temperature protection when you have sensors. Limiting on temperature is better than finding the thermal limit by damaging hardware.

# Step 3: set wheel and motor info

Set:

- battery series cell count
- wheel diameter
- motor pole pairs
- gearing, if you have any

Then check that VESC Tool's speed reading is close to real vehicle speed. A badly wrong reading almost always means one of pole pairs, wheel diameter or gearing is wrong.

# Step 4: set up sensors

If the motor has working Hall sensors, use them. A normal Hall setup should start smoothly from standstill, accelerate cleanly at low speed, and move into sensorless operation without a kick or crunch.

Don't lower the Hall to sensorless transition just because a lower number looks nicer. The observer needs enough back-EMF to track the rotor angle.

If the detected setup transitions cleanly, there's no reason to optimise it further.

## If it cogs in the transition

Check these in order:

1. Hall detection and table
2. Motor parameters (rerun detection cold if unsure)
3. Motor temperature compensation
4. Transition ERPM range

Only go to observer tuning if it's still rough after that.

# Step 5: set up throttle and brakes

Set up the input method you actually use: ADC throttle, UART, CAN, PPM or another app/package input.

Check:

- neutral is stable
- full throttle reaches the expected command
- brake direction is correct
- throttle and brake don't overlap unexpectedly
- behaviour on loss of signal is safe

Do all of this at conservative current first.

# Step 6: first test

Start with a low-power test and check:

- smooth startup
- smooth Hall operation
- clean sensorless transition
- no unexpected faults
- sensible motor current and battery current
- sensible motor and controller temperatures

Then raise load gradually. Don't jump from an unloaded test straight to maximum phase current.

## If the motor runs hot

Check phase current, how long the current is held, motor resistance, cooling and mechanical load. If you already enabled field weakening or overmodulation, check those too.

Don't use a higher switching frequency or observer changes as a substitute for correct thermal limits.

# Step 7: before chasing top speed

Work out what is actually limiting the setup first.

- Current limited: duty still has headroom but the motor can't accelerate harder. Voltage tricks won't add low-speed torque. Find the real current or power limit.
- Voltage limited: the controller reaches the motor voltage ceiling at high speed. Only here can overmodulation or field weakening help.
- Observer limited: the motor is rough or loses tracking while electrical headroom remains. Fix position estimation before adding speed.
- Thermal limited: temperature limiting is already cutting current. More aggressive FOC settings won't fix it.

# When to use the advanced guide

Move on to the [[Advanced VESC Tuning Guide]] when you specifically need:

- observer selection and tracking
- saturation compensation
- hall to sensorless transition optimisation
- V0/V7 sampling and switching frequency
- HFI
- MTPA
- overmodulation
- field weakening
- high-current or high-speed diagnostic work

If the setup already works correctly, you don't need any of those.