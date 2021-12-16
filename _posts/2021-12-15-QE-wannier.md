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
      ! System
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
    !! Atom-centred px,py orbitals
    !C : px;py
    !! Bond-centred px,py-orbitals
    f=0.5,0.5,0.25:px;py
    f=0.5,0.5,0.75:px;py
    end projections
    
    !Job Control
    exclude_bands : 1-2
    translate_home_cell = .true.
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
    
    !Post-Processing
    !restart = plot
    wannier_plot = .true.
    wannier_plot_format = xcrysden
    wannier_plot_supercell = 1 1 3

    bands_plot = .true.
    bands_num_points = 100
    bands_plot_format = gnuplot
    begin kpoint_path
        M 0.0 0.0 -0.5 G 0.0 0.0 0.0
        G 0.0 0.0  0.0 M 0.0 0.0 0.5
    end kpoint_path

    fermi_surface_plot = .true.
    fermi_surface_num_points = 100
    fermi_energy = -5.262

    write_hr=.true.
    write_rmn=.true.
    write_tb=.true.
    !end plot
    ```

    - 运行mpirun -np $NP wannier90.x -pp seedname

    **NOTE:** 其结果中包含有初始指定的projections函数(以alat为单位)：
    ! convert wannier center in cartesian coordinates (in unit of alat)

4. Run pw2wannier90 to compute the overlap between Bloch states and the projections for the
starting guess (written in the seedname.mmn and seedname.amn files).  

    `pw2wannier90.x < pw2wan.in > pw2wan.out`

    输入文件`pw2wan.in`如下：  

    ```fortran
    &inputpp
    outdir = './outdir'
    prefix = 'carbyne'
    seedname = 'carbyne'
    spin_component = 'none'
    write_mmn = .true.
    write_amn = .true.
    write_unk = .true.
    /
    ```  

    要在后续wannier90.x使用plot画MLWF函数，需要设置`write_unk=.true.`

    **NOTE**可能会遇到报错

    ```bash
     %%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%
     Error in routine  fft_type_set (6):
    there are processes with no planes. Use pencil decomposition (-pd .true.)
    %%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%
    ```

    需要使用`mpirun -np $NP pw2wannier90.x <pw2wan.in> pw2wan.out`并行计算，并且$NP太大时会导致部分核分配不到平面波，需要适当减小`\$NP`的值


5. Run wannier90 to compute the MLWFs.  
   `mpirun -np 28 wannier90.x seedname`  
   并在seedname.win中添加plot部分  

   ```fortran
   !restart = plot
   wannier_plot = .true.
   wannier_plot_format = xcrysden
   wannier_plot_supercell = 1 1 3
   
   bands_plot = .true.
   bands_num_points = 100
   bands_plot_format = gnuplot
   begin kpoint_path
           M 0.0 0.0 -0.5 G 0.0 0.0 0.0
           G 0.0 0.0  0.0 M 0.0 0.0 0.5
   end kpoint_path
   
   fermi_surface_plot = .true.
   fermi_surface_num_points = 100
   fermi_energy = -5.262
   
   write_hr=.true.
   write_rmn=.true.
   write_tb=.true.
   !end plot

   ```  

计算得到的wannier插值的能带图如下：  
![Wannier-band](https://xh125.github.io/images/post/wannier-band.png)
