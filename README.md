# CZ melt: crucible rotation sweep

[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.21955323.svg)](https://doi.org/10.5281/zenodo.21955323)

Twelve steady CFD cases of a Czochralski (CZ) silicon melt, with the crucible
counter-rotation varied from −1 to −10 rpm and everything else held fixed. I
made them as reference data for a surrogate / PINN model.

Each case is a converged 2D axisymmetric swirl solution exported on the same
8181-node mesh with the columns `r, z, u_r, u_z, u_swirl, p, T`.

The −3 rpm case is the shared anchor of the three sweeps. It is the same field
as the 8 rpm case in [CZ_Crystal_Sweep](https://github.com/bertwinshine/CZ_Crystal_Sweep)
and the 1745 K case in [CZ_Study_TempChange](https://github.com/bertwinshine/CZ_Study_TempChange).

## Setup

Geometry and boundary conditions follow Huang et al., AIP Advances 15, 105202
(2025), https://doi.org/10.1063/5.0271778.

Crucible radius 0.30 m, crystal radius 0.18 m, melt height 0.15 m.

| Zone | Thermal | Momentum |
|---|---|---|
| Crystal face | 1685 K | rotational, 0.837758 rad/s (+8 rpm) |
| Crucible wall | 1745 K | rotational, ω per case |
| Crucible bottom | `1700 + (r/0.30)·(1745−1700)` K | rotational, ω per case |
| Free surface | `1685 + ((r−0.18)/0.12)·(1745−1685)` K | swirl ramp, see below |
| Axis | | axis |

Solver: ANSYS Fluent 2026 R1 Student, 2D axisymmetric swirl, laminar, energy on,
Boussinesq (operating temperature 1717 K), gravity −9.81 in x, coupled
pressure-velocity, second-order discretisation, automatic pseudo time step.

The free-surface swirl is a linear ramp from the crystal edge speed to the
crucible wall speed. In this sweep the outer end follows the crucible, so each
case changes both the crucible wall and the ramp:

```
0.150796 [m/s] + ((RadialCoordinate - 0.18 [m]) / (0.30 [m] - 0.18 [m]))
                 * (OUTER [m/s] - 0.150796 [m/s])
```

| rpm | ω (rad/s) | OUTER, swirl at r = 0.30 (m/s) |
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

## Results

Mean velocity has a minimum near −4 rpm:

| crucible rpm | mean \|u\| (m/s) | max \|u\| (m/s) | p range (Pa) |
|---:|---:|---:|---:|
| −1 | 0.003128 | 0.012232 | 12.98 |
| −2 | 0.002880 | 0.010987 | 13.44 |
| −3 | 0.002598 | 0.009498 | 14.94 |
| −3.5 | 0.002495 | 0.008499 | 16.08 |
| −4 | 0.002457 | 0.008462 | 17.51 |
| −4.5 | 0.002507 | 0.009106 | 19.28 |
| −5 | 0.002641 | 0.010312 | 21.37 |
| −5.5 | 0.002841 | 0.011638 | 23.97 |
| −6 | 0.003112 | 0.013081 | 26.97 |
| −7 | 0.003847 | 0.016159 | 34.28 |
| −8 | 0.004775 | 0.019363 | 42.60 |
| −10 | 0.006683 | 0.025744 | 63.02 |

I added the −3.5 and −4.5 rpm cases to check that the minimum was real and not
a single odd point. Both came out higher than −4 rpm. A parabola through the
three middle points has its vertex at −3.96 rpm; the same fit on max velocity
gives −3.78 rpm.

The pressure range keeps rising across the sweep, so the minimum is in the flow
pattern and not in the driving force. My reading is that counter-rotation first
works against the buoyancy cell and partly cancels it, and beyond about −4 rpm
it starts driving the flow itself.

The change is gradual. The correlation of `u_r` between neighbouring cases stays
above 0.95 through the middle of the sweep, while −1 and −10 rpm correlate at
−0.243, so the two ends are different flow patterns.

## Certification

`certify_case.py` reads the rpm from the file name, works out every boundary
condition from it and checks the exported field against them.

```
python certify_case.py *.csv
```

All twelve cases pass. Typical margins:

| check | typical |
|---|---|
| crucible wall swirl | < 1e-6 % of target |
| crucible bottom swirl | < 1e-6 % of target |
| free-surface swirl ramp | max err ~1.5e-4 m/s |
| crystal face swirl | 0.06 % of edge speed |
| free-surface / bottom thermal ramps | < 0.03 K |
| max \|u_r\| on axis | 0.11 to 0.24 % of field max |

The axis check (radial velocity must be zero on the axis) also catches a swapped
`u_r` / `u_z` column, which Fluent sometimes does on export.

Two known deviations are tolerated:

- The corner node at r = 0.18 belongs to both the crystal face and the free
  surface. Fluent blends the two conditions there (swirl 0.147044 instead of
  0.150796, T 1685.55 K instead of 1685 K), so it is left out of the ramp checks.
- The −3 rpm case was set up with ω = −0.3141 instead of −0.3141593, so its wall
  swirl is 0.05 % off. The script prints this. It has no physical effect.

## Reduced-order model

The sweep is close to low rank. One POD mode holds 99.96 % of the swirl energy
and 99.6 % of the pressure energy; `u_r` and `u_z` need two modes for 99 %.

POD + RBF surrogate, leave-one-out error over the interior cases, as % of each
field's range:

| u_r | u_z | u_swirl | p | T |
|---|---|---|---|---|
| 0.225 % | 0.267 % | 0.043 % | 0.469 % | 0.245 % |

Holding out an end point is extrapolation: −1 rpm gives 12.5 % on pressure and
−10 rpm gives 6.0 %. Only use the model between −1 and −10 rpm.

## Limitations

- Laminar, Boussinesq, steady. No turbulence model, radiation, Marangoni
  convection, species transport or magnetic field.
- The free surface is a prescribed wall, not a solved meniscus.
- Only crucible rotation changes. Hot wall (1745 K) and crystal (+8 rpm) are
  fixed, so the data says nothing about interactions between parameters.
- Student-licence mesh, 8181 nodes.

This is a benchmark for method development, not a model of a real CZ puller.

## Files

```
CZ_crucible_m*rpm_steady.csv  12 cases, r,z,u_r,u_z,u_swirl,p,T
certify_case.py               boundary-condition and sanity checks
CITATION.cff                  citation info
```

## Credit

The ANSYS setup, sweep design and validation are my work. I wrote the Python
with help from an AI assistant and have checked and understand all of it.

Bertwin Kurisinkal Shine, TU Chemnitz
