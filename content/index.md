---
title: VESC Tuning & Tools
date: 2026-10-08
description: VESC setup, advanced FOC tuning and custom diagnostic tools.
---

# VESC Tuning & Tools

A set of notes and tools for VESC-based e-bike, scooter and custom EV setups.

## Guides

### [[Beginner VESC Setup Guide]]

A conservative setup flow for getting a motor detected, setting safe limits, configuring sensors and validating a normal setup without touching advanced FOC controls.

Start here if you mainly want the vehicle to run correctly.

### [[Advanced VESC Tuning Guide]]

Observer tuning, saturation compensation, sampling modes, sensorless transition optimization, HFI, MTPA, overmodulation and field weakening.

This assumes you already know how to configure and safely test a VESC.

### [[Custom VESC QML Dashboard]]

Custom VESC Tool QML dashboard with configurable gauge presets and an FOC diagnostic view for $I_d$, $I_q$, $V_d$, $V_q$, phase current, line current and modulation.

## Safety

Advanced motor-control settings can create destructive current spikes, loss of synchronism or controller failure when configured incorrectly.

Change one parameter at a time, start with conservative current limits and keep a known-good configuration backup.
