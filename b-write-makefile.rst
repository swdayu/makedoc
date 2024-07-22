编写 Makefile
==============

Make 工具可以自动确定哪些文件需要更新，并执行对应命令更新这些文件。只需要把文件关系
和构建命令描述在 makefile 文件中，make 就会自动根据文件关系和时间戳来更新文件。一个
简单的 makefile 文件由多个规则（rule）组成，每个规则的形式如下： ::

    target ... : prerequisite ...
        recipe
        ...
        ...

目标（target）通常是需要生成的目标文件名，或者需要进行的操作比如清理等不生成文件的伪
目标（Phony Target）。前置条件（prerequisite）是目标文件依赖的文件，操作（recipe）
是生成目标需要执行的一个或多个命令。这些命令可以放在同一行，或者分行但是每行必须用 Tab
键开始，或者设置 .RECIPEPREFIX 变量用其他字符开始。

通常操作命令是当规则中的任何一个前置条件发生变化时，才执行的用于生成目标文件的命令，但
一个规则也可以没有条件，这时候当目标文件不存在时操作命令会无条件执行。因此，一个规则实
际上是描述了目标文件如何构建、以及什么时候构建。

当 make 执行时，它会自动执行第一个目标（不是以点.开头的目标除非以点开头的目标包含一个或
多个斜杠/字符），这个目标称为默认目标（default goal）。当然，你可以在 make 命令行指定
默认执行的目标，或者设置 .DEFAULT_GOAL 变量来指定。只要目标文件不存在、或者前置条件中任
何一个文件的时间戳比目标文件新时（目标所依赖的文件发生了变化），就会重新生成目标文件。

在更新目标文件之前，make 首先会依次更新条件中的目标，也即目标依赖的目标会先更新，如果条
件是一个单纯的文件而不是一个目标，make 不会为这些单纯的文件做任何事情。

变量（Variable）可以简化 makefile 的书写，变量可以定义一个文本串，然后在所有需要使用这
个文本串的地方用变量代替。 ::

    objects = main.o kbd.o command.o display.o

    edit: $(objects)
        cc -o edit $(objects) # link

    main.o: main.c defs.h
        cc -c main.c # compile

    kbd.o: kbd.c defs.h command.h
        cc -c kbd.c

    comand.o: command.c defs.h command.h
        cc -c command.c

    display.o: display.c defs.h buffer.h
        cc -c display.c

    .PHONY: clean # clean 不是一个文件而是一个伪目标
    clean: # 这样即使存在 clean 文件这个 clean 目标仍然会被执行
        -rm -f edit $(objects) # 命令前加-表示忽略命令执行时遇到的任何错误

    clean: FORCE # 当不支持 .PHONY 时可以这样写，但区别是如果真的存在 FORCE 这个文件，clean 就不会执行了
        -rm -f edit $(objects)

    FORCE:

另外 make 提供了一些隐式的规则，可以自动将 .c 生成同名的 .o 文件（cc -c main.c -o main.o），
这样上面的规则可以简化： ::

    # 依赖的同名.c可以省略
    main.o : defs.h
    kbd.o : defs.h command.h
    command.o : defs.h command.h
    display.o : defs.h buffer.h

    # 还可以将相同项合并
    $(objects) : defs.h
    kbd.o command.o : command.h
    display.o : buffer.h


Makefile 文件
--------------

Makefile文件包含五种内容：

1. 显式规则（explicit rules）
2. 隐式规则（implicit rules）
3. 变量定义（variable definitions）
4. 指示语句（directive）
5. 注释（comment）

指示语句包括：包含另一个 makefile；条件语句；多行变量定义。注释以 # 开头，直到行尾或续行
（如果行末位用\结束），如果需要使用 # 字符，要用 \# 转义。但是在变量引用和函数调用中不能
使用注释，在变量引用和函数调用中 # 字符被当成是普通字符。规则中的操作命令，如果包含了注释
会直接传递到 shell 中，是否被当作注释由 shell 决定。指示语句 define 中包含注释会被当成
原始字符串，是否被当作注释，视这个变量展开使用的地方而定。

用 '反斜杠和换行' 可以续行，一般地 '反斜杠和换行' 被替换为空格，如果你不想要空格，可以用
下面 '$\ 和换行' 的技巧： ::

    var := one$\
        word

    # 它等价于:
    var := one$ word

    # 但是 '$ ' 空格字符的变量不存在，这个变量的引用为空，因此相当于：
    var := oneword

Makefile 的名字
----------------

默认，make 会一次查找 GNUmakefile，makefile，Makefile。如果没有找到这些文件，则不会读取
任何文件，必须在 make 命令行指定规则的目标，这时 make 仅使用隐式规则来执行。

当然，make 命令行可以直接指定一个 makefile 文件名，使用 `-f name` 或者 `--file=name`，
你可以指定一个或多个 makefile 文件，多个文件会按顺序拼接成一个完整的内容来解析。

包含 makefile 文件
-------------------

使用 include 指示命令来包含文件： ::

    include foo *.mk $(bar)

包含文件可以是变量引用和函数调用，它们会先进行展开，还可以包含文件名称匹配字符，比如上面的
例子，如果有三个 .mk 文件 a.mk，b.mk，c.mk，并且 $(bar) 的值是 bish bash，那么相当于：
include foo a.mk b.mk c.mk bish bash。如果文件名为空，相当于不包含文件，不会报错。

如果包含的文件没有包含斜杠字符或路径字符，并且在当前文件夹找不到，将依次找寻以下文件夹：
命令行选项 `-I` 或 `--include-dir` 指定的目录，/usr/gnu/include，/usr/local/include，
/usr/include。.INCLUDE_DIRS 变量包含当前搜寻的目录列表。如果你不想搜寻这些目录，可以设置
命令行选项 `-I-`。

包含的文件找不到不会马上报错，make 会继续处理 include 后续的文件，在这之后 make 会尝试更新
不是最新的 makefile，以及尝试创建不存在的 makefile。当 make 寻找生成这个文件的规则生成这个
文件时，如果没有找到这种规则，或者找到了规则但是执行操作命令失败，才会报包含的文件找不到的错
误。如果你不想 make 报这种错误，可以使用 -include 代替 include，为了兼容其他 make 实现，也
可以使用 sinclude。

如果定义了 MAKEFILES 环境变量，make 会读取任何其他 makefile 之前先加载这个变量里面定义的
makefile 文件。注意的是，默认目标不会从这些文件中选择，也不会在这些文件包含的文件中选择。另外
如果这些文件不存在，也不会报错。

MAKEFILES 环境变量的主要目的，是用来沟通递归调用中的 make。通常不应该在顶层 make 执行之前就
定义这个环境变量，这样会在外面搞乱 makefile。但是，如果你运行 make，没有指定的 makefile，这
个环境变量中定义的 makefile 可以做一些有用的事情帮助内置的隐式规则更好的工作，比如定义搜索的
路径。

更新 makefile 文件
-------------------

有时 makefile 是由其他文件生成的，比如 RCS 或者 SCCS 文件。在 make 读取 makefile 时，make
会把 makefile 文件名当作目标尝试更新这个 makefile 文件，确保这个 makefile 是最新的或者生成
这个 makefile 文件。如果打开了并行构建，makefile 的更新也会并行更新。

如果一个 makefile 存在这个规则或者有隐式规则表明怎样更新目标 makefile，那么这个规则会被执行。
这样，make 的执行顺序是，先处理完所有的 makefile，然后检查每个 makefile 名称对应的目标是否会
生成更新的 makefile，如果任意一个 makefile 有更新，make 会以干净的状态重新处理一遍这所有的
makefile。每一次重新执行，MAKE_RESTARTS 这个变量都会加一，记录重新执行的次数。

为了执行的效率，如果你确切的知道一个或多个 makefile 不存在更新，为了避免 make 查找隐式规则更
新这些文件，你可以显式的写一个同名目标名称的空操作命令规则。另外，如果指定一个双冒号的无条件有
操作命令的规则，这个目标文件会真正的被无条件更新（不管这个文件是否已经存在）。但是对于生成
makefile，这种规则每次 make 重入都无条件执行的话，会造成无限嵌套循环，所以 make 不会对 makefile
名称的这种双冒号规则无条件执行。伪目标也是相同的效果，make 也不会对对应的伪目标无条件更新： ::

    Makefile::
        recipe
        ...

    .PHONY: Makefile2
    Makefile2:
        recipe
        ...

可以利用上面的这一点，避免 make 对 makefile 进行更新，比如你确认默认的 Makefile 是不存在更新
的：Makefile:: ; 或者 .PHONY: Makefile 。

注意 make 的默认 makefile 或者用命令行指定的 makefile 可以不存在，并不是错误，make 并不总是
需要 makefile 文件，它可以执行隐式规则。但是 include 包含的 makefile 文件不存在会报错。

使用 -t -q -n 不会阻止 make 去更新 makefile 文件，即使这些选项指定不去执行操作命令，但是 make
仍然会执行构建 makefile 的操作命令。因此 make -f mfile -n foo 会首先更新 mfile，然后在新的
mfile 基础上执行对应的操作。

但是少数情况下，你可能想同时阻止对 makefile 文件的更新，你可以同时将 makefile 文件设置成 make
的命令行目标。例如 make -f mfile -n mfile foo，这时 make 会读取 mfile 文件，但是不会更新 mfile，
只会打印对应的命令，目标foo也会在未更新的 mfile 上执行。

继承另一个 makefile 的内容
--------------------------

通常，给同一个目标定义不同的操作命令是非法的。但是有一种其他的方式，通过使用模板规则，当 make
在当前 makefile 没有找到对应规则时，匹配当前 makefile 里定义的模板规则，找到对应的 makefile
文件。 ::

    foo:
        frobnicate > foo
    %: FORCE
        @$(MAKE) -f other_makefile $@
    FORCE: ;

例如执行 make foo 或执行当前文件的目标 foo，但执行 make bar 时会执行： ::

    bar: FORCE
        make -f other_makefile bar

如果 other_makefile 提供了更新 bar 的规则，就会执行这个规则。这个方法能够实行，是因为模板
目标 %，可以匹配任何目标。另外这个目标的条件设成 FORCE，保证匹配的目标即使已经存在也会被执行，
而且这个 FORCE 目标是一个空操作命令的目标，避免 make 寻找隐式规则生成这个目标，导致循环应用
% 匹配规则，再次生成 FORCE 目标自己，导致无限迭代循环。

Make 怎样读取一个 makefile 文件
--------------------------------

Make 用两个步骤完成工作。第一步，它读取所有的 makefile 文件，包括包含的 makefile，然后内部化
所有的变量和值以及隐式和显示规则，之后创建出所有目标和它们条件之间的一个完整的依赖图。第二步，
make 使用这些内化数据去决定那些目标需要更新，并执行对应规则的操作命令更新目标。

对这两个步骤的理解，对变量和函数展开的理解是非常关键的。我们常说的立即展开，说的是在第一阶段，
即 makefile 在读取时就会被展开；而延时展开，它的展开只有到这个结构被使用的时候才进行，即它或
者在第一阶段被引用在一个即时展开的上下文中，或者在第二阶段被使用时。

即时和延时的场景如下： ::

    immediate = deferred
    immediate ?= deferred
    immediate := immediate or ::= immediate
    immediate :::= immediate-with-escape # $会被转义成$$
    immediate += deferred or immediate
    immediate != immediate # 右边部分会被立即展开并且传给shell处理，处理的结果保存在左边的变量中

    define immediate
    deferred
    endef
    define immediate =
    deferred
    endef
    define immediate ?=
    deferred
    endef
    define immediate := or ::=
    immediate
    endef
    define immediate :::=
    immediate-with-escape
    endef
    define immediate +=
    deferred or immediate
    endef
    define immediate !=
    immediate
    endef

    immediate: immediate ; deferred
        deferred

规则的展开都是上面所示的相同的方式展开，这对所有的规则都适用，不管是显式规则、隐式规则、模板
规则、后缀规则、静态模板规则、还是简单的依赖条件定义。

另外条件语句是即时的，这就是说自动变量不能用在条件语句中，因为自动变量在规则操作命令被执行时
才设置。如果你要在条件中使用自动变量，必须把条件移到规则的操作命令中，并且使用 shell 的条件
语句语法。

在读取 makefile 文件时，makefile 是被以行为单位进行解析的，解析的步骤如下：

1. 读取一个逻辑行，包括用反斜杠转义的后续行
2. 移除行注释
3. 如果行以操作命令前缀字符（通常为Tab）开始，并且是在规则定义的上下文中，将当前行添加到规则的操作命令序列中
4. 扩展在即时扩展上下文中的当前行中的对应元素
5. 扫描行中的字符看是否包含:或者=，来分辨是赋值还是规则
6. 内部化改行读到的数据

另外，变量扩展可以扩展成一个完整的规则定义，但是内容需要定义成一行： ::

    myrule = target: ; echo built
    $(myrule)

如果变量定义成多行，变量的扩展会把空白字符忽略，规则的操作命令会错误的变成依赖条件。下面的例
子中，$(myrule) 会扩展成 target: echo built，不是想要的结果。如果要合适地展开多行内容的变
量，必须使用 eval 函数。 ::

    define myrule
    target:
        echo built
    endef
    $(myrule)

二次展开
---------

如前面介绍的 make 工作有两个阶段，第一阶段是读取阶段，第二阶段是目标更新阶段。make 还可对目标
的前置条件在目标更新阶段进行二次展开，为此必须在需要二次展开的前置条件之前定义一个特殊的目标
.SECONDEXPANSION。

一般情况下，二次展开没有什么效果，因为所有的变量和函数的引用都已经在 make 工作的第一阶段已经展
开过了。 ::

    # 第一次展开成 myfile: onefile $(TWOVAR)，第二次展开 myfile: onefile twofile
    .SECONDEXPANSION:
    ONEVAR = onefile
    TWOVAR = twofile
    myfile: $(ONEVAR) $$(TWOVAR)

    # 第一次展开成 onefile: top twofile: $(AVAR)，第二次展开 twofile: bottom
    .SECONDEXPANSION:
    AVAR = top
    onefile: $(AVAR)
    twofile: $$(AVAR)
    AVAR = bottom

    # 第一次展开 main: $($@_OBJS) main: $(main_OBJS) lib: $(lib_OBJS)，
    # 第二次展开 main: main.o try.o test.o lib: lib.o api.o
    .SECONDEXPANSION:
    main_OBJS := main.o try.o test.o
    lib_OBJS := lib.o api.o
    main lib: $$($$@_OBJS)

    .SECONDEXPANSION:
    foo: foo.1 bar.1 $$< $$^ $$+ # 展开成 foo: foo.1 bar.1，因为之前没有foo目标
    foo: foo.2 bar.2 $$< $$^ $$+ # 展开成 foo: foo.2 bar.2 foo.1 foo.1 bar.1 foo.1 bar.1
    foo: foo.3 bar.3 $$< $$^ $$+ # 展开成 foo: foo.3 bar.3 foo.1 foo.1 bar.1
    # foo.2 bar.2 foo.1 bar.1 foo.2 bar.2 foo.1 foo.1 bar.1 foo.1 bar.1

显式规则中，$$@ 和 $$% 展开成规则目标，$$< 展开成这个目标的第一个规则的第一个条件，$$^ 和 $$+
展开成规则的所有条件，不同的是 $$+ 不会合并相同项而是原样展开。另外 $$? 和 $$* 会展开成空字符
串。静态模板规则跟显式规则基本相同，不同的是 $$* 会被置成匹配词干。

隐式规则的二次展开，当 make 寻找隐式规则时，变量引用和函数调用会先被执行，然后再对 % 进行替换，
然后对 $^ 等自动变量进行展开。 ::

    .SECONDEXPANSION:
    foo: bar
    foo foz: fo%: bo%
    %oo: $$< $$^ $$+ $$*

像目标foo寻找隐式规则执行时，上面的规则相当于： ::

    foo: bar
    foo: boo
    foo: bar bar boo bar boo f

其中 $$< 扩展成 bar，$$^ 扩展成 bar boo，$$+ 也扩展成 bar boo，$$* 扩展成 f。再看下面的
例子： ::

    .SECONDEXPANSION:
    /tmp/foo.o:
    %.o: $$(addsuffix /%.c,foo bar) foo.h
        @echo $^

构建 /tmp/foo.o 时由于没有找到操作命令，因此会搜寻隐式规则，如果目标模板不包含目录，文件名会
分成目录部分 /tmp/ 和文件名部分 foo.o，然后用文件名部分进行匹配，将匹配词干替换%后，再加上目
标部分： ::

    %.o: $(addsuffix /%.c,foo bar) foo.h
        @echo $^
    %.o: foo/%.c bar/%.c foo.h
        @echo $^
    %.o: foo/%.c bar/%.c foo.h
        @echo $^
    /tmp/foo.o: /tmp/foo/foo.c /tmp/bar/foo.c foo.h
        @echo $^

如果你对目录识别感兴趣，再前置条件中可以用 $$* 代替 %: ::

    %.o: $$(addsuffix /$$*.c,foo bar) foo.h
        @echo $^

    # 依次扩展为：
    %.o: $(addsuffix /$*.c,foo bar) foo.h
        @echo $^
    %.o: foo/$*.c bar/$*.c foo.h
        @echo $^
    %.o: foo/foo.c bar/foo.c foo.h
        @echo $^

条件编译
---------

根据变量的值是否满足条件，条件指示符可以让 make 选择性的解析或忽略 makefile 文件定义的部分
内容，相当于 C/C++ 编程语言中的条件编译语句。例如： ::

    libs_for_gcc = -lgnu
    normal_libs =
    foo: $(objects)
    ifeq ($(CC),gcc)
        $(CC) -o foo $(objects) $(libs_for_gcc)
    else
        $(CC) -o foo $(objects) $(normal_libs)
    endif

    ifeq ($(CC),gcc)
    libs = $(libs_for_gcc)
    else
    libs = $(normal_libs)
    endif
    foo: $(objects)
        (CC) -o foo $(objects) $(libs)

详细的条件语句语法如下： ::

    conditional-directive
    text-if-true
    endif

    conditional-directive
    text-if-true
    else
    text-if-false
    endif

    conditional-directive-one
    text-if-one-is-true
    else conditional-directive-two
    text-if-two-is-true
    else
    text-if-one-and-two-are-false
    endif

其中条件提示符有四种不同的测试形式，除了不能以 tab 作为行的第一个字符外，其他形式的空白字符
都可以出现在条件指示符之前，参数列表之前，或者参数列表之后： ::

    ifeq (arg1,arg2)
    ifeq 'arg1' 'arg2'
    ifeq "arg1" "arg2"
    ifeq "arg1" 'arg2'
    ifeq 'arg1' "arg2"
    ifeq ($(strip $(foo)),)
    text-if-empty
    endif

    ifneq (arg1,arg2)
    ifneq 'arg1' 'arg2'
    ifneq "arg1" "arg2"
    ifneq "arg1" 'arg2'
    ifneq 'arg1' "arg2"

    ifdef variable-name # 检查变量是否有非空值，但不会递归的去进行检查，
    ifndef variable-name # 如果要确切的检查值是否为空可以使用 ifeq ($(foo),)
    bar = true
    foo = bar
    ifdef $(foo) # 展开变成 bar，bar 有非空值
    frobozz = yes
    endif
    bar =
    ifdef bar # 为 false 因为 bar 值为空
    frobozz = yes
    endif
    foo = $(bar)
    ifdef foo # foo 是有值的，这里不会递归地继续检查 $(bar)
    frobozz = yes
    else
    frobozz = no
    endif
    foo =
    ifdef foo # 为 false 因为 foo 值为空
    frobozz = yes
    else
    frobozz = no
    endif

Make 在读取 makefile 时进行条件语句的解析，因此自动变量不能作为条件的测试变量，因为它们在
规则命令运行时才被定义。为了避免混淆，不允许在一个 makefile 中开始条件，而在另一个 makefile
文件中结束条件。

测试 make 的命令行选项，注意 MAKEFLAGS 会被所有的单字符选项作为自己的抵押给单词，如果没有
单字符选项，第一个单词为空。为了不让第一个单词为空，可以在该变量的值前面加一个横杠字符。 ::

    archive.a: ...
    ifneq (,$(findstring t,$(firstword -$(MAKEFLAGS))))
        +touch archive.a # 加号让改行命令总会被执行，即使 make 命令行添加了不执行命令的选项
        +ranlib -t archive.a
    else
        ranlib archive.a
    endif
