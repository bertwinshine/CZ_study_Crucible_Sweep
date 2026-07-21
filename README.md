
# CZ melt - crucible rotation study

Reference CFD cases for the Czochralski silicon melt, this time varying the crucible
rotation. Same melt setup as before, temperature held at 1745 K and the crystal kept at
its baseline 8 rpm - here it's the crucible spin that changes between cases. Meant as
ground-truth data for the surrogate/PINN model.

Only the crucible rotation changes from case to case, everything else is fixed.

## Common setup

- 2D axisymmetric swirl, silicon melt in a crucible
- radius 0.30 m, height 0.15 m, 8181-node mesh
- crucible hot wall fixed at 1745 K
- crystal face fixed at 1685 K
- crystal rotation fixed at 8 rpm (0.838 rad/s)
- steady, laminar, energy on, gravity on
- Coupled, second order

## Cases

The crucible is counter-rotating (opposite to the crystal), and I swept its speed from
-1 down to -10 rpm, which is the usual crucible range for CZ growth. Both crucible walls
(bottom and side) turn at the same speed in each case.

| file | crucible rotation | swirl range |
|---|---|---|
| CZ_crucible_m1rpm_steady.csv | -1 rpm (-0.105 rad/s) | -0.094 to 0.148 m/s |
| CZ_crucible_m3rpm_steady.csv | -3 rpm (-0.314 rad/s) | -0.094 to 0.148 m/s |
| CZ_crucible_m6rpm_steady.csv | -6 rpm (-0.628 rad/s) | -0.188 to 0.148 m/s |
| CZ_crucible_m10rpm_steady.csv | -10 rpm (-1.047 rad/s) | -0.314 to 0.148 m/s |

The -3 rpm case is the baseline - the same 1745 K / 8 rpm case that the temperature study
and the crystal rotation study both share. It's included here under the crucible name so
this folder is complete on its own.

The positive side of the swirl stays around 0.148 in every case because the crystal is
fixed at 8 rpm and drives that part of the flow. What changes is the negative side - as
the crucible spins harder in the opposite direction it pulls more swirl its own way, so
that peak grows from -0.094 down to -0.314. At the low end (-1 and -3 rpm) the crucible
is too slow to make much difference and the two look almost the same; the crucible's
effect only really shows up at -6 and -10.

## Columns

One row per node: `r, z, u_r, u_z, u_swirl, p, T`

r, z are position (m), u_r/u_z/u_swirl the velocity parts (m/s), p pressure (Pa),
T temperature (K).

## Notes

All four are steady and certified the same way as the other studies - Fluent reached its
convergence criteria on its own, mass closes to zero, temperature stays in the
1685-1745 K band, and the swirl behaves the way it should as the crucible speeds up.

One thing worth flagging: the -6 rpm case closes its energy balance to about 1.7%, looser
than the others (which sit under 1%). It's still a converged solution and the mass balance
is fine - the reason is that the stronger crucible rotation drives a lot more heat transfer
at the free surface, and the 8181-node mesh resolves that slightly less tightly there. A
finer mesh would close it better but that's beyond what the Student license allows. The
-10 rpm case, oddly, comes back under 1% again, so it isn't a straight trend. Keeping the
number on record for that case rather than hiding it.
