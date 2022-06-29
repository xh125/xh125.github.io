---
layout:     post
title:      "QUANTUM ESPRESSO：用GRID进行phonon计算"
date:       2022-06-29 15:10:00
author:     "Xiehua"
header-img: "images/post/post-bg-understand.jpg"
tags:
    - QUANTUM ESPRESSO
    - DFPT
    - phonon
---  

## QE中用GRID进行phonon计算  

   1. 第一步，scf计算
   2. 第二步，A preparatory phonon run with start_irr=0, last_irr=0 calculates the displacement patterns:

   ```fortran
   ph0.in  phonons calculation
   &inputph
   tr2_ph=1.0d-16,
   prefix='carbyne',
   outdir='./'
  !  epsil=.true. !use for insulators
   ldisp=.true.
   nq1=10, nq2=1, nq3=1
   start_irr = 0
   last_irr  = 0
  !  amass(1) = 0.0
   fildyn='carbyne.dyn'
   fildvscf='dvscf'
  /
   ```  

   3. 第三步：ph.x is run for each representation of each q point.
   The code runs with different outdir and only the xml files are copied  in the same outdir (input input.#q.#irr, output output.#q.#irr)
