# Part I  基本概念 #

[TOC]

## 第一章  如何写一个简单的Makefile  ##
**GNU make**（还有其他的make的变体）做的就是管理源码编译为可执行文件的步骤。**make**定义了一种语言，描述了源码、中间文件、可执行文件之间的关系。**make**使用的指令通常保存在文件*makefile*中。如下是一个编译传统的"Hello, World"的程序的*makefile*文件：
```
hello: hello.c
	gcc hello.c -o hello
```
要想编译这个程序只需要在命令行中键入`make`并且执行。这会让*make*程序读入*makefile*文件并且构建它找到的第一个目标：
```
$ make
gcc hello.c -o hello
```
如果在命令行中指定目标，那么这个目标就会被更新。如果命令行没有给*make*指定目标，那么就会使用*makefile*文件中的第一个目标，这个称为**default goal**。

往往大多数的*makefile*中默认目标是构建程序。通常涉及到很多步骤。很多时候程序的源码是不完整的，必须要使用*flex*或者*bison*等工具生成源码，然后把源码编译成二进制目标文件（了例如C/C++生成`*.o`文件，Java生成`*.class`文件，等等）。接着，对于C/C++而言，链接器把目标文件绑定到一起生成一个可执行文件。



### 目标和必备条件 ###
本质上来说*makefile*包含一系列的规则来构建一个应用程序。*make*找到的第一个规则称为**default rule**。一个规则由3个部分组成：目标，必备条件和命令。
```
target: prereq1 prereq2
	commands
```
目标是文件或者其他必须要做的事情。必备条件或者依赖条件是目标被成功创建之前必须要存在的文件。命令就是那些用来从必备条件创建目标的shell命令。

如下是一个用来把C源文件`foo.c`编译成目标文件`foo.o`的规则：
```
foo.o: foo.c foo.h
	gcc -c foo.c
```
目标文件`foo.o`出现在分号之前。必备条件`foo.c`和`foo.h`在分号之后。命令通常出现在下一行且一个tab缩进。

当*make*计算一条规则时，它首先会寻找必备条件和目标所代表的文件。如果任意某个必备条件有一个与之关联的规则，那么*make*就会先去更新那个规则，处理完那些规则才会更新当前的目标文件。如果任意某个必备条件比目标文件要新，才会执行命令来重新构建目标。每个命令都会传递给shell并且在新的子shell中执行。如果任意某个命令产生了错误，目标的构建就会终止并且*make*会退出。一个文件比另外一个文件要新的评判标准是这个文件是最近被修改了。

这里有一个程序，用来统计输入中单词"fee"，"fie"，"foe"，"fum"出现的个数。它在`main()`中简单使用了一个*flex*扫描器。
```C
#include<stdio.h>
#include<stdlib.h>

extern int fee_count, fie_count, foe_count, fum_count;
extern int yylex( void );

int main( int argc, char ** argv)
{
	printf( "Please type some words(Ctrl-D to end):\n" );
	yylex();
	printf( "fee=%d\nfie=%d\nfoe=%d\nfum=%d\n", fee_count, fie_count, foe_count, fum_count );
	exit( 0 );
}
```
扫描器`lexer.l`非常简单：
```
%{
	int fee_count = 0;
	int fie_count = 0;
	int foe_count = 0;
	int fum_count = 0;
%}

%%
fee fee_count++;
fie fie_count++;
foe foe_count++;
fum fum_count++;
```

这个应用程序的*makefile*也是相当的简单：
```Makefile
count_words: count_words.o lexer.o -lfl
	gcc count_words.o lexer.o -lfl -ocount_words

count_words.o: count_words.c
	gcc -c count_words.c

lexer.o: lexer.c
	gcc -c lexer.c

lexer.c: lexer.l
	flex -t lexer.l > lexer.c


clean:
	-@rm -rf *.o lexer.c count_words 2>/dev/null || true
```

第一次执行*makefile*会有以下输出：
```
$ make
gcc -c count_words.c
flex -t lexer.l > lexer.c
gcc -c lexer.c
gcc count_words.o lexer.o -lfl -ocount_words
```

我们得到了可执行程序。当然，通常实际的程序包含的模块远比这个例子多。
应该注意到命令的执行顺序和*makefile*文件中规定的顺序相反。因为*makefile*通常是**自顶向下**的风格，通常目标的最通用的格式放在*makefile*的第一行，它的细节放在后面。



### 依赖检查 ###
*make*是如何决定做什么的？再浏览一遍前面的例子。

首先，*make*注意到命令行并没有包含目标，所以它决定生成默认目标--`count_words`。它检查必备条件，发现了3个：`count_words.o`, `lexer.o` 还有 `-lfl`。现在*make*思考如何编译`count_words.o`并且检查它的规则。同样*make*去检查它的必备条件，注意到`count_words.c`没有规则而且文件存在，因此*make*执行命令把`count_words.c`转换成`count_words.o`，命令如下：
```
	gcc -c count_words.c
```

目标到必备条件，到目标，到必备条件的链式结构是的*make*如何分析*makefile*决定执行哪个命令的标准。

*make*需要检查的下一个必备条件是`lexer.o`。同样规则的链式结构引导到`lexer.c`但是此时这个文件并不存在。*make*发现了从`lexer.l`文件产生`lexer.c`文件的规则。因此它运行`flex`程序。现在`lexer.c`文件存在了它可以运行`gcc`命令了。

最终*make*检查`-lfl`。`gcc`的选项`-l`表明必须要把系统库链接到这个程序中。**fl**指定的实际的库名称是**libfl.a**。对于这种语法，GNU make囊括了特殊的支持。当看到了必备条件是`-l<NAME>`形式的，*make*寻找`libNAME.so`形式的文件，如果没有找到，则寻找`libNAME.a`文件。在这里，*make*找到了`/usr/lib/libfl.a`并且执行了最后一个动作--链接。


### 最小化重建 ###
当运行我们的程序时，会发现除了打印fees, fies, foes, 和 fums，它还会打印输入文件的文本。这个不是我们所希望的。问题出在我们忘记了词法分析器中的某些规则，`flex`会把没有识别的文本打印出来。为了解决这个问题，我们仅仅添加一个*任何字符*的规则和一个新行规则：
```
%{
	int fee_count = 0;
	int fie_count = 0;
	int foe_count = 0;
	int fum_count = 0;
%}

%%
fee fee_count++;
fie fie_count++;
foe foe_count++;
fum fum_count++;
.
\n
```
编辑并保存文件重新构建我们的应用程序：
```
 % make
flex -t lexer.l > lexer.c
gcc -c lexer.c
gcc count_words.o lexer.o -lfl -ocount_words
```

注意到这一次`count_words.c`文件并没有重新编译。当*make*分析规则时，它发现`count_words.o`存在并且要比它的必备条件`count_words.c`文件更新，因此没有必要更新`count_words.o`文件。当分析`lexer.c`文件时，然而*make*发现必备条件`lexer.l`文件要比目标`lexer.c`更新，因此*make*必须要更新`lexer.c`。就这样，反过来导致更新`lexer.o`和`count_words`，现在我们的字符统计程序修复了。

```
 % ./count_words < lexer.l
Please type some words(Ctrl-D to end):
fee=3
fie=3
foe=3
fum=3
```