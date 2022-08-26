---
layout:     post
title:      "QE代码的bug（费米面下拟合能带电子数nelec错误）"
date:       2022-08-24 14:56:00
author:     "Xiehua"
header-img: "images/post/post-bg-understand.jpg"
tags:
    - QE
    - Ef
    - LVCSH
    - Wannier
---

## QE代码的bug（费米面下拟合能带电子数错误）

EPW计算的`Ef`有问题：只有在wannier拟合时，project的能带加`exclude_bands`能带包含了费米能以下的所有能带，在插值时才能得到正确的费米能级。可以选择将费米面下不需要拟合的能带全部排除出去，或者将不能排除出去的能带全部拟合。该问题是由于原始的在ephwann_shuffle.f90和ephwann_shuffle_mem.f90代码中对于`nelec`不能正确计算导致。

```fortran
  WRITE(stdout,'(/5x,a,f10.6,a)') 'Fermi energy coarse grid = ', ef * ryd2ev, ' eV'
  !
  IF (efermi_read) THEN
    !
    ef = fermi_energy
    WRITE(stdout,'(/5x,a)') REPEAT('=',67)
    WRITE(stdout, '(/5x,a,f10.6,a)') 'Fermi energy is read from the input file: Ef = ', ef * ryd2ev, ' eV'
    WRITE(stdout,'(/5x,a)') REPEAT('=',67)
    !
    ! SP: even when reading from input the number of electron needs to be correct
    already_skipped = .FALSE.
    IF (nbndskip > 0) THEN
      IF (.NOT. already_skipped) THEN
        IF (noncolin) THEN
          nelec = nelec - one * nbndskip
        ELSE
          nelec = nelec - two * nbndskip
        ENDIF
        already_skipped = .TRUE.
        WRITE(stdout, '(/5x,"Skipping the first ", i4, " bands:")') nbndskip
        WRITE(stdout, '(/5x,"The Fermi level will be determined with ", f9.5, " electrons")') nelec
      ENDIF
    ENDIF
    !
  ELSEIF (band_plot) THEN
    !
    WRITE(stdout, '(/5x,a)') REPEAT('=',67)
    WRITE(stdout, '(/5x,"Fermi energy corresponds to the coarse k-mesh")')
    WRITE(stdout, '(/5x,a)') REPEAT('=',67)
    !
  ELSE
    ! here we take into account that we may skip bands when we wannierize
    ! (spin-unpolarized)
    ! RM - add the noncolin case
    already_skipped = .FALSE.
    IF (nbndskip > 0) THEN
      IF (.NOT. already_skipped) THEN
        IF (noncolin) THEN
          nelec = nelec - one * nbndskip
        ELSE
          nelec = nelec - two * nbndskip
        ENDIF
        already_skipped = .TRUE.
        WRITE(stdout, '(/5x,"Skipping the first ", i4, " bands:")') nbndskip
        WRITE(stdout, '(/5x,"The Fermi level will be determined with ", f9.5, " electrons")') nelec
      ENDIF
    ENDIF
    !
    ! Fermi energy
    !
    ! Since wkf(:,ikq) = 0 these bands do not bring any contribution to Fermi level
    IF (ABS(degaussw) < eps16) THEN
      ! Use 1 meV instead
      efnew = efermig(etf, nbndsub, nkqf, nelec, wkf, 1.0d0 / ryd2mev, ngaussw, 0, isk_dummy)
    ELSE
      efnew = efermig(etf, nbndsub, nkqf, nelec, wkf, degaussw, ngaussw, 0, isk_dummy)
    ENDIF
    !
    WRITE(stdout, '(/5x,a,f10.6,a)') &
        'Fermi energy is calculated from the fine k-mesh: Ef = ', efnew * ryd2ev, ' eV'
    !
    ! if 'fine' Fermi level differs by more than 250 meV, there is probably something wrong
    ! with the wannier functions, or 'coarse' Fermi level is inaccurate
    IF (ABS(efnew - ef) * ryd2eV > 0.250d0 .AND. (.NOT. eig_read)) &
      WRITE(stdout,'(/5x,a)') 'Warning: check if difference with Fermi level fine grid makes sense'
    WRITE(stdout,'(/5x,a)') REPEAT('=',67)
    !
    ef = efnew
  ENDIF
```

其中nelec需要使用wannier拟合的能带中包含的电子占据数代替。
该问题容易导致计算得到的插值空间中的Ef是错误的，同时可能会导致如下报错：

```txt
Warning: check if difference with Fermi level fine grid makes sense
```

我们通过引入新的参数`nbndnfbfe`进行修正。

- 1. 在`epwcom.f90`中添加变量`nbndnfbfe`

```fortran
integer :: nbndnfbfe
!! number of band not wannier fitting below fermi energy
```

![nelec-1][0]

- 2. 在`epw_readin.f90`中添加对变量`nbndnfbfe`的设置
  - 2.1 在`USE epwcom, only :` 中添加`nbndnfbfe`
  ![nelec-2.png][1]
  - 2.2 在`NAMELIST / inputepw /`中添加`nbndnfbfe`
  ![nelec-3.png][2]
  - 2.3 在607行左右设置`nbndnfbfe`的初始值为`0`
  ![nelec-4.png][3]
  - 2.4 在1000行左右，将`nbndnfbfe`的值传递给所有核。
  `call mp_bcast(nbndnfbfe,meta_ionode_id,world_comm)`
  ![nelec-5.png][4]

- 3. 在`ephwann_shuffle.f90`和`ephwann_shuffle_mem.f90`中添加`nbndnfbfe`,并根据`nbndnfbfe`对`nelec`进行修正。

  - 3.1 在`ephwann_shuffle.f90`和`ephwann_shuffle_mem.f90`的`USE epwcom, only :` 中添加`nbndnfbfe`
  ![nelec-6.png][5]
  - 3.2 在`ephwann_shuffle.f90`和`ephwann_shuffle_mem.f90`的下列语句之间添加对`nelec`的修正：
  
  ```fortran
  WRITE(stdout,'(/5x,a,f17.10,a)') 'Fermi energy coarse grid = ', ef * ryd2ev, ' eV'	
  !
  IF (efermi_read) THEN
  ```

  改为：
  
  ```fortran
  WRITE(stdout,'(/5x,a,f17.10,a)') 'Fermi energy coarse grid = ', ef * ryd2ev, ' eV'
  !
  IF (noncolin) THEN
    nelec = nelec - one * nbndnfbfe
  ELSE
    nelec = nelec - two * nbndnfbfe
  ENDIF
  !
  IF (efermi_read) THEN
  ```

 ![nelec-7.png][6]
 ![nelec-8.png][7]

- 4. 则在epw的输入文件中可以使用参数`nbndnfbfe`来指定费米面以下未被Wannier拟合且未被`exclude_bands`的能带条数来避免这个bug。输入文件如下：

```fortran
wannierize = .true.
nbndsub = 2
nbndnfbfe = 3
bands_skipped = 'exclude_bands = 1:2'
```

- 5. 修改后代码以及软件编译参考：[QUANTUM ESPRESSO v7.1 修改版安装][10]

[0]:https://xh125.github.io/images/QE-change/nelec-1.png
[1]:https://xh125.github.io/images/QE-change/nelec-2.png
[2]:https://xh125.github.io/images/QE-change/nelec-3.png
[3]:https://xh125.github.io/images/QE-change/nelec-4.png
[4]:https://xh125.github.io/images/QE-change/nelec-5.png
[5]:https://xh125.github.io/images/QE-change/nelec-6.png
[6]:https://xh125.github.io/images/QE-change/nelec-7.png
[7]:https://xh125.github.io/images/QE-change/nelec-8.png

[10]:https://xh125.github.io/2022/07/01/QE-v7.1-install/
[21]:https://github.com/xh125/LVCSH-new/blob/main/docs/QE_change_code/QE_change_code/v7.1/Originalcode/PW/src/summary.f90
[31]:https://github.com/xh125/LVCSH-new/blob/main/docs/QE_change_code/QE_change_code/v7.1/PW/src/summary.f90  
