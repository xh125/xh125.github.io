---
layout:     post
title:      "修改QE输出跃迁偶极矩Vmef"
date:       2022-08-24 14:56:00
author:     "Xiehua"
header-img: "images/post/post-bg-understand.jpg"
tags:
    - QE
    - Vmef
    - LVCSH
---

## 修改QE中EPW的print部分输出跃迁偶极矩矩阵

在EPW的输出文件中添加在稠密倒空间中插值的跃迁偶极矩矩阵元到输出文件，用于进行光激发初始电子和空穴状态的设置以及后续用于光学性质的计算。

需要修改的文件包括`printing.f90`以及`ephwann_shuffle.f90`

### 1.printing.f90中添加`subroutine print_sh()`

```fortran
    !-----------------------------------------------------------------------
    subroutine print_sh()
    !-----------------------------------------------------------------------
    !---------------------------------------------------------------------------------
    ! Added by xiehua, used to print the vmef of dmef to the output.
    !---------------------------------------------------------------------------------    
    ! Write dmef or vmef to the files
    ! vmef is in units of Ryd * bohr
    ! dmef is in units of 1/a.u. (where a.u. is bohr)
    ! v_(k,i) = 1/m <ki|p|ki> = 2 * dmef (:, i,i,k)
    ! 1/m  = 2 in Rydberg atomic units    
    ! vmef = 2 *dmef
    ! ! ... RY for "Rydberg" atomic units (e^2=2, m=1/2, hbar=1)   
    !Lets gather the velocities from all pools
    USE kinds,         ONLY : DP
    USE io_global,     ONLY : stdout
    USE modes,         ONLY : nmodes
    USE epwcom,        ONLY : nbndsub,vme
    USE elph2,         ONLY : etf, ibndmin,ibndmax, nkqf, xqf, nbndfst,     &
                              nkf, epf17, xkf, nkqtotf, wf, nktotf, nqtotf, &
                              vmef, dmef
    USE constants_epw, ONLY : ryd2mev, ryd2ev, two, zero, czero
    USE mp,            ONLY : mp_barrier, mp_sum
    USE mp_global,     ONLY : inter_pool_comm
    USE mp_world,      ONLY : mpime
    USE io_global,     ONLY : ionode_id
    USE division,      ONLY : fkbounds
    USE poolgathering, ONLY : poolgather2,poolgatherc4  
    !USE polaron,       ONLY : epfall
    !
    ! Local variables
    INTEGER :: lower_bnd
    !! Lower bounds index after k or q paral
    INTEGER :: upper_bnd
    !! Upper bounds index after k or q paral
    INTEGER :: ik
    !! K-point index
    INTEGER :: ikk
    !! K-point index
    INTEGER :: ikq
    !! K+q-point index
    integer :: iq
    !! q-point index
    INTEGER :: ibnd
    !! Band index
    INTEGER :: jbnd
    !! Band index
    INTEGER :: pbnd
    !! Band index
    INTEGER :: nu
    !! Mode index
    INTEGER :: mu
    !! Mode index
    INTEGER :: n
    !! Number of modes
    INTEGER :: ierr
    !! Error status
    REAL(KIND = DP) :: ekk
    !! Eigenenergies at k
    
    REAL(KIND = DP) :: xkf_all(3, nkqtotf)
    !! Collect k-point coordinate from all pools in parallel case
    REAL(KIND = DP) :: etf_all(nbndsub, nkqtotf)
    !! Collect eigenenergies from all pools in parallel case    
    COMPLEX(KIND = DP), ALLOCATABLE :: dmef_all(:, :, :, :)
    !! dipole matrix elements on the fine mesh among all pools
    COMPLEX(KIND = DP), ALLOCATABLE :: vmef_all(:, :, :, :)
    !! velocity matrix elements on the fine mesh among all pools      

    xkf_all = zero
    etf_all = zero
    
#if defined(__MPI)
      !
      ! Note that poolgather2 works with the doubled grid (k and k+q)
      !
      CALL poolgather2(3,       nkqtotf, nkqf, xkf, xkf_all)
      CALL poolgather2(nbndsub, nkqtotf, nkqf, etf, etf_all)
      !CALL mp_sum(epfall, inter_pool_comm )
      CALL mp_barrier(inter_pool_comm)      
      IF (vme == 'wannier') THEN
        ALLOCATE(vmef_all(3, nbndsub, nbndsub, nkqtotf), STAT = ierr)
        IF (ierr /= 0) CALL errore('printing', 'Error allocating vmef_all', 1)
        vmef_all(:, :, :, :) = czero
        CALL poolgatherc4(3, nbndsub, nbndsub, nkqtotf, 2 * nkf, vmef, vmef_all)
      ELSE
        ALLOCATE(dmef_all(3, nbndsub, nbndsub, nkqtotf), STAT = ierr)
        IF (ierr /= 0) CALL errore('printing', 'Error allocating dmef_all', 1)
        dmef_all(:, :, :, :) = czero
        CALL poolgatherc4(3, nbndsub, nbndsub, nkqtotf, 2 * nkf, dmef, dmef_all)
      ENDIF
  
#else
      xkf_all = xkf
      etf_all_= etf 
      IF (vme == 'wannier') THEN
        ALLOCATE(vmef_all(3, nbndsub, nbndsub, nkqtotf), STAT = ierr)
        IF (ierr /= 0) CALL errore('printing', 'Error allocating vmef_all', 1)
        vmef_all = vmef
      ELSE
        ALLOCATE(dmef_all(3, nbndsub, nbndsub, nkqtotf), STAT = ierr)
        IF (ierr /= 0) CALL errore('printing', 'Error allocating dmef_all', 1)
        dmef_all = dmef
      ENDIF
#endif    
      
      
      ! Only master writes 
      IF (mpime == ionode_id) then
        !---------------------------------------------------------------------
        ! print Enk(nband,nkf)
        !---------------------------------------------------------------------
        write(stdout,"(5X,a)") 'Electron eigenvalues enk[eV]'
          do ik = 1, nktotf
            !
            ikk = 2*ik - 1
            ikq = ikk + 1
            !
            WRITE(stdout, '(5x, "ik = ", i7, " coord.: ", 3f12.7)') ik, xkf_all(:, ikk)
            write(stdout,"(5X,a)") ' iband   enk[eV] '
            do ibnd = 1, nbndfst
              ekk = etf_all(ibndmin - 1 + ibnd, ikk)
              write(stdout,"(i9,f18.10)") ibndmin - 1 + ibnd,ekk*ryd2ev
            enddo
            WRITE(stdout, '(5x, a/)') REPEAT('-', 78)
          enddo
        !-----------------------------------------------------------------------
        !
        !-----------------------------------------------------------------------
        ! print wf(nmodes,nqtotf)
        !-----------------------------------------------------------------------
        write(stdout,"(5X,a)") 'phonon eigenvalues omega(nmodes,nqtotf)[meV]'
        do iq = 1, nqtotf
          WRITE(stdout, '(5x, "iq = ", i7, " coord.: ", 3f12.7)') iq, xqf(:,iq)
          write(stdout,"(5X,a)") ' imode   omega(imode,iq)[meV] '
          do nu = 1, nmodes
            write(stdout,"(i9,f18.10)") nu,ryd2mev*wf(nu,iq)
          enddo
          WRITE(stdout, '(5x, a/)') REPEAT('-', 78)
        enddo
        !-----------------------------------------------------------------------
        !          
        
        
        
        !-----------------------------------------------------------------------
        !---------------------------------------------------------------------------------
        ! Added by xiehua, used to print the vmef of dmef to the output.
        !---------------------------------------------------------------------------------    
        ! Write dmef or vmef to the files
        ! vmef is in units of Ryd * bohr
        ! dmef is in units of 1/a.u. (where a.u. is bohr)
        ! v_(k,i) = 1/m <ki|p|ki> = 2 * dmef (:, i,i,k) 
        ! vmef = 2 *dmef
        ! ! ... RY for "Rydberg" atomic units (e^2=2, m=1/2, hbar=1)   
        !Lets gather the velocities from all pools
        !-----------------------------------------------------------------------
        
        if(vme == 'wannier') then
          write(stdout,"(/5X,a)") 'Velocity matrix elements (vmef) (Ryd*bohr)'
        else
          write(stdout,"(/5X,a)") 'Dipole   matrix elements (dmef) (1/bohr)'
        endif
        do ik = 1,nktotf
          ikk = 2*ik-1
          write(stdout,"(/5x,'ik = ', i7 ,' coord.:',3f12.7)") ik,xkf_all(:,ikk)
          if (vme == 'wannier') then
            write(stdout,"(3A5,2A17,6A20)") "ik","ibnd","jbnd", "Enk[eV]", "Emk[eV]", "Re(v_x)","Im(v_x)", &
                  "Re(v_y)","Im(v_y)","Re(v_z)","Im(v_z)"
          else
            write(stdout,"(3A5,2A17,6A20)") "ik","ibnd","jbnd","Enk[eV]"," Emk[eV]","Re(d_x)","Im(d_x)",&
                  "Re(d_y)","Im(d_y)","Re(d_z)","Im(d_z)"
          endif
          do ibnd=ibndmin,ibndmax
            do jbnd=ibndmin,ibndmax
              if (vme == 'wannier') then
                write(stdout,"(3i5,2f17.10,3(2E20.10))") &
                ik, ibnd , jbnd, etf_all(ibnd,ikk)*ryd2ev,etf_all(jbnd,ikk)*ryd2ev,&
                vmef_all(:,ibnd,jbnd,ikk)
              else 
                write(stdout,"(3i5,2f17.10,3(2E20.10))") &
                ik, ibnd , jbnd , etf_all(ibnd,ikk)*ryd2ev,etf_all(jbnd,ikk)*ryd2ev,&
                dmef_all(:,ibnd,jbnd,ikk)
              endif
            enddo
          enddo
        enddo      
      ENDIF ! master node

      IF (vme == 'wannier') THEN
        deallocate(vmef_all, STAT = ierr)
        IF (ierr /= 0) CALL errore('printing', 'Error deallocating vmef_all', 1)
      ELSE
        deallocate(dmef_all, STAT = ierr)
        IF (ierr /= 0) CALL errore('printing', 'Error deallocating dmef_all', 1)
      ENDIF
      
    !---------------------------------------------------------------------------------    
    !-----------------------------------------------------------------------
    end subroutine print_sh
    !-----------------------------------------------------------------------
```

### 2.ephwann_shuffle.f90中添加`call print_sh()`  

- 78行中添加`print_sh`

    ```fortran
    USE printing,         ONLY : print_gkk, plot_band, plot_fermisurface,print_sh
    ```
    ![][0]

- 1774行添加`if(prtgkk) call print_sh()`

    ```fortran
    IF (.NOT. iterative_bte) CALL transport_coeffs(ef0, efcb)
    ENDIF ! if scattering
    !
    !--------------------------------------------------------------------------------
    ! Added by xiehua, used to print the vmef of dmef to the output.
    !--------------------------------------------------------------------------------   
    ! Write dmef or vmef to the files
    ! vmef is in units of Ryd * bohr
    ! dmef is in units of 1/a.u. (where a.u. is bohr)
    ! v_(k,i) = 1/m <ki|p|ki> = 2 * dmef (:, i,i,k) 
    ! vmef = 2 *dmef
    ! ! ... RY for "Rydberg" atomic units (e^2=2, m=1/2, hbar=1)   
    !Lets gather the velocities from all pools
    if(prtgkk) call print_sh() 

    ! Now deallocate
    DEALLOCATE(epf17, STAT = ierr)
    ```

    ![][1]



[0]:https://xh125.github.io/images/QE-change/shuffle-1.png
[1]:https://xh125.github.io/images/QE-change/shuffle-2.png
[2]:https://github.com/xh125/LVCSH-new/blob/main/docs/QE_change_code/QE_change_code/v7.1/Originalcode/PW/src/summary.f90
[3]:https://github.com/xh125/LVCSH-new/blob/main/docs/QE_change_code/QE_change_code/v7.1/PW/src/summary.f90  

