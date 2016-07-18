---
title: "__isnanf引发的血案"
date: 2016-07-18 15:41:25
categories:
tags: [android, ndk]
---

最近遇到了4.x版本 android 上由**__isnanf**引发的血案，过程就不表了。典型出错日志(4.2.2)：
```bash
07-18 11:58:20.445 W/ExceptionHandler(15303): Caused by: java.lang.UnsatisfiedLinkError: Cannot load library: soinfo_relocate(linker.cpp:975): cannot locate symbol "__isnanf" referenced by "libmylibrary.so"...
```

在4.2.2版本的 [bionic c](http://androidxref.com/4.2.2_r1/xref/bionic/libm/include/math.h) 中：
```c
#define	isnan(x)					\
    ((sizeof (x) == sizeof (float)) ? isnanf(x)		\
    : (sizeof (x) == sizeof (double)) ? isnan(x)	\
    : __isnanl(x))
```
在后来的版本中，以[5.1.1](http://androidxref.com/5.1.1_r6/xref/bionic/libm/include/math.h)为例，它变成了：
```c
#define	isnan(x)					\
   ((sizeof (x) == sizeof (float)) ? __isnanf(x)	\
    : (sizeof (x) == sizeof (double)) ? isnan(x)	\
    : __isnanl(x))
```

我们看下 ndk 中的头文件(以r10e)为例: platform-21 中使用的是 __isnanf，platform-21 以下版本的是 isnanf。

有此可见在低版本的 android 上 __isnanf 不是一定存在的，使用它是不安全的，同时由于 ndk 的不同版本所包含的 libm.so 并没有与具体的 platform上的 libm.so 保持一致，在链接时并不能发现这一问题。所以针对这个问题，有两个方面要注意：

1) 编译完动态库之后用如下命令查看下动态符号表，确认是否引用了 __isnanf 符号：
```bash
arm-linux-androideabi-objdump -T {your_library}
```

2）如果你觉得1) 这种方式麻烦，你可以采用如下方式将 isnan 转为安全的 __builtin_isnan
```c
#include <math.h>
+#undef isnan
+#define isnan(x) __builtin_isnan(x)
```

在调查以上问题时，还 get 到了一个很偏门的 c 语言小知识：
在一些 libc 头文件会以如下方式(函数名外边套一对圆括号)声明函数：
```c
int (isnan)(double) __NDK_FPABI_MATH__ __pure2;
```

其目的在于抑制宏替换，示例如下：

```c
/* the macro */
#define isdigit(c) ...

/* the function */
int (isdigit)(int c) /* avoid the macro through the use of parentheses */
{
  return isdigit(c); /* use the macro */
}
```

机理请见下文：
>7.1.4 Use of library functions

>Any function declared in a header may be additionally implemented as a function-like macro deﬁned in the header, so if a library function is declared explicitly when its header is included, one of the techniques shown below can be used to ensure the declaration is not affected by such a macro. Any macro deﬁnition of a function can be suppressed locally by enclosing the name of the function in parentheses, because the name is then not followed by the left parenthesis that indicates expansion of a macro function name. For the same syntactic reason, it is permitted to take the address of a library function even if it is also deﬁned as a macro.