# Chinese_noted_AFL
## afl-fuzz.c
对源文件添加注释，和自己的个人见解，并随着对其深入不断修正。  
并在文件头记录了关于fuzz test的笔记和关于AFL的想法  
***
## afl-tmin.c
在原来的基础上添加了文件夹操作命令 `-d`（判断是不是文件夹），-i 后面接文件夹路径（如果需要文件夹的话，需要 `-d` 模式为 1）  
详细说明：`https://www.cnblogs.com/wayne-tao/p/11964565.html`  
***
## afl-cmin
提高了鲁棒性，如果 `-o` 文件夹不是空的话，可以自动将其保存，然后重新生成，而不是被提示文件夹不为空233  
详细说明：`https://www.cnblogs.com/wayne-tao/p/11971922.html`  
***
## afl-fuzz.c笔记
文件记录了我看源码时候的一些笔记，现在还不成系统，边看边写。
