## 1.Gcc相关
### 1.1GCC编译过程
一个C/C++文件一共经历四个步奏：1.**预处理(preprocessing)**:生成后缀为**.i**的文件；将include的文件插入原文件，将宏定义展开，根据条件编译命令选择要使用的代码；2.**编译(compilation)**：对后缀为**.i**的文件进行预处理生成后缀为**.s**的文件，**翻译成汇编代码；**3.**汇编(assembly)**：对后缀为**.s**的文件进行处理生成后缀为**.o**的文件，生成**机器码；**4.**链接(linking)**：**将不同的object文件与库文件进行相互链接；**可以一次性讲多个**.o**文件链接到同一个输出中。

### 1.2GCC常用命令   
**ESco**（以上四个过程对应的-option） l (指定链接哪一个库文件)    L (指定链接时库文件的目录)
``` makefile
gcc -E -o main.i main.c	#进行代码预处理，也可以查看.c文件包含什么头文件
gcc -S -o main.s main.i	#编译生成汇编文件
gcc -c -o main.o main.s	#将汇编文件处理为机器码文件
gcc -o    main   main.o	#对object文件进行链接

gcc -shared -o libsub.so sub.o sub2.o sub3.o #使用-shared生成动态库文件
gcc -o test main.o -lsub -L /libsub.so #将生成的动态库库链接进去


/* 注意如果需要编译ARM板运行的文件需要进行交叉编译 */
arm-buildroot-linux-gnueabihf-gcc -o hello hello.c
/* 多个.c文件一起编译链接 */
gcc -o test main.c sub.c
```
  同时可以使用**-Wall**打开警告选项；**-O**开启优化编译；附加可选符在前面。

  使用**#include< >**包含文件则只在**标准库目录开始搜索**；如果以**#include" "**包含文件，则先从用户的工作目录开始搜索，再搜索标准库目录。

## 2.Makefile相关

作用：提高编译效率。当修改头文件或者源文件时，只重新编译修改过的文件。makefile文件无后缀，就名为makefile，调用make语句系统自动执行当前文件夹中的makefile文件。

### 2.1Makefile语法

如果依赖文件也是源文件，则先把此依赖文件相关全部处理完成。

``` makefile
源文件:依赖文件 #（1.如果依赖文件比源文件新；2.或者源文件不存在，就执行下一行）
	 命令
#如果make后面不加目标语句，默认执行第一个，即test
test: a.o b.o c.o
	gcc -o test a.o b.o c.o
%.o : %.c  #通配符%的使用
	gcc -c -o $@(源文件) $<(第一个依赖文件) ($^(所有依赖文件))
clean:
	rm *.o test
.PHONY: clean #定义为假想目标，源文件不用与目标文件比较谁更新，即每次调用都会执行（防止文件夹内有同名文件）

#变量分为即时变量，延时变量
A := xxx #A的值立即确定
A  = xxx #A的值使用到的时候才确定
A ?= xxx #只有变量第一次被定义才有用
@echo $(A) #使用此变量,加上@打印出来的不显示命令
```
### 2.2Makefile函数
``` makefile
$(filter pattern,$(var)) #从list中找出符合pattern格式的值,若取出不符合的为:filter-out
C = $(filter %/, $(A))

$(patsubst pattern,replacement,$(var)) #将符合格式的文件替换为其他格式
files = a.c b.c c.c d.c e.c 
files2 = $(patsubst %.c,%.d,$(file))

A = a b c d/
B = $(foreach f, $(A), $(f).o) #对A(list)里面的每个变量执行最后的公式，其中f为形参

files = $(wildcard *.c) #从当前文件夹中输出符合此格式的文件

```