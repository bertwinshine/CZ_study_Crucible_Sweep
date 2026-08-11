
# CZ melt — crucible rotation study

Reference CFD cases for the Czochralski silicon melt, varying the crucible
counter-rotation. Twelve steady 2D axisymmetric swirl cases from −1 to −10 rpm,
everything else held fixed. Meant as ground-truth data for the surrogate/PINN
model.

Each case is a full converged field on a common mesh, exported as
`r, z, u_r, u_z, u_swirl, p, T` at 8181 nodes. Every case is checked against the
boundary conditions it should have been run with before it enters the dataset —
`certify_case.py` does that and all twelve pass.

## What varies

Only the crucible rotation. The crystal stays at its baseline +8 rpm and the
crucible wall stays at 1745 K.

Because the free surface sits between the crystal and the crucible wall, its
prescribed swirl has to follow the crucible speed — it ramps linearly from the
crystal edge value to whatever the crucible wall is turning at. So each case
changes two coupled things: the crucible wall condition and the outer end of the
free-surface ramp.

| rpm | ω (rad/s) | free-surface swirl at r = 0.30 |
|---:|---:|---:|
| −1 | −0.104720 | −0.031416 |
| −2 | −0.209440 | −0.062832 |
| −3 | −0.314159 | −0.094248 |
| −3.5 | −0.366519 | −0.109956 |
| −4 | −0.418879 | −0.125664 |
| −4.5 | −0.471239 | −0.141372 |
| −5 | −0.523599 | −0.157080 |
| −5.5 | −0.575959 | −0.172788 |
| −6 | −0.628319 | −0.188496 |
| −7 | −0.733038 | −0.219911 |
| −8 | −0.837758 | −0.251327 |
| −10 | −1.047198 | −0.314159 |

Free-surface swirl expression, with the last value taken from the table:

```
0.150796 [m/s] + ((RadialCoordinate - 0.18 [m]) / (0.30 [m] - 0.18 [m]))
                 * (OUTER [m/s] - 0.150796 [m/s])
```

## Case setup

Geometry: crucible radius 0.30 m, crystal radius 0.18 m, melt height 0.15 m.

| Zone | Thermal | Momentum |
|---|---|---|
| Crystal face | 1685 K | rotational, 0.837758 rad/s (+8 rpm) |
| Crucible wall | 1745 K | rotational, ω per case |
| Crucible bottom | `1700 + (r/0.30)·(1745−1700)` K | rotational, ω per case |
| Free surface | `1685 + ((r−0.18)/0.12)·(1745−1685)` K | swirl ramp, see above |
| Axis | — | axis |

Solver: 2D axisymmetric swirl, laminar, energy on, Boussinesq with operating
temperature 1717 K, gravity in x = −9.81, coupled pressure–velocity, second-order
discretisation, automatic pseudo time step. ANSYS Fluent 2026 R1 Student.

## What the sweep shows

There is a **minimum in mean velocity near −4 rpm**.

| crucible rpm | mean \|u\| (m/s) | max \|u\| (m/s) | p range (Pa) |
|---:|---:|---:|---:|
| −1 | 0.003128 | 0.012232 | 12.98 |
| −2 | 0.002880 | 0.010987 | 13.44 |
| −3 | 0.002598 | 0.009498 | 14.94 |
| −3.5 | 0.002495 | 0.008499 | 16.08 |
| **−4** | **0.002457** | **0.008462** | 17.51 |
| −4.5 | 0.002507 | 0.009106 | 19.28 |
| −5 | 0.002641 | 0.010312 | 21.37 |
| −5.5 | 0.002841 | 0.011638 | 23.97 |
| −6 | 0.003112 | 0.013081 | 26.97 |
| −7 | 0.003847 | 0.016159 | 34.28 |
| −8 | 0.004775 | 0.019363 | 42.60 |
| −10 | 0.006683 | 0.025744 | 63.02 |

Velocity falls as counter-rotation increases to −4 rpm, then rises. The −3.5 and
−4.5 cases were run specifically to test whether that was real or a single-point
artefact — both came out above −4, and a parabola through the three central
points puts the vertex at −3.96 rpm with positive curvature. The maximum velocity
agrees independently, with its own vertex at −3.78 rpm.

The pressure range rises monotonically across the whole sweep, so the minimum is
in the flow structure rather than in the driving force. The reading is that
increasing counter-rotation first opposes and partially cancels the
buoyancy-driven cell, and past about −4 rpm starts to drive the flow itself.

It is a smooth minimum, not a discontinuity. The cross-case correlation of `u_r`
between adjacent cases stays above 0.95 through the middle of the sweep. But the
two ends really are different flow structures: `u_r` at −1 rpm correlates at
−0.243 with −10 rpm. The change between them is gradual, not abrupt.

## Certification

`certify_case.py` reads the rpm from the filename, re-derives every boundary
condition from it, and checks the exported field against them, so a case whose
setup does not match its label fails rather than entering the dataset unnoticed.

```
python certify_case.py steady/*.csv
```

All twelve pass. Typical margins:

| check | typical |
|---|---|
| crucible wall swirl | < 1e-6 % of target |
| crucible bottom swirl | < 1e-6 % of target |
| free-surface swirl ramp | max err ~1.5e-4 m/s |
| crystal face swirl | 0.06 % of edge speed |
| free-surface / bottom thermal ramps | < 0.03 K |
| axis symmetry, max \|u_r\| on axis | 0.11 – 0.24 % of field max |

Two things the checks are deliberately tolerant about, both understood:

- **The corner node at r = 0.18** is shared between the crystal face and the free
  surface, and the solver blends the two conditions there. It reads 0.147044
  instead of 0.150796 for swirl and 1685.55 instead of 1685 for temperature. It
  is excluded from the ramp checks rather than being treated as an error.
- **The −3 rpm case** was set up with ω rounded to −0.3141 rather than
  −0.3141593, so its wall swirl is 0.05 % off the exact value. The velocity
  checks are relative and print the percentage, so this shows in the output
  rather than being hidden. It is well inside anything that matters physically,
  but it is a real record of how that case was configured.

The axis-symmetry check is worth singling out. On the symmetry axis the radial
velocity has to vanish — that is a consequence of axisymmetry, not a numerical
tolerance — and the same check catches a `u_r`/`u_z` column transposition at
export time, which is otherwise invisible field by field.

## Reduced-order model

The sweep is close to low-rank: one POD mode captures 99.96 % of the swirl energy
and 99.6 % of the pressure energy; `u_r` and `u_z` need two modes for 99 %.

A POD + RBF surrogate over the twelve cases, validated leave-one-out, gives (mean
over interior cases, as % of each field's range):

| u_r | u_z | u_swirl | p | T |
|---|---|---|---|---|
| 0.225 % | 0.267 % | 0.043 % | 0.469 % | 0.245 % |

Holding out an endpoint instead is extrapolation and behaves accordingly — −1 rpm
gives 12.5 % on pressure, −10 rpm gives 6.0 %. **The model interpolates inside
−1 to −10 rpm and should not be used outside it.**

## Scope and limits

- Laminar, Boussinesq, steady, prescribed free surface. No turbulence model, no
  radiation, no Marangoni convection, no species transport, no magnetic field.
- The free surface is a prescribed wall, not a solved meniscus.
- One parameter varies. Crucible rotation is swept; hot-wall temperature and
  crystal rotation are held at 1745 K and +8 rpm. Nothing here says anything
  about how those parameters interact — that needs cases where more than one
  moves at a time.
- Student-licence mesh, 8181 nodes.

This is a benchmark dataset for method development, not an industrially
representative model of a real puller.

## Files

```
steady/            12 certified cases, r,z,u_r,u_z,u_swirl,p,T
certify_case.py    boundary-condition and sanity checks
```

The ANSYS modelling, the sweep design and the validation are mine. The Python was
written with AI assistance — I understand what it does and can walk through it.
