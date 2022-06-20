---
layout:     post
title:      "QUANTUM ESPRESSO：vc-relax"
date:       2021-12-14 16:56:00
author:     "Xiehua"
header-img: "images/post/post-bg-understand.jpg"
tags:
    - QUANTUM ESPRESSO
    - vc-relax
---

## QUANTUM ESPRESSO：[vc-relax](http://www.quantum-espresso.org/Doc/INPUT_PW.html#idm32)  

vc-relax:对晶体的晶格常数和原子位置进行结构优化，需要设置[**`cell_dynamics`**](http://www.quantum-espresso.org/Doc/INPUT_PW.html#idm1080),[**`press_conv_thr`**](http://www.quantum-espresso.org/Doc/INPUT_PW.html#idm1108)等参数。由于后续需要进行phonon计算，最好在relax过程中将相应的计算精度设置的高一些，否则容易在phonon计算中出现虚频，包括[**`conv_thr`**](https://www.quantum-espresso.org/Doc/INPUT_PW.html#idm771)、[**`etot_conv_thr`**](https://www.quantum-espresso.org/Doc/INPUT_PW.html#idm117)、[**`forc_conv_thr`**](https://www.quantum-espresso.org/Doc/INPUT_PW.html#idm123)参数。对于二维材料，在计算中可以使用参数[**`assume_isolated='2D'`**](https://www.quantum-espresso.org/Doc/INPUT_PW.html#idm547)  对于低维材料体系，在优化时对于k点设置要设置的多一点，否则容易导致结构优化的晶格常数不准确，以及phonon计算中的声子谱不收敛。例如：对于石墨烯，需要设置到36\*36\*1,同时对于`smearing`设置成`mp`比较合适.


### 采用`ibrav=0`设置结构的 vc-relax输入文件 
*note:*采用分数坐标时，坐标给到小数点后10位比较合适，在小数点位数不够时，容易在phonon计算中，对于q点对称性的计算会有问题。 最好将`outdir`统一设置为`outdir='./'`,由于后续epw计算中需要使用`outdir='./'`

```fortran
&CONTROL
    calculation   = "vc-relax"  
    restart_mode  = "from_scratch"
    prefix        = "carbyne"
    outdir        = "./"
    pseudo_dir    = "./pseudo/"
    verbosity     = "high"
    tprnfor       = .true.  
    tstress       = .true.
    etot_conv_thr =  1.0d-8
    forc_conv_thr =  1.0d-7
/

&SYSTEM
    ibrav       = 0
    nat         = 2
    ntyp        = 1
    nbnd        = 22
    occupations = 'fixed'
    ecutwfc     =  50
    ecutrho     =  400
/

&ELECTRONS
    conv_thr         =  1.000e-12
    electron_maxstep =  200
    mixing_beta      =  0.7
    startingpot      = "atomic"
    startingwfc      = "atomic+random"
/

&IONS
    ion_dynamics = "bfgs"
/

&CELL
    cell_dofree    = "x"
    cell_dynamics  = "bfgs"
    press_conv_thr =  0.01
/

K_POINTS {automatic}
 200  1  1  0 0 0

ATOMIC_SPECIES
C      12.01070  C.pbe-n-kjpaw_psl.1.0.0.UPF

CELL_PARAMETERS (angstrom)
   2.565600000   0.000000000   0.000000000
   0.000000000  10.000000000   0.000000000
   0.000000000   0.000000000  10.000000000

ATOMIC_POSITIONS (angstrom)
C             0.0000000000       5.0000000000       5.0000000000    0   0   0
C             1.2640800000       5.0000000000       5.0000000000    1   0   0

```  

计算过程中使用下列命令可以查看优化过程中的收敛情况：  

查看优化过程中各原子总的受力情况:

```bash
grep " Total force =" vc-relax.out
```

查看优化过程中能量的变化过程

```bash
grep ! vc-relax.out 
```  

查看优化过程中压力张量大小

```bash
grep -A 12 " Total force =" vc-relax.out 
```

计算结束后，运行

```bash
awk  '/Begin final coordinates/,/End final coordinates/{print $0}' vc-relax.out
```

得到以下输出（或在输出文件中可以找到）（vc-relax的结果）

```bash
Begin final coordinates
     new unit-cell volume =   1731.30152 a.u.^3 (   256.55242 Ang^3 )
     density =      0.15548 g/cm^3

CELL_PARAMETERS (angstrom)
   2.565524159   0.000000000   0.000000000
   0.000000000  10.000000000   0.000000000
   0.000000000   0.000000000  10.000000000

ATOMIC_POSITIONS (angstrom)
C             0.0000000000        5.0000000000        5.0000000000    0   0   0
C             1.2639655349        5.0000000000        5.0000000000    1   0   0
End final coordinates
```

### 采用 $ibrav \neq 0$ 以及A, B, C, cosAB, cosAC, cosBC的结构进行结构优化  

```fortran
&CONTROL
    calculation   = "vc-relax"  
    restart_mode  = "from_scratch"
    prefix        = "carbyne"
    outdir        = "./outdir/"
    pseudo_dir    = "./pseudo/"
    verbosity     = "high"
    tprnfor       = .true.  
    tstress       = .true.
    etot_conv_thr =  1.0d-6
    forc_conv_thr =  1.0d-5
/

&SYSTEM
    ibrav       = 6
    A           = 10.0
    C           = 2.5656
    nat         = 2
    ntyp        = 1
    nbnd        = 22
    occupations = 'fixed'
    ecutwfc     =  50
    ecutrho     =  400
/

&ELECTRONS
    conv_thr         =  1.000e-9
    electron_maxstep =  200
    mixing_beta      =  0.7
    startingpot      = "atomic"
    startingwfc      = "atomic+random"
/

&IONS
    ion_dynamics = "bfgs"
/

&CELL
    cell_dofree    = "z"
    cell_dynamics  = "bfgs"
    press_conv_thr =  0.01
/

K_POINTS {automatic}
 1 1 100  0  0  0 

ATOMIC_SPECIES
C      12.01070  C.pbe-n-kjpaw_psl.1.0.0.UPF

ATOMIC_POSITIONS (angstrom)
C             5.0000000000       5.0000000000       0.0000000000    0   0   0
C             5.0000000000       5.0000000000       1.2640800000    0   0   1
```  

计算结束后，运行

```bash
awk  '/Begin final coordinates/,/End final coordinates/{print $0}' vc-relax.out
```

得到以下输出（或在输出文件中可以找到）（vc-relax的结果）

```bash
Begin final coordinates
     new unit-cell volume =   1731.30161 a.u.^3 (   256.55243 Ang^3 )
     density =      0.15548 g/cm^3

CELL_PARAMETERS (alat= 18.89726125)
   1.000000000   0.000000000   0.000000000
   0.000000000   1.000000000   0.000000000
   0.000000000   0.000000000   0.256552429

ATOMIC_POSITIONS (angstrom)
C             5.0000000000        5.0000000000        0.0000000000    0   0   0
C             5.0000000000        5.0000000000        1.2640744811    0   0   1
End final coordinates
```

### 采用 $ibrav \neq 0$ 以及celldm(i), i=1,6的结构进行结构优化  

```fortran
&CONTROL
    calculation   = "vc-relax"  
    restart_mode  = "from_scratch"
    prefix        = "carbyne"
    outdir        = "./outdir/"
    pseudo_dir    = "./pseudo/"
    verbosity     = "high"
    tprnfor       = .true.  
    tstress       = .true.
    etot_conv_thr =  1.0d-6
    forc_conv_thr =  1.0d-5
/

&SYSTEM
    ibrav       = 6
    celldm(1)   = 18.89726125
    celldm(3)   = 0.256552429
    nat         = 2
    ntyp        = 1
    nbnd        = 22
    occupations = 'fixed'
    ecutwfc     =  50
    ecutrho     =  400
/

&ELECTRONS
    conv_thr         =  1.000e-9
    electron_maxstep =  200
    mixing_beta      =  0.7
    startingpot      = "atomic"
    startingwfc      = "atomic+random"
/

&IONS
    ion_dynamics = "bfgs"
/

&CELL
    cell_dofree    = "z"
    cell_dynamics  = "bfgs"
    press_conv_thr =  0.01
/

K_POINTS {automatic}
 1 1 100  0  0  0 

ATOMIC_SPECIES
C      12.01070  C.pbe-n-kjpaw_psl.1.0.0.UPF

ATOMIC_POSITIONS (angstrom)
C             5.0000000000       5.0000000000       0.0000000000    0   0   0
C             5.0000000000       5.0000000000       1.2640800000    0   0   1
```  

计算结束后，运行

```bash
awk  '/Begin final coordinates/,/End final coordinates/{print $0}' vc-relax.out
grep -B 40 "Begin final coordinates" vc-relax.out
```

得到以下输出（或在输出文件中可以找到）（vc-relax的结果）

```bash
Begin final coordinates
     new unit-cell volume =   1731.29310 a.u.^3 (   256.55117 Ang^3 )
     density =      0.15548 g/cm^3

CELL_PARAMETERS (alat= 18.89726125)
   1.000000000   0.000000000   0.000000000
   0.000000000   1.000000000   0.000000000
   0.000000000   0.000000000   0.256551169

ATOMIC_POSITIONS (angstrom)
C             5.0000000000        5.0000000000        0.0000000000    0   0   0
C             5.0000000000        5.0000000000        1.2640744811    0   0   1
End final coordinates
```

#### Notes  

如果设置了`cell_dofree    = "ibrav"`,优化过程保持布拉维格子的种类不变，vc-relax.out中会有优化后的优化后的celldm,见：  
```bash
grep -B 40 "Begin final coordinates" vc-relax.out
```

```fortran
ibrav =      7
 celldm(1) =     10.37600462
 celldm(3) =      1.97244178
Input lattice vectors:
     0.49999668    -0.49999668     0.98621435
     0.49999668     0.49999668     0.98621435
    -0.49999668    -0.49999668     0.98621435
New lattice vectors in INITIAL alat:
     0.49999668    -0.49999668     0.98621435
     0.49999668     0.49999668     0.98621435
    -0.49999668    -0.49999668     0.98621435
New lattice vectors in NEW alat (for information only):
     0.50000000    -0.50000000     0.98622089
     0.50000000     0.50000000     0.98622089
    -0.50000000    -0.50000000     0.98622089

```
