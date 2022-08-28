---
layout:     post
title:      "修改EPW输出复数形式的gmnvkq"
date:       2022-08-24 14:56:00
author:     "Xiehua"
header-img: "images/post/post-bg-understand.jpg"
tags:
    - QE
    - gmnvkq
    - LVCSH
---
## 修改printing.f90中的subroutine print_gkk，输出复数形式的$g_{mnv}(k,q)$

**要求：** 不需要根据简并特点求平均，为了得到能量量纲的电声耦合大小，输出的值为：$\sqrt{\hbar/2\omega_{qv}}g_{mnv}(k,q)$

对`printing.f90`做如下修改：

```fortran
    USE constants_epw, ONLY : ryd2mev, ryd2ev, two, zero, czero    ! Line 32

    complex(kind=dp),allocatable :: epc_cmp(:,:,:,:)   ! Line 92
    !! complex gmnvkq-vertex with out "SYMMETRIZE"
    !! "SYMMETRIZE": actually we simply take the averages over
    !! degenerate states, it is only a convention because g is gauge-dependent!    

    allocate(epc_cmp(nbndfst,nbndfst,nmodes,nktotf),STAT = ierr)  ! Line 104
    IF (ierr /= 0) CALL errore('print_gkk', 'Error allocating epc_cmp', 1)

    epc_cmp(:,:,:,:) = czero ! Line 109

    ! First do the average over bands and modes for each pool
    DO ik = 1, nkf
      ikk = 2 * ik - 1
      ikq = ikk + 1
      !
      DO nu = 1, nmodes
        wq = wf(nu, iq)
        DO ibnd = 1, nbndfst
          DO jbnd = 1, nbndfst
            gamma = (ABS(epf17(jbnd, ibnd, nu, ik)))**two
            IF (wq > 0.d0) THEN
              gamma = gamma / (two * wq)
              epc_cmp(ibnd, jbnd, nu, ik + lower_bnd - 1) = epf17(jbnd, ibnd, nu, ik)/sqrt(two * wq)
            ELSE
              gamma = 0.d0
              epc_cmp(ibnd, jbnd, nu, ik + lower_bnd - 1) = czero
            ENDIF
            gamma = DSQRT(gamma)
            ! gamma = |g| [Ry]
            epc(ibnd, jbnd, nu, ik + lower_bnd - 1) = gamma
          ENDDO ! jbnd
        ENDDO   ! ibnd
      ENDDO ! loop on modes
      !


      CALL mp_sum(epc_cmp, inter_pool_comm )  !Line 214

    ! Only master writes
    IF (mpime == ionode_id) THEN
      !
      WRITE(stdout, '(5x, a)') ' Electron-phonon vertex |g| (meV)'
      !
      WRITE(stdout, '(/5x, "iq = ", i7, " coord.: ", 3f12.7)') iq, xqf(:, iq)
      DO ik = 1, nktotf
        !
        ikk = 2 * ik - 1
        ikq = ikk + 1
        !
        WRITE(stdout, '(5x, "ik = ", i7, " coord.: ", 3f12.7)') ik, xkf_all(:, ikk)
        WRITE(stdout, '(5x, a)') ' ibnd     jbnd     imode   enk[eV]    enk+q[eV]  &
        &  omega(q)[meV]      |g|[meV]      Re[gmnvkq][meV]     Im[gmnvkq][meV]'
        WRITE(stdout, '(5x, a)') REPEAT('-', 118)
        !
        DO ibnd = 1, nbndfst
          ekk = etf_all(ibndmin - 1 + ibnd, ikk)
          DO jbnd = 1, nbndfst
            ekq = etf_all(ibndmin - 1 + jbnd, ikq)
            DO nu = 1, nmodes
              WRITE(stdout, '(3i9, 2f12.6, 1f20.10, 3e20.10)') ibndmin - 1 + ibnd, ibndmin - 1 + jbnd, &
                   nu, ryd2ev * ekk, ryd2ev * ekq, ryd2mev * wf(nu, iq), ryd2mev * epc(ibnd, jbnd, nu, ik), &
                   ryd2mev*epc_cmp(ibnd, jbnd, nu, ik)
            ENDDO
          ENDDO
          !
        ENDDO
        WRITE(stdout, '(5x, a/)') REPEAT('-', 118)
        !
      ENDDO
    ENDIF ! master node
    !
    DEALLOCATE(epc, STAT = ierr)
    IF (ierr /= 0) CALL errore('print_gkk', 'Error deallocating epc', 1)
    DEALLOCATE(epc_sym, STAT = ierr)
    IF (ierr /= 0) CALL errore('print_gkk', 'Error deallocating epc_sym', 1)
    deallocate(epc_cmp,stat = ierr)
    IF (ierr /= 0) CALL errore('print_gkk', 'Error deallocating epc_sym', 1)
    !
```

### 修改后代码以及软件编译参考：[QUANTUM ESPRESSO v7.1 修改版安装][10]

**Note:** 如遇到问题，欢迎在评论区留言。评论系统采用了[Disqus系统][1]，需要翻墙才能加载。

[1]:https://disqus.com/
[10]:https://xh125.github.io/2022/07/01/QE-v7.1-install/