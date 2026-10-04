---
title: "Quantum Espresso"
date: 2026-09-30T19:53:33+05:30
draft: false
author: "Zhongwei Lu"
tags:
  - QM software
  - plane wave
  - example
image: /images/quantum-espresso/qe-pdos.png
description: "Example calc with Quantum Espresso"
toc: true
mathjax: true
---

## Density of State with QE

Quantum Espresso (QE, https://www.quantum-espresso.org/) is an open-source, plane-wave based, DFT software. 

Here, QE is used to generate Density of State plots for Pr \\( _{2} \\) O \\( _{3} \\).

![alt text](/images/quantum-espresso/bluk.png)

### Convergence

Using a (8,8,8) k grid and a fixed ecutrho/ecutwfc=8 ratio, the ecutwfc was converged first.

![alt text](/images/quantum-espresso/ecutwfc_convergence_dual_axis.png)

The ecutwfc is converged at 120 Ry. A higher ecutrho/ecutwfc ratio did not change the energy.

The k_grid convergenced is benchmarked with Geometry Optimization of the unit cell.

```Python
import numpy as np
from ase import Atoms
from ase.io.trajectory import Trajectory
from ase.eos import EquationOfState
from ase.io import read
from ase.units import kJ
from ase.filters import ExpCellFilter, FrechetCellFilter
from ase.optimize import LBFGS, QuasiNewton
from ase.calculators.espresso import Espresso, EspressoProfile


# Pseudopotentials from SSSP Efficiency v1.3.0
pseudopotentials = {'Pr': 'Pr.paw.pbe.z_13.atompaw.wentzcovitch.v1.0.legacy.upf', 'O': 'O.paw.pbe.z_6.ld1.psl.v0.1.upf'}

# Optionally create profile to override paths in ASE configuration:
profile = EspressoProfile(
    command='mpirun -np $SLURM_NTASKS /shared/apps/easybuild/x86_64/amd/zen4/software/QuantumESPRESSO/7.5-foss-2025b/bin/pw.x',
    pseudo_dir='/shared/home1/c.c22015584/Example-QuantumEspresso/mix-sssp-prec-pbe-lib-v2/library'
)

#Pr 90 720; O 50 600
#opt wfc 120 rho 960
wfc = 120
rho = 8 * wfc
input_data = {
    'system': {'ecutwfc': wfc, 'ecutrho': rho},
    #'disk_io': 'low',  # Automatically put into the 'control' section
    'tprnfor': True,
    'tstress': True,
    #'mixing_ndim': 25,
    'electron_maxstep': 150,
}


espresso = Espresso(profile=profile, pseudopotentials=pseudopotentials,
        kpts=(15,15,15),
        input_data=input_data)

oxide = read("EntryWithCollCode75481.cif")

cell = oxide.get_cell()
traj = Trajectory('oxide.traj', 'w')


with espresso.socketio(unixsocket='ase-espresso') as calc:

    oxide.calc = calc

    ecf = FrechetCellFilter(oxide, constant_volume=True)
    qn = QuasiNewton(ecf)
    traj_opt = Trajectory('opt_x.traj', 'a', oxide)
    qn.attach(traj_opt)

    qn.run(fmax=0.01)
    energy = oxide.get_potential_energy()

    traj.write(oxide)

```

![alt text](/images/quantum-espresso/kpoint_convergence.png)


Finally, the full density of state analysis with dos.x,

![alt text](/images/quantum-espresso/tot_dos.png)

and the projected density of state analysis with pdos.x.

![alt text](/images/quantum-espresso/combined_pdos_vertical.png)


More to add: Phonon.
 


