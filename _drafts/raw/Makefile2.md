

# Makefile

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