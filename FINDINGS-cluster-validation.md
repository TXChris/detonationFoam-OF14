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
| `./Allwmake` against OpenFOAM Foundation 14 | built clean, GCC 15.2.0. **Not** `-march=native`, as this row first said: the flag was set in the environment and wmake does not read it (`wmake/rules/linux64Gcc/c++Opt` is `-O3` only), and the library carries no AVX instruction at all (`objdump` census, 2026-09-18). Baseline x86-64. |
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

0. ~~**The numerics are not the cause. Ruled out, measured.**~~ **WRONG, and
   corrected below: the numerics ARE the cause.** The claim was that the same
   case with vanLeer reconstruction instead of Minmod tracks the baseline to
   within 0.1 %, and over the stretch it was measured on that is true:
   1736.5 against 1734.9 m/s over 0.012-0.020 m, and 1858.9 against 1822.6
   over 0.020-0.028 m. But the vanLeer run was still going. It parts from the
   baseline at 30 mm and transitions to detonation at 40 mm. The measurement
   stopped 12 mm short of the answer. See "The limiter decides whether this
   case detonates" at the end of this report.
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
4. **Something in the port or the solver.** Still last, and now the live
   question is narrower: not whether the port is wrong, but which limiter the
   tutorial ought to ship.

What can be said without ambiguity is narrower and still worth saying: the
tutorial ships no expected wave speed, so none of the three can be told apart
by a user who runs it, and a run that is behaving exactly as designed is
indistinguishable from one that is not.

Peak pressure and temperature over the whole domain at 18.5 us are 10.02 bar
and 4522 K, against a predicted von Neumann state of 21.15 bar and 1924 K and
a CJ state of 11.66 bar and 3410 K. Those are domain maxima and include the
5000 K driver region, so they do NOT characterise the front and no conclusion
is drawn from them here. Front-resolved profiles are the next measurement.

## A second, unrelated solver agrees with the baseline

*Read with the section after it. The comparison stands as measured, but the
conclusion drawn from it does not. Both of these runs carry heavy dissipation
where the shock lives: the baseline reconstructs rho, U and T with Minmod, the
most diffusive common MUSCL limiter, and the stock solver's momentum
convection is `Gauss upwind`, first order (its species, kinetic energy and
pressure flux are `Gauss vanLeer`). Two schemes that are both dissipative on
the momentum equation agreeing that the wave decays is weaker evidence than it
looked, and the section after this one shows what happens when that
dissipation is reduced.*

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

Two runs of the full `1D_NH3_O2_cracking_0.3_detonation_OF14` case, 20000
cells over 0.1 m, 20 ranks each, both with `SW_position_limit` 0.09999 m so
they end when the wave crosses the domain.

| | scheme | at | still to answer |
|---|---|---|---|
| job 684, n1c | vanLeer | 53.7 mm, t = 2.08e-5 s | does the overdriven wave settle at CJ |
| job 683, n1b | stock `multicomponentFluid` | 48.3 mm, t = 2.18e-5 s | is its rise at 43 mm an oscillation or its own transition |

The Minmod baseline (job 671) is not running: it was killed by a 10 h wall
limit at 59.3 mm. It needs resubmitting with a longer limit before "Minmod
does not detonate in this tube" can be stated rather than inferred.

## The limiter decides whether this case detonates

This supersedes item 0 above, which said numerical dissipation had been ruled
out. It had not been. The vanLeer run that produced that null result was still
going when the result was written, and it was reporting agreement from the
only stretch where the two limiters agree.

Same case, same 20000 cells over 0.1 m, same mechanism, same thermo, same
initial fields, 20 ranks. `diff` over `system/fvSolution`,
`system/controlDict`, `constant/solverTypeProperties` and
`constant/chemistryProperties` is empty. Three lines of `system/fvSchemes`
differ:

```
reconstruct(rho)    Minmod;      ->  vanLeer;
reconstruct(U)      MinmodV;     ->  vanLeerV;
reconstruct(T)      Minmod;      ->  vanLeer;
```

Leading-shock speed as a least-squares slope over each 2 mm window, against
D_CJ = 2419.2 m/s:

| window (m) | Minmod | vanLeer |
|---|---|---|
| 0.012 - 0.014 | 1835.8 (75.9 %) | 1835.1 (75.9 %) |
| 0.016 - 0.018 | 1699.6 (70.3 %) | 1701.4 (70.3 %) |
| 0.020 - 0.022 | 1789.1 (74.0 %) | 1887.3 (78.0 %) |
| 0.024 - 0.026 | 1822.7 (75.3 %) | 1847.3 (76.4 %) |
| 0.028 - 0.030 | 1726.9 (71.4 %) | 1836.8 (75.9 %) |
| 0.032 - 0.034 | 1690.8 (69.9 %) | 1855.7 (76.7 %) |
| 0.036 - 0.038 | 1743.9 (72.1 %) | 2091.4 (86.4 %) |
| 0.038 - 0.040 | 1721.1 (71.1 %) | 2413.6 (99.8 %) |
| 0.040 - 0.042 | 1691.0 (69.9 %) | **3244.6 (134.1 %)** |
| 0.044 - 0.046 | 1655.6 (68.4 %) | 2840.7 (117.4 %) |
| 0.048 - 0.050 | 1598.4 (66.1 %) | 2745.3 (113.5 %) |
| 0.052 - 0.054 | 1577.0 (65.2 %) | 2637.6 (109.0 %) |
| 0.058 - 0.060 | 1551.3 (64.1 %) | still running |

Out to 20 mm the two agree to 0.1 %, and out to 28 mm to 2 %. That is the
stretch item 0 was measured on.

From 30 mm they separate. vanLeer climbs through 86 % of CJ at 37 mm and 100 %
at 39 mm, spikes to 134 % at 41 mm, and relaxes through 117, 113 and 109 %.
An overshoot past CJ followed by a decay toward it is the signature of
deflagration-to-detonation transition with an overdriven phase. Minmod, in the
same case at the same instant, is at 70 % and still falling, and reaches 64 %
at 59 mm.

**So the sub-CJ wave is a property of the limiter, not of the case.** With the
tutorial's default the wave decays; with a less diffusive limiter of the same
family it detonates.

### What this still does not show

- **vanLeer has not been shown to settle at CJ.** It is at 109 % and falling
  with 46 mm of tube left. Overdriven relaxation is the reading, but it is not
  a measurement until the run reaches the end of the domain.
- **Minmod has not been shown never to detonate.** That run was killed by a
  10 h wall limit at 59 mm, not by reaching an answer.
- **One mesh.** dx = 5 um in all three runs, so dissipation and resolution are
  not separated. The induction zone is resolved at 25 to 34 cells, but the
  transition length is a different scale. The experiment that separates them
  is Minmod on a finer mesh: if Minmod detonates at 2.5 um, the default is
  under-resolved rather than over-dissipative. That run has not been done.

### What is worth sending upstream

Not "the tutorial is wrong". The sharper point, and the one a user cannot
reach on their own:

**This tutorial's outcome is scheme-dependent, and it ships no expected
result.** A user who runs it with the defaults gets a decaying 64 % wave, a
user who changes one line in `fvSchemes` gets a detonation, and nothing in the
case tells either of them which was intended. One line in the readme stating
the expected wave speed, and a note that the Minmod default is the reason it
is what it is, would close that.

Measurement, tracks and the analysis script:
`rde_engine/benchmarks/detonationfoam-limiter/`.

## The side patches are not irrelevant either

The tutorial mesh is 20000 x 1 x 1 cells and its `bottom` and `top` patches
are `wall`, with `zeroGradient` on every field, so each carries 20000 faces
on a problem that has none in that direction. Two consequences, measured on
the cluster (rde_engine `benchmarks/openfoam-hotpath`, jobs 763 and 765, six
ranks, the production ISAT settings, 678 steps to 0.2 µs):

- **Cost.** Every field evaluates 40000 boundary faces per update, and the
  energy boundary condition builds a species mixture per face and per
  adjacent cell each time. Declaring the two patches `empty` took the step
  from 0.1808 to 0.0942 s: 1.92x.
- **Answer.** The written fields are not the same: T at the front differs by
  up to 1 % at 0.2 µs, p by 0.2 %, and the fields away from the front are
  byte-identical. The side faces sit in the reconstruction limiter's stencil,
  so a 1-D result depends on a patch type that should not enter it.

Every wave speed in this report was measured with the `wall` sides. The same
case with `empty` sides, Minmod, 20 ranks, is running to the end of the tube
(job 768) and its track will be added here. Until then the tables above are
"with the tutorial's patches", which is one more thing a readme line would
have settled.
