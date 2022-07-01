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

vc-relax:对晶体的晶格常数和原子位置进行结构优化，需要设置[**`cell_dynamics`**](http://www.quantum-espresso.org/Doc/INPUT_PW.html#idm1080),[**`press_conv_thr`**](http://www.quantum-espresso.org/Doc/INPUT_PW.html#idm1108)等参数。由于后续需要进行phonon计算，最好在relax过程中将相应的计算精度设置的高一些，否则容易在phonon计算中出现虚频，包括[**`conv_thr`**](https://www.quantum-espresso.org/Doc/INPUT_PW.html#idm771)、[**`etot_conv_thr`**](https://www.quantum-espresso.org/Doc/INPUT_PW.html#idm117)、[**`forc_conv_thr`**](https://www.quantum-espresso.org/Doc/INPUT_PW.html#idm123)参数。对于二维材料，在计算中可以使用参数[**`assume_isolated='2D'`**](https://www.quantum-espresso.org/Doc/INPUT_PW.html#idm547)  对于低维材料体系（**该参数的设置好像会导致在Wannier拟合中导致WFs虚部太大，可能是由于该参数的设置导致的，在Graphene的能带拟合中，注释掉该参数后，得到的WFs变成实数。**），在优化时对于k点设置要设置的多一点，否则容易导致结构优化的晶格常数不准确，以及phonon计算中的声子谱不收敛。例如：对于石墨烯，需要设置到36\*36\*1,同时对于`smearing`设置成`mp`比较合适，参考[QE_forrum中Graphene的phonon计算的相关内容](https://lists.quantum-espresso.org/pipermail/users/2016-April/034958.html)。并且，`ecutwfc`和`ecutrho`也不能使用[Standard solid-state pseudopotentials(SSP)](https://www.materialscloud.org/discover/sssp/table/efficiency),使用未收敛的截断能，导致计算得到的声子谱在$\Gamma$点出现虚频。  


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

查看优化过程中各原子的受力情况：

```bash
grep -A 20 "Forces acting on atoms (cartesian axes, Ry/au):" vc-relax.out
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

采用A, B, C, cosAB, cosAC, cosBC或者celldm进行vc-relax后，如果优化中设置了`cell_dofree    = "ibrav"`，会输出新的ibrav以及celldm(i)的值。采用下列命令将可以得到新的ibrav参数和celldm(i):  
```bash
grep -B 40 "Begin final coordinates" vc-relax.out
```



```bash
     Energy error            =      1.3E-12 Ry
     Gradient error          =      3.0E-31 Ry/Bohr
     Cell gradient error     =      6.9E-04 kbar
ibrav =      4
 celldm(1) =      4.66232677
 celldm(3) =      8.10636498
Input lattice vectors:
     1.00000020     0.00000000     0.00000000
    -0.50000010     0.86602557     0.00000000
     0.00000000     0.00000000     8.10636657
New lattice vectors in INITIAL alat:
     1.00000020     0.00000000     0.00000000
    -0.50000010     0.86602557     0.00000000
     0.00000000     0.00000000     8.10636657
New lattice vectors in NEW alat (for information only):
     1.00000000     0.00000000     0.00000000
    -0.50000000     0.86602540     0.00000000
     0.00000000     0.00000000     8.10636498
Discrepancy in bohr =     0.000000    0.000000    0.000000

     bfgs converged in   2 scf cycles and   1 bfgs steps
     (criteria: energy <  1.0E-08 Ry, force <  1.0E-07Ry/Bohr, cell <  1.0E-03kbar)

     End of BFGS Geometry Optimization

     Final enthalpy           =     -36.8880451464 Ry

     File ./outdir/graphene.bfgs deleted, as requested
Begin final coordinates
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
如果设置了`cell_dofree    = "ibrav"`，则可以使用下列命令查到新的celldm(i)
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

#### Notes1  

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

QE中Fortran代码如下：

```fortran  
IF(enforce_ibrav) CALL remake_cell( ibrav, alat, at(1,1),at(1,2),at(1,3), new_alat )

SUBROUTINE remake_cell(ibrav, alat, a1,a2,a3, new_alat)
  USE kinds, ONLY : DP
  USE io_global, ONLY : stdout
  IMPLICIT NONE
  INTEGER,INTENT(in) :: ibrav
  REAL(DP),INTENT(in)  :: alat
  REAL(DP),INTENT(out) :: new_alat
  REAL(DP),INTENT(inout) :: a1(3),a2(3),a3(3)
  REAL(DP) :: e1(3), e2(3), e3(3)
  REAL(DP) :: celldm_internal(6), lat_internal, omega
  ! Better not to do the following, or it may cause problems with ibrav=0 from input
!  ibrav = at2ibrav (a(:,1), a(:,2), a(:,3))
  ! Instead, let's print a warning and do nothing:
  IF(ibrav==0)THEN
    WRITE(stdout,'(a)') "WARNING! With ibrav=0, cell_dofree='ibrav' has no effect. "
    RETURN
  ENDIF
  !
  CALL  at2celldm (ibrav,alat,a1, a2, a3,celldm_internal)
  WRITE(stdout,'("ibrav = ",i6)') ibrav
  WRITE(stdout,'(" celldm(1) = ",f15.8)') celldm_internal(1)
  IF( celldm_internal(2) /= 0._dp) WRITE(stdout,'(" celldm(2) = ",f15.8)') celldm_internal(2)
  IF( celldm_internal(3) /= 0._dp) WRITE(stdout,'(" celldm(3) = ",f15.8)') celldm_internal(3)
  IF( celldm_internal(4) /= 0._dp) WRITE(stdout,'(" celldm(4) = ",f15.8)') celldm_internal(4)
  IF( celldm_internal(5) /= 0._dp) WRITE(stdout,'(" celldm(5) = ",f15.8)') celldm_internal(5)
  IF( celldm_internal(6) /= 0._dp) WRITE(stdout,'(" celldm(6) = ",f15.8)') celldm_internal(6)
```  

如果vc-relax中没有要求优化中ibrav不变，则不会输出ibrav以及celldm(i)

#### Notes2  

对于类似于石墨烯超胞等原子应该满足严格的分数坐标时，采用vc-relax可能导致原子位置偏移。这是可以在vc-relax计算中，在[&IONS](https://www.quantum-espresso.org/Doc/INPUT_PW.html#idm882)中设置参数：[trust_radius_min](https://www.quantum-espresso.org/Doc/INPUT_PW.html#idm1044)、[trust_radius_ini](https://www.quantum-espresso.org/Doc/INPUT_PW.html#idm1049)等参数，以降低优化过程中原子的移动幅度。对优化后的结构，导入MS查看，可以更好的指导晶体结构的对称性。

#### Notes3  

对于超胞的石墨烯结构，由于在优化过程中设置了`cell_dofree    = "ibrav"`，会导致原子的受力总是对称的，并且不为零。

```bash

```
