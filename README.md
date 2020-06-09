# Chinese_noted_AFL
## afl-fuzz.c
1. 对源文件添加注释，和自己的个人见解，并随着对其深入不断修正并在文件头记录了关于fuzz test的笔记和关于AFL的想法。  
2. 对out_dir文件夹的操作进行了小修改，当文件夹内有 valuable 的内容的时候，不是停止程序，而是做备份，然后让程序继续。[afl-fuzz.c的out_dir修改](https://www.cnblogs.com/wayne-tao/p/12129385.html)

## afl-tmin.c
在原来的基础上添加了文件夹操作命令 `-d`（判断是不是文件夹），-i 后面接文件夹路径（如果需要文件夹的话，需要 `-d` 模式为 1） 
详细说明：[afl-tmin魔改](https://www.cnblogs.com/wayne-tao/p/11964565.html)

## afl-cmin
提高了鲁棒性，如果 `-o` 文件夹不是空的话，可以自动将其保存，然后重新生成，而不是被提示文件夹不为空233  
详细说明：[afl-cmin魔改](https://www.cnblogs.com/wayne-tao/p/11971922.html)

## afl-fuzz.c笔记
afl-fuzz.c笔记文件记录了看源码时候记录的一些笔记，现在还不成系统，边看边写，在看雪上也有（上篇、下篇），欢迎点赞收藏。  
[AFL笔记上篇](https://bbs.pediy.com/thread-257399.htm)  
[AFL笔记下篇](https://bbs.pediy.com/thread-259982.htm)  
总共分为五个部分：  
Ⅰ、文件引用：主要是对所引用头文件的解释，包括自定义头文件和标准库的头文件。  
Ⅱ、预备工作：从预处理、变量、结构体三个方面进行记录。  
Ⅲ、fuzzing的整体结构：对fuzzing过程中的大过程进行划分，其中主要记录关于输入命令的while，文件变异方式（六种），fuzzing的策略，以及语料库的更新。  
Ⅳ、关键函数实现原理：按照大的功能块的划分，记录一些关键函数的理解。  
Ⅴ、main函数：对main函数的循环进行完整解释。  
Ⅵ、附录，一些看过比较好的文章分享。

## 近期的计划是：
一、对ijon进行复现（还差trace模式有点问题）（~~目前解决maze问题 && SMB马里奥video模式~~）  
二、实现AFL、MOpt-afl、Angora、QSYM、ijon+afl对LAVA-M数据集的测试，然后编写易用的一键式脚本（后面写论文肯定要用的[笔记上](https://www.cnblogs.com/wayne-tao/p/12813355.html)、[笔记中](https://www.cnblogs.com/wayne-tao/p/12834525.html)还剩最后一部分）  
三、测试那个256-AFL和一般的AFL有多少改进（https://gamozolabs.github.io/fuzzing/2018/09/16/scaling_afl.html）  
