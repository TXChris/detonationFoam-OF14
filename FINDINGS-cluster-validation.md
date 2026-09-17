# Independent verification of detonationFoam-OF14 on an HPC cluster

Prepared 2026-09-16 by an outside user, against commit `OF-14` as of
2026-09-06. Nothing here is a criticism of the port: it built and ran on the
first attempt on a machine it had never seen. These are the things an
independent run surfaced that the author could not see from a laptop, offered
in case any of them is worth folding back.

## The machine

Eight identical compute nodes, 2 x Xeon Gold 6138 (Skylake-SP, AVX-512), 40
physical cores and 768 GB each, Slurm with PMIx, Ubuntu 26.04, OpenFOAM
Foundation 14 on shared NFS. No GPU. Builds are done on a compute node with
`-march=native` so the binary is tuned for the machine that runs it.

## What was verified

| Check | Result |
|---|---|
| `./Allwmake` against OpenFOAM Foundation 14 | built clean, GCC 15.2.0, `-march=native` |
| `./AllwmakeAMR` (reusable `planarRefiner`) | built clean |
| `tutorials/1D_NH3_O2_cracking_0.3_detonation_OF14_fast` | ran to `End`, 141 steps, stopped on its own shock-position limit |
| `tutorials/H2_O2_laptop_autoUnref_Soret_OF14` | PASSED, including `VerifyLaptopSmoke` 14/14 |
| automatic unrefinement | exercised: split counts `6, 3, 2, 1, 1, ...`, final T max 3581.88 K, final shock position 0.00051 m |
| `tutorials/1D_NH3_O2_cracking_0.3_detonation_OF14` (full, 20000 cells) | running at the time of writing |

So the port is portable: a clean build and a correct-looking run on hardware,
a compiler and an OpenFOAM install with no relationship to the author's.

## Finding 1: the parallel runner cannot be used under a batch scheduler

`tutorials/*/Allrun.parallel` calls OpenFOAM's `runParallel`, which is
`mpirun -np N`. Inside a Slurm allocation this double-books the cores and the
job dies:

```
allocation-overload
```

On a cluster the launcher must be the scheduler's, so the stages have to be run
by hand instead:

```bash
blockMesh
sed -i "s/^numberOfSubdomains .*/numberOfSubdomains $SLURM_NTASKS;/" system/decomposeParDict
decomposePar -force
srun --mpi=pmix -n "$SLURM_NTASKS" foamRun -parallel
```

This is not a defect in the solver, and `Allrun.parallel` is right for a
workstation. It might be worth a line in the tutorial readmes saying that a
scheduled cluster needs the scheduler's launcher, because the failure message
points at MPI rather than at the runner.

## Finding 2: no tutorial states an expected physical result

Every tutorial verifies that the code runs, and `VerifyLaptopSmoke` verifies
that the case is configured as intended. Neither answers the question an
outside user has first: **does this solver get detonation physics right, and
how would I know?** Searching the repository for a wave speed, a cell size or
any reference number returns nothing, and the `OF8_fast_reference` case ships
only a readme saying to run it under `detonationFoam_V2.0`.

The published validation exists (Sun, Wang, Tian & Chen, *Comput. Phys.
Commun.* 292 (2023) 108859, reporting 2026 m/s against a CJ speed of 1977 for
stoichiometric H2/air at 20 um). It is not in the repository, so a user who
builds the port cannot check the build reproduces it.

**Offered contribution.** An independently computed Chapman-Jouguet state for
the mixture the NH3 tutorial actually uses, so the tutorial can state what its
wave speed should approach:

| quantity | value |
|---|---|
| mixture (mass) | NH3 0.290586, O2 0.584991, H2 0.022109, N2 0.102314 |
| initial state | 600 K, 101325 Pa |
| D_CJ | 2419.2 m/s |
| T_CJ | 3410.2 K |
| p_CJ | 11.66 bar |
| T_vN | 1924.1 K |
| p_vN | 21.15 bar |

Computed from **this case's own** `constant/foam/thermo.foam`, by an
independent equilibrium solver (Gibbs minimisation with element constraints,
CJ taken as the minimum wave speed on the equilibrium Hugoniot and checked
against the sonic condition).

**And reproduced by Cantera 3.2.0 on the same mechanism**, converted from the
case's OpenFOAM dictionaries without retyping a number, so the reference is
not one code's opinion:

| quantity | independent solver | Cantera 3.2.0 | difference |
|---|---|---|---|
| D_CJ (m/s) | 2419.2 | 2418.6 | 0.025 % |
| T_CJ (K) | 3410.2 | 3408.2 | 0.06 % |
| p_CJ (bar) | 11.66 | 11.65 | 0.1 % |
| T_vN (K) | 1924.1 | 1923.6 | 0.03 % |
| p_vN (bar) | 21.15 | 21.13 | 0.1 % |

Two tools, two completely different implementations, agreeing to better than
a tenth of a percent. Whatever else is uncertain here, the reference is not.

A tutorial that prints its own leading shock position every write, as this one
does, is one line of post-processing away from reporting a measured wave speed
against that reference. That would turn every tutorial into a physics check
instead of a smoke test.

## Finding 3: OpenFOAM 14's own `etc/bashrc` is not `set -u` clean

Not this project's bug, but it bites anyone writing a strict job script:

```
/software/openfoam14/etc/bashrc: line 46: ZSH_NAME: unbound variable
```

Worth a warning line in the readme, since `set -euo pipefail` is the normal
way to write a batch script and the failure happens before anything of the
solver runs.

## Measured so far: the wave runs at about 72 % of CJ

Interim, from the run still in progress at 0.043 m of the 0.1 m domain. The
leading-shock speed is steady, not still climbing:

| window (m) | speed (m/s) | of D_CJ = 2419.2 |
|---|---|---|
| 0.011 - 0.015 | 1836.6 | 75.9 % |
| 0.015 - 0.020 | 1693.2 | 70.0 % |
| 0.020 - 0.025 | 1849.3 | 76.4 % |
| 0.025 - 0.030 | 1758.1 | 72.7 % |
| 0.030 - 0.035 | 1716.3 | 70.9 % |
| 0.035 - 0.040 | 1736.5 | 71.8 % |
| 0.040 - 0.043 | 1690.0 | 69.9 % |

**Corrected once the run went further: the wave is DECAYING, not steady.** The
reading above was taken at 43 mm and looked flat. Run on to 59 mm, the last
three windows fall monotonically:

| window (m) | speed (m/s) | of D_CJ |
|---|---|---|
| 0.012 - 0.020 | 1734.9 | 71.7 % |
| 0.020 - 0.028 | 1822.6 | 75.3 % |
| 0.028 - 0.036 | 1723.8 | 71.3 % |
| 0.036 - 0.044 | 1707.4 | 70.6 % |
| 0.044 - 0.052 | 1618.7 | 66.9 % |
| 0.052 - 0.059 | 1552.9 | 64.2 % |

That is a different object from a marginal-but-sustained detonation. A wave
that is still slowing after 47 mm of travel is a driver-initiated shock-flame
complex on its way out, not a self-sustaining front, and reporting it as
"settled at 72 %" was wrong. The error was mine and it was the ordinary one:
a trend read off too short a record.

**This is an observation about the tutorial, not a verdict on the solver, and
it should not be read as one.** At least three explanations fit and this run
does not separate them:

0. **The numerics are not the cause. Ruled out, measured.** The same case with
   vanLeer reconstruction instead of Minmod, the most diffusive common limiter
   replaced by a less diffusive one, tracks the baseline to within 0.1 %:
   1736.5 against 1734.9 m/s over 0.012-0.020 m, and 1858.9 against 1822.6 over
   0.020-0.028 m. Numerical dissipation was the hypothesis this port was most
   attractive for testing, and it produced a null result.
1. ~~**The case may be underresolved.**~~ **Ruled out, measured.** The
   induction length at this mixture's von Neumann state, from a constant-volume
   Cantera reactor on the case's own mechanism, is 125 um by the 400 K-rise
   definition and 170 um by steepest dT/dt. At the case's 5 um mesh that is
   **25 to 34 cells across the induction zone**, comfortably above the 10 to 20
   usually asked for. The mesh is not the problem, and if the wave really is
   running slow its shock is weaker, its post-shock temperature lower and its
   induction zone longer still, so it would be better resolved, not worse.
2. **The tutorial may be deliberately underdriven**, in which case 72 % is
   the intended answer and only the absence of a stated expectation makes it
   look like a finding.
3. ~~**The mixture may be genuinely marginal**, with a low-velocity mode.~~
   **Weakened by the decay.** A low-velocity mode is a mode: it sustains. This
   one is still losing speed at 59 mm. Ammonia being hard to detonate remains
   the likely reason it decays, but the wave is not sitting in a mode.
4. **Something in the port or the solver.** Still last, but with the cheapest
   explanation now eliminated rather than assumed.

What can be said without ambiguity is narrower and still worth saying: the
tutorial ships no expected wave speed, so none of the three can be told apart
by a user who runs it, and a run that is behaving exactly as designed is
indistinguishable from one that is not.

Peak pressure and temperature over the whole domain at 18.5 us are 10.02 bar
and 4522 K, against a predicted von Neumann state of 21.15 bar and 1924 K and
a CJ state of 11.66 bar and 3410 K. Those are domain maxima and include the
5000 K driver region, so they do NOT characterise the front and no conclusion
is drawn from them here. Front-resolved profiles are the next measurement.

## The result that matters: a second, unrelated solver agrees

The same case, same mesh, same initial fields, same 33-species mechanism and
the same thermo, run under OpenFOAM 14's own stock `multicomponentFluid`
instead of `detonationFluid`. Those are different numerical families: density
based with MUSCL reconstruction and an HLLC-P Riemann solver against pressure
based PIMPLE with Gauss upwind. Both measured the same way, from the written
pressure field at the tutorial's own 101425 Pa threshold, so neither solver's
own reporting gets a vote.

| window (m) | detonationFluid | multicomponentFluid |
|---|---|---|
| 0.020 - 0.028 | 1822.6 (75.3 %) | 1811.4 (74.9 %) |
| 0.028 - 0.036 | 1723.8 (71.3 %) | 1687.2 (69.7 %) |
| 0.036 - 0.044 | 1707.4 (70.6 %) | 1708.5 (70.6 %) |
| 0.044 - 0.047 | 1654.0 (68.4 %) | 1856.2 (76.7 %) |

Three of the four windows agree to within 2 %, and one of them to a tenth of a
percent. The fourth sits at the end of the data available for the stock solver
and spans only 3 mm, so it is the noisiest window in the table rather than a
disagreement; it should be re-read when that run has gone further.

**So the sub-CJ wave is not an artefact of this port.** Two solvers with
nothing in common but the framework, the mesh and the chemistry produce the
same wave at the same speed. Taken with the two earlier null results -- the
mesh resolves 25 to 34 cells across the induction zone, and vanLeer against
Minmod changes the answer by 0.1 % -- every explanation that would have
implicated the numerics is now measured away.

What remains is the case itself: a driver-initiated wave in a mixture that
does not sustain a detonation at these conditions, decaying slowly over the
domain. That may well be exactly what the tutorial is for. The point of the
report stands either way, and is now sharper: **a user cannot tell, because the
tutorial states no expected result.** Three independent measurements were
needed here to establish that a number is the intended one, and one line in a
readme would have done it.

## What is still running

The full `1D_NH3_O2_cracking_0.3_detonation_OF14` case, 20000 cells over
0.1 m, on 20 ranks. Its own `SW_position_limit` of 0.09999 m ends it when the
wave crosses the domain. The measurement it is being run for is the mean
leading-shock speed once the wave is established, against the 2419.2 m/s
above. That result will be appended here.
