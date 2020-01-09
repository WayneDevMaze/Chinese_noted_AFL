带图笔记Blog地址：[https://www.cnblogs.com/wayne-tao/p/11889718.html](https://www.cnblogs.com/wayne-tao/p/11889718.html)
## cmin笔记  
cmin是一个用来减小输入集合的工具
### cmin用法
```c
//afl-cmin -i 测试用例文件夹 -o 筛选后的测试用例文件夹 [可能会用到其他操作] 测试程序文件
//这里的用法如下
afl-cmin -i fuzz_in -o fuzz_in_cmin ./afl_test
```
### cmin介绍
在docs的README中是这样描述`cmin tool`的：
>Before using this corpus for any other purposes, you can shrink it to a smaller size using the afl-cmin tool. The tool will find a smaller subset of files offering equivalent edge coverage.  

大概意思就是：
>cmin工具可以减小corpus的大小，这个工具可以得到一个更小的子集（原case集合的子集），这个子集可以提供同样的edge coverage效果。

### cmin原理