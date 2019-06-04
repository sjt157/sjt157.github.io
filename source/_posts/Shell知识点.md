---
title: Shell知识点
date: 2018-03-11 17:32:26
tags: Shell
categories: Shell
---

1. /dev/null  位桶（bit bucket）传送到此文件的数据会被丢掉,可以用来测试一个文件是否包含某个pattern。
```
        if grep pattern myfile >  /dev/null
        then ... 找到模式
        else ...没找到模式
        fi
```
2. Shell脚本的命令行参数超多9时，用数字框起来${10}
3. ^$ 用来匹配空字符串或行列，
   `cc -E foo.c | grep -v '^$'> foo.out`  用来排除空行
4. 实时查看当前进程中使用的shell种类 
   `ps | grep $$ | awk '{print $4}'`
5. 正则表达式对于程序执行时的locale环境相当敏感；方括号表达式里的范围应避免使用，该用字符集，例如[[:alnum:]]比较好。
6. -k选项指定排序字段，-t选择字段界定符
```                
        sort -t_ -k1,1 -k2,2 << EOF
        > one_two
        > one_two_three
        > ont_two_four
        > ont_two_five
        > EOF
        one_two
        one_two_three
        ont_two_five
        ont_two_four
```
      可以看出sort并不稳定，因为输入与输出不一致，但我们可以通过--stable选项补救该问题。
7. 为什么看不到/etc/passwd,因为默认是隐藏的。
8. awk与cut是提取字段的好工具。
   
         awk -F: '{print $5}' or cut -d: -f5
9. `2> /dev/null` 是丢弃标准错误信息的输出
10. myvar=赋值并不会将myvar删除，只不过是将其设为null字符串，unset myvar则会完全删除它。
11. `rm -fr /$MYPROGRAM` 若MYPROGRAM未定义，就会有灾难发生！！！
12. POSIX标准化字符串长度运算符：返回字符长度。
```        
        x=supercalifraglistcexpialidoucius  著名的特殊单词
        echo There are ${#x} characters in $x
```
13. set命令 如果未給任何选项，会设置位置参数的值，并将之前存在的任何值丢弃。
```
        [root@localhost ~]# set -- hello "hi there" greetings
        [root@localhost ~]# echo there are $# total arguments 计数
        there are 3 total arguments
        [root@localhost ~]# for i in $*
        > do echo i is $i
        > done
        i is hello  注意内嵌的空白已经消失
        i is hi
        i is there
        i is greetings
        [root@localhost ~]# for i in $@  没有双引号 $*和$@一样
        > do echo i is $i
        > done
        i is hello
        i is hi
        i is there
        i is greetings
        [root@localhost ~]# for i in "$*"
        > do echo i is $i
        > done
        i is hello hi there greetings
        [root@localhost ~]# for i in "$@"; do echo i is $i; done
        i is hello
        i is hi there
        i is greetings
        [root@localhost ~]# shift  截去第一个参数
        [root@localhost ~]# echo there are now $# total arguments
        there are now 2 total arguments  证明消失
        [root@localhost ~]# for i in "$@"; do echo i is $i; done
        i is hi there
        i is greetings
        [root@localhost ~]# shift
        [root@localhost ~]# echo there are now $# total arguments
        there are now 1 total arguments
```
14. 特殊变量$$ 可在编写脚本时用来建立具有唯一性的文件名，多半是临时的，根据Shell的进程编号建立文件名，不过，mktemp也能做。
15. set -e
当命令以非零状态退出时，则退出shell。主要作用是，当脚本执行出现意料之外的情况时，立即退出，避免错误被忽略，导致最终结果不正确。说明set -e 选项对set.sh起作用。脚本作为一个进程去描述set -e选项的范围应该是：set -e选项只作用于当前进行，不作用于其创建的子进程。
set -e 命令用法总结如下：1.当命令的返回值为非零状态时，则立即退出脚本的执行。2.作用范围只限于脚本执行的当前进行，不作用于其创建的子进程。
3.另外，当想根据命令执行的返回值，输出对应的log时，最好不要采用set -e选项，而是通过配合exit 命令来达到输出log并退出执行的目的。
16. shell 脚本中set-x 与set+x的区别
linux shell 脚本编写好要经过漫长的调试阶段，可以使用sh -x 执行。但是这种情况在远程调用脚本的时候，就有诸多不便。
又想知道脚本内部执行的变量的值或执行结果，这个时候可以使用在脚本内部用 set -x 。set去追踪一段代码的显示情况，执行后在整个脚本有效，set -x 开启，set +x关闭，set -o 查看

17. shell if条件判断中的-z到-d的意思
```

[ -a FILE ] 如果 FILE 存在则为真。
[ -b FILE ] 如果 FILE 存在且是一个块特殊文件则为真。
[ -c FILE ] 如果 FILE 存在且是一个字特殊文件则为真。
[ -d FILE ] 如果 FILE 存在且是一个目录则为真。
[ -e FILE ] 如果 FILE 存在则为真。
[ -f FILE ] 如果 FILE 存在且是一个普通文件则为真。
[ -g FILE ] 如果 FILE 存在且已经设置了SGID则为真。
[ -h FILE ] 如果 FILE 存在且是一个符号连接则为真。
[ -k FILE ] 如果 FILE 存在且已经设置了粘制位则为真。
[ -p FILE ] 如果 FILE 存在且是一个名字管道(F如果O)则为真。
[ -r FILE ] 如果 FILE 存在且是可读的则为真。
[ -s FILE ] 如果 FILE 存在且大小不为0则为真。
[ -t FD ] 如果文件描述符 FD 打开且指向一个终端则为真。
[ -u FILE ] 如果 FILE 存在且设置了SUID (set user ID)则为真。
[ -w FILE ] 如果 FILE 如果 FILE 存在且是可写的则为真。
[ -x FILE ] 如果 FILE 存在且是可执行的则为真。
[ -O FILE ] 如果 FILE 存在且属有效用户ID则为真。
[ -G FILE ] 如果 FILE 存在且属有效用户组则为真。
[ -L FILE ] 如果 FILE 存在且是一个符号连接则为真。
[ -N FILE ] 如果 FILE 存在 and has been mod如果ied since it was last read则为真。
[ -S FILE ] 如果 FILE 存在且是一个套接字则为真。
[ FILE1 -nt FILE2 ] 如果 FILE1 has been changed more recently than FILE2, or 如果 FILE1 exists and FILE2 does not则为真。
[ FILE1 -ot FILE2 ] 如果 FILE1 比 FILE2 要老, 或者 FILE2 存在且 FILE1 不存在则为真。
[ FILE1 -ef FILE2 ] 如果 FILE1 和 FILE2 指向相同的设备和节点号则为真。
[ -o OPTIONNAME ] 如果 shell选项 “OPTIONNAME” 开启则为真。
[ -z STRING ] “STRING” 的长度为零则为真。
[ -n STRING ] or [ STRING ] “STRING” 的长度为非零 non-zero则为真。
[ STRING1 == STRING2 ] 如果2个字符串相同。 “=” may be used instead of “==” for strict POSIX compliance则为真。
[ STRING1 != STRING2 ] 如果字符串不相等则为真。
[ STRING1 < STRING2 ] 如果 “STRING1” sorts before “STRING2” lexicographically in the current locale则为真。
[ STRING1 > STRING2 ] 如果 “STRING1” sorts after “STRING2” lexicographically in the current locale则为真。
[ ARG1 OP ARG2 ] “OP” is one of -eq, -ne, -lt, -le, -gt or -ge. These arithmetic binary operators return true if “ARG1” is equal to, not equal to, less than, less than or equal to, greater than, or greater than or equal to “ARG2”, respectively. “ARG1” and “ARG2” are integers.
```

18. Shell变量表达式
![image](https://github.com/sjt157/MarkDownPhotos/raw/master/DockerMiaoSha/5.png)
 
举个栗子：
```

#!/bin/bash

str="a b c d e f g h i j"

echo "the source string is "${str}                         #源字符串
echo "the string length is "${#str}                        #字符串长度
echo "the 6th to last string is "${str:5}                  #截取从第五个后面开始到最后的字符
echo "the 6th to 8th string is "${str:5:2}                 #截取从第五个后面开始的2个字符
echo "after delete shortest string of start is "${str#a*f} #从开头删除a到f的字符
echo "after delete widest string of start is "${str##a*}   #从开头删除a以后的字符
echo "after delete shortest string of end is "${str%f*j}   #从结尾删除f到j的字符
echo "after delete widest string of end is "${str%%*j}     #从结尾删除j前面的所有字

```

