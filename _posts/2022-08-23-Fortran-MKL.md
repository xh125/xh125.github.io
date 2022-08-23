---
layout:     post
title:      "Fortran中使用Inetl MKL数学库"
date:       2022-08-23 14:56:00
author:     "Xiehua"
header-img: "images/post/post-bg-understand.jpg"
tags:
    - Fortran
    - Intel MKL
---

## Fortran中使用Intel MKL数学库

### Linux中使用MKL库

```makefile
#MKL libraries
MKLLIB           = ${MKLROOT}/lib/intel64
MKLINCLUDE       = -I${MKLROOT}/include -I${MKLROOT}/include/intel64/lp64 
BLAS95_LIB       = ${MKLLIB}/libmkl_blas95_lp64.a
LAPACP95_LIB     = ${MKLLIB}/libmkl_lapack95_lp64.a
BLACS_LIBS       = ${MKLLIB}/libmkl_blacs_intelmpi_lp64.a
SCALAPACK_LIBS   = ${MKLLIB}/libmkl_scalapack_lp64.a 
lp64_lib         = ${MKLLIB}/libmkl_intel_lp64.a
thread_lib       = ${MKLLIB}/libmkl_intel_thread.a
sequential_lib   = ${MKLLIB}/libmkl_sequential.a
core_lib         = ${MKLLIB}/libmkl_core.a
MKL_dynamic_Link = ${BLAS95_LIB} ${LAPACP95_LIB} -L${MKLLIB} \
                   -lmkl_intel_lp64 -lmkl_intel_thread -lmkl_core \
                   -liomp5 -lpthread -lm -ldl
MKL_static_Link  = ${BLAS95_LIB} ${LAPACP95_LIB} -Wl,--start-group \
                   ${lp64_lib} ${sequential_lib} ${core_lib} \
	       	         -Wl,--end-group \
		               -liomp5 -lpthread -lm -ldl


#---------------------------------------------------------------
# Redefining Pattern Rules
#---------------------------------------------------------------
%.o : %.f90
	$(F90) $(F90FLAGS) ${MKLINCLUDE} -c $<

%.o : %.c
	$(CC) $(CFLAGS) -c $<

%.fh : %.h
	$(CPP) $(CPPFLAGS) $< -o $*.fh


.PHONY: all

all: $(exe)

##
${exe} : ${OBJS}	bindir tools
#	${F90} -o ${exe} ${OBJS} ${MKL_dynamic_Link}
	${F90} -o ${exe} ${OBJS} ${MKL_static_Link}
	cp -f ${exe} ../bin             
```

### Window中在Visual Studio中使用Intel MKL库

- 设置运行时库（Runtime Library）
- 设置Intel MKL

![VS-MKL][2]

- 使用BLAS95和LAPACK95库
	在Additional Dependencies 中添加mkl_lapack95.lib mkl_blas95.lib
	或者（mkl_lapack95_ilp64.lib mkl_blas95_ilp64.lib）
![VS-MKL95][3]




[2]:https://xh125.github.io/images/Fortran/VS-MKL.png
[3]:https://xh125.github.io/images/Fortran/VS-MKL95.png
