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
against the sonic condition). That solver is gated against Cantera at 0.02 %
on D_CJ, T_vN and p_vN for H2/air, so the number above is a cross-tool result
rather than one code's opinion.

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

## What is still running

The full `1D_NH3_O2_cracking_0.3_detonation_OF14` case, 20000 cells over
0.1 m, on 20 ranks. Its own `SW_position_limit` of 0.09999 m ends it when the
wave crosses the domain. The measurement it is being run for is the mean
leading-shock speed once the wave is established, against the 2419.2 m/s
above. That result will be appended here.
