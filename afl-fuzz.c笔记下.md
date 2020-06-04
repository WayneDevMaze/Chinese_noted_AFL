## **下篇序：**
上一篇已经将前三部分写完了，但是留有一些疑问点，所以下篇将分三部分：对上篇遗留问题的解决、关键函数实现原理介绍、main函数注释（逐行贴出代码块，因为main是整个alf的核心流程，所以需要一行行的详解），这一篇可能对代码的分析偏多一点，因为以后我的论文应该也是要在afl的基础上进行改进，所以就需要对代码多进行分析。其实像mopt-afl对代码的改进就主要是在fuzz_one函数也就是mutation阶段。  
最后是附录，写有我看过的一些实用的相关文章。

------------------------------------------------------------
## 接上篇遗留的问题
先来说说上一篇未解决的问题（可以在[上篇]*链接* 中用搜索🔍【？？】快速定位，这几个问题主要是在代码细节的理解上，有错误之处还望斧正）：
#### 1. 数组`eff_map`  
>关于`effector map`，在看源码数据变异这一部分的时候，一定会注意的是在 bitflip 8/8 的时候遇到一个叫`eff_map`的数组，这个数组的大小是`EFF_ALEN(len)`，这个数是多大？  

当时没想出来这么写是个什么意思，后来转过弯来，其实那不是变量也不是普通的函数，`EFF_ALEN`是一个宏定义的函数：
``` C++
#define EFF_APOS(_p)          ((_p) >> EFF_MAP_SCALE2)
#define EFF_REM(_x)           ((_x) & ((1 << EFF_MAP_SCALE2) - 1))
#define EFF_ALEN(_l)          (EFF_APOS(_l) + !!EFF_REM(_l))
#define EFF_SPAN_ALEN(_p, _l) (EFF_APOS((_p) + (_l) - 1) - EFF_APOS(_p) + 1)

  /* Initialize effector map for the next step (see comments below).
  Always flag first and last byte as doing something. */

  eff_map = ck_alloc(EFF_ALEN(len));
```
如果把这一块的宏定义拆开，变成函数的形式就是这样：
``` C++
/*此部分我写了下伪代码，宏定义虽然写代码的时候好用，有点像内联函数（个人感觉）
但是不得不说全是大写字母有点难理解，所以我又给拆分成了通俗的代码，
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
前两个的宏函数本质知道了，但是分别代表的什么意思呢？（代码中给出注释如下）

     EFF_APOS      - position of a particular file offset in the map. 在map中的特定文件偏移位置。（）
     EFF_ALEN      - length of a map with a particular number of bytes. 根据特定数量的字节数，计算得到的文件长度。
     EFF_SPAN_ALEN - map span for a sequence of bytes. 跳过一块bytes

     EFF_REM 代码没有给出解释，我的理解是
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
#### 2. bitflip
>如果加减某数之后效果与之前某bitflip效果相同，认为此次变异在上一阶段已经执行过，此次不再执行?  

这里当时标记出来是因为没有理解透彻，没怎么搞懂作者是怎么实现的，后来经过仔细分析几个变量的关系，才缕清楚，这里把8/8变异这一段重新贴一下(16和32的变异大同小异)，并把重要部分做解释：
``` C++

```

#### 3. 为什么不是当前的尾拼接随机文件的头


#### 4. 
------------------------------------------------------------
## **Ⅳ、关键函数实现原理**
这部分详细解释三个重要函数fuzz_one、、
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
eff_map的类型是u8（上篇解释过这种看不懂的，可以去config里找，这是一个无符号整形（0～255）），并且u8类型只有这一个是map命名形式的，在后面的注释中如果出现Effector map，说的就是这个变量。主要的作用就是标记，在初始化数组的地方有这么一段注释**Initialize effector map for the next step (see comments below). Always flag first and last byte as doing something.**把第一个和最后一个字节单独标记出来用作其他用途，这里其实就说明了，这个map是标记作用，那是标记什么呢，怎么在变异阶段互相影响的呢？
##### 【2】eff_map 代码实现 & 运用的主要思路：
**初始化：**  
****
****
****
##### 【3】关于设计思想的思考以及跟fuzz_one相关的帮助函数（helper function）：


### 2. init_forkserver函数

### 3. 

------------------------------------------------------------
## **Ⅴ、main函数**
main函数逐块分析，这部分可能会有点长，大家随意看看就好，算是自己看代码的碎碎念吧，因为不少在前面已经解释过了，这里算是总结了，**有很多总结文章对源代码的解读比我更好，形式也比我的好看**，我这算是献丑了233333）  

``` C++
int main(int argc, char** argv) {
//局部变量
  s32 opt;
  u64 prev_queued = 0;
  u32 sync_interval_cnt = 0, seek_to;
  u8  *extras_dir = 0;
  u8  mem_limit_given = 0;//内存限定
  u8  exit_1 = !!getenv("AFL_BENCH_JUST_ONE");//设定只跑一次
  char** use_argv;//命令后面的一系列参数

  struct timeval tv;
  struct timezone tz;

  SAYF(cCYA "afl-fuzz " cBRI VERSION cRST " by <lcamtuf@google.com>\n");

  doc_path = access(DOC_PATH, F_OK) ? "docs" : DOC_PATH;

  gettimeofday(&tv, &tz);
  srandom(tv.tv_sec ^ tv.tv_usec ^ getpid());

//while循环，进行命令行参数判定
  while ((opt = getopt(argc, argv, "+i:o:f:m:t:T:dnCB:S:M:x:Q")) > 0)

    switch (opt) {

      case 'i': /* input dir 输入文件夹 */

        if (in_dir) FATAL("Multiple -i options not supported");
        in_dir = optarg;

        if (!strcmp(in_dir, "-")) in_place_resume = 1;

        break;

      case 'o': /* output dir 输出文件夹 */

        if (out_dir) FATAL("Multiple -o options not supported");
        out_dir = optarg;
        break;

      case 'M': { /* master sync ID */

          u8* c;

          if (sync_id) FATAL("Multiple -S or -M options not supported");
          sync_id = ck_strdup(optarg);

          if ((c = strchr(sync_id, ':'))) {

            *c = 0;

            if (sscanf(c + 1, "%u/%u", &master_id, &master_max) != 2 ||
                !master_id || !master_max || master_id > master_max ||
                master_max > 1000000) FATAL("Bogus master ID passed to -M");

          }

          force_deterministic = 1;

        }

        break;

      case 'S': 

        if (sync_id) FATAL("Multiple -S or -M options not supported");
        sync_id = ck_strdup(optarg);
        break;

      case 'f': /* target file */

        if (out_file) FATAL("Multiple -f options not supported");
        out_file = optarg;
        break;

      case 'x': /* dictionary 指定字典，变异阶段会用*/

        if (extras_dir) FATAL("Multiple -x options not supported");
        extras_dir = optarg;
        break;

      case 't': { /* timeout 因为后面阶段是while循环，这里可以通过设置t来实现定时停止fuzzing*/

          u8 suffix = 0;

          if (timeout_given) FATAL("Multiple -t options not supported");

          if (sscanf(optarg, "%u%c", &exec_tmout, &suffix) < 1 ||
              optarg[0] == '-') FATAL("Bad syntax used for -t");

          if (exec_tmout < 5) FATAL("Dangerously low value of -t");

          if (suffix == '+') timeout_given = 2; else timeout_given = 1;

          break;

      }

      case 'm': { /* mem limit 内存限制*/

          u8 suffix = 'M';

          if (mem_limit_given) FATAL("Multiple -m options not supported");
          mem_limit_given = 1;

          if (!strcmp(optarg, "none")) {

            mem_limit = 0;
            break;

          }

          if (sscanf(optarg, "%llu%c", &mem_limit, &suffix) < 1 ||
              optarg[0] == '-') FATAL("Bad syntax used for -m");

          switch (suffix) {

            case 'T': mem_limit *= 1024 * 1024; break;
            case 'G': mem_limit *= 1024; break;
            case 'k': mem_limit /= 1024; break;
            case 'M': break;

            default:  FATAL("Unsupported suffix or bad syntax for -m");

          }

          if (mem_limit < 5) FATAL("Dangerously low value of -m");

          if (sizeof(rlim_t) == 4 && mem_limit > 2000)
            FATAL("Value of -m out of range on 32-bit systems");

        }

        break;

      case 'd': /* skip deterministic */

        if (skip_deterministic) FATAL("Multiple -d options not supported");
        skip_deterministic = 1;
        use_splicing = 1;
        break;

      case 'B': /* load bitmap */

        /* This is a secret undocumented option! It is useful if you find
           an interesting test case during a normal fuzzing process, and want
           to mutate it without rediscovering any of the test cases already
           found during an earlier run.

           To use this mode, you need to point -B to the fuzz_bitmap produced
           by an earlier run for the exact same binary... and that's it.

           I only used this once or twice to get variants of a particular
           file, so I'm not making this an official setting. */

        if (in_bitmap) FATAL("Multiple -B options not supported");

        in_bitmap = optarg;
        read_bitmap(in_bitmap);
        break;

      case 'C': /* crash mode */

        if (crash_mode) FATAL("Multiple -C options not supported");
        crash_mode = FAULT_CRASH;
        break;

      case 'n': /* dumb mode */

        if (dumb_mode) FATAL("Multiple -n options not supported");
        if (getenv("AFL_DUMB_FORKSRV")) dumb_mode = 2; else dumb_mode = 1;

        break;

      case 'T': /* banner */

        if (use_banner) FATAL("Multiple -T options not supported");
        use_banner = optarg;
        break;

      case 'Q': /* QEMU mode 这个模式是用于没有源码的情况下进行fuzz*/

        if (qemu_mode) FATAL("Multiple -Q options not supported");
        qemu_mode = 1;

        if (!mem_limit_given) mem_limit = MEM_LIMIT_QEMU;

        break;

      default:

        usage(argv[0]);

    }

  if (optind == argc || !in_dir || !out_dir) usage(argv[0]);
//设置信号句柄
  setup_signal_handlers();
//检测asan选项
  check_asan_opts();

  if (sync_id) fix_up_sync();
//检查in out文件夹是否相同
  if (!strcmp(in_dir, out_dir))
    FATAL("Input and output directories can't be the same");

//设置环境变量
  if (dumb_mode) {

    if (crash_mode) FATAL("-C and -n are mutually exclusive");
    if (qemu_mode)  FATAL("-Q and -n are mutually exclusive");

  }

  if (getenv("AFL_NO_FORKSRV"))    no_forkserver    = 1;
  if (getenv("AFL_NO_CPU_RED"))    no_cpu_meter_red = 1;
  if (getenv("AFL_NO_ARITH"))      no_arith         = 1;
  if (getenv("AFL_SHUFFLE_QUEUE")) shuffle_queue    = 1;
  if (getenv("AFL_FAST_CAL"))      fast_cal         = 1;

  if (getenv("AFL_HANG_TMOUT")) {
    hang_tmout = atoi(getenv("AFL_HANG_TMOUT"));
    if (!hang_tmout) FATAL("Invalid value of AFL_HANG_TMOUT");
  }

  if (dumb_mode == 2 && no_forkserver)
    FATAL("AFL_DUMB_FORKSRV and AFL_NO_FORKSRV are mutually exclusive");

  if (getenv("AFL_PRELOAD")) {
    setenv("LD_PRELOAD", getenv("AFL_PRELOAD"), 1);
    setenv("DYLD_INSERT_LIBRARIES", getenv("AFL_PRELOAD"), 1);
  }

  if (getenv("AFL_LD_PRELOAD"))
    FATAL("Use AFL_PRELOAD instead of AFL_LD_PRELOAD");

  save_cmdline(argc, argv);

  fix_up_banner(argv[optind]);

  check_if_tty();

//获取cpu逻辑核数
  get_core_count();
//尝试绑定空闲cpu
#ifdef HAVE_AFFINITY
  bind_to_free_cpu();
#endif /* HAVE_AFFINITY */
//确保 core dump 不发生
  check_crash_handling();
//cpu的调度管理
  check_cpu_governor();

  setup_post();
  //创建共享内存 && 各种覆盖状态
  setup_shm();
  //初始化统计计数桶
  init_count_class16();
//设置文件夹；主要是out目录的相关操作
  setup_dirs_fds();
//read测试用例，读取种子
  read_testcases();
//自动加载字典
  load_auto();

  pivot_inputs();
//加载用户通过-x指定的字典
  if (extras_dir) load_extras(extras_dir);

  if (!timeout_given) find_timeout();
//检查是否有@@，通过文件
  detect_file_args(argv + optind + 1);
//标准输入流
  if (!out_file) setup_stdio_file();

  check_binary(argv[optind]);
//开始第一遍dry_run
  start_time = get_cur_time();

  if (qemu_mode)
    use_argv = get_qemu_argv(argv[0], argv + optind, argc - optind);
  else
    use_argv = argv + optind;
//校验初始种子，对种子进行处理
  perform_dry_run(use_argv);
//更新队列
  cull_queue();
//显示ui界面，更新，显示状态
  show_init_stats();

//开始真正的fuzz
  seek_to = find_start_position();
//写到 out/fuzz_stats
  write_stats_file(0, 0, 0);
  save_auto();
//在main函数中一共只用了两次goto，都是为了结束afl的fuzz过程；
//还用到stop_soon变量，这是个标志变量，表示是否按下Ctrl-c，所以ctrl-c是用来停止afl的
  if (stop_soon) goto stop_fuzzing;

  /* Woop woop woop */

  if (!not_on_tty) {
    sleep(4);
    start_time += 4000;
    if (stop_soon) goto stop_fuzzing;
  }

  while (1) {

    u8 skipped_fuzz;
//更新队列
    cull_queue();

    if (!queue_cur) {
//现在在进行第几轮的循环
      queue_cycle++;
//现在fuzz的是第几个值
      current_entry     = 0;
      cur_skipped_paths = 0;
      queue_cur         = queue;

      while (seek_to) {
        current_entry++;
        seek_to--;
        queue_cur = queue_cur->next;
      }
//显示状态
      show_stats();

      if (not_on_tty) {
        ACTF("Entering queue cycle %llu.", queue_cycle);
        fflush(stdout);
      }

      /* If we had a full queue cycle with no new finds, try
         recombination strategies next. */
//队列没有更新的话，没有产生interesting的种子
      if (queued_paths == prev_queued) {

        if (use_splicing) cycles_wo_finds++; else use_splicing = 1;

      } else cycles_wo_finds = 0;

      prev_queued = queued_paths;
//并行fuzz
      if (sync_id && queue_cycle == 1 && getenv("AFL_IMPORT_FIRST"))
        sync_fuzzers(use_argv);

    }

    skipped_fuzz = fuzz_one(use_argv);

    if (!stop_soon && sync_id && !skipped_fuzz) {
      
      if (!(sync_interval_cnt++ % SYNC_INTERVAL))
        sync_fuzzers(use_argv);

    }

    if (!stop_soon && exit_1) stop_soon = 2;
//如果按下ctrl-c跳出循环
    if (stop_soon) break;
//队列继续，进行到下一节点
    queue_cur = queue_cur->next;
    current_entry++;

  }//while部分在这里结束

  if (queue_cur) show_stats();

  /* If we stopped programmatically, we kill the forkserver and the current runner. 
     If we stopped manually, this is done by the signal handler. */
  if (stop_soon == 2) {
      if (child_pid > 0) kill(child_pid, SIGKILL);
      if (forksrv_pid > 0) kill(forksrv_pid, SIGKILL);
  }
  /* Now that we've killed the forkserver, we wait for it to be able to get rusage stats. */
  if (waitpid(forksrv_pid, NULL, 0) <= 0) {
    WARNF("error waitpid\n");
  }

  write_bitmap();
  write_stats_file(0, 0, 0);
  save_auto();

/*利用goto强制跳到结束，作者用了不少goto跳转，比如在变异阶段的skip_bitflip等，虽然最开始学c的时候，老师也说用goto不好，但是实际情况是用goto真香，在跳转中用的好很好用。*/
stop_fuzzing:

  SAYF(CURSOR_SHOW cLRD "\n\n+++ Testing aborted %s +++\n" cRST,
       stop_soon == 2 ? "programmatically" : "by user");

  /* Running for more than 30 minutes but still doing first cycle? */
  /* 运行超过三十分钟，还是第一轮fuzz，就给出提示，说明刚刚的fuzz过程8太行。
  这时候可以考虑考虑什么情况下会这样，用例数量太多？用例不够精简？程序复杂？程序设置了陷阱让你一直在里面跑来跑去？*/
  if (queue_cycle == 1 && get_cur_time() - start_time > 30 * 60 * 1000) {

    SAYF("\n" cYEL "[!] " cRST
           "Stopped during the first cycle, results may be incomplete.\n"
           "    (For info on resuming, see %s/README.)\n", doc_path);

  }
  /* 善后工作 */
  fclose(plot_file);
  destroy_queue();
  destroy_extras();
  ck_free(target_path);
  ck_free(sync_id);

  alloc_report();

  OKF("We're done here. Have a nice day!\n");

  exit(0);

}
```

--------------------------------------------------------------
## **Ⅵ、附录**
#### AFL教程类

#### Fuzz相关有用文章

#### 杂七杂八看过的有价值的文章

#### 写在最后
下篇本来很快能完成的，但是中间被导师安排了一些杂七杂八的事情，就一直拖到现在，还好现在已经进入正轨了，期间有同学发私信一起讨论，如果大家也想一起讨论，或者对文章内容有异议欢迎留言，或者发邮件 wayne-tao(at)Outlook.com，我也是上学期末对时候下定决心搞这方面的，再加上本科不是搞安全的，所以进步很慢。  
文章和注释源码 git地址：，除了上下两篇对 afl.c 文件对分析，还有两篇对cmin和tmin的分析，这两个因为没多少技术含量就不放看雪了，感兴趣可以看一下我对博客（对两个工具对分析和小修改），
目前一方面搞fuzz，一方面也在搞web安全（毕竟新入门的水平比较菜，这个好入门），后面也会学一学二进制，因为经常是发现一些崩溃情况，但是不知道怎么入手分析就很烦，感觉入门还早，继续努力把吧。  
最后，谢谢大家看我的文章，希望大家都有所收获，早发文章。