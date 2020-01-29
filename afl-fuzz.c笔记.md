# **afl-fuzz.c笔记**

--------------------------------------------------------------

## **前言：**
本文适合对象：已经对afl的流程有一定了解，自己跑过afl的各个功能；具有一定C编程基础，浏览过afl源码或者某个模块的源码。笔记分为五个大块对 afl-fuzz.c 进行分析：文件引用、预备工作、fuzzing的整体结构、关键函数实现原理、main函数。当然其中涉及到很多其他的头文件，也会对相关部分进行详述。  

--------------------------------------------------------------

## **Ⅰ、文件引用**
### 【一】自定义头文件  
这部分几个头文件都比较重要，源码都是需要看的  
```c
#include "config.h"  
#include "types.h"  
#include "debug.h"  
#include "alloc-inl.h"  
#include "hash.h"  
```  
1. 配置文件，包含各种宏定义，属于通用配置。比如bitflip变异时收集的`token`的长度和数量会在此文件中进行定义。  
2. 类型重定义，一些在 `afl-fuzz.c` 中看不太懂的类型，可以在这里看看是不是有相关定义，比如 `u8` 在源码中经常出现，实际上在这个头文件可以看出 `typedef uint8_t  u8`，所以其对应的类型应该是 `uint8_t` ，对应的是 C99 标准里的无符号字符型  
3. 调试，宏定义各种参数及函数，比如显示的颜色，还有各种自定义的函数，如果改AFL，这些东西相当于没有编译器情况下的 *"高端printf(滑稽脸)"*，比如最常见的 `OKF("We're done here. Have a nice day!\n");` 其中的 `OKF` 就是一个输出代表成功信息的函数  
4. 内存相关，提供错误检查、内存清零、内存分配等常规操作，“内存器的设计初衷不是为了抵抗恶意攻击，但是它确实提供了便携健壮的内存处理方式，可以检查 **use-after-free** 等”  
5. 哈希函数，文件中实现一个参数为 `const void* key, u32 len, u32 seed` 返回为 `u32` 的静态内联函数

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
1. `<dirent.h>` 文件操作相关头文件，可以参考在 `afl-tmin.c` 中的修改，查看其具体用途  
2. `<sched.h>` 任务调度相关头文件  

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
```c
#include "types.h"  
#include "debug.h"  
#include "alloc-inl.h"  
```
这三个都需要查看源码，对后续查看、按需修改 **afl源码** 一定都会很有帮助。  

--------------------------------------------------------------

## **Ⅱ、预备工作**  
这一部分，差不多前三百多行左右，到枚举类型结束。
### 【一】环境判断预处理  
1. **Linux环境下独有的宏定义**  
```c
#ifdef __linux__
#  define HAVE_AFFINITY 1
#endif /* __linux__ */
```
在之后代码中，可以用 `ifdef HAVE_AFFINITY` 判断是否是Linux环境，这里的 AFFINITY 是亲和性的意思，跟cpu的亲和性，**与运行性能有关**。
在定义之后的源代码中一共出现了五次 `HAVE_AFFINITY`，因为是特定条件下的宏定义，所以也可以当是跟cpu操作相关的bool用，都是成对出现，比如：
```c
#ifdef HAVE_AFFINITY
static s32 cpu_aff = -1;/* Selected CPU core */
#endif /* HAVE_AFFINITY */
```
只有在启用这种亲和性的情况下，才会定义 cpu_aff 用来标记选择的cpu核。

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
/* Lots of globals, but mostly for the status UI and other things where it really makes no sense to haul them around as function parameters. */  
可以看出作者很谦虚的指出，这部分的全局变量大部分是跟 UI 展示的状态部分有关的，还有一部分是*没必要*去把他们当作函数参数 (有些情况会造成代码冗余)。  

**这部分从四个方面考虑：**
#### 1.变量类型  
这部分跟头文件 `"types.h"` 息息相关，变量类型不清楚的时候就去那个头文件找答案。比如：  
>u8:  `typedef uint8_t  u8;`  
>u32: `typedef uint32_t u32;`  
>u64: 下面代码的意思，我猜测是为了兼容32和64不同结构情况下的定义  
```c
#ifdef __x86_64__
typedef unsigned long long u64;
#else
typedef uint64_t u64;
#endif /* ^__x86_64__ */  
```
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
在之后的整个fuzzing过程中，都会维护一些queue_entry类型的队列。

#### 4.枚举
枚举类型的好处是可读性强，易于使用，作者定义了三部分枚举类型，用来表示自己想要展现的状态。*具体用法会分析后举例说明*:

1. Fuzzing 的状态，比如有地方的用一个数组，需要区分01234···序列代表的含义，就可以用这个代替。这里涉及了很多种不同的 fuzz 种子变异策略
```c
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
还有就是要注意宏定义的内容，对afl的环境配置、整体的流程的分析都很有帮助。  


--------------------------------------------------------------


## **Ⅲ、fuzzing 的整体结构**
后面的代码就是真正的 `afl fuzz` 过程代码，在开始分析详细的代码之前，需要从整体结构上了解掌握 **afl-fuzz.c** 的主要流程，这也是师兄给的主要任务。这里先不管 `main` 函数之前的函数是怎么实现原理的，先就通过简单的 `main` 函数的分析，结合各函数的原注释进行主要流程分析。  
### 【一】设置 `main` 函数内的变量  
设置各种main函数需要的主要局部变量，一部分跟要调用函数的参数有关，一部分跟main函数主体相关。
```c
  s32 opt;
  u64 prev_queued = 0;
  u32 sync_interval_cnt = 0, seek_to;
  u8  *extras_dir = 0;
  u8  mem_limit_given = 0;//是否内存限制
  u8  exit_1 = !!getenv("AFL_BENCH_JUST_ONE");
  char** use_argv;//用户的输入参数

  struct timeval tv;
  struct timezone tz;

  //显示，这就是之前提到的自定义的调试头文件
  SAYF(cCYA "afl-fuzz " cBRI VERSION cRST " by <lcamtuf@google.com>\n");

  //文件路径
  doc_path = access(DOC_PATH, F_OK) ? "docs" : DOC_PATH;
  
  //时间
  gettimeofday(&tv, &tz);
  srandom(tv.tv_sec ^ tv.tv_usec ^ getpid());
```
这部分的设置变量，主要是main内使用，跟之外的都无关系，魔改的时候要注意这里。  
### 【二】while循环读取来自命令行的参数输入  
```c
while ((opt = getopt(argc, argv, "+i:o:f:m:t:T:dnCB:S:M:x:Q")) > 0)`  
```
while的判断条件是“命令行”里是否还有输入的参数，while内的命令判断是通过 **`switch - case`** 实现的：  
>i:输入文件夹，包含所有的测试用例 testcase  
>o:输出文件夹，用来存储所有的中间结果和最终结果  
>M:设置主（Master）Fuzzer  
>S:设置从属（Slave）Fuzzer  
>f:testcase的内容会作为afl_test的stdin    
>x:设置用户提供的tokens  
>t:设置程序运行超时的时间，单位为ms  
>m:设置分配的内存空间  
>d:  
>B:  
>C:  
>n:  
>T:  
>Q:
>default: 如果输入错误，提示使用手册（当进行魔改的时候，可以在这个函数 `usage(argv[0]);` 里进行修改）  
对afl进行魔改的时候，需要外界输入参数的话，可以从这里先入手，然后倒推到开头，整理出afl实现过程中的参数变化过程。
  
### 【三】初始化，环境设置和检查，为fuzz做准备  
#### 1. tuple  
首先应该知道的是，AFL是根据二元tuple(跳转的源地址和目标地址)来记录分支信息，从而获取target的执行流程和代码覆盖情况。起始阶段 `fuzzer` 会进行一系列的准备工作，为记录插桩得到的目标程序执行路径，即 `tuple` 信息。  
```c
/* SHM with instrumentation bitmap  */
EXP_ST u8* trace_bits;
EXP_ST u8  virgin_bits[MAP_SIZE],//Regions yet untouched by fuzzing
           virgin_tmout[MAP_SIZE],//Bits we haven't seen in tmouts
           virgin_crash[MAP_SIZE];//Bits we haven't seen in crashes
```
其中 `trace_bits` 记录当前的tuple信息；
`virgin_bits` 用来记录总的tuple信息；
`virgin_tmout` 记录fuzz过程中出现的所有目标程序的timeout时的tuple信息；
`virgin_crash` 记录fuzz过程中出现的crash时的tuple信息；
  
#### 2. forkserve  
初始化阶段有个重要的函数调用不得不提 `init_forkserver`（关于此函数的详细解读在之后的函数实现原理part会详细说明），因为这个函数操作就是初始化 `forkserver`。下面说一下forkserver为什么重要，以及到底是用来干嘛的。  
fuzzing的大致思路是，对输入的文件不断变异，然后输入喂给target执行，检查是否会造成崩溃，这其中会涉及大量的fork和执行target过程。（在“[文件/数据]变异”部分就会见识到有多多了）  
AFL实现的这套 `fork server` 机制就是为了提高效率，把 `fork` 相关的操作集成起来，fuzzer不需要对fork负责，只需要与fork server交互即可。关于这部分的具体运行原理在之后会有详细说明。
  
#### 3. 目录处理 && 环境检查  
对输入目录输出目录进行检查，检查cpu，检查内核数量等等。
  
### 【四】[文件/数据]变异  
正好这里涉及了对输入文件的变异策略，可以单独先提一下。在AFL的fuzz过程中，维护了一个 testcase 队列，每次把队列里的文件取出来之后，对其进行变异，下面就讲一下各个阶段的变异是怎样的。  
注： `bitflip、arithmetic、interest、dictionary` 是 `deterministic fuzzing` 过程，属于dumb mode(-d) 和主 fuzzer(-M) 会进行的操作； `havoc、splice` 与前面不同是存在随机性，是所有fuzz都会进行的变异操作。文件变异是具有启发性判断的，应注意“避免浪费，减少消耗”的原则，即之前变异应该尽可能产生更大的效果，比如 `eff_map` 数组的设计；同时减少不必要的资源消耗，变异可能没啥好效果的话要及时止损。  
#### 1.**`bitflip`**，位反转，顾名思义按位进行翻转，0变1，1变0。  
>`STAGE_FLIP1` 每次翻转一位(`1 bit`)，按一位步长从头开始。  
>`STAGE_FLIP2` 每次翻转相邻两位(`2 bit`)，按一位步长从头开始。
>`STAGE_FLIP4` 每次翻转相邻四位(`4 bit`)，按一位步长从头开始。
>`STAGE_FLIP8` 每次翻转相邻八位(`8 bit`)，按八位步长从头开始，也就是说，每次对一个byte做翻转变化。 
>`STAGE_FLIP16`每次翻转相邻十六位(`16 bit`)，按八位步长从头开始，每次对一个word做翻转变化。   
>`STAGE_FLIP32`每次翻转相邻三十二位(`32 bit`)，按八位步长从头开始，每次对一个dword做翻转变化。  

这一部分在 `5135` 行的 `#define FLIP_BIT(_ar, _b) do {***}` 有详细的代码实现。  
##### <`token` - 自动检测>
源码中有一段关于这部分的注释，意思是说在进行为翻转的时候，程序会随时注意翻转之后的变化。比如说，对于一段 `xxxxxxxxIHDRxxxxxxxx` 的文件字符串，当改变 `IHDR` 任意一个都会导致奇怪的变化，这个时候，程序就会认为 `IHDR` 是一个可以让fuzzer很激动的“神仙值”--token。
>    /* While flipping the least significant bit in every byte, pull of an extra trick to detect possible syntax tokens. In essence, the idea is that if you have a binary blob like this:
>       xxxxxxxxIHDRxxxxxxxx
>      ...and changing the leading and trailing bytes causes variable or no changes in program flow, but touching any character in the "IHDR" string always produces the same, distinctive path, it's highly likely that "IHDR" is an atomically-checked magic value of special significance to the fuzzed format.
>      We do this here, rather than as a separate stage, because it's a nice way to keep the operation approximately "free" (i.e., no extra execs).
>       Empirically, performing the check when flipping the least significant bit is advantageous, compared to doing it at the time of more disruptive changes, where the program flow may be affected in more violent ways.
>       The caveat is that we won't generate dictionaries in the -d mode or -S mode - but that's probably a fair trade-off.
>       This won't work particularly well with paths that exhibit variable behavior, but fails gracefully, so we'll carry out the checks anyway.
>      */ 
>
其实token的长度和数量都是可以控制的，在 `config.h` 中有定义，但是因为是在头文件宏定义的，修改之后需要重新编译使用。

>/* Maximum number of auto-extracted dictionary tokens to actually use in fuzzing (first value), and to keep in memory as candidates. The latter should be much higher than the former. */  
```c
 #define USE_AUTO_EXTRAS 50  
 #define MAX_AUTO_EXTRAS (USE_AUTO_EXTRAS * 10)  
```  
##### <`effector map` - 生成>  
在这里值得一提的是 `effector map`，在看源码数据变异这一部分的时候，一定会注意的是在 bitflip 8/8 的时候遇到一个叫 `eff_map` 的数组，这个数组的大小是 `EFF_ALEN(len)` （也就是【？？】），数组元素只有 0/1 两种值，很明显是标记的意思，到底是标记什么呢？  
要想明白 `effector map` 的原理需要了解三个点：  
>1. 为什么是 8/8 的时候出现？因为 `8bit`（比特）的时候是 `1byte`（字节），如果一个字节的翻转都无法带来路径变化，此byte极有可能是不会导致crash的数据，所以之后应该用一种思路避开无效byte。  
>2. 标记是干什么用的？根据上面的分析，就很好理解了，标记好的数组可以为之后的变异服务，相当于提前“踩雷（踩掉无效byte的雷）”，相当于进行了启发式的判断。无效为0，有效为1。  
>3. 达到了怎样的效果？要知道判断的时间开销，对不停循环的fuzzing过程来说是致命的，所以 `eff_map` 利用在这一次8/8的判断中，通过不大的空间开销，换取了可观的时间开销。(暂时是这样分析的，具体是否真的节约很多，不得而知)

#### 2.**`arithmetic`**，算术，实际操作就是加加减减。 
`bitflip` 结束之后，就进入 `arithmetic` 阶段，目标大小和阶段与 `bitflip` 非常类似：
>arith 8/8，每次8bit进行加减运算，8bit步长从头开始，即对每个byte进行整数加减变异；  
>arith 16/8，每次16bit进行加减运算，8bit步长从头开始，即对每个word进行整数加减变异；  
>arith 32/8，每次32bit进行加减运算，8bit步长从头开始，即对每个dword进行整数加减变异；  

其中对于加减变异的上限，在 `config.h` 中有所定义：
>/* Maximum offset for integer addition / subtraction stages: */  
```c
#define ARITH_MAX 35
```  
*注：跟bitflip相同的，如果需要修改此值，在头文件中修改完之后，要进行编译才会生效。*  

在这里对整数目标进行+1，+2，+3...+35，-1，-2，-3...-35的变异。由于整数存在大端序和小端序两种表示，AFL会对这两种表示方式都进行变异。  
前面也提到过AFL设计的巧妙之处，AFL尽力不浪费每一个变异，也会尽力让变异不冗余，从而达到快速高效的目标。AFL会跳过某些arithmetic变异：
1. 在 `eff_map` 数组中对byte进行了 0/1 标记，如果一个整数的所有 bytes 都被判为无效，那么就认为整数无效，跳过此数的变异；  
2. 【？？】如果加减某数之后效果与之前某bitflip效果相同，认为此次变异在上一阶段已经执行过，此次不再执行；  

#### 3.**`interest`**，把一些“有意思”的特殊内容替换到原文件中。  
interest的三个步骤跟arithmetic相同：
>interest 8/8，每次8bit进行加减运算，8bit步长从头开始，即对每个byte进行替换；  
>interest 16/8，每次16bit进行加减运算，8bit步长从头开始，即对每个word进行替换；  
>interest 32/8，每次32bit进行加减运算，8bit步长从头开始，即对每个dword进行替换；  

用于替换的叫做 `interesting values` ，是AFL预设的特殊数：
>/* Interesting values, as per config.h */
```c
static s8 interesting_8[] = { INTERESTING_8 }; 
static s16 interesting_16[] = { INTERESTING_8,INTERESTING_16 };
static s32 interesting_32[] = { INTERESTING_8, INTERESTING_16, INTERESTING_32 };
```
其中 `interesting values` 在 `config.h` 中的设定：
```c
/* List of interesting values to use in fuzzing. */
#define INTERESTING_8 \
  -128,          /* Overflow signed 8-bit when decremented  */ \
  -1,            /*                                         */ \
   0,            /*                                         */ \
   1,            /*                                         */ \
   16,           /* One-off with common buffer size         */ \
   32,           /* One-off with common buffer size         */ \
   64,           /* One-off with common buffer size         */ \
   100,          /* One-off with common buffer size         */ \
   127           /* Overflow signed 8-bit when incremented  */

#define INTERESTING_16 \
  -32768,        /* Overflow signed 16-bit when decremented */ \
  -129,          /* Overflow signed 8-bit                   */ \
   128,          /* Overflow signed 8-bit                   */ \
   255,          /* Overflow unsig 8-bit when incremented   */ \
   256,          /* Overflow unsig 8-bit                    */ \
   512,          /* One-off with common buffer size         */ \
   1000,         /* One-off with common buffer size         */ \
   1024,         /* One-off with common buffer size         */ \
   4096,         /* One-off with common buffer size         */ \
   32767         /* Overflow signed 16-bit when incremented */

#define INTERESTING_32 \
  -2147483648LL, /* Overflow signed 32-bit when decremented */ \
  -100663046,    /* Large negative number (endian-agnostic) */ \
  -32769,        /* Overflow signed 16-bit                  */ \
   32768,        /* Overflow signed 16-bit                  */ \
   65535,        /* Overflow unsig 16-bit when incremented  */ \
   65536,        /* Overflow unsig 16 bit                   */ \
   100663045,    /* Large positive number (endian-agnostic) */ \
   2147483647    /* Overflow signed 32-bit when incremented */
```
可以看到，基本是些会造成溢出的数值。与前面的思想相同的，本着“避免浪费，减少消耗”的原则，`eff_map`数组中已经判定无效的就此轮跳过；如果 `interesting value` 达到的效果跟 `bitflip` 或者 `arithmetic` 效果相同，也被认为是重复消耗，跳过。  

#### 4.**`distionary`**，字典，会把自动生成或者用户提供的token替换、插入到原文件中。
此阶段已经是确定性变异 `deterministic fuzzing` 的结尾：
>user extras (over)，从头开始，将用户提供的tokens依次替换到原文件中  
>user extras (insert)，从头开始，将用户提供的tokens依次插入到原文件中
>auto extras (over)，从头开始，将自动检测的tokens依次替换到原文件中   

其中 “用户提供的tokens” 是一开始通过 -x 选项指定的，如果没有则跳过对应的子阶段；“自动检测的tokens” 是第一个阶段 `bitflip` 生成的。  

#### 5.**`havoc`**，“大破坏”，对原文件进行大量变异。  
`havoc` 意味着随机的开始，与后面的 `splice` 是任何模式下都要进行的变异，具体来说 havoc 包含了多轮变异，每一轮都是组合拳：  

>   随机选取某个bit进行翻转
    随机选取某个byte，将其设置为随机的interesting value
    随机选取某个word，并随机选取大、小端序，将其设置为随机的interesting value
    随机选取某个dword，并随机选取大、小端序，将其设置为随机的interesting value
    随机选取某个byte，对其减去一个随机数
    随机选取某个byte，对其加上一个随机数
    随机选取某个word，并随机选取大、小端序，对其减去一个随机数
    随机选取某个word，并随机选取大、小端序，对其加上一个随机数
    随机选取某个dword，并随机选取大、小端序，对其减去一个随机数
    随机选取某个dword，并随机选取大、小端序，对其加上一个随机数
    随机选取某个byte，将其设置为随机数
    随机删除一段bytes
    随机选取一个位置，插入一段随机长度的内容，其中75%的概率是插入原文中随机位置的内容，25%的概率是插入一段随机选取的数
    随机选取一个位置，替换为一段随机长度的内容，其中75%的概率是替换成原文中随机位置的内容，25%的概率是替换成一段随机选取的数
    随机选取一个位置，用随机选取的token（用户提供的或自动生成的）替换
    随机选取一个位置，用随机选取的token（用户提供的或自动生成的）插入
>
之后AFL会生成一个随机数，作为变异组合的数量，每次从上面随机选取作用于当前文件。

#### 6.**`splice`**，“拼接”，两个文件拼接到一起得到一个新文件。  
具体来说，AFL会在文件队列中随机选择一个文件与当前文件进行对比，如果差别不大就重新再选；如果差异明显，就随机选取位置两个文件都一切两半。最后将当前文件的头与随机文件的尾拼接起来得到新文件【为什么不是当前的尾拼接随机文件的头【？？】】。当然了本着“减少消耗”的原则拼接后的文件应该与上一个文件对比，如果未发生变化应该过滤掉。

#### 7.**`cycle`**，循环往复，一波变异结束后的文件，会在队列结束后下一轮中继续变异下去  
AFL状态栏右上角的 `cycles done` 意味着完成的循环数，每次循环是对整个队列的再一次变异，不过只有第一次 cycle 才会进行 `deterministic fuzzing`，之后的只有随机性变异了。

### 【五】fuzzing策略  
这部分跟 `mutation` 相关，会多写一点(不断更新)，将会分为两部分进行展开：**现有AFL的fuzzing的策略**和**论文相关魔改AFL的fuzzing策略**。
#### 1.**AFL**现在的fuzzing策略  
在上一节中已经说完了文件/数据的各种变异类型，其实还有一个点没说，就是一次变异操作到底是怎样的流程：（一次变异到底是变一个比特还是从一比特到32比特都结束了才算事【？？】）
>`fuzz_one` ：进行一次测试用例变异过程
`common_fuzz_stuff` ：变异完成后的一般操作处理
`write_to_testcase` ：变异后的内容写入测试文件
`run_target` ：运行目标进程
`save_is_insteresting` ：判断是否保存这个测试用例
>`has_new_bits` ：判断测试用例是否产生新状态
>
#### 2.相关论文的一些fuzzing策略  
##### MOpt

接下来将会讨论一篇论文的魔改AFL——Mopt的fuzzing策略，也就是关于这篇文章的演讲让我对之后afl的学习充满了信心。【？？】
这篇文章是针对havoc的改进？？？

##### 针对堆溢出的fuzz策略  

##### afl-smart  

### 【六】语料库更新  
随着fuzzing的不断进行，cycle一轮又一轮，势必会产生更多的变异测试用例，这些用例并不是一股脑全都放进队列里的，对于没有新状态（由tuple信息可得）的测试用例应该抛弃；而且对于较优的用例应该更有可能被利用（这也是启发性的体现）。  
#### 1.判定新状态 【？？】与插桩有关，待之后详写  
#### 2.更优的测试用例  
其实在循环的fuzzing过程中，到现在还有一个问题未解决，我们现在知道了什么样的文件应该留下，但是有没有可能扔掉的文件也很有用呢？有没有一个系统的标准来判定，两个文件到底哪个更优？【？？】  

--------------------------------------------------------------


## **Ⅳ、关键函数实现原理**


--------------------------------------------------------------


## **Ⅴ、main函数**