---
title: Advanced VESC Tuning Guide
---

# Preface

> At the end of the day, max speed is determined either by a current or a back-EMF wall.

## Important!

+ Don't change values you are not comfortable using or don't know what they do, you can very well blow up your VESC with badly configured control loops.

+ You also only get marginal gains from using all of this, don't go into this expecting a torque boost of 10%, 2-3% more performance + a more efficient/smooth observer is realistic.

+ Only Change one setting at a time, changing multiple settings at once will leave you wondering what setting fucked up what.

# Equations

+ These are needed for certain settings to see validity/value the effectiveness of them. 
+ $p$ stands for pole pairs, the rest of the values are either measurements from VESC or from your specific wheel setup.
+ All calculations use default SI units, hope you paid attention in physics class.

**Speed/Motor Specs**

+ KV from Flux Linkage: $K_V = 60/(\sqrt{3} \cdot 2\pi\,\lambda\,p)$ [rpm/V]
+ No-load Speed from Diameter: $v = 0.1885 \cdot K_V \cdot V_{pack} \cdot D_{wheel}$ [km/h]

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

You don't need to understand every single equation for this guide to make sense (not even all are used) but it helps to understand most of it.

# Motor Detection

We do a detection so we can:

1. Compare the values to the first ever detection to see if anything has changed that would indicate failure (worse Motor R, Flux Linkage decrease)
2. Build Values for this tune using the formulas above

## Preparation

> Only run the detection cold to ensure correct magnetic flux/resistance.

Before running any Motor Detection be sure to go to *Motor Cfg -> FOC -> Sensorless -> Temp Comp* and set it to on if you have a temperature sensor, this will recalculate for the values gained during the detection and sets *Temp Comp Base Comp* which compensates for increased resistance during riding. (Be sure to select the correct temp sensor)

Do this if u feel confident, i wouldn't mess if it if you don't have the experience:

* Set the *Max Power Loss* to 15% of your motor rating (tweak this value based on your setup, lower might be better on motors with less mass) while selecting your motor type by clicking on *Override (Advanced)* and then do that.

Apparently VESC sets the recommended amperage based on: $I = \sqrt{P/(1.5R)}$, so 800 W on 50.3 mΩ gives 103 A, if you use values like 800W on 2kW motors your motor will saturate and show wrong values.

## Running it

+ Always run the detection in the same conditions, then if R, L and lambda don't repeat within ~2% your wiring, the connectors are bad/you performed the detection with the wheel on the ground or heated it up. 

+ Running this test every few weeks and comparing cold values helps you diagnose motor/config issues early and helps prevent damage so don't ignore your motor.

### Parameter test

Here is an example of correctly read parameters on a 52V 3T 2kW e-bike hub motor:

+ kV: $60 / (\sqrt{3} \cdot 2\pi \cdot 0.020439Wb \cdot 23p)$ = **11.7 kV**
+ Unloaded Speed: $0.1885 \cdot 11.7kV \cdot 58.8V \cdot 0.7366m$ = **95.52km/h**

If that unloaded top speed is nowhere near what the motor should do, then lambda, pole pairs or wheel size is wrong, slight deviation is ok from unloaded top speed if it lines up with GPS speed due to multiple motor driving factors. (overmodulation, MTPA set to $Iq$ target or field weakening change unloaded top speed so beware)

## Testing

If everything runs clean without much vibration after setting up your throttle and current limits for your setup you are good to go, however consider using this guide even if you don't have issues with vibrations and cogging.

# Sensorless Interpolation

> VESC automatically transitions into sensorless operation from sensored at 2k-3k ERPM, optimizing the observer is cruicial for all of this to work.

Set *Hall Sensor Interpolation ERPM* to 250, halve or double your *Observer Gain* (This only helps on observers with lambda_comp) and set the *ERPM limit* to be higher than the max ERPM and the tracking should already be much better, if that doesn't fix it, continue on and tune the observer.

## Observer Types

Go to *Motor Cfg -> FOC -> Advanced*, and test the different observers during the transition period. Ortega works better for highly salient motors which is almost never the case for outrunners, but mxlemming works much nicer with motors that have rapid saturation and high pole counts, mxv is the better version of mxlemming moreso.

+ `FOC_OBSERVER_ORTEGA_ORIGINAL`
	+ Original VESC Observer, no adjustable gain and shouldn't be used unless you like hand tuning motor parameters.
- `FOC_OBSERVER_MXLEMMING`
	- New Observer type made by mxlemming, should be tested first.
- `FOC_OBSERVER_ORTEGA_LAMBDA_COMP`
	- Same as Ortega with Flux Linkage compensation, useless for most.
- `FOC_OBSERVER_MXLEMMING_LAMBDA_COMP`
	- This Observer works quite well for a lot of setups and should be taken into consideration, however use MXV for setups that are pushed into magnetic saturation. (aka pushing more amps than intended for the motor type)
- `FOC_OBSERVER_MXV`
	- Same as mxlemming but with advanced math for flux linkage compensation at high amperage.
- `FOC_OBSERVER_MXV_LAMBDA_COMP`
	- This and the \_LIN version of this algo are the best two observers, they use advanced math and algos with separate parameters for gain which makes them amazing for avoiding and fixing motor crunch.
- `FOC_OBSERVER_MXV_LAMBDA_COMP_LIN`
	- This Observer uses a low-pass filter to estimate flux much closer, try using it if you run your motor into heavy saturation, else don't.

## Environment Compensation

All of these changes should've already removed cogging on 95% of setups, if you still experience issues with cogging you might want to check compensation first.

Go to *Motor Cfg -> FOC -> Sensorless* and set *Saturation Compensation Factor* to 15% using *Factor* first, if that doesn't help against vibrations play around with that value for a bit, +-10% is ok, avoid using this if not needed because extra heat could get introduced.

In the same section, check if *Temp Comp Base Temp* has been set during the Motor Detection, if it isn't make sure it is enabled or you have temp sensors, then rerun the detection.

## Optimizing the Transition Window

> If your motor is still cogging or vibrating during transition periods, you might consider switching to halls only or setting the transition window high, instead of following this section, normal setups should still follow through.

Now we work on getting the transition window as low as possible, since halls experience noise from EMI emitted by the phases, it is best to transition to sensorless as low as is possible with the back-EMF presented. 

The observer needs back-EMF above its error terms, you can either manually try guessing them or doing them like this:

* Dead time error, note *Dead Time Compensation* and don't change it, use it's value for some BEMF voltage floor calculations.
* IR drop model error, at current $I$ the term is $I \cdot R_{err}$, so your **resistance error**, not your resistance. With temp comp set right that's a few percent, with a wrong base temp it's 1.6V.

Add the two together and that's your floor:

- Floor = $V_{dc} \cdot t_{dead} \cdot f_{sw} + I \cdot R_{err}$
- 2kW E-bike hub: $58.8 \cdot 0.16\mu S \cdot 34kHz$ = 0.32V, plus 5% R error at 135A = $135 \cdot 0.0503 \cdot 0.05$ = 0.339V, so **0.659V**

Want back-EMF at 5x that, so 3.295V:

* ERPM = $(3.295 / \lambda) \cdot 60 \div (2\pi)$
* 2kW E-bike hub: $3.295 / 0.020439 \cdot 60 \div (2\pi)$ = **1145 ERPM**, so 7 km/h (120 rad/s).

Now you can set your *Sensored Transition ERPM* to 1145 and your *Sensorless Transition ERPM* to 1800 (for this example, yours will differ), if this causes issues bump up the ERPM on both by a few hundred to counteract errors during calculation, this is verified working on my 35H 2kW setup deep into motor saturation.

## HFI and VSS

Just don't use it. This exists so a sensorless motor can make torque at zero speed, you will just get extra noise, new parameters to tune in and more headaches.

(I have managed to get this working on a outrunner setup and will document it here someday)

# MTPA

>  Saliency $= (L_q - L_d) / L$
 > $I_d = [\lambda - \sqrt{\lambda^2 + 8\,\Delta L^2 I_s^2}] / (4\,\Delta L)$

Saliency tells you reluctance torque is possible however it doesn't tell you whether it is big enough to actually use, this ratio shows it better:

+ Reluctance to magnet torque ratio: $(L_q - L_d) \cdot I_s / \lambda$

Well under 1 (near 0.1-0.2) means the magnet term is dominating and MTPA is unnecessary. Near 1 means the two are comparable and MTPA actually brings more performance. (this doesn't apply for the average SPMSM/BLDC e-scooter motor and mostly IPMSM motors)

2kW hub at 135A, using the high-current $L_q-L_d$ of 15.86µH:

+ Ratio: $15.86\mu H \cdot 135A / 0.020439Wb$ = **0.105**
+ $I_d$: $-13.7A$, $I_q$: $134.3A$
+ Torque: 95.19Nm to 95.71Nm, so **0.55%**

For small ratios the gain is roughly ratio$^2$ / 2, so it scales as the square, with half your saliency you quarter the gain. That is why 22.9% saliency gets you nothing here, 23 pole pairs multiplying a healthy magnet term limits the reluctance term.

## Using it Anyways

Why you should (or shouldn't) keep it on even if you don't benefit torque wise:

Per VESC PR #91, $L_q-L_d$ is separate from MTPA and **only affects the observer if MTPA is not used**, and the observer accounts for saliency and tracks better when it's non-zero.

So with MTPA on, $L_q-L_d$ only feeds a path worth half a percent and an inaccurate value changes no observer behavior then, given saliency is current dependent, enabling MTPA sounds like a good idea, although this is extremely setup dependent and you should test if it makes your setup heat up, cog or track worse before considering.

## Iq Target vs Iq Measured

The only difference is that commanded and actual current disagree:
- throttle transients -> milliseconds, irrelevant
- any limiter clipping you -> temp throttling, battery current, absolute max

Don't use $Iq$ Target use $Iq$ Measured, it can introduce unintended amounts of field weakening on a setup where MTPA is already on something that's not meant for it.

# Zero Vector Frequency and Control Sample Mode

> _V0 Only_ runs the controllers and estimators at HALF your switching frequency, _V0 and V7_ runs them at the full frequency and needs phase shunts.

**Run 24 kHz with V0 and V7 if your hardware supports it,** it beats a higher frequency on V0 Only for both of the things that make the observer read signals cleaner.

2kW hub, 58.8V, 0.12µS dead time:

- 34 kHz V0 Only: 0.24V dead time error, 17 kHz control rate
- 24 kHz V0 and V7: 0.17V, 24 kHz control rate

Switching frequency pulls the observer two ways at once, which is why this isn't obvious. Dead time error grows with frequency and lands on your BEMF floor, but control rate is your estimator update rate and more is better, in turn lowering frequency while enabling V0/V7 gives improvements on both.

**If you ever raise switching frequency, disable V0/V7 first,** above ~40 kHz it can hang the CPU and get dangerous, blown mosfets and drivers is a very real possibility, this has personally happened to me running sensorless openloop and HFI at high frequencies even on V0 only.

# Overmodulation

> Linear SVM caps phase voltage at $V_{dc}/\sqrt{3}$, six-step at $2V_{dc}/\pi$

Overmodulation lets the modulator clip the reference vector once it falls outside what the six switching states can reach. The gap between those two limits is only 10.3%, so nothing more than that is available at full six-step, and you only get it if you're actually against the voltage limit.

Different levels of unloaded overmodulation 29" hub 11.7kV on 14S: 
+ 1.00: 95.5 km/h, 
+ 1.15: 105.3km/h

Real increases on this setup:
+ 1.00: 62.1km/h,
+ 1.15: 67.2km/h,

Not a significant increase since this setup is current limited, so it is recommended not to use it on set ups like these if you have temperature issues.

Going above 1.15 is not recommended, as it will just lead to massive torque ripple, iron loss and copper loss without any more noticeable gain, the first few percent of gains are usually pretty free so stay under or at 1.15.

Use:
- 1.0 if you're current limited rather than voltage limited, reason being unneeded harmonic losses
- 1.15 works great, most of the speed for a fraction of the distortion.
- Past 1.2 is diminishing returns, high losses and for extreme setups that can handle this

Deep overmodulation also means applied voltage is no longer equal commanded voltage, which is where issues with the observer can start to arise, this only matters near top speed where back-EMF is large, but if you have tracking issues at top speed and nowhere else, you should check this setting.

# Field Weakening

> Only does something when you're against the voltage limit, below that it costs you $I_q$/torque and does nothing.

Your voltage ceiling isn't just back-EMF. The inverter has to supply three things:

- $V_q = RI_q + \omega_e\lambda$
- $V_d = -\omega_e L_q I_q$
- Modulation depth: $m = \sqrt{V_d^2 + V_q^2} / (V_{dc}/\sqrt{3})$

The $\omega_e L_q I_q$ is the main term, on a 2kW hub at 65 km/h it's 6.9V out of 26.7V total, so 26% of your applied voltage, which will keep growing with speed and current. Field weakening makes it smaller by injecting negative $I_d$, which subtracts from $V_q$ through $\omega_e L_d I_d$.

You don't have to totally understand this, but if your duty cycle never stabilises at 95% or the configured max duty, it means you never reach your true modulation depth and this setting will do nothing for you.

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

## Dangers of Field Weakening

From the firmware comment in `foc_run_fw`: requesting more weakening than the motor can achieve makes the current controller put almost all voltage into $V_d$, and then the $I_q$ controller has no headroom left to overcome the d-axis coupling. $I_q$ falls short, nothing notices, weakening keeps going, $I_q$ falls further.

_FW Backoff_ breaks the loop by feeding $I_q$ error back into the setpoint, scaling the whole ramp down. This is the fix for the runaway complaints you'll find on older firmware.

Nowadays this is not an issue anymore on newer vesc firmware versions like 7.0, as they have the $Iq$ buffer set to 2-5% which fixes all this.

## Cost

Field Weakening doesn't directly cause heat, instead the reallocation of $Iq$ into $Id$ causes the motor to need more q-axis current to overcome drag, which in turn demands more $Iq$ aka heat.

- $I_s^2 = I_d^2 + I_q^2$

At 80A total with 40A of weakening, $I_q$ is 69A, which is 14% of your torque current lost, that is alright near the voltage ceiling where speed was voltage-limited anyway.

Unlike overmodulation, field weakening adds no harmonics. $I_d$ is a clean DC quantity in the rotating frame, no need to distort the voltage waveform.