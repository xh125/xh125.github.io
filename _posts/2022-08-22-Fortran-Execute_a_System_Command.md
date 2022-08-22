---
layout:     post
title:      "Fortran调用系统命令"
date:       2022-08-22 21:56:00
author:     "Xiehua"
header-img: "images/post/post-bg-understand.jpg"
tags:
    - Fortran
    - Execute a System Command
---

## Fortran中调用系统命令/其它可执行文件

**转载自：**[博客园https://www.cnblogs.com/erichf/p/13905140.html][0]

fortran内调用系统命令/其它可执行文件时，有两种方法：

- 在fortran 2008中新增内部subroutine程序： `execute_command_line`  (参考：《Modern Fortran explained incorporating Fortran 2018》中的9.18.4)
- 使用`subroutine system` 或者 `function system`（不同系统不一样？ ），对intel fortran 中可使用 `systemqq`,`system()`是一个非标准扩展，但它也是我到目前为止使用的所有编译器的内在过程。同样，不必使用任何模块来调用它。 `system()`既可以用作函数，也可以用作子程序，具体取决于编译器。
例如：在Sun Studio中，system为一个function，使用方法是：[参考：https://docs.oracle.com/cd/E19957-01/805-4942/6j4m3r90i/index.html][1]

```fortran
    character*8 string / 'ls s*' /
    INTEGER*4 status, system
    status = system( string )
    if ( status .ne. 0 ) stop 'system: error'
    end
```

在gfortran中，The SYSTEM subroutine (and function) are a GNU extension.

```fortran
program SystemTest
  call system("ls")
end program SystemTest
```

推荐使用`execute_command_line`，使用时，注意在命令前后加上双引号

```fortran
program SystemTest
integer :: i
 call execute_command_line ("ls", exitstat=i)
end program SystemTest
```

- 参考：[Execute a system command][2]

- 需要输出/输出文件时，可以使用重定向：

```fortran
call system('aa.exe < bb.txt')
call system('aa.exe > bb.txt')
```

- 其他调用形式：

```fortran
call system("start /wait name.exe")
```

ifport 是 Intel Fortran 提供的一个扩展模块，包含各种面向操作系统的程序接口。例如各种命令行操作，文件夹操作，多文件通配符操作（列举文件夹下所有文件）等等。最后要注意的是，一旦你 use ifport 了，你的代码就只能在 Intel Fortran 上运行了。

### fortran：使用内部函数fdate得到系统时间
```fortran
program main
        use ifport, only : fdate
        implicit none
        
        print*, fdate()
end program main
```

[0]:https://www.cnblogs.com/erichf/p/13905140.html
[1]:https://docs.oracle.com/cd/E19957-01/805-4942/6j4m3r90i/index.html
[2]:https://rosettacode.org/wiki/Execute_a_system_command#Fortran
