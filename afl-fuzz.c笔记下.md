## 下篇前言：
上一篇已经将前三部分写完了，但是留有一些疑问点，所以下篇将分三部分：**对上篇遗留问题的解决**、**关键函数实现原理介绍**、**main函数注释**（逐行贴出代码块，因为main是整个alf的核心流程，所以需要一行行的详解），这一篇可能对代码的分析偏多一点，因为以后我的论文应该也是要在afl的基础上进行改进，所以就需要对代码多进行分析。其实像mopt-afl对代码的改进就主要是在fuzz_one函数上也就是mutation阶段。  
最后是附录，写有我看过的一些实用的相关文章、博客、帖子之类的，多是实用总结型的。

------------------------------------------------------------

## 接上篇遗留的问题
先来说说上一篇未解决的问题（[上篇](https://bbs.pediy.com/thread-257399.htm) ），这几个问题主要是在代码细节的理解上，有错误之处还望斧正）：
#### 1. effector map  
>关于`effector map`，在看源码数据变异这一部分的时候，一定会注意的是在 bitflip 8/8 的时候遇到一个叫`eff_map`的数组，这个数组的大小是`EFF_ALEN(len)`，这个数是多大？  

当时没想出来这么写是个什么意思，后来转过弯来，其实那不是变量也不是普通的函数，`EFF_ALEN`是一个宏定义的函数：
``` C++
#define EFF_APOS(_p)          ((_p) >> EFF_MAP_SCALE2)
#define EFF_REM(_x)           ((_x) & ((1 << EFF_MAP_SCALE2) - 1))
#define EFF_ALEN(_l)          (EFF_APOS(_l) + !!EFF_REM(_l))
#define EFF_SPAN_ALEN(_p, _l) (EFF_APOS((_p) + (_l) - 1) - EFF_APOS(_p) + 1)
//为eff_map分配大小为EFF_ALEN(len)的内存
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
```
宏函数本质知道了，但是分别代表的什么意思呢？（代码中给出的注释如下）
>EFF_APOS      - position of a particular file offset in the map. 在map中的特定文件偏移位置。
>EFF_ALEN      - length of a map with a particular number of bytes. 根据特定数量的字节数，计算得到的文件长度。
>EFF_SPAN_ALEN - map span for a sequence of bytes. 跳过一块bytes

``` C++
type(_l) EFF_ALEN(_l){
    /*这里的 !! 是一个两次否，目的是归一化（又是一个骚操作，这个作者写代码真的是净整些这种，主要还是自己菜，菜是原罪）
    比如 r = !!a，如果a是整数0，则r=0，如果a是整数非0，则r=1。
    
    在a不是整数的情况下一般不这么用，但这里都是默认_l为整数的，毕竟字符型转成ascii码那不也是整数吗。*/
    return (EFF_APOS(_l) + !!EFF_REM(_l))
}
type(_p) EFF_SPAN_ALEN(_p, _l){
    return (EFF_APOS((_p) + (_l) - 1) - EFF_APOS(_p) + 1)
}
```
现在重新看`eff_map = ck_alloc(EFF_ALEN(len));`，len来自于队列当前结点queue_cur的成员len，是input length（输入长度），所以这里分配给eff_map的大小是 (文件大小/8) 向下取整，这里的 `8 = 2^EFF_MAP_SCALE2`。比如文件17bytes，那么这里的EFF_ALEN(_l)就是3。
#### 2. ARITHMETIC
>如果加减某数之后效果与之前某bitflip效果相同，认为此次变异在上一阶段已经执行过，此次不再执行?  

这里当时标记出来是因为没有理解透彻，没怎么搞懂作者是怎么实现的，后来经过仔细分析几个变量的关系，才缕清楚，这里以 `ARITHMETIC` 的8/8变异这一段为例(16和32的变异大同小异)，把重要部分做解释：
``` C++
  stage_name  = "arith 8/8";//当前进行的状态，这个在fuzz的时候用来在状态栏展示
  stage_short = "arith8";//同上，是简短的状态名
  stage_cur   = 0;
  stage_max   = 2 * len * ARITH_MAX;
  /*ARITH_MAX就是加减变异的最大值限制35，
  文件大小len bytes，
  然后进行 +/- 操作乘以2，
  每个byte要进行的 +/- 操作各35次，
  所以这个stage_max意思就是将要进行多少次变异，但是之后要是没有进行有效变异就要给减去*/
  stage_val_type = STAGE_VAL_LE;
  orig_hit_cnt = new_hit_cnt;//暂存用于最后的计算
  for (i = 0; i < len; i++) {//循环len次
    u8 orig = out_buf[i];
    /* Let's consult the effector map... */
    //如果当前i byte在eff_map对应位置是0，就跳过此次循环，进入for循环的下一次
    //并且此byte对应的变异无效，所以要减 2*ARITH_MAX
    if (!eff_map[EFF_APOS(i)]) {
      stage_max -= 2 * ARITH_MAX;
      continue;
    }
    stage_cur_byte = i;//当前byte
    for (j = 1; j <= ARITH_MAX; j++) {
      //分别进行 +/- 操作 j = 1~35
      u8 r = orig ^ (orig + j);
      /* Do arithmetic operations only if the result couldn't be a product of a bitflip. */
      //只有当arithmetic变异跟bitflip变异不重合时才会进行
      if (!could_be_bitflip(r)) {//判断函数就是对是否重合进行判断的
        stage_cur_val = j;
        out_buf[i] = orig + j;
        if (common_fuzz_stuff(argv, out_buf, len)) goto abandon_entry;
        stage_cur++;
      } else stage_max--;//如果没有进行变异，stage_max减一，因为这里属于无效操作
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
  new_hit_cnt = queued_paths + unique_crashes;//如果8/8期间有新crash的话会加到这里
  stage_finds[STAGE_ARITH8]  += new_hit_cnt - orig_hit_cnt;//这期间增加了的
  stage_cycles[STAGE_ARITH8] += stage_max;//如果之前没有有效变异的话stage_max这里就已经变成0了
```

#### 3. 文件拼接
>为什么不是当前的尾拼接随机文件的头  

这个其实想想当时挺蠢的，一堆文件，拿出一个一分为二，把当前文件的头和其他文件的尾拼接，这是原来的策略，这样多次之后，自然会有之前的那个拿了尾的文件，用这个的头跟其他的尾拼接了。所以没必要此次拼接的时候，非得生成两个文件，因为这样的话后面还会出现，就有点重复了。

#### 4. fuzz流程
>一次变异到底是变一个比特还是从一比特到32比特都结束了才算事  

这个看理解，我跟同学总结的说法是：一次变异是指一个比特的改变也算变异，根据编译策略，经过很多次变异之后，fuzz_one结束叫做一轮变异。

------------------------------------------------------------

## **Ⅳ、关键函数实现原理**
这部分详提一下三个重要函数 `fuzz_one、init_forkserver、show_stats`  
### 1. fuzz_one函数
这个函数的主要功能就是变异，每一阶段变异的原理，在上一篇已经分析过，这里着重说一下作者加的 `eff_map` 这个变量，看代码的话，在变异fuzz_one阶段，这是个贯穿始终的数组，刚开始第一遍看代码其实不是理解的很深刻，现在理解的比较多了，也能理解作者为什么要加这个数组了：  
**fuzz_one函数将近两千行，每一个变异阶段都有自己的功能，怎么把上一阶段的信息用于下一阶段，需要一个在函数内通用的局部变量**，可以看到fuzz_one一开始的局部变量有很多，有很多类型：
``` C++
  s32 len, fd, temp_len, i, j;
  u8  *in_buf, *out_buf, *orig_in, *ex_tmp, *eff_map = 0;
  u64 havoc_queued,  orig_hit_cnt, new_hit_cnt;
  u32 splice_cycle = 0, perf_score = 100, orig_perf, prev_cksum, eff_cnt = 1;
  u8  ret_val = 1, doing_det = 0;
  u8  a_collect[MAX_AUTO_EXTRA];
  u32 a_len = 0;
```
eff_map的类型是u8（*上篇解释过这种看不懂的，可以去types.h里找，这是一个uint8_t = unsigned char 形，8比特（0~255）*），并且u8类型只有这一个是map命名形式的，在后面的注释中如果出现Effector map，说的就是这个变量。主要的作用就是标记，在初始化数组的地方有这么一段注释**Initialize effector map for the next step (see comments below). Always flag first and last byte as doing something.**把第一个和最后一个字节单独标记出来用作其他用途，这里其实就说明了，这个map是标记作用，那是标记什么呢，标记当前byte对应的map块是否需要进行阶段变异。如果是0意味着不需要变异，比如一开始分析的 `arithmetic 8/8` 阶段。  

### 2. init_forkserver函数
用于fork server进行的初始化，这部分跟插桩息息相关，只有插桩模式才会运行这部分，可以配合afl-as.h文件一起看。  


### 3. show_stats函数
如果说在AFL运行的界面，看到那个不懂的变量，或者想单独看那个参数是怎么来的，就从这个函数入手就对了，对应着界面的字符串定位到函数的位置，然后从变量再定位到相应位置，就可以很轻松的理解参数的含义和求法。

------------------------------------------------------------

## Ⅴ、main函数
main函数逐块分析，这部分可能会有点长，大家随意看看就好，算是我自己看代码的碎碎念吧，因为不少在前面已经解释过了，这里算是总结了，**有很多总结文章对源代码的解读比我更好，形式也比我的好看**，我这算是献丑了233333）main函数代码太长，我放在这里：[afl-fuzz.c文件里的main函数代码注释版](https://github.com/WayneDevMaze/Chinese_noted_AFL/blob/master/main%E5%87%BD%E6%95%B0.md)了，可以点进去看。  

--------------------------------------------------------------

## Ⅵ、附录

#### AFL教程类
1. [持续更新的AFL相关小细节](https://www.cnblogs.com/wayne-tao/tag/AFL%E5%AD%A6%E4%B9%A0/)
>这个是我从去年年底一开始接触AFL一直到现在记录的平时遇到的一些小细节，一般情况下就很机录在这个博客里，**王婆卖瓜自卖自夸**哈哈哈哈哈

2. [利用AFL对upx从头开始模糊测试](https://www.cnblogs.com/WangAoBo/p/8280352.html)
>这是一篇比较简单的实践类，非常适合刚开始的新手对AFL进行初探。但是我最早的时候跟着这个帖子进行测试，发现并没有达到作者的效果，所以大家就跟着练练手就好啦，只要跑出来的界面🆗就行。

3. [从头开始的fuzzing流程](https://foxglovesecurity.com/2016/03/15/fuzzing-workflows-a-fuzz-job-from-start-to-finish/)
>很详细的教程，配合2.可以加快对afl工作原理的理解，我是这么觉得的。如果对英文吃力，可以看这篇[翻译](https://blog.csdn.net/abcdyzhang/article/details/53487683)

4. [afl-fuzz.c源码讲解](https://bbs.pediy.com/thread-254705.htm)
>这位前辈写的源码分析，对我而言收获颇多，希望能帮到大家。

5. [AFL整体结构](https://bbs.pediy.com/thread-249912.htm)
>版主写的一篇对AFL从整体结构上进行剖析的文章，写的也很好，很有收获。

6. [256多线程运行AFL](https://gamozolabs.github.io/fuzzing/2018/09/16/scaling_afl.html)
>😳一个对AFL进行多线程操作的文章，算是从工程上进行性能改进的类型，有机会可以实现一下。

7. [Awesome AFL](https://github.com/Microsvuln/Awesome-AFL)
>一个搜集的项目，作者收集了很多 不同的AFL变种，或者AFL启发的超级魔改，这个项目的好处是从没停更，一直整的挺好的。

8. [AFL生态圈](https://bbs.pediy.com/thread-251051.htm)
> 这位师傅写的翻译，对很多AFL类，以及其他fuzzer工具的介绍，比较仔细全面

9. [AFL插桩讲解](https://www.dazhuanlan.com/2020/03/04/5e5f13ab06638/)
>可以看到上下篇只对afl-fuzz.c进行讲解，插桩部分并没有讲解，因为我也还没仔细研究。有这方面需要的可以看一下这篇文章，对插桩进行了详细解释

#### Fuzz相关有用收藏
1. [Fuzzer tools](https://blackarch.org/fuzzer.html)
>一些fuzzer工具集合，内容是前几年比较经典的一些fuzzer，不包括最近顶会的那些fuzzer

2. [fuzz总结性优秀文章分类](https://bbs.pediy.com/thread-249986.htm)
>这是看雪上的一篇质量挺高的总结性的文章

3. [顶会fuzz论文收集与总结](https://github.com/bsauce/Some-Papers-About-Fuzzing)
>作者收集的一些文章是比较经典的顶会文章，而且做了思维导图，可以说是很用心了，~~ 但是貌似停更很久了，不知道还会不会继续更新。 ~~ 跟作者聊过，现在是不更新了，不过还有一个项目在维护🔜

4. [顶会fuzzing论文分类](https://github.com/wcventure/FuzzingPaper)
>这是 3. 的作者写的另一个一直在维护的论文分类project，后面我跟同学也会参与其中一起维护，后面对论文的理解相关的东西也会同步更新到这里。我跟郭老板的想法是把近十年的论文abstract都过一遍然后进行分类，希望能从中找到启发。

5. [大手子rk700](http://rk700.github.io/)
>写的关于fuzz的文章都挺好的，最早的几篇对afl的理解也很有帮助。[他的这篇分析写的很好](https://rk700.github.io/2017/12/28/afl-internals/)


#### 写在最后
下篇本来是跟着上篇写的，但是中间被安排了一些杂七杂八的事情，就一直拖到现在，还好现在已经进入正轨了，期间看雪上也有发私信一起讨论的，如果大家也想一起讨论，或者对文章内容有异议欢迎留言，或者发邮件 wayne-tao(at)Outlook.com，我也是上学期末对时候下定决心搞这方面的，再加上本科不是搞安全的，所以进步很慢，大家共同进步。  
文章和注释源码我放在git上了地址： [https://github.com/WayneDevMaze/Chinese_noted_AFL](https://github.com/WayneDevMaze/Chinese_noted_AFL) **欢迎大家star**，除了上下两篇对 afl.c 文件对分析，还有两篇对cmin和tmin的分析，这两个因为没多少技术含量就不放看雪了，感兴趣可以看一下我对博客（对两个工具对[分析](https://www.cnblogs.com/wayne-tao/p/11889718.html)和修改[tmin](https://www.cnblogs.com/wayne-tao/p/11964565.html)、[cmin](https://www.cnblogs.com/wayne-tao/p/11971922.html)），目前一方面搞fuzz，一方面也在搞web安全（毕竟新入门的水平比较菜，这个好入门），后面也会深入一下二进制，因为经常是发现一些崩溃情况，但是不知道怎么入手分析就很烦，感觉入门还早，继续努力把吧。  
最后，谢谢大家看我的文章，希望大家都能有所收获，早发文章。