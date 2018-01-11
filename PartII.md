# Part II  规则 #
上一章中，我们编写了一些规则来编译、连接我们的字符统计程序。每一个规则定义了一个目标，那就是要更新的一个文件。每一个目标文件取决于一系列的必备条件，它们也是文件。当需要更新目标时，如果必备条件中任何一个文件要比目标文件有最近的更新，那么*make*将会执行规则中的命令脚本。因为一个规则中的目标可能是另一个规则中的必备条件，目标和必备条件形成了一个依赖关系的链式结构或者依赖图，简称依赖路径。构建依赖图并处理依赖路径直到更新要求的目标，这就是*make*所做的一切。

因为规则在*make*中如此重要，有好几种不同类型的规则。

- 明确的规则（Explicit rules） 就像上一章中的，指定了一个明确的目标，如果目标要比任何一个必备条件旧，就需要更新目标。这是你经常要写的规则。
- 模式规则（Pattern rules） 使用通配符而不是明确的文件名。这就使得*make*可以在任何时候将一个符合模式的目标文件更新。
- 隐含规则（Implicit rules） 是*make*内置的规则，在规则数据库中找到的模式规则或者后缀规则。有了内置的规则数据使得编写*makefiles*格外的简单，因为对于许多常用的任务*make*已经知道了文件类型，后缀和更新目标的程序。
- 静态模式规则（Static pattern rules） 类似于常规的模式规则，除了一点，它们只针对于一列目标文件。

## 明确的规则 ##
你所写的绝大部分的规则都是明确的规则，即指明具体的文件作为目标和必备条件。**一个规则可以有多个目标**。这就意味这每一个目标都和另外其他的目标拥有相同的必备条件的集合。如果目标过时了，将会执行相同的动作来更新每一个目标。例如：
```
vpath.o variable.o: make.h config.h getopt.h gettext.h dep.h
```
表明`vpath.o` 和 `variable.o`都依赖于同样的C头文件集合，它和下面的等价：
```
vpath.o: make.h config.h getopt.h gettext.h dep.h
variable.o: make.h config.h getopt.h gettext.h dep.h
```

两个目标会进行独立的处理。

一条规则不必一次性定义完整。每一次*make*找到一个目标文件，就会将目标和必备条件添加到依赖图中。如果目标已经存在于依赖图中，任何其他的必备条件都会添加到那个目标文件的依赖关系中。最常见的情形是，对于把很长的一行打断，提升*makefile*文件的可读性很有用。
```
vpath.o: vpath.c make.h config.h getopt.h gettext.h dep.h
vpath.o: filedef.h hash.h job.h commands.h variable.h vpath.h
```

更复杂的情形中，组成必备条件的文件是由不同的方式管理的：
```
# 确保在vpath.c编译之前创建lexer.c文件
vpath.o: lexer.c
...
# 使用特殊的标记来编译 vpath.c 文件
vpath.o: vpath.c
	$(COMPILE.c) $(RULE_FLAGS) $(OUTPUT_OPTION) $<
...
# 包含某个程序创建的依赖关系
include auto-generated-dependencies.d
```
第一个规则是说只要`lexer.c`更新了就必须要更新`vpath.o`。这个规则也确保了必要条件总是在目标更新之前先更新。
