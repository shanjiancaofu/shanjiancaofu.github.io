---
title: Makefile - 介绍
date: 2025-07-16 21:00:00 +0800
categories: [Linux]
tags: [Linux, Makefile, make, 编译]
---

# 【Linux】Makefile（一）- 介绍
本篇博客是作者在学习Linux方面知识过程中，对Makefile不够了解，从而产生了对Makefile有一个全面的认识的想法，在知道《跟我一起写Makefile》此书后，作者学习过程中整理的笔记。


## **makefile**介绍:

make 命令执行时，需要一个 Makefile 文件，以告诉 make 命令需要怎么样的去编译和
链接程序。 规则是：

1）如果这个工程没有编译过，那么我们的所有 C 文件都要编译并被链接。

2）某几个 C 文件被修改，只编译被修改的 C 文件，并链接目标程序。

3）头文件被改变了，需要编译引用了这几个头文件的 C 文件，并链接目标程序。


### **规则：**

target ... : prerequisites ...

command

**解释：**

target 也就是一个目标文件，可以是 Object File，也可以是执行文件。还可以是一个标签（Label）。

prerequisites 是要生成那个 target 所需要的文件或是目标。

command是make需要执行的命令。

**依赖关系:**

target 这一个或多个的目标文件依赖于prerequisites 中的文件，其生成规则定义在 command 中。


### **示例：**

edit : main.o  command.o \

display.o

cc -o edit main.o  command.o \

display.o

main.o : main.c defs.h

cc -c main.c

command.o : command.c defs.h command.h

cc -c command.c

display.o : display.c defs.h buffer.h

cc -c display.c

clean :

rm edit main.o  command.o display.o \

反斜杠（\）是换行符的意思,便于 Makefile 的易读,

目标文件（target）包含：执行文件 edit 和中间目标文件（*.o），

依赖文件（prerequisites）就是冒号后面的那些 .c 文件和 .h 文件

 **Tab** **键作为开头**

make 并不管命令是怎么工作的，他只管执行所定义的命令,

Makefile 中只有行注释，和 UNIX 的 Shell 脚本一样，其注释是用“#”字符。

clean 不是一个文件，它只不过是一个动作名字,类似于标签（lable),

要执行其后的命令，就要在 make 命令后明显得指出这个lable 的名字


### 使用变量:

如果 makefile 很复杂，那么我们就有可能会忘掉一个需要加入的地方，而导致编译失败。

为了 makefile 的易维护，在 makefile 中我们可以使用变量。makefile 的变量也就是一个字符串，可以理解成 C 语言中的宏。

objects = main.o  command.o \

display.o

edit :  $(objects)

cc -o edit  $(objects)

main.o : main.c defs.h

cc -c main.c

command.o : command.c defs.h command.h

cc -c command.c

display.o : display.c defs.h buffer.h

cc -c display.c

clean :

rm edit  $(objects)

声明一个变量，叫 objects，在 makefile 中以“$(objects)”的方式来使用这个变量，如果有新的 .o 文件加入，我们只需简单地修改一下 objects 变量就可以了。


### 自动推导：

我们的 make 会自动识别，并自己推导命令。

只要 make 看到一个[.o]文件，它就会自动的把[.c]文件加在依赖关系中，如果 make找到一个 whatever.o，那么 whatever.c，就会是 whatever.o 的依赖文件。就不用写cc -c whatever.c。


### [.o]和[.h]的依赖文件收拢：

格式：

右边对应每一个[.h]文件，左边是需要右边[.h]文件的对应的[.o]文件

示例：

$(objects) : defs.h

kbd.o command.o files.o : command.h

display.o insert.o search.o files.o : buffer.h

这种风格，让我们的 makefile 变得很简单，但文件依赖关系有些凌乱。

一是文件的依赖关系看不清楚，二是如果文件一多，要加入几个新的.o 文件。


### 清空目标文件的规则：

每个 Makefile 中都应写一个清空目标文件（.o 和执行文件）的规则，这不仅便于重

编译，也利于保持文件的清洁。

一般的风格是：

clean:

rm edit $(objects)


更为稳健的做法是：

.PHONY : clean

clean :

-rm edit $(objects)


.PHONY 意思表示 clean 是一个“伪目标”，。而在 rm 命令前面加了一个小减号的意思是，也许某些文件出现问题，但不管，继续做后面的事。

clean 的规则不要放在文件的开头，clean 放在文件的最后。


### **Makefile**文件名：

默认的情况下，make 命令会在当前目录下按顺序找寻文件名为“GNUmakefile”、“makefile”、“Makefile”的文件，找到了解释这个文件。

在这三个文件名中，最好使用“Makefile”这个文件名，因为，这个文件名第一个字符为大写，这样有一种显目的感觉。最好不要用“GNUmakefile”，这个文件是 GNU 的 make 识别的。

也可以使用别的文件名来书写Makefile，

要指定特定的 Makefile，你可以使用 make 的“-f”和“--file”参数，


### 引用其他Makefile

在 Makefile 使用 include 关键字可以把别的 Makefile 包含进来，include 的语法是： include <filename>

filename 可以是当前操作系统 Shell 的文件模式（可以包含路径和通配符）

在 include前面可以有一些空字符，但是绝不能是[Tab]键开始


include foo.make *.mk $(bar)

等价于：

include foo.make a.mk b.mk c.mk e.mk f.mk


如果文件都没有指定绝对路径或是相对路径的话，make 会在当前目录下首先寻找，

如果当前目录下没有找到，那么，make 还会在下面的几个目录下找：

1、如果 make 执行时，有“-I”或“--include-dir”参数，那么 make 就会在这个参数 所指定的目录下去寻找。

2、目录<prefix>/include（一般是：/usr/local/bin 或/usr/include）存在的话，make 也会去找。

### 环境变量MAKEFILES

如果你的当前环境中定义了环境变量 MAKEFILES，那么，make 会把这个变量中的值做一个类似于 include 的动作。慎用。


### make 的工作方式

make工作的执行步骤如下：

1、读入所有的 Makefile。

2、读入被 include 的其它 Makefile。

3、初始化文件中的变量。

4、推导隐晦规则，并分析所有规则。

5、为所有的目标文件创建依赖关系链。

6、根据依赖关系，决定哪些目标要重新生成。

7、执行生成命令。
