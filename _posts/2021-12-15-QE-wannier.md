---
layout:     post
title:      "QUANTUM ESPRESSO：wannier"
date:       2021-12-15 16:56:00
author:     "Xiehua"
header-img: "images/post/post-bg-understand.jpg"
tags:
    - QUANTUM ESPRESSO
    - wannier
---

## QUANTUM ESPRESSO：[Wannier90拟合](http://www.wannier.org/)

从bandfat数据分析，在带隙附近的能带主要由C原子px,py轨道贡献  
进行wannier计算的步骤如下：

1. 用pw.x运行‘scf’计算
2. 用pw.x进行'nscf'计算，需要列出所有k点的坐标，和权重，使用kmesh.pl生成

   ```bash
   cp scf.in nscf.in

   calculation = 'nscf'
   nbnd = 22
   !K_POINTS{automatic}
   !1 1 200 0 0 0

   kmesh.pl 1 1 200 >>nscf.in
   pw.x <nscf.in>nscf.out
   ```

3. 运行wannier90.x -pp (预处理pre-process，或在输入文件内写postproc_setup = .true.)生成seedname.nnkp。  
  
    - 构建输入文件carbyne.win  

      ```fortran
      begin unit_cell_cart
      Ang
       10.000000   0.000000   0.000000
        0.000000  10.000000   0.000000
        0.000000   0.000000   2.56551169
      end unit_cell_cart
      
      begin atoms_cart
      Ang
      C       5.0000000000       5.0000000000       0.0000000000
      C       5.0000000000       5.0000000000       1.2640744811
      end atoms_cart
      
      ```

    - 添加k-points信息  

     ```bash
     echo "mp_grid = 1 1 200">>${seedname}.win
     echo "begin kpoints">>${seedname}.win
     kmesh.pl 1 1 200 wannier >>${seedname}.win
     echo "end kpoints">>${seedname}.win
     ```

    - 添加wannier拟合参数

    ```fortran
    !System
    num_wann  = 4
    num_bands = 20
    
    !Projection
    begin projections
    C : px;py
    end projections
    
    !Job Control
    exclude_bands : 1-2
    !restart =
    
    !disentanglement
    dis_win_min  = -10.0
    !dis_win_max  = 12.0
    dis_froz_min = -11.0
    dis_froz_max = -0.5
    dis_num_iter = 1000
    dis_mix_ratio= 0.5
    dis_conv_tol = 1.0E-10
    dis_conv_window = 5
    
    !Wannierise
    num_iter = 10000
    conv_tol = 1.0E-10
    conv_window     = 10
    guiding_centres = .true.
    ```

    - 运行wannier90.x -pp seedname

