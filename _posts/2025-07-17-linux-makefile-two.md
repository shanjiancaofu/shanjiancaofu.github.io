---
title: Makefile - 书写规则
date: 2025-07-17 21:00:00 +0800
categories: [Linux]
tags: [Linux, Makefile, 规则, 伪目标]
---

# 【Linux】Makefile（二）- 书写规则

## 书写规则

### 规则示例

foo.o : foo.c defs.h # foo 模块

cc -c -g foo.c

1、文件的依赖关系，foo.o 依赖于 foo.c 和 defs.h 的文件，如果 foo.c 和 defs.h 的

文件日期要比 foo.o 文件日期要新，或是 foo.o 不存在，那么**依赖关系发生**。

2、如果生成或更新 foo.o 文件。也就是那个 cc 命令，其说明了如何生成 foo.o

这个文件。（ foo.c 文件 include 了 defs.h 文件）


### 规则的语法

targets : prerequisites

command

...

或是这样：

targets : prerequisites ; command

command

...

targets 是文件名，以空格分开，可以使用通配符。

command 是命令行，如果其不与“target:prerequisites”在一行，那么，必须**以[Tab键]开头**，如果和 prerequisites 在一行，那么用**分号做为分隔**。

命令太长，可以使用反斜框（‘\’）作为换行符


### 通配符以及使用通配符

通配符：

“*”：

代替了你一系列的文件

“*.c”表示所以后缀为 c 的文件

如果文件名中有通配符，如：“*”，那么用转义字符“\”，如“\*”来表示真实的“*”字符，而不是任意长度的字符串。

“?”

“[...]”

“~”：

“~/test”，就表示当前用户的$HOME 目录下的 test 目录。

“~hchen/test”，则表示用户 hchen 的宿主目录下的 test 目录

“$?”是一个自动化变量

lpr -p $?

通配符在变量中展开，也就是让 objects 的值是所有[.o]的文件名的集合，由关键字“wildcard”指出

objects := $(wildcard *.o)


### 文件搜寻

在一些大的工程中，有大量的源文件，通常的做法是把这许多的源文件分类，并存放在不同的目录中。

当 make 需要去找寻文件的依赖关系时，可以在文件前加上路径，但更好的是把一个路径告诉 make，让 make 自动去找。

我们通过特殊变量“VPATH”去实现。

通常make 只会在当前的目录中去找寻依赖文件和目标文件。

如果定义了这个变量，make会在当前目录找不到的情况下，到所指定的目录中去找寻文件了。

VPATH = src:../headers

上面的的定义指定两个目录，“src”和“../headers”，make 会按照这个顺序进行搜索。目录由“冒号”分隔。

另一个设置文件搜索路径的方法是使用 make 的“vpath”关键字，它是全小写的，不是变量，这是一个 make 的关键字。

可指定不同的文件在不同的搜索目录中。它的使用方法有三种：

1、vpath <pattern> <directories>

为符合模式<pattern>的文件指定搜索目录<directories>。

2、vpath <pattern>

清除符合模式<pattern>的文件的搜索目录。

3、vpath

清除所有已被设置好了的文件搜索目录。

<pattern>需要包含“%”字符，“%”的意思是匹配零或若干字符，例如，“%.h”表示所有以“.h”结尾的文件。

**例如：**

vpath %.h ../headers

该语句表示，要求 make 在“../headers”目录下搜索所有以“.h”结尾的文件。

如果某文件在当前目录没有找到的话。

可以连续地使用 vpath 语句，以指定不同搜索策略。如果连续的 vpath 语句中出现了相同的<pattern>，或是被重复了的<pattern>，make 会按照 vpath 语句的先后顺序来执行搜索。

例如：

vpath %.c foo

vpath % blish

vpath %.c bar

其表示“.c”结尾的文件，先在“foo”目录，然后是“blish”，最后是“bar”目录。


### 伪目标

“clean”的目标，这是一个“伪目标”，如：

clean:

rm *.o temp

“伪目标”并不是一个文件，只是一个标签，只是通过显示地指明这个“目标”才能让其生效

“伪目标”的取名不能和文件名重名

为了避免和文件重名的这种情况，我们可以使用一个特殊的标记“.PHONY”来显示地指明一个目标是“伪目标”

.PHONY : clean

**伪目标同样也可成为依赖**

.PHONY: cleanall cleanobj cleandiff

cleanall : cleanobj cleandiff

rm program

cleanobj :

rm *.o

cleandiff :

rm *.diff


“make cleanall”将清除所有要被清除的文件。“cleanobj”和“cleandiff”这两个伪目标有点像“子程序”的意思。

我们可以输入“make cleanall”、“make cleanobj”、“make cleandiff”命令来达到清除不同种类文件的目的。


### 多目标

Makefile 的规则中的目标可以不止一个，其支持多目标，有可能我们的多个目标同时依赖于一个文件，并且其生成的命令大体类似。

例如：

bigoutput littleoutput : text.g

generate text.g -$(subst output,,$@) > $@


上述规则等价于：

bigoutput : text.g

generate text.g -big > bigoutput

littleoutput : text.g

generate text.g -little > littleoutput

其中，-$(subst output,,$@)中的“$”表示执行一个 Makefile 的函数，函数名为 **subst**，后面的为参数。

这里的这个函数是截取字符串的意思，“$@”表示目标的集合，就像一个数组，“$​@”依次取出目标，并执于命令。


### 静态模式

静态模式可以更加容易地定义多目标的规则，可以让我们的规则变得更加的有弹性和灵活，语法：

<targets ...>: <target-pattern>: <prereq-patterns ...>

<commands>

....

例子：

objects = foo.o bar.o

all: $(objects)

$(objects): %.o: %.c

$$(CC) -c$$(CFLAGS) $< -o $@

等价于下面的规则：

foo.o : foo.c

$$(CC) -c$$(CFLAGS) foo.c -o foo.o

bar.o : bar.c

$$(CC) -c$$(CFLAGS) bar.c -o bar.o

目标从$object 中获取，“%.o”表明要所有以“.o”结尾的目标，依赖模式“%.c”则取模式“%.o”的“%”，也就是“foo bar”，并加下“.c”的后缀，依赖目标就是“foo.c bar.c”，

“$<”表示所有的依赖目标集（也就是“foo.c bar.c”），

“$@”表示目标集（也就是“foo.o bar.o”）


### 自动生成依赖性

main.o : main.c defs.h

一个比较大型的工程，你必需清楚哪些 C 文件包含了哪些头文件，添加在Makefile中。

在加入或删除头文件时，也需要小心地修改 Makefile，非常繁重且容易出错。

使用 C/C++编译的一个功能，

“-M”的选项，即自动找寻源文件中包含的头文件

cc -M main.c

其输出是：

main.o : main.c defs.h

使用 GNU 的 C/C++编译器，“-MM”参数

gcc -MM main.c
