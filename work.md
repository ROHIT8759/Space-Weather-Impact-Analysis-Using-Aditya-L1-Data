Project 2 — Space Weather Impact Analysis Using Aditya-L1 Data
From the solar wind at L1 to the geomagnetic storm at Earth
Hands-on session brief · Aditya-L1 MAG + ASPEX-SWIS · OMNI SYM-H / AE · 10–11 October 2024
The same CME that Project 1 dissects drove one of the strongest geomagnetic storms of solar cycle 25. Aditya-L1 measured that driver 1.5 million km upstream of Earth. Your job is to work out how much of the storm at Earth you can determine from that upstream measurement alone.
The problem
Space-weather forecasting rests on one idea: a spacecraft parked upstream of Earth sees the solar wind before it arrives, so it can warn us. Aditya-L1 is that spacecraft for India.
But an upstream measurement is not a forecast until you answer three questions:
•	When does what L1 saw actually reach Earth?
•	Which part of the solar wind does the magnetosphere care about?
•	How much of the storm on the ground can that upstream measurement actually explain?
Your question for the session: Given only what Aditya-L1 measured, could you have predicted the storm that followed — and how close would you have got?
Why the timing is the hard part
•	Aditya-L1 sits about 259 Earth radii sunward of Earth. The plasma it measures takes tens of minutes to cross that gap.
•	The delay is not a constant. It depends on the solar-wind speed — and during this event the speed more than doubles.
•	Assume a fixed 40 minutes and you smear the whole event by tens of minutes, which destroys any attempt to line up cause with effect.
•	So the delay has to be computed minute by minute, from the speed the spacecraft actually measured.
The rule for this project
Everything we analyse comes from Aditya-L1. The OMNI indices (SYM-H, AE) appear in exactly one role: as the ground truth against which an Aditya-L1-based prediction is scored. No OMNI solar-wind quantity is used as an input anywhere.
This is a real constraint, and it shapes the notebook in two visible ways:
•	the storm onset is determined from Aditya’s own dynamic-pressure jump, not from the ground index;
• the ring-current model is started from a neutral value rather than from the observed pre-storm level, so that no ground data leaks into the prediction.
If you delete the OMNI cell, every analysis still runs. You simply lose the ability to mark your own work.
The physics, in plain words
The chain from the solar wind to a compass needle has four links.
1. Southward field opens the magnetosphere. When the interplanetary field points south (Bz < 0 in GSM), it meets Earth’s northward dayside field head-on and reconnects with it. The magnetosphere stops being a closed bubble.
2. The energy input has a number. The dawn–dusk merging electric field measures how fast that opening happens. Large positive Ey means strong driving.
3. A ring current builds up. Energised ions drift westward around the Earth. Their current produces a magnetic field that opposes the dipole at the surface.
4. The ground measures the depression. That is SYM-H: the globally averaged low-latitude magnetic disturbance, at 1-minute resolution. The deeper it goes, the bigger the storm.
The merging electric field is:
• Get this sign right or everything downstream inverts. SWIS measures Vx < 0 (the wind blows anti-sunward), so the minus sign makes Ey positive when the field points south — that is, positive when the magnetosphere is being driven.
• A sign error here is silent: the code still runs, the plot still looks plausible, and every correlation comes out backwards.
The one complication worth knowing
• Dynamic pressure does something different: it squeezes the magnetosphere rather than energising it, and a squeeze raises SYM-H.
• So the arrival of the shock produces a sharp positive spike before the storm proper — the storm sudden commencement.
• Any honest model of SYM-H therefore needs two terms: a ring-current term driven by Ey, and a compression term driven by the square root of the dynamic pressure.
Storm classification by minimum SYM-H
Minimum SYM-H
Class
Below −50 nT
Moderate
Below −100 nT
Intense
Below −250 nT
Super
The data you will use
Source
What it gives
Cadence
Role
Aditya-L1 MAG
Field vector in GSE and GSM
10 s
The driver
Aditya-L1 ASPEX-SWIS
Proton density, speed, velocity components, thermal speed
5 s
The driver
OMNI SYM-H / AE
Ring-current depression and auroral activity on the ground
1 min
The answer sheet only
• Event window: 10–11 October 2024, the same interval Project 1 works on.
• Sections 1–8 are identical to Project 1’s — same code, same figures. Compare notes with that group before the analyses diverge; you are both starting from the same reduced data.
• Important distinction: Aditya-L1 data are a measurement. The propagation to Earth is a model. Keep the two separate in your head, and say which is which in your report.
What we will do, step by step
Stage 1 — Get the data in and check it (Sections 1–8)
• Identical to Project 1: read MAG and SWIS, quality-control both, drop flagged samples.
• Build the common 1-minute table and the derived quantities — including Ey, the quantity this project lives on.
• Produce the seven-panel Aditya-L1 overview and read it.
Stage 2 — Carry the solar wind from L1 to Earth (Part A, Sections 9–11)
• Work out where the bow shock actually is, using two standard models.
• Compute the delay minute by minute from the measured speed and the spacecraft’s measured position.
• Compare it against a naive fixed-distance delay, and see which part of the calculation actually matters.
• Apply the shift: relabel each measurement with the time it reaches Earth, sort (fast plasma overtakes slow), re-bin, and bridge the holes the shift itself creates.
• Plot before and after, and check by eye that the displacement matches the delay you computed.
• Determine the storm onset from Aditya alone — the sharpest rise in dynamic pressure — and establish the quiet reference levels.
The delay — flat-plane convection: treat each structure as a plane perpendicular to the Sun–Earth line, carried anti-sunward at the measured speed.
Where the magnetopause sits — Shue et al. (1998). It moves with both pressure and Bz, and the result is in Earth radii:
Where the bow shock sits — Farris & Russell (1994). The shock stands off the magnetopause by an amount set by the magnetosonic Mach number:
The Mach number it needs — magnetosonic, so both the Alfvén and the sound speed enter:
• Notice the singularity. As M_ms approaches 1 the bow-shock formula diverges, and inside the magnetic cloud the flow really is barely super-magnetosonic. The notebook clips the result and substitutes a median there.
• That clipping is a modelling choice, not a measurement. Say so in your report.
Stage 3 — Put driver and response on one axis (Part B, Sections 12–13)
• Fetch SYM-H and AE from OMNI and put them on exactly the same 1-minute grid.
• Build the central figure of the project: shifted solar wind above, ground response below, with the onset marked at the time Aditya gave you.
• Read cause and effect off it directly, and write down four things with times: the pressure jump, the southward turning, the lag between deepest Bz and deepest SYM-H, and where SYM-H turns around.
Stage 4 — Predict SYM-H from Aditya-L1 alone (Part C, Section 14)
• Integrate the Burton (1975) / O’Brien & McPherron (2000) equation, which treats the ring current as a leaky bucket: filled by Ey, draining with a field-dependent time constant, plus the pressure term for the compression.
• Drive it with nothing but the shifted Aditya-L1 data. No ground data enters the calculation.
• Compare against the observed SYM-H and quantify: correlation, RMS error, timing error at the minimum.
• Diagnose the mismatch — too shallow, too deep, right depth but wrong time, or wrong recovery rate. Each failure mode points at a different part of the chain.
The leaky bucket. The ring current is filled by the merging electric field and drains on a timescale that itself depends on the driving:
The injection term — zero until the driving passes a threshold, then linear in Ey:
The decay time, in hours — stronger driving makes the ring current leak faster:
The pressure correction, which turns the ring-current term into the index actually measured on the ground:
• The first three equations are the ring current; the last adds the magnetopause current, which is compression rather than energy input. That term is what produces the sudden commencement.
• Coefficients are from O’Brien & McPherron (2000), fitted to a statistical ensemble of storms — not to this one. Where the model disagrees with the observation, that fit is the first suspect.
What you should come out with
A propagation result:
• The delay from Aditya-L1 to Earth, and how much it varied across the event. You should find it roughly halves as the shock passes — which is exactly why a constant delay will not do.
• The fraction of minutes whose arrival order is reversed by the shift — fast plasma overtaking slow. A physical effect, not a bug.
• A storm onset time derived from Aditya data alone, and the warning it would have given.
What to hand in
1. The driver–response figure, with the storm onset marked at the time Aditya-L1 gave you, and the four readings you took from it.
2. The prediction figure with the correlation, the RMS error and the timing error, plus a paragraph on where and why the model fails.
3. A short discussion of the propagation delay: how much it varied, and how much it would have mattered to assume a constant 40 minutes.
References
• Burton, R. K., McPherron, R. L. & Russell, C. T. (1975), An empirical relationship between interplanetary conditions and Dst, JGR 80, 4204.
• O’Brien, T. P. & McPherron, R. L. (2000), An empirical phase space analysis of ring current dynamics, JGR 105, 7707.
• Shue, J.-H. et al. (1998), Magnetopause location under extreme solar wind conditions, JGR 103, 17691.
• Farris, M. H. & Russell, C. T. (1994), Determining the standoff distance of the bow shock, JGR 99, 17681.
• Gonzalez, W. D. et al. (1994), What is a geomagnetic storm?, JGR 99, 5771.
• OMNI data and documentation: [https://omniweb.gsfc.nasa.gov/](https://omniweb.gsfc.nasa.gov/)

