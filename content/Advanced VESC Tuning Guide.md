---
title: Advanced VESC Tuning Guide
date: 2026-10-08
description: Advanced VESC FOC tuning, observer diagnostics, saturation compensation, sampling, MTPA, overmodulation and field weakening.
tags:
  - vesc
  - foc
  - motor-control
  - advanced
---

# Preface

> At the end of the day, max speed is determined either by a current or a back-EMF wall.

## Important!

+ Don't change values you are not comfortable using or don't know what they do, you can very well blow up your VESC with badly configured control loops.

+ You also only get marginal gains from using all of this, don't go into this expecting a torque boost of 10%, 2-3% more performance + a more efficient/smooth observer is realistic.

+ Only Change one setting at a time, changing multiple settings at once will leave you wondering what setting fucked up what.

## Before tuning, check what's limiting you

Current (A) limited:
+ Duty/modulation still has headroom at maximum load
	+  More voltage won't help torque, nothing helps other than a new battery.

Voltage (V) limited:
* Duty/modulation reaches its ceiling
	+ Overmodulation/Field Weakening can increase speed.

Observer limited:
+ Motor becomes rough despite electrical headroom
	+ Fix tracking/observer model first.

Thermal limited:
+ Current is being reduced by temperature limits
	+ More FOC tuning won't fix it/will make it slightly better.

# Equations

+ These are needed for certain settings to see validity/value the effectiveness of them. 
+ $p$ stands for pole pairs, the rest of the values are either measurements from VESC or from your specific wheel setup.
+ All calculations use default SI units, hope you paid attention in physics class.

**Speed/Motor Specs**

+ KV from Flux Linkage: $K_V = 60/(\sqrt{3} \cdot 2\pi\,\lambda\,p)$ [rpm/V]
+ No-load Speed from Diameter: $v = 0.1885 \cdot K_V \cdot V_{pack} \cdot D_{wheel}$ [km/h]
+ Electrical Speed: $\omega_e = v_{kmh} \cdot p / (3.6\,r)$ [rad/s]

**Important Constants/Formulas**

Observer Tracking/Losses:
+ Back-EMF: $V_{emf} = 2\pi\,(\mathrm{ERPM}/60)\,\lambda$
+ Copper Loss: $P_{cu} = 1.5\,I^{2}R$
+ Observer Gain: $g_{obs} = 0.001/\lambda^{2}$
+ Dead Time Error: $\Delta V = V_{dc}\,t_{dead}\,f_{sw}$
+ BEMF Floor: $V_{floor} = \Delta V + I \cdot R_{err}$
+ ERPM from target BEMF: $ERPM = (5 V_{floor}/\lambda) \cdot 60 / (2\pi)$

MTPA/Current Distribution:
+ Saliency: $s = (L_q - L_d)/L$
+ Reluctance to Magnet Ratio: $\xi = \Delta L \cdot I_s / \lambda$
+ Optimal d-axis Current: $I_d = [\lambda - \sqrt{\lambda^{2} + 8\Delta L^{2}I_s^{2}}]/(4\Delta L)$
+ Resulting q-axis Current: $I_q = \sqrt{I_s^{2} - I_d^{2}}$
+ Torque with MTPA: $T = 1.5\,p\,[\lambda I_q + \Delta L \cdot |I_d| \cdot I_q]$
+ Torque without MTPA: $T = 1.5\,p\,\lambda\,I_s$

Field Weakening/Voltage:
+ q-axis Voltage: $V_q = RI_q + \omega_e\lambda + \omega_e L_d I_d$
+ d-axis Voltage: $V_d = RI_d - \omega_e L_q I_q$
+ Modulation Depth: $m = \sqrt{V_d^{2} + V_q^{2}} / (V_{dc}/\sqrt{3})$
+ Total Current: $I_s^{2} = I_d^{2} + I_q^{2}$

You don't need to understand every single equation for this guide to make sense (not even all are used) but it helps to understand most of it.

# Motor Detection

We do a detection so we can:

1. Compare the values to the first ever detection to see if anything has changed that would indicate failure (worse Motor R, Flux Linkage decrease)
2. Build Values for this tune using the formulas above

## Preparation

> Only run the detection cold to ensure correct magnetic flux/resistance.

Before running any Motor Detection be sure to go to *Motor Cfg -> FOC -> Sensorless -> Temp Comp* and set it to on if you have a temperature sensor. Check afterwards that *Temp Comp Base Temp* matches the temperature your motor actually was during detection, if it doesn't set it yourself. (Be sure to select the correct temp sensor)

Do this if u feel confident, i wouldn't mess with it if you don't have the experience:

* Set the *Max Power Loss* to the max acceptable peak loss during acceleration, tweak this value based on your setup, lower might be better on motors with less mass while selecting your motor type by clicking on *Override (Advanced)* and then do that.

VESC sets the detection current based on: $I = \sqrt{P/(1.5R)}$, so 800 W on 50.3 mΩ gives 103 A, if you use values like 800W on 2kW motors your motor will saturate and you get high-current values instead of normal ones, which is desired on setups that run into a lot.

## Running it

+ Always run the detection at the same current and in the same conditions, then if R, L and lambda don't repeat within ~2% your wiring, the connectors are bad/you performed the detection with the wheel on the ground or heated it up. 

+ Running this test every few weeks and comparing cold values helps you diagnose motor/config issues early and helps prevent damage so don't ignore your motor.

### Parameter test

Here is an example of correctly read parameters on a 52V 4T 2kW e-bike hub motor:

+ kV: $60 / (\sqrt{3} \cdot 2\pi \cdot 0.020439Wb \cdot 23p)$ = **11.7 rpm/V**
+ Unloaded Speed: $0.1885 \cdot 11.7 \cdot 58.8V \cdot 0.7366m$ = **95.52km/h**

If that unloaded top speed is nowhere near what the motor should do, then lambda, pole pairs or wheel size is wrong, slight deviation is ok from unloaded top speed if it lines up with GPS speed due to multiple motor driving factors. (overmodulation, MTPA set to $Iq$ target or field weakening change unloaded top speed so beware)

## Testing

If everything runs clean without much vibration after setting up your throttle and current limits for your setup you are good to go, however consider using this guide even if you don't have issues with vibrations and cogging.

# Sensorless Interpolation

> VESC typically blends into sensorless operation from sensored at 2k-3k ERPM, optimizing the observer is crucial for all of this to work.

Set *Hall Sensor Interpolation ERPM* to 250 from 500 (setting it too low can cause issues with negative speeds), halve your *Observer Gain* (lower is better at low speed, higher tracks faster) and set *Max ERPM* higher than the default, the tracking should already be much better, if that doesn't fix it, continue on and tune the observer.

## Observer Types

Go to *Motor Cfg -> FOC -> Advanced*, and test the different observers during the transition period. Mxlemming works much nicer with motors that have rapid saturation and high pole counts, mxv is the better version of mxlemming (due to better processing of flux).

+ `FOC_OBSERVER_ORTEGA_ORIGINAL`
	+ Original VESC Observer, uses the observer gain and relies heavily on your motor parameters being right, shouldn't be used unless you like hand tuning motor parameters.
- `FOC_OBSERVER_MXLEMMING`
	- New Observer type made by mxlemming, doesn't use observer gain at all.
- `FOC_OBSERVER_ORTEGA_LAMBDA_COMP`
	- Same as Ortega with Flux Linkage compensation, useless for most.
- `FOC_OBSERVER_MXLEMMING_LAMBDA_COMP`
	- This Observer works quite well for a lot of setups and should be taken into consideration, however use MXV for setups that are pushed into magnetic saturation. (aka pushing more amps than intended for the motor type)
- `FOC_OBSERVER_MXV`
	- Same as mxlemming but with the proper circle clamp, no flux linkage compensation though, it assumes your lambda is right and doesn't use observer gain.
- `FOC_OBSERVER_MXV_LAMBDA_COMP`
	- This and the \_LIN version of this algo work the best with my setup, circle clamp plus it estimates your flux linkage live instead of trusting the config, which makes them amazing for avoiding and fixing motor crunch.
- `FOC_OBSERVER_MXV_LAMBDA_COMP_LIN`
	- This Observer uses a low-pass filter to estimate flux much closer, try using it if you run your motor into heavy saturation, else don't.

## Environment Compensation

All of these changes should've already removed cogging on 95% of setups, if you still experience issues with cogging you might want to check compensation first.

Go to *Motor Cfg -> FOC -> Sensorless* and set *Saturation Compensation Factor* to 0-5% using *Factor* first.

Factor thinks both your L and lambda dropped by that percentage times how close you are to max current. So 15% at full current means the observer thinks your magnets just lost 15% of their flux. Compare two detections at different currents to see how much your lambda actually drops, on my motor it was about 3.5% from 44A to 103A, so anything above ~5% overcorrects.

This doesn't add heat by itself, it only changes what the observer thinks, but a wrong value throws the angle off and that will do it.

# Zero Vector Frequency and Control Sample Mode

> _V0 Only:_ runs the controllers and estimators at HALF your switching frequency _V0 and V7: runs them at the full frequency and needs phase shunts.
> V0 and V7: Interpolated uses transforms to estimate vectors, rarely needed

**You should run 24kHz/30kHz with V0 and V7 if your hardware supports it,** it beats a higher frequency on V0 Only for two observer factors.

2kW hub, 58.8V, 0.16µS dead time (100100 default):

- 34 kHz V0 Only: 0.32V dead time error, 17 kHz control rate
- 24 kHz V0 and V7: 0.23V dead time error, 24 kHz control rate

Switching frequency changes the observer two ways at once, dead time error grows with frequency and lands on your BEMF floor, but control rate is your estimator update rate and more is better, however lowering frequency while enabling V0/V7 gives improvements on both.

**If you ever raise switching frequency, disable V0/V7 first,** above ~40 kHz it can hang the CPU and blown fets is a very real possibility, which you don't need to risk since sampling that high isn't that beneficial anymore.

# Optimizing the Sensorless Transition Window

> If your motor is still cogging or vibrating during transition periods, you might consider switching to halls only or setting the transition window high, instead of following this section, normal setups should still follow through.

Now we work on getting the transition window as low as possible, the lower it is the more of your speed range still works if a hall sensor ever dies mid ride, so it's best to transition to sensorless as low as the back-EMF allows.

The observer needs back-EMF above its error terms, you can either manually try guessing them or doing them like this:

* Dead time error, note *Dead Time Compensation* and don't change it, use it's value for the BEMF voltage floor calculation.
* IR drop model error, at current $I$ the term is $I \cdot R_{err}$, so your **resistance error**, not your resistance. With temp comp set right that's a few percent, with a wrong base temp it's 1.6V.

Add the two together and that's your floor:

- Floor = $V_{dc} \cdot t_{dead} \cdot f_{sw} + I \cdot R_{err}$
- 2kW E-bike hub: $58.8 \cdot 0.16\mu S \cdot 34kHz$ = 0.32V, plus 5% R error at 135A = $135 \cdot 0.0503 \cdot 0.05$ = 0.34V, so **0.66V**

Want back-EMF at 5x that, so 3.30V:

* ERPM = $(3.30 / \lambda) \cdot 60 \div (2\pi)$
* 2kW E-bike hub: $3.30 / 0.020439 \cdot 60 \div (2\pi)$ = **1540 ERPM**, so 9.3 km/h (161 rad/s).

5x is a factor i chose since it yields the cleanest transitions with enough noise buffer, lower factors like 4x and 3x would work great too though.

Now set your *Sensored ERPM Start* around that and your *Sensorless ERPM* a few hundred above it (for this example, yours will differ). *Sensored ERPM Start* is the end where it's still fully on halls, *Sensorless ERPM* is where it's fully on the observer.

Then walk both down a few hundred at a time until it stops being smooth, and go back one step. The 5x is conservative, mine ended up at 1100 to 1800 which is below the calculated value, verified working on my 35H 2kW setup deep into motor saturation from 25°C to 80°C.

# HFI

> HFI finds the rotor position from standstill by injecting a high frequency signal, it's a *Sensor Mode*, so selecting it replaces your halls instead of backing them up.

If you have working halls, don't use it. This is for hall-less motors or ones where the halls are dead.

## What you need first

+ *45 Deg V0V7 HFI (Silent)* isn't silent without high side phase shunts, most cheap hardware doesn't have it so you can forget it on those. Else enable *V0 and V7* in *Control Sample Mode*
+ A good L and some $L_q-L_d$. The tooltip says the mode relies on a good inductance measurement on top of some difference between Lq and Ld.
+ *Zero Vector Frequency* at 32 kHz or lower more doesn't run better sometimes worse, the tooltip gives that as the max for completely silent operation.

If it doesn't track well, the tooltip's own advice is to move your L up and down by 1-5%.

## Getting it running

Use *45 Deg V0V7 HFI (Silent)* in *Motor Cfg -> FOC -> General -> Sensor Mode*. Stock settings worked for me, the only thing I had to change was *HFI Start Voltage*.

1. Pull *HFI Start Voltage* up until it finds the rotor every time, mine ended at 30V. Turn the wheel by hand to a different position between starts, if it only fails sometimes it's still too low.
2. Then move *HFI Run Voltage* and *HFI Max Voltage* together by 2V in whichever direction is more stable, mine sit at 4V and 6V.
	- Higher: stronger signal and easier tracking, but more noise and more heat sitting at standstill
	- Lower: quieter, but the signal gets weaker and tracking suffers
3. Set *HFI Ambiguity Resolve Mode* to *Id Double Pulse*.
4. Leave the rest alone unless something specific is wrong.

My other values for reference, mostly untouched:

- *Sensorless ERPM HFI*: 1100
- *HFI Reset ERPM*: 500
- *HFI Samples*: 32
- *HFI Gain*: 0.120
- *HFI Ambiguity Resolve Current*: 10A, *Threshold*: 0%
- *HFI Current Hysteresis*: 5A

*Sensorless ERPM HFI* is where HFI hands over to the observer, so the BEMF floor calculation from the transition section applies to it the same way.

# MTPA

> Saliency $= (L_q - L_d) / L$
> $I_d = [\lambda - \sqrt{\lambda^2 + 8\,\Delta L^2 I_s^2}] / (4\,\Delta L)$

Saliency tells you reluctance torque is possible however it doesn't tell you whether it is big enough to actually use, this ratio shows it better:

+ Reluctance to magnet torque ratio: $(L_q - L_d) \cdot I_s / \lambda$

Well under 1 (near 0.1-0.2) means the magnet term is dominating and MTPA is unnecessary. Near 1 means the two are comparable and MTPA actually brings more performance. Motor type doesn't decide this, the ratio does, some scooter motors have way more saliency than you'd expect.

2kW hub at 135A, using the high-current $L_q-L_d$ of 15.86µH:

+ Ratio: $15.86\mu H \cdot 135A / 0.020439Wb$ = **0.105**
+ $I_d$: $-13.7A$, $I_q$: $134.3A$
+ Torque: 95.19Nm to 95.71Nm, so **0.55%**

For small ratios the gain is roughly ratio$^2$ / 2, so it scales as the square, with half your saliency you quarter the gain. That is why 14.3% saliency gets you nothing here, 23 pole pairs multiplying a healthy magnet term drowns the reluctance term.

Use the $L_q-L_d$ from a detection near the current you're evaluating at, saliency on these motors collapses under load, this one reads 23.10µH at low detection current and 15.86µH at 103A.

## Using it Anyways

The observer always uses $L_q-L_d$ to blend between $L_d$ and $L_q$ depending on your current, no matter if MTPA is on or off (you can see this in `foc_observer_update`). So a wrong $L_q-L_d$ hurts your tracking either way, and turning MTPA off doesn't protect you from it.

MTPA itself is basically free, with $I_q$ Measured it can't hurt, so leaving it on is fine, just don't expect any torque from it on a motor like this.

## Iq Target vs Iq Measured

Both only differ when commanded and actual current disagree:
- throttle transients -> milliseconds, irrelevant
- any limiter clipping you -> temp throttling, battery current, absolute max

In that second case Target calculates $I_d$ for current that isn't actually flowing, which is basically unwanted field weakening at the exact moment your motor is already too hot. Use $I_q$ Measured.

# Overmodulation

> Linear SVM caps phase voltage at $V_{dc}/\sqrt{3}$, six-step at $2V_{dc}/\pi$

The inverter uses SVPWM to fake a sine wave, in turn it only lets 57.7% bus voltage pass through with clean tops, overmodulation flattens the top of the sines and makes it approach six-step, letting the bus voltage rise to 63.7% @ 1.15 overmodulation.

That gap is only 10.3%, so that's all you can ever get out of it, and only if you're actually against the voltage limit. In the code the factor tops out at $2/\sqrt{3} = 1.1547$ which is exactly six-step, so 1.15 is already basically the max and anything above it adds extreme distortion for minimal gain.

Different levels of unloaded overmodulation 29" hub 11.7 rpm/V on 14S: 
+ 1.00: 95.5 km/h
+ 1.15: 105.3km/h

Real increases on this setup:
+ 1.00: 62.1km/h
+ 1.15: 67.2km/h

That's 8% out of the 10.3% possible, so this setup is actually hitting the voltage wall at top speed and overmodulation pays off here.

The catch is that the flattening adds harmonics, which means extra copper loss, iron loss and torque ripple without extra torque. The first few percent are basically free, going all the way square costs a lot of distortion for the last bit of speed.

Use:
- 1.0 if you're current limited rather than voltage limited (duty never gets near max), you'd pay harmonic losses for nothing.
- 1.10 for good gains without distorting too much, good middle ground.
- 1.15 if you hit the voltage wall, most of the speed for a fraction of the distortion.
- Anything above 1.15 does nothing, six-step is the physical limit.

Deep overmodulation also means applied voltage is no longer equal commanded voltage, which is where issues with the observer can start to arise, this only matters near top speed where back-EMF is large, but if you have tracking issues at top speed and nowhere else, you should check this setting.

# Field Weakening

> Only does something when you're against the voltage limit, below that it costs you $I_q$/torque and does nothing.

Your voltage ceiling isn't just back-EMF. The inverter has to supply three things:

- $V_q = RI_q + \omega_e\lambda$
- $V_d = -\omega_e L_q I_q$
- Modulation depth: $m = \sqrt{V_d^2 + V_q^2} / (V_{dc}/\sqrt{3})$

The $\omega_e L_q I_q$ is the main term, on a 2kW hub at 65 km/h it's 6.9V out of 26.7V total, so 26% of your applied voltage, which will keep growing with speed and current. Field weakening makes it smaller by injecting negative $I_d$, which subtracts from $V_q$ through $\omega_e L_d I_d$.

You don't have to totally understand this, but if your duty cycle never gets near your configured max duty, you never reach the voltage wall and this setting will do nothing for you.

## How it actually works

It doesn't work how most people think, it ramps linearly with duty:

- Threshold: _FW Duty Start_ $\times$ _Max Duty_
- Injected current: maps linearly from 0 at the threshold to _FW Current Max_ at _Max Duty_

So _FW Duty Start_ is a fraction of max duty, not an absolute duty. At 0.80 with max duty 0.95 the threshold is 0.76, and at 83% duty you're only 37% of the way up the ramp, so a configured 50A gives you 18A at that amount of duty.

It's also self-limiting, weakening lowers the voltage you need, which lowers duty, which pulls the injection back down. It settles at roughly what's needed rather than running to the configured maximum (**This does not mean to pull the amps up to 100A and pray it regulates itself**).

## Tuning it

- _FW Duty Start_: Set it so the threshold lands just below where you start running out of voltage and hit the Back-EMF Wall.
- _FW Current Max_: the top of the ramp, not what gets applied immediately.
- _FW Ramp Time_: controls how fast it comes on, 150-500ms are sensible depending on how fast u need it to engage.
- _FW Backoff_: **leave this non-zero/stock.** See section below.

To find your _FW Current Max_, raise it 10A at a time and do a top speed run each time, same charge, same road, same tuck. Keep going while top speed goes up, the moment it stops going up or drops, go back one step. That's your number, past it you're giving away more torque than the extra voltage buys you.

## Dangers of Field Weakening

From the firmware comment in `foc_run_fw`: requesting more weakening than the motor can achieve makes the current controller put almost all voltage into $V_d$, and then the $I_q$ controller has no headroom left to overcome the d-axis coupling. $I_q$ falls short, nothing notices, weakening keeps going, $I_q$ falls further.

_FW Backoff_ is supposed to break the loop by feeding $I_q$ error back into the setpoint. Don't rely on it alone though, the way it's written it reacts when $I_q$ is above its target, while the runaway happens when $I_q$ falls short, so im not sure what what happens there.

Also remember $I_d$ is real phase current on top of your torque current, 110A of field weakening plus 100A of torque is 149A through your FETs.

## Cost

Field Weakening doesn't directly cause heat, since $I_d$ takes a share of your existing current instead of adding to it:

- $I_s^2 = I_d^2 + I_q^2$

The heat comes from what it unlocks, more speed means more drag, aerodynamic drag scales with $v^2$, drag power by $v^3$ so you need more $I_q$  at more $-I_d$ to hold it. If it doesn't make you faster, you get torque loss + more heat.

At 80A total with 40A of weakening, $I_q$ is 69A, which is 14% of your torque current lost, that is alright near the voltage ceiling where speed was voltage-limited anyway.

Unlike overmodulation, field weakening adds no harmonics. $I_d$ is a clean DC quantity in the rotating frame, no need to distort the voltage waveform.

# Speed Tracker

If your bike cogs when it hits a profile speed limit but is fine unlimited, look here. At the limit a speed loop takes over, and if *Speed Tracker Position Source* is on *Observer* and you lowered your observer gain, it reads a laggy speed and starts hunting.

Im still not sure why anything here introduces issues with speed limiting but I'll find the issue soon.