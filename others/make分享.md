

# Make && Makefile



代码变成可执行文件，叫做*编译*。对于一般的C文件，一般有以下过程

```shell
# 编译生成.o文件
cc -c main.c
# 链接生成可执行文件 main
cc main.o -o main
```

但是当整个项目的文件越来越多，文件之间的依赖越来越复杂，手动的编译就不仅不方便而且还容易出错，因此就出现了所谓的*自动化构建工具*，`make`就是其中之一，其他常见的还有`cmake`、`ninja`等

通过编写`Makefile`文件，可以告诉make如何通过调用正确的编译工具对目标进行构建。同时也定义了不同目标文件之间的依赖关系（如`include`操作），因此可以知道构建时正确的先后顺序

通过在命令行中执行`make`命令，并使用适当的参数可以对整个项目进行构建/安装/清理

```shell
make all/install/clean
```

## Makefile

`Makefile`文件是make的核心，其中定义了每一个目标的构建方式以及依赖关系，典型的makefile由多个所谓的*Rule*构成：

```makefile
target … : prerequisites …
        recipe
        …
        …
```

其中`target`为需要构建的目标文件名，`prerequisites`为依赖，而`recipe`则是具体构建`target`所需要执行的命令

针对*Rule*的格式有几个要点

1. `recipe`一般为命令行，也就是shell指令，用于构建`target`，可以有多个
2. ==原则上==当`recipe`执行完毕的时候，应该有一个名为`target`的文件被生成
3. 当依赖（也就是prerequisites）不存在，或者时间戳比`target`要更新的时候，说明`target`需要进行更新，此时执行`make target`将会对`target`进行重新构建，构建的过程会先对依赖进行递归检查，判断是否需要==先更新依赖==
4. 如果执行`make`时没有参数，那么makefile中的第一个Rule将会被执行

## 举个栗子🌰

一个典型的makefile可能如下所示
```makefile
edit : main.o kbd.o command.o display.o \
       insert.o search.o files.o utils.o
        cc -o edit main.o kbd.o command.o display.o \
                   insert.o search.o files.o utils.o

main.o : main.c defs.h
        cc -c main.c
kbd.o : kbd.c defs.h command.h
        cc -c kbd.c
command.o : command.c defs.h command.h
        cc -c command.c
display.o : display.c defs.h buffer.h
        cc -c display.c
insert.o : insert.c defs.h buffer.h
        cc -c insert.c
search.o : search.c defs.h buffer.h
        cc -c search.c
files.o : files.c defs.h buffer.h command.h
        cc -c files.c
utils.o : utils.c defs.h
        cc -c utils.c
clean :
        rm edit main.o kbd.o command.o display.o \
           insert.o search.o files.o utils.o
```

其中定义了每一个文件的生成规则以及依赖，在这个例子中的依赖是由`.c`文件对`.h`文件的`include`关系所决定的。

当我们执行

```shell
make edit
```

将会生成一个名为`edit`的可执行文件

但是这个文件看起来有些繁琐？每一次新增一个`.c`文件，都需要更改大量的地方，如果可以像C语言中的宏一样，做到一处定义，随处可用就好了。

## 变量

于是我们就有了所谓的*变量*(*Variable*)

变量通过等号`=`来进行定义和赋值，同时使用`$(variable_name)`的方式来在makefile中进行引用：

```makefile
foo = $(bar)
bar = $(ugh)
ugh = Huh?
all:
	echo $(foo)

# ---- 输出 ----
# Huh?
```

由于make在执行的时候是分为两个阶段执行的，第一个阶段是读取所有的makefile文件，对其中的变量进行*展开*，同时构建一个所有target以及其依赖的依赖图（*dependency graph*），第二步再进行是否更新的判断，并进行必要的更新/构建

因此，使用 `=` 定义的变量可以在引用后再赋值，同时也是==延迟计算==的，也就是在真正用到的时候才会进行计算

此处的例子

1. 首先碰到`echo $(foo)`，需要计算`foo`
2. `foo = $(bar)`，需要计算`bar`
3. `bar = $(ugh)`，需要计算`ugh`
4. `ugh = Huh?`，所以`foo = Huh?`
5. 命令变为`echo Huh?`，也就是在命令行中输出`Huh?`

但是这种延迟求值可能并不是我们想要的，有时候还可能会导致一些问题。

```makefile
# 在原有的CFLAGS中添加 -O 参数
CFLAGS = $(CFLAGS) -O
```

会导致`CFLAGS`变量递归定义

另一个问题，如果变量的定义中使用了*函数*，那么包含在变量值中的函数总是会在变量被展开的时候执行

```makefile
files = $(shell echo *.c)
```

那么之后每一次通过`$(files)`来访问files的时候`echo *.c`都会被执行一遍。

### 简单变量

于是就又了*简单变量*，简单变量通过`:=`来进行赋值，在make扫描时第一次遇到定义的时候就会进行计算，这也就意味着简单变量不是延迟计算的，也就无法使用还未定义的值

```makefile
x = foo
y := $(x)
x = later

all:
	echo $(y)
```

`y`最终的值是`foo bar`而不是`later bar`

有了变量之后我们就可以对上述的例子进行一定程度上的简化：

```makefile
objects = main.o kbd.o command.o display.o \
          insert.o search.o files.o utils.o

edit : $(objects)
        cc -o edit $(objects)
main.o : main.c defs.h
        cc -c main.c
kbd.o : kbd.c defs.h command.h
        cc -c kbd.c
command.o : command.c defs.h command.h
        cc -c command.c
display.o : display.c defs.h buffer.h
        cc -c display.c
insert.o : insert.c defs.h buffer.h
        cc -c insert.c
search.o : search.c defs.h buffer.h
        cc -c search.c
files.o : files.c defs.h buffer.h command.h
        cc -c files.c
utils.o : utils.c defs.h
        cc -c utils.c
clean :
        rm edit $(objects)
```

这样每次新增`.c`文件的时候，只需要更改`objects`这一处就好了（当然还有新增的Rule）。

但是仅仅如此还是很麻烦，因为对于所有的`.o`文件，在他的依赖列表中几乎必然会有对应的`.c`文件。

因此make支持了所谓的*隐含规则*(*implict rule*)。如果对于`.o`文件省略了recipe，那么就会将`cc -c name.c -o name.o`作为隐含规则，添加到recipe中。如果使用这种隐含规则的话，那么对应的`.c`文件也会被加入到`prerequisites`当中，所以就可以在`prerequiesites`中省略这一个文件，就会变成这样：

```makefile
objects = main.o kbd.o command.o display.o \
          insert.o search.o files.o utils.o

edit : $(objects)
        cc -o edit $(objects)

main.o : defs.h
kbd.o : defs.h command.h
command.o : defs.h command.h
display.o : defs.h buffer.h
insert.o : defs.h buffer.h
search.o : defs.h buffer.h
files.o : defs.h buffer.h command.h
utils.o : defs.h

clean :
        rm edit $(objects)
```

相较于最开始的版本就简化了很多。

## 虚拟目标

可能有人发现了，`clean`所对应的recipe，仅仅是进行了删除操作，并没有生成一个名为`clean`的文件。这就会导致每一次执行`make clean`的时候，`rm`指令必然会被执行。同时，如果真的有一个名为clean的文件，那么`rm`可能永远都不会执行，因为`clean`文件可能一直都没有更新。类似于clean这种target，仅仅只是一个过程，或者一个动作，并不会生成一个名为target的目标，我们将其称之为*虚拟目标*(phony target)。通过将其加入到`.PHONY`的依赖中，可以解决上述的问题

```makefile
objects = main.o kbd.o command.o display.o \
          insert.o search.o files.o utils.o

edit : $(objects)
        cc -o edit $(objects)

main.o : defs.h
kbd.o : defs.h command.h
command.o : defs.h command.h
display.o : defs.h buffer.h
insert.o : defs.h buffer.h
search.o : defs.h buffer.h
files.o : defs.h buffer.h command.h
utils.o : defs.h

.PHONY : clean
clean :
        rm edit $(objects)
```

### 进一步优化

目前为止的makefile中，有一些文件的依赖列表是相同的，比如`display.o`/`insert.o`/`search.o`以及`kdb.o`/command.o`，所以make支持将多个target写在一个rule中

```makefile
kbd.o command.o files.o: defs.h command.h
display.o insert.o search.o: defs.h buffer.h
main.o utils.o defs.h
files.o : buffer.h
```

> 可以发现`files.o`在两个地方都定义了依赖列表，这两个地方的依赖列表最终会被整合在一起形成一个单独的依赖列表。

如果有多个target对应同一个依赖列表的话，可以简单的将其展开就可以得到正确的值，但是有时候在recipe中还需要知道target的名字，这样写在一起的话就无法获得。为了解决这个问题make提供了一些自动变量(*automatic variables*)，此处用到的是`$@`，用于recipe中表示当前对应的==单个==target的名字

```makefile
bigoutput littleoutput : text.g
        generate text.g -$(subst output,,$@) > $@
```

等价于

```makefile
bigoutput : text.g
        generate text.g -big > bigoutput
littleoutput : text.g
        generate text.g -little > littleoutput
```

## 函数

函数的一般形态如下

```makefile
$(function arguments)
# -- or --
${function arguments}
```

上一小节的例子中使用了`$(subst output,,$@)`对文本进行了替换，除了`subst`之外，make还提供了很多有用的函数。

- `$(subst FROM, TO, TEXT)` 

  将==TEXT==中的==FROM==，全部替换为==TO==

  ```makefile
  # 将所有的ee，替换为EE
  $(subst ee,EE,feet on the street) 
  # 输出
  fEEt on the strEEt
  ```

- `$(patsubst PATTERN, REPLACEMENT, TEXT)`

  将==TEXT==中，符合==PATTERN==的字符串，替换为==REPLACEMENT==

  ```makefile
  $(patsubst %.c,%.o,x.c.c bar.c)
  # 输出
  x.o.o bar.o
  ```

- `$(filter pattern..., text)`，挑选出text中符合==某一个==pattern的数据

  ```makefile
  sources := foo.c bar.c baz.s ugh.h
  foo: $(sources)
          cc $(filter %.c %.s,$(sources)) -o foo
  # 最终执行
  cc foo.c bar.c baz.s -o foo
  ```

- `$(wildcard PATTERN)`

- `$(foreach var, list, text)`

  相当于python中的`[text(var) for var in list]`，对于list中的每一个var，将其带入text中进行展开，返回得到的结果
  
  ```makefile
  dirs := a b c d
  files := $(foreach dir,$(dirs),$(wildcard $(dir)/*))
  a/* b/* c/* 
  # 最终files为a,b,c,d文件夹下所有的文件
  ```
  
- `$(call variable, param, param,...)`

  `variable`是一个变量，call将会通过params将其展开，在varible中可以用`$(1)`、`$(2)`等来表示是第几个param

  一个简单的栗子🌰

  ```makefile
  reverse = $(2) $(1)
  
  foo = $(call reverse,a,b)
  b a
  ```

  一个有丶难的栗子🌰

  ```makefile
  map = $(foreach a,$(2),$(call $(1),$(a)))
  o = $(call map, origin, o map MAKE)
  # -- 展开后为 --
  o = $(foreach a,o map MAKE,$(origin $(a)))
  
  # --- 输出 ---
  # % file file default
  # file表明是在一个makefile中定义的
  # default说明是内嵌变量
  ```

- shell函数

  ```makefile
  contents := $(shell cat foo)
  ```

  对于shell命令的执行结果的**标准输出**取出来，并将所有的换行/回车替换为空格，赋值给左边的变量

  ```
  ll `which cc`
  ```

  

- `$(flavor variable)`

  用于表示变量的类型

  1. undefine 变量从未定义

     > 值为空，和未定义是不同的，对于值为空的变量，使用`?=`赋值依然会为空

  2. recursive 递归定义的变量，也就是 =

  3. simple 简单变量，也就是 :=

## 编写Rule

#### 顺序相关依赖

前面已经简单介绍了一个makefile的例子，以下将着重分析Rule的编写方法以及需要注意的点

一个Rule的语法就像下面这样

```makefile
targets : prerequisites
        recipe
        …
```

也可以这样

```makefile
targets : prerequisites ; recipe
        recipe
        …
```

一般来说，决定某一个target是否需要更新要看依赖的两个条件

1. 是否存在（先后顺序）
2. 是否更新

某些情况下只需要考虑先后顺序，而不需要考虑是否更新。

考虑如下情况，当我们的target被保存在一个单独的文件夹中，这个文件夹可能在make运行之前都不存在，在这种情况下，就想要首先创建这个文件夹，之后再将target保存在其中。但是每当文件夹中添加/删除/重命名文件的时候，文件夹的时间戳就会改变，如果简单的将其加入依赖列表中就会导致每次向其中添加文件的时候都会引发所有target的重新构建。

因此引入了顺序相关依赖（*order-only-prerequisistes*）。这种依赖只关心先后顺序，而不关心依赖的时间戳是否更新。

```makefile
OBJDIR := objdir
OBJS := $(addprefix $(OBJDIR)/,foo.o bar.o baz.o)

$(OBJDIR)/%.o : %.c
        $(COMPILE.c) $(OUTPUT_OPTION) $<

all: $(OBJS)

$(OBJS): | $(OBJDIR)

$(OBJDIR):
        mkdir $(OBJDIR)
```

#### 静态规则

静态规则的基本格式如下，主要是通过将target从target==s==取出，然后传入`target-pattern`进行匹配，匹配的部分，称之为*stem*，传入`prereq-patterns`来构造对应的依赖

```makefile
targets …: target-pattern: prereq-patterns …
        recipe
        …
```

直接给出一个例子

```makefile
objects = foo.o bar.o

all: $(objects)

$(objects): %.o: %.c
        $(CC) -c $(CFLAGS) $< -o $@
foo.o: foo.c
	$(CC) -c 
```

其中`$<`也是一个自动变量，表示依赖列表中的==第一个==

## 编写Recipe

对于每一个在recipe中执行的命令，make都会将其打印出来。有时候这个特性是多余的，比如`echo`命令

```makefile
all:
	echo 要开始编译了！

# 将会打印出两句话
echo 要开始编译了！# 命令本身
要开始编译了！     # echo的输出
```

为了消除对某一条命令本身的输出，可以在该条命令前加上`@`

`@echo 要开始编译了！`将不会打印命令本身而只有echo的输出

#### Recipe的执行

某一个target的多个recipe是在各自独立的shell中执行的，所以类似于切换目录或者是设置环境变量等操作都无法从一个recipe影响到另一个recipe。正确的做法是需要在一起执行的recipe通过`&&` 或者`;`写在同一行

```makefile
foo : bar/lose
        cd $(<D) && gobble $(<F) > ../$@
```

> 这里使用了`&&`进行连接是因为可以在目录`bar`不存在的时候直接中断运行，而不是继续尝试执行后面的`gobble`
>
> 其中`$(<D)`以及`$(<F)`也是自动变量，分别表示==第一个==依赖的文件夹部分和文件名部分



多个recipe在执行的时候是顺序执行的，当前一个recipe完成之后make会查看exit status，如果是0（没有任何异常），才会执行下一条recipe，如果出错make则会放弃当前rule的执行，甚至可能会放弃所有rule的执行。

> 在shell中通过`$?`查看上一条命令的exit status

但是有些情况下，recipe执行出错并没有什么大问题，比如创建文件夹的时候文件夹已经存在，抑或是删除文件的时候文件不存在。因此可以在某一条recipe前加上==-==来让make忽略这一条recipe所抛出的error

```makefile
clean:
		-rm -f *.o
```



#### 并行执行

由于上面所说的，同一个Rule下的不同recipe是在独立的shell中执行的，所以非常容易想到通过并行执行来提高make的速度。通过在调用make指令的时候传入`-j`参数可以告诉make尝试去并行执行指令。也可以指定具体的并行数，如`-j32`。当然并行执行的时候所有的输出都可能会乱掉，这个没有什么好办法避免。同时如果有多个recipe需要从标准输入中进行读取，那么可能会出错。一般情况下都是可以无脑并行的。

> 可以通过`make -j$(nproc)`来自动的选择并行数，其中`nproc`会返回当前进程可用的cpu核心数目

## 高级用法

#### 变量

还有别的几种变量

- `variable ?= default_value`

  用于设定变量的默认值，当变量之前没有被==定义==过的时候，才会设置为目标值

  ```makefile
  MACHINE ?= $(shell uname -m)
  ```

- `variable += new_value`

  用于向variable中添加新的元素，会自动添加空格

  ```makefile
  objects = main.o foo.o bar.o utils.o
  objects += another.o
  ```

#### 内建变量

有一些make中内置的变量进行使用，常见的有

- `$(CC)` c compiler，c编译器
- `$(CXX)` c++ 编译器
- `$(CPP)` c preprocessor c预处理器

可以非常方便的通过设置环境变量中的这些值来使用不同的编译器来编译。

```shell
# /home/name/clang_envs.sh
export CC=/usr/bin/clang
export CXX=/usr/bin/clang++
export CPP=/usr/bin/clang-cpp

# /home/name/gcc_envs.sh
export CC=/usr/bin/gcc
export CXX=/usr/bin/g++
export CPP=/usr/bin/gcc
```

想要用不同的编译器来编译就

```shell
. /home/name/clang_envs.sh
make
```

#### 加入其他的makefile

通过在一个makefile文件中使用`include filenames...`可以导入别的makefile，这种情况一般多见于，被include的makefile中包含了一些所有makefile公用的变量。

Makefile include src.mk



#### Pattern Rules

```makefile
%.o : %.c
        $(CC) -c $(CFLAGS) $(CPPFLAGS) $< -o $@
```



## 其他

#### 运行参数

- `-e`

  使用系统环境变量的定义覆盖Makefile中的同名变量定义，一般不建议使用。如果不加这个参数。

- `-f`

  指定makefile文件。一般来说make寻找makefile的顺序是`GNUmakefile` -> `makefile` -> `Makefile`，一旦指定了`-f`参数，则会取消对默认文件名的搜索。

- `-n`

  空操作，只打印重建命令，但是并不实际执行

- `-o`

  将某个文件排除在更新判断之外。比如在一个.h文件中添加了一个宏，但是这个宏不会对其他已有的文件产生影响，那么就可以通过`-o`参数进行排除，防止整个项目重新构建。多个文件需要用多个`-o`

- `-k`

  `--keep-going`，遇到问题接着跑，主要是用来调试的，否则的话每一次只遇到一个错误就直接停止了。
