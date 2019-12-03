# **afl-fuzz.c笔记**

## **Ⅰ、文件引用**
### 【一】自定义头文件  
这部分几个头文件都比较重要，源码都需要看  
```c
#include "config.h"
#include "types.h"
#include "debug.h"
#include "alloc-inl.h"
#include "hash.h"
```  
>* 配置文件，包含各种宏定义，属于通用配置  
>* 类型重定义，一些在 `afl-fuzz.c` 中看不太懂的类型，可以在这里看看是不是有相关定义，比如 `u8` 在源码中经常出现，实际上在这个头文件可以看出 `typedef uint8_t  u8`，所以其对应的类型应该是 `uint8_t` ，对应的是 C99 标准里的无符号字符型  
>* 调试，宏定义各种参数及函数，比如显示的颜色，还有各种自定义的函数，如果改AFL，这些东西相当于没有编译器情况下的 *"高端printf(滑稽脸)"*，比如最常见的 `OKF("We're done here. Have a nice day!\n");` 其中的 `OKF` 就是一个输出代表成功信息的函数  
>* 内存相关，提供错误检查、内存清零、内存分配等常规操作，“内存器的设计初衷不是为了抵抗恶意攻击，但是它确实提供了便携健壮的内存处理方式，可以检查 **use-after-free** 等”  
>* 哈希函数，文件中实现一个参数为 `const void* key, u32 len, u32 seed` 返回为 `u32` 的静态内联函数

### 【二】标准库头文件  
标准库文件，基本跟 Windows 下的 C 编程都差不多，只有个别是Linux C下才会用到的  
```c
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include <errno.h>
#include <signal.h>
#include <dirent.h>
#include <ctype.h>
#include <fcntl.h>
#include <termios.h>
#include <dlfcn.h>
#include <sched.h>
```
>* `<dirent.h>` 文件操作相关头文件，可以参考在 `afl-tmin.c` 中的修改，查看其具体用途  
>* `<sched.h>` 任务调度相关头文件  

### 【三】Linux C编程特有的头文件
```c
#include <sys/wait.h>
#include <sys/time.h>
#include <sys/shm.h>
#include <sys/stat.h>
#include <sys/types.h>
#include <sys/resource.h>
#include <sys/mman.h>
#include <sys/ioctl.h>
#include <sys/file.h>
```
这一部分，引用了一些Linux环境下的特殊头文件，跟上一部分会有一些重叠，但是各司其职，实际编程两边的函数都可以，但是偏向于用这部分。对应Linux环境的头文件位置 `/usr/include/sys/`  

### 【四】文件引用总结
需要深入了解看源代码的头文件是

>#include "types.h"  
>#include "debug.h"  
>#include "alloc-inl.h"  

这三个都需要查看源码，对后续查看、按需修改 **afl源码** 一定都会很有帮助。  

----------------------------------------------------------------------------------------------------------------------------


## **Ⅱ、预备工作：预处理、变量、结构体**  
这一部分，差不多前三百多行左右，到枚举类型结束。
### 【一】环境判断预处理  
1. **Linux环境下独有的宏定义**  
```c
#ifdef __linux__
#  define HAVE_AFFINITY 1
#endif /* __linux__ */
```
在之后代码中，可以用 `ifdef HAVE_AFFINITY` 判断是否是Linux环境，这里的 AFFINITY 是亲和性的意思，跟cpu的亲和性，**与运行性能有关**。

2. **AFL_LIB库处理**
```c
#ifdef AFL_LIB
#  define EXP_ST
#else
#  define EXP_ST static
#endif /* ^AFL_LIB */
```
不常用，当afl被编译成库的时候，用来保证变量的输出。`//A toggle to export some variables`

### 【二】全局变量
整个部分代码块最开始的注释块：

> /* Lots of globals, but mostly for the status UI and other things where it really makes no sense to haul them around as function parameters. */  

可以看出作者很谦虚的指出，这部分的全局变量大部分是跟 UI 展示的状态部分有关的，还有一部分是**没必要**去把他们当作函数参数 ( 又些情况会造成代码冗余 )。  
**这部分从四个方面考虑：**

#### 1.变量类型  
这部分跟头文件 `"types.h"` 息息相关，变量类型不清楚的时候就去那个头文件找答案。  
>u8: `typedef uint8_t  u8;`  
>u32: `typedef uint32_t u32;`  
>u64: 下面代码的意思，我猜测是为了兼容32和64不同结构情况下的定义  
>>```c
>>#ifdef __x86_64__
>>typedef unsigned long long u64;
>>#else
>>typedef uint64_t u64;
>>#endif /* ^__x86_64__ */  
>>```
>s32: `typedef int16_t  s16;`  

#### 2.global变量  
这里作者的变量定义是按代码块的结构写的， **`【此部分每块变量的含义，待补】`**

#### 3.结构体 - 测试用例队列
```c
struct queue_entry {
  u8* fname;//测试用例的文件名  
  u32 len;//输入长度， 

  u8  cal_failed,// Calibration failed? 
      trim_done,// Trimmed?      
      was_fuzzed,//是否已经经过fuzzing  
      passed_det,// Deterministic stages passed?     
      has_new_cov,// Triggers new coverage?           
      var_behavior,// Variable behavior? 
      favored,     // Currently favored? 
      fs_redundant;// Marked as redundant in the fs?   

  u32 bitmap_size, // Number of bits set in bitmap     
      exec_cksum;  // Checksum of the execution trace  

  u64 exec_us,// Execution time (us)
      handicap,// Number of queue cycles behind    
      depth;// Path depth    

  u8* trace_mini;  // Trace bytes, if kept
  u32 tc_ref;      // Trace bytes ref count            

  struct queue_entry *next,//队列下一结点 
                     *next_100;       // 100 elements ahead 

};
```

#### 4.枚举
枚举类型的好处是可读性强，易于使用，作者定义了三部分枚举类型，用来表示自己想要展现的状态。*具体用法会分析后举例说明*:

1. Fuzzing 的状态，比如有地方的用一个数组，需要区分01234···序列代表的含义，就可以用这个代替。这里涉及了很多种不同的 fuzz 种子变异策略，1为转换到32位转换等···
>```c
>enum {
>  /* 00 */ STAGE_FLIP1,
>  /* 01 */ STAGE_FLIP2,
>  /* 02 */ STAGE_FLIP4,
>  /* 03 */ STAGE_FLIP8,
>  /* 04 */ STAGE_FLIP16,
>  /* 05 */ STAGE_FLIP32,
>  /* 06 */ STAGE_ARITH8,
>  /* 07 */ STAGE_ARITH16,
>  /* 08 */ STAGE_ARITH32,
>  /* 09 */ STAGE_INTEREST8,
>  /* 10 */ STAGE_INTEREST16,
>  /* 11 */ STAGE_INTEREST32,
>  /* 12 */ STAGE_EXTRAS_UO,
>  /* 13 */ STAGE_EXTRAS_UI,
>  /* 14 */ STAGE_EXTRAS_AO,
>  /* 15 */ STAGE_HAVOC,
>  /* 16 */ STAGE_SPLICE
>};
>```
比如 `DI(stage_finds[STAGE_FLIP1]), DI(stage_cycles[STAGE_FLIP1]),`其中 **`stage_finds`** 的定义`static u64 stage_finds[32]`是一个大数组，用来保存不同的fuzz stage状态下 **`【?】模式发现【?】待补`**  

2. 状态的值类型，这里是用来辅助、修饰上一个枚举fuzzing状态的，用来识别当前的状态枚举对应变异方式的类型
>```c
>enum {
>  /* 00 */ STAGE_VAL_NONE,
>  /* 01 */ STAGE_VAL_LE,
>  /* 02 */ STAGE_VAL_BE
>};
>```

3. 执行状态（针对错误fault），每个状态都有其特殊含义 **【此部分每个状态的含义，待补】**  
>```c
>enum {
>  /* 00 */ FAULT_NONE,
>  /* 01 */ FAULT_TMOUT,
>  /* 02 */ FAULT_CRASH,
>  /* 03 */ FAULT_ERROR,
>  /* 04 */ FAULT_NOINST,
>  /* 05 */ FAULT_NOBITS
>};
>```

### 【三】预处理总结  
这部分最重要的是对于全局变量四部分的理解，afl-fuzz.c文件最常用的变量都在这里面了，特别是测试用例的队列、枚举代表的含义。  


------------------------------------------------------------------------------------------------------------------


## **Ⅲ、fuzzing 的整体结构**
后面的代码就是真正的 `afl fuzz` 过程代码，在开始分析详细的代码之前，需要从整体结构上了解掌握 **afl-fuzz.c** 的主要流程，这也是师兄给的主要任务。这里先不管 `main` 函数之前的函数是怎么实现原理的，先就通过 `main` 函数的分析，结合各函数的原注释进行主要流程分析。  
### 【一】设置 `main` 函数内的局部变量  

### 【二】while循环读取来自命令行的参数输入  

### 【三】环境设置和检查，为fuzz做准备  

### 【四】开始第一遍fuzz  

### 【五】开始真正的fuzz

### 【六】结束fuzz的后处理


----------------------------------------------------------------------------------------------------------------------------


## **函数**
***
## **main函数**