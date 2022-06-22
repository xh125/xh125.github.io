---
layout:     post
title:      "QUANTUM ESPRESSO：Convergence-tests"
date:       2022-06-22 16:56:00
author:     "Xiehua"
header-img: "images/post/post-bg-understand.jpg"
tags:
    - QUANTUM ESPRESSO
    - Convergence-tests
    - ecutwfc
    - ecutrho
    - k-points
    - smearing、degauss
---

## QUANTUM ESPRESSO参数设置收敛性测试：

`pw.x`的参数在设置的时候，需要进行收敛性测试，以使得计算得到的结果满足收敛性条件，常见的需要进行收敛性测试的参数包括：平面波截断能[**`ecutwfc`**](https://www.quantum-espresso.org/Doc/INPUT_PW.html#idm298)、电荷密度和电势的截断能[**`ecutrho`**](https://www.quantum-espresso.org/Doc/INPUT_PW.html#idm301)(在使用超软赝势和PAW赝势时需要进行收敛性测试)、[**`k-point`**](https://www.quantum-espresso.org/Doc/INPUT_PW.html#idm1487)、对于金属进行布里渊区积分时的展宽系数[**`degauss`**](https://www.quantum-espresso.org/Doc/INPUT_PW.html#idm379)的收敛性测试。使用状态方程（Equation of State）拟合得到晶格常数。

### （1） ecutwfc的收敛性测试

收敛性测试脚本：  

```bash
#!/bin/bash
#BSUB -J ecuttest
#BSUB -q privateq-zw
##BSUB -q publicq
#BSUB -n 28
#BSUB -R "span[ptile=28]"
#BSUB -o %J.out
#BSUB -e %J.err

CURDIR=$PWD
rm -f nodelist >& /dev/null
for host in `echo $LSB_HOSTS`
do
echo $host >> nodelist
done
NP=`cat nodelist | wc -l`
rm nodelist

module use /share/home/zw/xiehua/modulefiles
module load Quantum_Espresso/7.0

rm -f etot_vs_ecut.dat
for ecut in $(seq 40 5 100)
do
	erho=$[8 * $ecut]
	cat >scf.$ecut.in<< EOF
&CONTROL
    calculation   = "scf"  
    restart_mode  = "from_scratch"
    prefix        = "graphene"
    outdir        = "./"
    pseudo_dir    = "./pseudo/"
    verbosity     = "high"
    tprnfor       = .true.
    tstress       = .true.
    etot_conv_thr =  1.0d-8
    forc_conv_thr =  1.0d-7
/

&SYSTEM
    ibrav       = 4
    nat         = 2
    ntyp        = 1
!    a           = 2.464
!    c           = 15.0
 celldm(1) =      4.66148920
 celldm(3) =      6.08086614
!    nbnd        = 16
    assume_isolated = "2D"
    occupations = "smearing"
    smearing    = "mp"    
    degauss     =  1.0d-2
    ecutwfc     =  $ecut
    ecutrho     =  $erho
/

&ELECTRONS
    conv_thr         =  1.000e-12
    electron_maxstep =  200
    mixing_beta      =  7.00000e-01
    startingpot      = "atomic"
    startingwfc      = "atomic+random"
/

!&IONS
!    ion_dynamics = "bfgs"
!/

!&CELL
!    cell_dofree    = "ibrav+2Dxy"
!    cell_dynamics  = "bfgs"
!    press_conv_thr =  0.001
!/

K_POINTS {automatic}
 36  36  1  0 0 0

ATOMIC_SPECIES
C      12.01070  C.pbe-n-kjpaw_psl.1.0.0.UPF

ATOMIC_POSITIONS {crystal}
C       0.333333333333333   0.666666666666667   0.500000
C       0.666666666666667   0.333333333333333   0.500000
EOF

mpirun -np $NP pw.x -nk 7 <scf.$ecut.in>scf.$ecut.out

grep -e 'kinetic-energy cutoff' -e ! scf.$ecut.out | \
        awk '/kinetic-energy/ {ecut=$(NF-1)}
             /!/              {print ecut, $(NF-1)}' >> etot_vs_ecut.dat

done
```

![etot-vs-ecutwfc](https://xh125.github.io/images/post/etot-vs-ecutwfc.png)

### (2) 在得到平面波截断能的收敛能后测试电势的截断能收敛性

```bash
#!/bin/bash
#BSUB -J ecuttest
#BSUB -q privateq-zw
##BSUB -q publicq
#BSUB -n 28
#BSUB -R "span[ptile=28]"
#BSUB -o %J.out
#BSUB -e %J.err

CURDIR=$PWD
rm -f nodelist >& /dev/null
for host in `echo $LSB_HOSTS`
do
echo $host >> nodelist
done
NP=`cat nodelist | wc -l`
rm nodelist

module use /share/home/zw/xiehua/modulefiles
module load Quantum_Espresso/7.0

rm -f etot_vs_ecut.dat
for rho in $(seq 4 1 16)
do
	erho=$[80 * $rho]
	cat >scf.$erho.in<< EOF
&CONTROL
    calculation   = "scf"  
    restart_mode  = "from_scratch"
    prefix        = "graphene"
    outdir        = "./"
    pseudo_dir    = "./pseudo/"
    verbosity     = "high"
    tprnfor       = .true.
    tstress       = .true.
    etot_conv_thr =  1.0d-8
    forc_conv_thr =  1.0d-7
/

&SYSTEM
    ibrav       = 4
    nat         = 2
    ntyp        = 1
!    a           = 2.464
!    c           = 15.0
 celldm(1) =      4.66148920
 celldm(3) =      6.08086614
!    nbnd        = 16
    assume_isolated = "2D"
    occupations = "smearing"
    smearing    = "mp"    
    degauss     =  1.0d-2
    ecutwfc     =  80
    ecutrho     =  $erho
/

&ELECTRONS
    conv_thr         =  1.000e-12
    electron_maxstep =  200
    mixing_beta      =  7.00000e-01
    startingpot      = "atomic"
    startingwfc      = "atomic+random"
/

!&IONS
!    ion_dynamics = "bfgs"
!/

!&CELL
!    cell_dofree    = "ibrav+2Dxy"
!    cell_dynamics  = "bfgs"
!    press_conv_thr =  0.001
!/

K_POINTS {automatic}
 36  36  1  0 0 0

ATOMIC_SPECIES
C      12.01070  C.pbe-n-kjpaw_psl.1.0.0.UPF

ATOMIC_POSITIONS {crystal}
C       0.333333333333333   0.666666666666667   0.500000
C       0.666666666666667   0.333333333333333   0.500000
EOF

mpirun -np $NP pw.x -nk 7 <scf.$erho.in>scf.$erho.out

grep -e 'charge density cutoff' -e ! scf.$erho.out | \
        awk '/charge density cutoff/ {ecut=$(NF-1)}
             /!/              {print ecut, $(NF-1)}' >> etot_vs_erho.dat

done
```

![etot-vs-ecutrho](https://xh125.github.io/images/post/etot-vs-rho.png)