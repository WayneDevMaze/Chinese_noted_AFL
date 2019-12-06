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

--------------------------------------------------------------


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

1. Fuzzing 的状态，比如有地方的用一个数组，需要区分01234···序列代表的含义，就可以用这个代替。这里涉及了很多种不同的 fuzz 种子变异策略
>```c
>/* Fuzzing stages */
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
比如 `DI(stage_finds[STAGE_FLIP1]), DI(stage_cycles[STAGE_FLIP1]),`其中 **`stage_finds`** 的定义`static u64 stage_finds[32]`是一个大数组，用来保存不同的 `fuzz stage` 状态下的模式发现。  

>**`#文件变异[可以先略过，回头再看]`**  
>
>正好这里涉及了对输入文件的变异策略，可以单独先提一下。在AFL的fuzz过程中，维护了一个 testcase 队列，每次把队列里的文件取出来之后，对其进行变异，下面就讲一下各个阶段的变异是怎样的。
>### 【1】**`bitflip`**，位反转，顾名思义按位进行翻转，0变1，1变0。  
>  
>>`STAGE_FLIP1` 每次翻转一位(`1 bit`)，按一位步长从头开始。  
>>`STAGE_FLIP2` 每次翻转相邻两位(`2 bit`)，按一位步长从头开始。 
>>`STAGE_FLIP4` 每次翻转相邻四位(`4 bit`)，按一位步长从头开始。 
>>`STAGE_FLIP8` 每次翻转相邻八位(`8 bit`)，按八位步长从头开始，也就是说，每次对一个byte做翻转变化。 
>>`STAGE_FLIP16`每次翻转相邻十六位(`16 bit`)，按八位步长从头开始，每次对一个word做翻转变化。   
>>`STAGE_FLIP32`每次翻转相邻三十二位(`32 bit`)，按八位步长从头开始，每次对一个dword做翻转变化。  
>
>这一部分在 `5135` 行的 `#define FLIP_BIT(_ar, _b) do {***}` 有详细的代码实现。  
>#### <`token` - 自动检测>
>源码中有一段关于这部分的注释，意思是说在进行为翻转的时候，程序会随时注意翻转之后的变化。比如说，对于一段 `xxxxxxxxIHDRxxxxxxxx` 的文件字符串，当改变 `IHDR` 任意一个都会导致奇怪的变化，这个时候，程序就会认为 `IHDR` 是一个可以让fuzzer很激动的“神仙值”--token，
>```c
>    /* While flipping the least significant bit in every byte, pull of an extra trick to detect possible syntax tokens. In essence, the idea is that if you have a binary blob like this:
>       xxxxxxxxIHDRxxxxxxxx
>      ...and changing the leading and trailing bytes causes variable or no changes in program flow, but touching any character in the "IHDR" string always produces the same, distinctive path, it's highly likely that "IHDR" is an atomically-checked magic value of special significance to the fuzzed format.
>      We do this here, rather than as a separate stage, because it's a nice way to keep the operation approximately "free" (i.e., no extra execs).
>       Empirically, performing the check when flipping the least significant bit is advantageous, compared to doing it at the time of more disruptive changes, where the program flow may be affected in more violent ways.
>       The caveat is that we won't generate dictionaries in the -d mode or -S mode - but that's probably a fair trade-off.
>       This won't work particularly well with paths that exhibit variable behavior, but fails gracefully, so we'll carry out the checks anyway.
>      */
>```  
>#### <`effector map` - 生成>  
>  
>### 【2】**`arithmetic`**，算术，实际操作就是加加减减。 
>  
>>  
>    
>### 【3】**`interest`**，把一些“有意思”的特殊内容替换到原文件中。  
>  
>> 
>   
>### 【4】**`distionary`**，字典，会把自动生成或者用户提供的token替换、插入到原文件中。
>    
>>  
>   
>### 【5】**`havoc`**，“大破坏”，对原文件进行大量变异。  
>>  
>### 【6】**`splice`**，“拼接”，两个文件拼接到一起得到一个新文件。  

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


--------------------------------------------------------------


## **Ⅲ、fuzzing 的整体结构**
后面的代码就是真正的 `afl fuzz` 过程代码，在开始分析详细的代码之前，需要从整体结构上了解掌握 **afl-fuzz.c** 的主要流程，这也是师兄给的主要任务。这里先不管 `main` 函数之前的函数是怎么实现原理的，先就通过 `main` 函数的分析，结合各函数的原注释进行主要流程分析。  
### 【一】设置 `main` 函数内的变量  
设置各种main函数需要的主要局部变量，一部分跟要调用函数的参数有关，一部分跟main函数主体相关。
```c
  s32 opt;
  u64 prev_queued = 0;
  u32 sync_interval_cnt = 0, seek_to;
  u8  *extras_dir = 0;
  u8  mem_limit_given = 0;
  u8  exit_1 = !!getenv("AFL_BENCH_JUST_ONE");
  char** use_argv;

  struct timeval tv;
  struct timezone tz;

  SAYF(cCYA "afl-fuzz " cBRI VERSION cRST " by <lcamtuf@google.com>\n");

  doc_path = access(DOC_PATH, F_OK) ? "docs" : DOC_PATH;

  gettimeofday(&tv, &tz);
  srandom(tv.tv_sec ^ tv.tv_usec ^ getpid());
```
这部分的设置变量，主要是main内使用，跟之外的都无关系，魔改的时候要注意这里。  
### 【二】while循环读取来自命令行的参数输入  
>`while ((opt = getopt(argc, argv, "+i:o:f:m:t:T:dnCB:S:M:x:Q")) > 0)`  

while的判断条件是“命令行”里是否还有输入的参数，while内的命令判断是通过 **`switch - case`** 实现的：  
>i:输入文件夹，包含所有的测试用例 testcase  
>o:输出文件夹，用来存储所有的中间结果和最终结果  
>M:   
>S:  
>f:  
>x:  
>t:   
>m:  
>d:  
>B:  
>C:  
>n:  
>T:  
>Q:
>default: 如果输入错误，提示使用手册（当进行魔改的时候，可以在这个函数 `usage(argv[0]);` 里进行修改）  

### 【三】环境设置和检查，为fuzz做准备  
设置信号句柄，  

### 【四】开始第一遍fuzz  


### 【五】开始真正的fuzz


### 【六】结束fuzz的后处理


--------------------------------------------------------------


## **关键函数实现原理**


--------------------------------------------------------------


## **main函数**