带图笔记Blog地址：[https://www.cnblogs.com/wayne-tao/p/11889718.html](https://www.cnblogs.com/wayne-tao/p/11889718.html)
## tmin笔记
### tmin用法
```c
//afl-tmin -i 需要精简的文件 -o 精简后的文件 [其他操作] 测试程序
//这里用到的用例
afl-tmin -i fuzz_in/testcase_bin -o fuzz_in_tmin/testcase_tmin ./afl_test
```
### tmin介绍
官方docs的README对`tmin tool`的描述如下:  
>Oh, one more thing: for test case minimization, give afl-tmin a try. The toolcan be operated in a very simple way:  
$ ./afl-tmin -i test_case -o minimized_result -- /path/to/program [...]  
The tool works with crashing and non-crashing test cases alike. In the crash mode, it will happily accept instrumented and non-instrumented binaries. In the non-crashing mode, the minimizer relies on standard AFL instrumentation to make the file simpler without altering the execution path.  

按照描述，大概意思是：  
>工具是为了创造一个更小的文件，同时又能达到同样的效果。当然操作也会比cmin简单。

### tmin原理