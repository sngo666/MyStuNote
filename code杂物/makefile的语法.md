# makefile入门

makefile用以定义一个工程的编译规则。一个工程中充满了复杂的文件结构，对于build的顺序和详细设置也有开发者自己的考量，我们就需要makefile作为与编译器沟通的门户。
makefile的本质是执行定义的命令，并不会关心命令如何执行，相当于代开发者之笔在命令行输入指令。

* 定义

1. 默认情况下makefile 的第一个目标是终极目标。
2. 依赖： 目标文件需要由哪些文件生成。
3. 命令： 通过执行文件由依赖文件生成目标文件，每条命令要求有一个tab保持缩进。
4. all: Makefile文件默认只生成第一个目标文件即完成编译，但是我们可以通过all指定需要生成的文件。
5. 使用`make clean`以清除所有的生成目标用以重新编译。

* 基本结构

makefile的规则如下所示：
>
> ```makefile
> target : prerequisites
> command
> ```

`target`是目标文件，可以是Object File，也可以是执行文件，还可以是一个标签（不懂）。
`prerequisites`就是，要生成的target所需要的文件或是目标(依赖关系中的被依赖对象)。
`command`也就是make要执行的命令。
`\`反斜杠表示换行符，为了方便结构上更加易于阅读。

* 变量

使用`$`来取变量的值，变量名多于一个字符，使用"()"。
一些其他符号的表示：
`$^`表示所有的依赖文件
`$@`表示生成的目标文件
`$<`代表第一个依赖文件

预定义变量：
> `CC = gcc`
使用`CC`代指c编译器的名称。
> `@echo "clean done!"`
回显问题，Makefile中的指令都会被打印出来，如果不想打印出来命令部分，可以使用`@`去除回显。

* 工作方式

make会首先在工程目录下寻找名字叫Makefile或者makefile的文件。
如果能够找到，它会寻找文件中的第一个目标文件，把这个文件作为最终的目标文件。
如果依赖文件不存在，或是依赖文件的修改时间要比当前目标文件新，他会执行后面的命令生成目标文件。
如此往下遍历，直到最终满足生成目标文件的条件并生成目标文件。

GNU的manke非常强大，它可以自动推导文件自己文件依赖关系后面的命令，当make看到一个`.o`文件时，他就会自动把同名的`.c`加在依赖关系之中。

* 使用赋值

使用"="进行赋值，变量的值是整个Makefile中最后被制定的值（即当右边的式子中有变量的引用时，引用的是全局中该变量的最终值）
使用":="对左边变量赋予当前位置等号右边的值（与"="相对）
使用"?="表示如果被赋值变量没有被赋过值（截止到执行当前表达式为止），赋值予等号右边的值。

例：
`objects = main.o kbd.o`  
在下方使用变量：
`cc -o edit $(objects)`

* make的自动推导

GNU的make功能非常强大，它能够自动推导文件以及文件依赖关系背后的命令，所以我们没有必要在每个.o文件后面写上类似的命令。当make看到一个命令时，会把自动把.c文件加在依赖关系中。
但是不建议利用自动推导功能写的过于简略，不方便阅读以及后续的更新开发。

* 清空目标文件的规则

一个合理的例子：

> ```makefile
> .PHONY :clean
> clean: -rm edit $(objects)
> ```

rm前面所加的减号表示，忽略执行中可能出现的各种问题或者警告。
clean一般放在文件的末尾，不然可能会变成make的主要目标。
.PHONY的意思表示clean是一个“伪目标”。

## MAKEFILE总述

* makefile主要组成

Makefile主要包含五个部分：显式规则、隐晦规则、变量定义、文件指示、注释。
显示规则说明了要生成的一个或者多个目标文件，并指出文件的依赖关系，以及要生成的命令。
由于make具有自动推导的功能，隐晦规则可以让我们比较粗糙地简略地书写。
变量的定义，变量类似于宏定义。
文件指示分成三个部分，一个是在Makefile中引用另外一个Makefile，类似于“include”，另一个是根据情况指定Makefile中的有效部分，第三部分是定义的多行命令。
在Makefile中的命令必须以[Tab]键开始。