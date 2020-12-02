对主要代码进行注解，有错误之处或者疑义，欢迎讨论[issues](https://github.com/WayneDevMaze/Chinese_noted_AFL/issues)
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
//这里说一下main函数的那两个参数
//argc是参数的个数
//argv[0]是程序地址
//argv[1]是第一个参数
//argv[2]是第二个参数。。。以此类推
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