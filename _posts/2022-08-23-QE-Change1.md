---
layout:     post
title:      "修改QE输出wannier90.x可用的cell和ionic positions坐标"
date:       2022-08-23 14:56:00
author:     "Xiehua"
header-img: "images/post/post-bg-understand.jpg"
tags:
    - QE
    - Wannier
---

## 修改QE输出wannier90.x可用的cell和ionic positions坐标

**修改源文件：** /qe/pw/src/summary.f90

对于QE7.1版本：
在225-243之间（`Call print_ps_info ( )`之前）添加如下代码：

```fortran
  !
  ! ... print cell and ionic positions for wannier90.x
  !
  write(stdout,"(A)") "Begin Write cell and positions for Wannier90.x"
  write(stdout,"(A)") "begin unit_cell_cart"
  write(stdout,"(A)") "Bohr"
  write(stdout,"(5X,3f12.7)") ((alat*at (ipol, apol) , ipol = 1, 3) , apol = 1, 3)
  write(stdout,"(A)") "end unit_cell_cart"
  write(stdout,"(/,A)") "begin atoms_cart"
  write(stdout,"(A)") "Bohr"
  WRITE( stdout, '(5x,a6,6x,3f12.7)') &
             ( atm(ityp(na)), (alat*tau(ipol,na), ipol=1,3), na=1,nat)  
  write(stdout,"(A)") "end atoms_cart"
  write(stdout,"(A)") "End Write cell and positions for Wannier90.x"
  !
  ! ...end print cell and ionic positions for wannier90.x
  ! 
```

![summary-change][1]

[修改前的代码][2]
[修改后的代码][3]

采用修改后的代码可以直接得到能用于Wannier90.x计算用得结构信息：

```bash
awk '/Begin Write/,/End Write/' scf.out|awk '/begin unit/,/end atoms/'>>carbyne.win
```

```bash
Begin Write cell and positions for Wannier90.x
begin unit_cell_cart
Bohr
    4.8484202   0.0000000   0.0000000
    0.0000000  28.3458919   0.0000000
    0.0000000   0.0000000  28.3458919
end unit_cell_cart

begin atoms_cart
Bohr
    C          -0.0000026  14.1729459  14.1729459
    C           2.3890085  14.1729459  14.1729459
end atoms_cart
End Write cell and positions for Wannier90.x
```

[1]:https://xh125.github.io/images/QE-change/summary.png
[2]:https://github.com/xh125/LVCSH-new/blob/main/docs/QE_change_code/QE_change_code/v7.1/Originalcode/PW/src/summary.f90
[3]:https://github.com/xh125/LVCSH-new/blob/main/docs/QE_change_code/QE_change_code/v7.1/PW/src/summary.f90  
