---
layout:     post
title:      "QUANTUM ESPRESSO安装"
date:       2021-12-14 16:56:00
author:     "Xiehua"
header-img: "images/post/post-bg-understand.jpg"
tags:
    - QUANTUM ESPRESSO
    - 安装
---

## QUANTUM ESPRESSO的安装  

1. 安装好intel编译器 module load intel/2019  (注意，使用compiler/intel/2019.0.045 编译不能通过)
2. tar -zxvf qe-6.8-ReleasePack.tgz
3. cd qe-6.8
4. ./configure  MPIF90=mpiifort CC=mpiicc --enable-openmp --enable-parallel --with-scalapack=intel

5. make -j20 all  
    - (1)报错：

   ```bash
    make[1]: Entering directory `/public/home/users/fjirsm004e/soft/qe/qe-6.8/install'
    cd ../external/devxlib; \
        if test ! -e configure; then \
        wget "https://gitlab.com/max-centre/components/devicexlib/-/archive/master/   devicexlib-master.tar.gz" -O devxlib.tar.gz || curl "https://gitlab.com/max-centre/   components/devicexlib/-/archive/master/devicexlib-master.tar.gz" -o devxlib.tar.gz ; \
        tar xzf devxlib.tar.gz --strip-components=1 -C . ; \
        rm devxlib.tar.gz ; \
        fi; \
        touch make.inc; \
        make clean; \
        export F90FLAGS=""; \
        ./configure FC=ifort CC=mpiicc \
                    --with-cuda=no \
                    --with-cuda-cc= \
                    --with-cuda-runtime= \
                    --disable-parallel \
                    --enable-cuda-env-check=no; \
        make all
    --2021-08-08 12:18:08--  https://gitlab.com/max-centre/components/devicexlib/-/archive/master/   devicexlib-master.tar.gz
    Resolving gitlab.com (gitlab.com)... failed: Name or service not known.
    wget: unable to resolve host address ‘gitlab.com’
    
    ```  

- 解决方法：将install/extlibs_makefile中的第96行注释，并手动下载devicexlib-master.tar.gz到../external/devxlib并修改文件名为devxlib.tar.gz  

 ```bash
 #wget $(DEVXLIB_URL) -O devxlib.tar.gz || curl $(DEVXLIB_URL) -o devxlib.tar.gz ; \
 mv devicexlib-master.tar.gz devxlib.tar.gz ; \
```  

- (2)make all_currents gwl couple gui  
- (3)make w90 epw  
    需要从官网下载wannier90-3.1.0.tar.gz并拷贝到目录archive中并改名为v3.1.0  
- (4)make want  
  下载源码want-want-2.6.1.tar.gz 拷贝到archive并修改为 want-2.6.1.tar.gz  
- (5)make d3q  
   手动下载d3q-qe6.8-latest.tgz拷贝到archive  
   make d3q  
   报错  
   修改qe-6.8/d3q-1.1.8-qe6.8/src中rand部分的代码  
   vi d3q-1.1.8-qe6.8/src/cmdline_param.f90  
- (6)make gipaw  
  需要手动克隆github安装安装  
  
  ```bash  
   make[1]: Entering directory `/public/home/users/fjirsm004e/soft/qe/qe-6.8/install'
    *** Unable to download qe-gipaw. Test whether curl or wget is installed and working,
    *** if you have direct access to internet. If not, copy into archive/ the file
    *** located here https://github.com/dceresoli/qe-gipaw/archive/6.8.tar.gz
  ```  

从github上目前下载不到该安装包  

从github克隆项目并手动安装  

Build instructions:  
From Quantum-Espresso distribution  

1. Configure and compile QE as usual, then:  
2. make gipaw  

Stand-alone  

1. Configure and compile QE as usual, then:  
2. git clone <https://github.com/dceresoli/qe-gipaw.git>  
3. cd qe-gipaw  
4. ./configure --with-qe-source="QE folder containing make.inc"  
5. make  

- (7)make yambo  
目前不能编译成功
