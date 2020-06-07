## 下篇前言：
上一篇已经将前三部分写完了，但是留有一些疑问点，所以下篇将分三部分：**对上篇遗留问题的解决**、**关键函数实现原理介绍**、**main函数注释**（逐行贴出代码块，因为main是整个alf的核心流程，所以需要一行行的详解），这一篇可能对代码的分析偏多一点，因为以后我的论文应该也是要在afl的基础上进行改进，所以就需要对代码多进行分析。其实像mopt-afl对代码的改进就主要是在fuzz_one函数上也就是mutation阶段。  
最后是附录，写有我看过的一些实用的相关文章、博客、帖子之类的，多是实用总结型的。

------------------------------------------------------------
## 接上篇遗留的问题
先来说说上一篇未解决的问题（[上篇](https://bbs.pediy.com/thread-257399.htm) ），这几个问题主要是在代码细节的理解上，有错误之处还望斧正）：
#### 1. 数组`eff_map`  
>关于`effector map`，在看源码数据变异这一部分的时候，一定会注意的是在 bitflip 8/8 的时候遇到一个叫`eff_map`的数组，这个数组的大小是`EFF_ALEN(len)`，这个数是多大？  

当时没想出来这么写是个什么意思，后来转过弯来，其实那不是变量也不是普通的函数，`EFF_ALEN`是一个宏定义的函数：
``` C++
#define EFF_APOS(_p)          ((_p) >> EFF_MAP_SCALE2)
#define EFF_REM(_x)           ((_x) & ((1 << EFF_MAP_SCALE2) - 1))
#define EFF_ALEN(_l)          (EFF_APOS(_l) + !!EFF_REM(_l))
#define EFF_SPAN_ALEN(_p, _l) (EFF_APOS((_p) + (_l) - 1) - EFF_APOS(_p) + 1)

eff_map = ck_alloc(EFF_ALEN(len));
```
如果把这一块的宏定义拆开，变成函数的形式就是这样：
``` C++
/*此部分我写了下伪代码，宏定义虽然写代码的时候好用，但是不得不说全是大写字母有点难理解，所以我又给拆分成了通俗的代码，
这部分跟上下文关系不大，看懂这部分的代码，其他类似的也很好懂了。*/
type(_p) EFF_APOS(_p){
    /* >> 是位移操作，把参数的二进制形式进行右移，每移动一位就减小2倍，所以这个函数的意思就是：
    返回传入参数_p除以（2的EFF_MAP_SCALE2次方）。
    同理另一个方向就是左移，是放大2的幂的倍数。*/
    return (_p / 8);
    // EFF_MAP_SCALE2 在文件config.h中出现，值为3，所以这里就是除以8的意思。
}
type(_x) EFF_REM(_x){
    //这里 & 是按位与，所以求的是 _x 与 ((1 << EFF_MAP_SCALE2) - 1)进行按位与，实际上就是跟 7 以二进制形式按位与
    return (_x & 7)
}
/*
  宏函数本质知道了，但是分别代表的什么意思呢？（代码中给出注释如下）
     EFF_APOS      - position of a particular file offset in the map. 在map中的特定文件偏移位置。（）
     EFF_ALEN      - length of a map with a particular number of bytes. 根据特定数量的字节数，计算得到的文件长度。
     EFF_SPAN_ALEN - map span for a sequence of bytes. 跳过一块bytes
*/
type(_l) EFF_ALEN(_l){
    /*这里的 !! 是一个两次否，目的是归一化（又是一个骚操作，这个作者写代码真的是净整些这种，主要是自己菜）
    比如 r = !!a，如果a是整数0，则r=0，如果a是整数非0，则r=1。
    
    在a不是整数的情况下一般不这么用，但这里都是默认_l为整数的，毕竟字符型转成ascii码那不也是整数吗。*/
    return (EFF_APOS(_l) + !!EFF_REM(_l))
}
type(_p) EFF_SPAN_ALEN(_p, _l){
    /*
    */
    return (EFF_APOS((_p) + (_l) - 1) - EFF_APOS(_p) + 1)
}
```
#### 2. ARITHMETIC
>如果加减某数之后效果与之前某bitflip效果相同，认为此次变异在上一阶段已经执行过，此次不再执行?  

这里当时标记出来是因为没有理解透彻，没怎么搞懂作者是怎么实现的，后来经过仔细分析几个变量的关系，才缕清楚，这里以 `ARITHMETIC` 的8/8变异这一段为例(16和32的变异大同小异)，把重要部分做解释：
``` C++
  stage_name  = "arith 8/8";
  stage_short = "arith8";
  stage_cur   = 0;
  stage_max   = 2 * len * ARITH_MAX;
  stage_val_type = STAGE_VAL_LE;
  orig_hit_cnt = new_hit_cnt;
  for (i = 0; i < len; i++) {
    u8 orig = out_buf[i];
    /* Let's consult the effector map... */
    if (!eff_map[EFF_APOS(i)]) {//这里会用到 effector map
      stage_max -= 2 * ARITH_MAX;
      continue;
    }
    stage_cur_byte = i;
    for (j = 1; j <= ARITH_MAX; j++) {
      u8 r = orig ^ (orig + j);
      /* Do arithmetic operations only if the result couldn't be a product of a bitflip. */
      if (!could_be_bitflip(r)) {
        stage_cur_val = j;
        out_buf[i] = orig + j;
        if (common_fuzz_stuff(argv, out_buf, len)) goto abandon_entry;
        stage_cur++;
      } else stage_max--;
      r =  orig ^ (orig - j);
      if (!could_be_bitflip(r)) {
        stage_cur_val = -j;
        out_buf[i] = orig - j;
        if (common_fuzz_stuff(argv, out_buf, len)) goto abandon_entry;
        stage_cur++;
      } else stage_max--;
      out_buf[i] = orig;
    }
  }
  new_hit_cnt = queued_paths + unique_crashes;
  stage_finds[STAGE_ARITH8]  += new_hit_cnt - orig_hit_cnt;
  stage_cycles[STAGE_ARITH8] += stage_max;
```

#### 3. 文件拼接
>为什么不是当前的尾拼接随机文件的头  

这个其实想想当时挺蠢的，一堆文件，拿出一个一分为二，把当前文件的头和其他文件的尾拼接，这是原来的策略，这样多次之后，自然会有之前的那个拿了尾的文件，用这个的头跟其他的尾拼接了。所以没必要此次拼接的时候，非得生成两个文件，因为这样的话后面还会出现，就有点重复了。

#### 4. fuzz流程
>一次变异到底是变一个比特还是从一比特到32比特都结束了才算事  

这个看理解，我跟同学总结的说法是：一次变异是指一个比特的改变也算变异，根据编译策略，经过很多次变异之后，fuzz_one结束叫做一轮变异。
------------------------------------------------------------
## **Ⅳ、关键函数实现原理**
这部分详细解释三个重要函数fuzz_one、init_forkserver、show_stats  
### 1. fuzz_one函数
这个函数的主要功能就是变异，每一阶段变异的原理，在上一篇已经分析过，这里着重说一下作者加的 `eff_map` 这个变量，看代码的话，在变异fuzz_one阶段，这是个贯穿始终的数组，刚开始第一遍看代码其实不是理解的很深刻，现在理解的比较多了，也能理解作者为什么要加这个数组了：
##### 【1】eff_map 主要功能，用于标记：
fuzz_one函数将近两千行，每一个变异阶段都有自己的功能，怎么把上一阶段的信息用于下一阶段，需要一个在函数内通用的局部变量，可以看到fuzz_one一开始的局部变量有很多，有很多类型：
``` C++
  s32 len, fd, temp_len, i, j;
  u8  *in_buf, *out_buf, *orig_in, *ex_tmp, *eff_map = 0;
  u64 havoc_queued,  orig_hit_cnt, new_hit_cnt;
  u32 splice_cycle = 0, perf_score = 100, orig_perf, prev_cksum, eff_cnt = 1;

  u8  ret_val = 1, doing_det = 0;

  u8  a_collect[MAX_AUTO_EXTRA];
  u32 a_len = 0;
```
eff_map的类型是u8（上篇解释过这种看不懂的，可以去types.h里找，这是一个uint8_t = unsigned char 形，8比特（0~255）），并且u8类型只有这一个是map命名形式的，在后面的注释中如果出现Effector map，说的就是这个变量。主要的作用就是标记，在初始化数组的地方有这么一段注释**Initialize effector map for the next step (see comments below). Always flag first and last byte as doing something.**把第一个和最后一个字节单独标记出来用作其他用途，这里其实就说明了，这个map是标记作用，那是标记什么呢，怎么在变异阶段互相影响的呢？
##### 【2】eff_map 代码实现 & 运用的主要思路：
**初始化：**  
****
****
****

### 2. init_forkserver函数

### 3. show_stats函数

------------------------------------------------------------
## Ⅴ、main函数
main函数逐块分析，这部分可能会有点长，大家随意看看就好，算是我自己看代码的碎碎念吧，因为不少在前面已经解释过了，这里算是总结了，**有很多总结文章对源代码的解读比我更好，形式也比我的好看**，我这算是献丑了233333）下篇代码太长，我放在这里[]()了  

--------------------------------------------------------------
## **Ⅵ、附录**
#### AFL教程类
[持续更新的AFL相关小细节](https://www.cnblogs.com/wayne-tao/tag/AFL%E5%AD%A6%E4%B9%A0/)
>这个是我从去年年底一开始接触AFL一直到现在记录的平时遇到的一些小细节，一般情况下就很机录在这个博客里，**王婆卖瓜自卖自夸**哈哈哈哈哈

[]()
>

#### Fuzz相关有用收藏
[Fuzzer tools](https://blackarch.org/fuzzer.html)
>一些fuzzer工具集合，内容是前几年比较经典的一些fuzzer，不包括最近顶会的那些fuzzer

[fuzz总结性优秀文章分类](https://bbs.pediy.com/thread-249986.htm)
>这是看雪上的一篇质量挺高的总结性的文章

#### 杂七杂八看过的有价值的收藏
[顶会fuzz论文收集](https://github.com/bsauce/Some-Papers-About-Fuzzing)
>作者收集的一些文章是比较经典的顶会文章，而且做了思维导图，可以说是很用心了，但是貌似停更很久了，不知道还会不会继续更新。

[]()
>



#### 写在最后
下篇本来是跟着上篇写的，但是中间被安排了一些杂七杂八的事情，就一直拖到现在，还好现在已经进入正轨了，期间看雪上也有发私信一起讨论的，如果大家也想一起讨论，或者对文章内容有异议欢迎留言，或者发邮件 wayne-tao(at)Outlook.com，我也是上学期末对时候下定决心搞这方面的，再加上本科不是搞安全的，所以进步很慢，大家共同进步。  
文章和注释源码我放在git上了地址：[https://github.com/WayneDevMaze/Chinese_noted_AFL](https://github.com/WayneDevMaze/Chinese_noted_AFL)**欢迎大家star**，除了上下两篇对 afl.c 文件对分析，还有两篇对cmin和tmin的分析，这两个因为没多少技术含量就不放看雪了，感兴趣可以看一下我对博客（对两个工具对分析和小修改），目前一方面搞fuzz，一方面也在搞web安全（毕竟新入门的水平比较菜，这个好入门），后面也会深入一下二进制，因为经常是发现一些崩溃情况，但是不知道怎么入手分析就很烦，感觉入门还早，继续努力把吧。  
最后，谢谢大家看我的文章，希望大家都能有所收获，早发文章。