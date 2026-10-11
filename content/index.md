---
title: VESC Tuning & Tools
date: 2026-10-08
---

# VESC Tuning and Tools

Personal set of notes and tools for my VESC based e-bike or other scooter/VESC builds.

## Pages

### [[Advanced VESC Tuning Guide]]

Observer tuning, saturation compensation, sampling modes, sensorless transition optimization, HFI, MTPA, overmodulation and field weakening.

*Assumes you already know how to configure and test a VESC and or have sufficient motor control knowledge.

### [[Custom VESC QML Dashboard]]

Custom VESC Tool QML dashboard with configurable gauges including values for $I_d$, $I_q$, $V_d$, $V_q$, phase current, line current, weakening current, voltage and modulation/$V_{dq}$. Also has a custom battery gauge much better compared to the stock one.

## Warning

Advanced motor-control settings can create dangerous current spikes, loss of synchronism (bad Pid, Observer, Ki/Kp) or controller failure when configured incorrectly.