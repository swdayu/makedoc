内置名称和惯例
==============

内置目标
---------

.PHONY
    定义伪目标
.SUFFIXES
    后缀规则中定义已知后缀类型
.DEFAULT
    定义默认规则，或者清掉前面所有的默认规则
.INTERMEDIATE
    设置成中间文件
.NOTINTERMEDIATE
    取消对应的中间文件，或取消所有的中间文件设置
.SECONDARY
    设置成辅助文件
.SECONDEXPANSION
    开启二次展开
.IGNORE
    忽略规则中操作命令执行时的错误
.SILENT
    所有操作命令或指定目标的操作命令在执行前不回显
.EXPORT_ALL_VARIABLES
    把所有的 make 变量都传递到环境中，相当于声明一个不带参数的 export
.ONESHELL
    每个规则的所有命令在一个 shell 进程中执行
.NOTPARALLEL
    带前置条件则对应的目标不会并行执行，不带前置条件关闭当前 make 的并行执行
.WAIT
    后面的前置条件必须等前面前置条件都完成后才执行
.DELETE_ON_ERROR
    前置条件中的目标，当构建时出错时删除目标文件
.PRECIOUS
    前置条件中的目标，即使构建时出错也不会被删除
.POSIX
    进入POSIX严格模式
.LOW_RESOLUTION_TIME
    标记一个文件的时间戳是低精度的，只要进行秒级别的比较

因为 cp -p 会丢弃源文件 src 秒级别后面的时间，即使更新了 dst，它的时间戳还是比 src 更
老，因此需要设置成 .LOW_RESOLUTION_TIME 告诉 make 对于这个目标，只需要进行低精度时间戳
的比较。另外，由于归档格式的限制，归档成员文件的时间戳都是低精度的，但是 make 会自动处理
归档成员文件的低精度时间戳。 ::

    .LOW_RESOLUTION_TIME: dst
    dst: src
        cp -p src dst

内置变量
---------

.DEFAULT_GOAL
    设置默认目标，只能设置一个目标，初始值是第一个用户目标
.RECIPEPREFIX
    规则操作命令行的第一个字符，默认是空表示使用 Tab 键
.VARIABLES
    当前已经定义的全局变量列表，只读
.FEATURES
    当前的make支持的特性
.INCLUDE_DIRS
    包含 makefile 文件的搜索目标列表，只读
.EXTRA_PREREQS
    给目标添加额外的前置条件，但这些前置条件不会被扩展到自动变量中
.LOADED
    当前成功加载的动态库文件列表
MAKE
    当前的make程序
MAKEFILE_LIST
    当前 make 解析的 makefile 文件列表，make 会在读取一个 makefile 之前将它加到这个列
    表
MAKE_RESTARTS
    由于有 makefile 文件的更新，当前的 make 实例重新启动的次数，初始值为 0，不应该
    export 这个值
MAKELEVEL
    make 递归的嵌套深度，顶层 make 的值是 0
MAKE_TERMOUT
    make 是否将 stdout 输出到终端，如果设置了会被 export，只读
MAKE_TERMERR
    make 是否将 stderr 输出到终端，如果设置了会被 export，只读
MAKE_HOST
    当前 make 运行的主机平台
MAKE_VERSION
    当前 make 的版本
CURDIR
    make 在启动之后，处理完 -C 选项，就会设置好这个变量
DESTDIR
    仅用于 install 或 uninstall 目标的变量，可以在 make 命令行设置安装目录，如 make
    DESTDIR=/tmp/stage install
MAKEFILES
    预加载的 makefile 文件列表
MAKEFLAGS
    make 的命令行选项和命令行参数
MFLAGS
    仅老版本使用，为了兼容而存在
MAKEOVERRIDES
    如果不想正常的传递命令行参数，可以将这个变量设为空，但是 POSIX 严格模式下不生效
GNUMAKEFLAGS
    GNU make 与其他 make 混用时，你可能将 GNU make 特有的命令行选项放到这个变量下
OUTPUT_OPTION
    每个生成目标文件的规则，都是用了这个目标文件输出选项
SHELL
    指定 make 使用的 shell 程序
.SHELLFLAGS
    指定 shell 程序的命令行选项
.SHELLSTATUS
    上一次 shell 执行完的错误代码
MAKESHELL
    MS系统会优先使用这个环境变量中的值
MAKE_TMPDIR
    环境变量
MAKECMDGOALS
    命令行中指定的目标列表
VPATH
    定义目标和前置条件中文件的搜索目录类型，用空白字符或冒号分隔
vpath
    操作提示符，为特定的一类文件指定搜索目录列表，它的优先级比 VPATH 高
GPATH
    如果一个文件是通过目录搜索找到的，并且这个目录在 GPATH 定义的目录列表中，那么这个文
    件会更新到对应的目录下
.LIBPATTERNS
    库文件名称的模板，默认是 lib%.so lib%.a
\-lname
    前置条件中可以添加一个库文件，库文件有特定的搜索方式

默认目标的使用： ::

    ifeq ($(.DEFAULT_GOAL),)
    $(warning no default goal is set)
    endif
    .PHONY: foo
    foo: ; @echo $@
    $(warning default goal is $(.DEFAULT_GOAL))
    .DEFAULT_GOAL :=
    .PHONY: bar
    bar: ; @echo $@
    $(warning default goal is $(.DEFAULT_GOAL))
    .DEFAULT_GOAL := foo

如果在全局设置 .EXTRA_PREREQS 变量，其中的目标会添加到所有的目标中，除了那些定义了自己
版本的 .EXTRA_PREREQS 变量的目标。 ::

    myprog: myprog.o file1.o file2.o
        $(CC) $(CFLAGS) $(LDFLAGS) -o $@ $^ $(LDLIBS)
    myprog: .EXTRA_PREREQS = $(CC)

当前make可能支持的特性有：

archives
    支持归档文件
check-symlink
    支持 -L 选项
else-if
    支持 else if 非嵌套条件语句
extra-prereqs
    支持 .EXTRA_PREREQS 特殊目标
grouped-target
    支持组目标
guile
    支持 GNU Guile 扩展语言
jobserver
    支持 jobserver 增强并行构建
jobserver-fifo
    支持使用命名管道方式的增强并行构建
load
    支持动态库的加载
notintermediate
    支持 .NOTINTERMEDIATE 特殊目标
oneshell
    支持 .ONESHELL 特殊目标
order-only
    支持 order-only 的前置条件
output-sync
    支持并行执行时的打印同步
second-expansion
    支持二次展开
shell-export
    支持将 make 变量导出到 shell 函数
shortest-stem
    在匹配模板规则时，支持最短词干优先的匹配方法
target-specific
    支持基于目标或者模板目标的变量定义
undefine
    支持取消变量定义提示符

内置隐式规则变量
----------------

AR - ar,
    archive-maintaining program
AS - as,
    program for compiling assembly files
CC - cc,
    program for compiling C programs
CXX - g++,
    program for compiling C++ program
CPP - $(CC) -E,
    program for running the C preprocessor, with results to stdout
FC - f77,
    program compiling or preproccessing Fortran and Ratfor programs
M2C - m2c,
    program to use to compile Modula-2 source code
PC - pc,
    program for compiling Pascal programs
CO - co,
    program for extracting a file from RCS
GET - get,
    program for extracting a file from SCCS
LEX - lex,
    program to use to turn Lex grammars into source code
YACC - yacc,
    program to use to turn Yacc grammars into source code
LINT - lint,
    program to use to run lint on source code
MAKEINFO - makeinfo,
    program to convert a Texinfo source file into an Info file
TEX - tex,
    program to make TEX DVI files from TEX source
TEXI2DVI - texi2dvi,
    program to make TEX DVI files from Texinfo source
WEAVE - weave,
    program to translate Web into TEX
CWEAVE - cweave,
    program to translate C Web into TEX
TANGLE - tangle,
    program to translate Web into Pascal
CTANGLE - ctangle,
    program to translate C Web into C
RM - rm -f,
    command to remove a file
ARFLAGS - rv,
    flags to give the archive-maintaining program
ASFLAGS - ,
    extra flags to give to the assembler
CFLAGS - ,
    extra flags to give to the C compiler
CXXFLAGS - ,
    extra flags to give to the C++ compiler
COFLAGS - ,
    extra flags to give to the RCS co program
CPPFLAGS - ,
    extra flags to give to the C preprocessor and programs that use it
FFLAGS - ,
    extra flags to give to the Fortran compiler
GFLAGS - ,
    extra flags to give to the SCCS get program
LDFLAGS - ,
    extra flags to give to compilers when they are supposed to invoke the linker,
    such as -L
LDLIBS - ,
    library flags or names given to compilers when they are supposed to invoke
    the linker, such as -lfoo
LOADLIBES - ,
    deprecated but still supported, same as LDLIBS
LFLAGS - ,
    extra flags to give to Lex
YFLAGS - ,
    extra flags to give to Yacc
PFLAGS - ,
    extra flags to give to the Pascal compiler
RFLAGS - ,
    extra flags to give to the Fortran compiler for Ratfor programs
LINTFLAGS - ,
    extra flags to give to lint

常用伪目标
----------

all

clean

install

mostlyclean 
    清除大多数但还是会保留一些文件，比如 GCC 不会删除 libgcc.a，因为基本都有用到这个文
    件，而且构建这个文件需要花大量时间

distclean

realclean

clobber
    删除更多文件，甚至一些 makefile 都不能构建的文件

print
    打印发生改变的所有源文件

tar
    创建 tar 文件

shar
    创建一个 shell 归档文件

dist
    创建一个发布文件，可能时一个 tar 文件，shar 文件，压缩文件等等

TAGS
    更新当前程序的 tags 列表

check

test
    执行程序的自测试

内核构建系统
------------

Liunx 内核 Makefile 文件分为五个部分：

Makefile
    顶层 makefile
.config
    kernel 配置文件
arch/$(SRCARCH)/Makefile
    平台架构相关的 Makefile
scripts/Makefile.*
    通用规则等可用于所有 kbuild makefile 中的公共文件
kbuild Makefiles
    每个子模块的 makefile

顶层 Makefile 读取 .config 配置文件，负责构建内核映像 vmlinux 和各模块，它根据内核的
源文件目录树来递归地构建这些文件。需要构建的子目录列表依靠 kernal 配置文件配置。同时顶
层 Makefile 还会包含平台架构相关的文件 ``arch/$(SRCARCH)/Makefile``，该文件描述了特
定平台架构的相关信息。

每个子目录定义自己的 kbuild Makefile，用来描述当前模块的构建方式，在具体构建时，make
根据 config 中的配置来构建这个模块。

模块中要构建的目标添加到 obj-y 这个变量中，例如： ::

    obj-y += foo.o

这告诉 kbuild 当前模块有一个名为 foo.o 的目标文件需要构建，foo.o 可以从 foo.S foo.c
或者 foo.cpp 构建出来。

Kduild 会编译所有 ``$(obj-y)`` 中的文件，然后调用 ``$(AR) rcSTP`` 将这些文件打包到一
个 built-in.a 文件中，最后会被链接到最终的 vmlinux 文件中，通过 scripts/link-vmlinux.sh
脚本。

``$(obj-y)`` 中的文件顺序是重要的，但是允许重复，这些文件按顺序构建到 built-in.a 文件
中，后面的文件如果与前面的文件名重复会被忽略。最后的链接顺序也是重要的，因为一些函数的会
被其模块调用存在依赖关系。

目标文件还可以放到 lib-y 中，基于这些目标文件最后会生成一个库文件 lib.a，通常 lib-y 的
使用通常仅限于 lib/ 或者 ``arch/*/lib`` 目录。

一个 Makefile 文件仅负责它自己目录下的目标的构建，子目录下文件的构建应该依赖于子目录下
的 Makefile。kbuild 会自动递归去构建子目录，只要将子目录添加到 obj-y 中，例如： ::

    obj-y += ext2/

模块顶层的 built-in.a 会将所有子目录下的 built-in.a 包含到它自己里面，lib.a 也一样。

**extra-y 和 always-y**

可以将构建 vmlinux 需要的，但不会被包含到 built-in.a 中的目标，放到 extra-y 中。例如： ::

    # arch/x86/kernal/Makefile
    extra-y += vmlinux.lds

该文件位于 ``arch/$(SRCARCH)/kernel/vmlinux.lds``。

always-y 指定那些总是会被构建的目标，例如： ::

    # ./Kbuild
    offsets-file := include/generated/asm-offsets.h
    always-y += $(offsets-file)

**编译选项**

ccflags-y asflags-y ldflags-y
    这些选项仅用于当前模块调用 cc/as/ld 时使用，即设置它们的那个 Makefile 文件对应的模
    块。它们分别有已经过时的老名称 EXTRA_CFLAGS，EXTRA_AFLAGS，EXTRA_LDFLAGS，这些老
    名称暂时仍是支持的。

subdir-ccflags-y subdir-asflags-y
    这些选项在当前文件和所有的子目录文件中生效，subdir- 版本的选项会优先于上面不带
    subdir- 的版本添加到命令行选项中。

ccflags-remove-y asflags-remove-y
    这些选项用于对命令行选项进行移除。

CFLAGS_$@ AFLAGS_$@
    仅对指定的目标文件生效，即在构建这个目标文件时这个选项才生效。这种选项的优先级比
    -remove-y 版本高，在 -remove-y 中移除的选项可以使用它加回来。

Kbuild 会跟踪的依赖文件包括：

1. 所有的前置条件，包括 .c 和 .h
2. 在上面前置条件中使用的任何CONFIG_宏配置
3. 用于编译目标文件的命令行

因此，如果你修改了例如 ``$(CC)`` 的一个选项，所有受影响的文件都会重新编译。

**定制规则**

下面是定制规则使用的一些变量，所有定制规则都必须使用相对路径。

$(src)
    这个变量保存的是相对于 srctree 源代码根目录的，当前 Makefile 的路径。

$(obj)
    这个变量保存的是相对于 objtree 构建根目录的，当前模块中的文件的输出路径。

$(kecho)
    可以打印信息，但是使用了 make -s 选项时，除了警告和错误信息，其他都不会打印。另外当
    打印执行的命令信息时，如果 KBUILD_VERBOSE 没有设置，只会打印缩短版本的命令信息，这
    样需要定义两个版本的命令： ::

        # lib/Makefile
        quiet_cmd_crc32 = GEN $@
              cmd_crc32 = $< > $@
        $(obj)/crc32table.h: $(obj)/gen_crc32table
            $(call cmd,crc32)
        # 这样 KBUILD_VERBOSE= 设置为空时，当更新 $(obj)/crc32table.h 文件会打印：
        GEN lib/crc32table.h

命令变化的检测
--------------

当一个规则被执行时，目标文件的时间戳会与它的每个前置条件中的文件的时间戳比较。GNU make
会自动更新目标文件，只要人一个前置条件中的文件的时间戳比它更新。

但是，目标文件在当命令行发生变化时也因该重新构建。这时 GNU make 不支持的，kbuild 提供了
if_changed 宏来实现这个功能，例如： ::

    quiet_cmd_<command> = ...
          cmd_<command> = ...
    <target>: <source(s)> FORCE
        $(call if_changed,<command>)

任何使用 if_changed 的目标都必须出现在 ``$(targets)`` 列表中，否则命令的检查会失败，
对应的 target 总是会被执行。对于出现在 obj-y/m，lib-y/m，extra-y/m，always-y/m，
hostprgs，userprogs 中的目标，kbulid 会自动将它们添加到 ``$(targets)`` 中。其他的目
标，必须手动添加。另外添加到 ``$(targets)`` 中的目标不需要添加 ``$(obj)/`` 前缀。

一个目标规则中，if_changed 只能使用一次，它会把当前执行的命令保存到对应 .cmd 文件中，多
次调用会导致覆盖和不一样的结果。

支持函数
---------

as-option
    用来给 ``$(CC)`` 添加一个编译 .S 文件时的选项，例如： ::

        ccflags-y += $(call as-option,-Wa$(comma)-isa=$(isa-y),)

    表示当 ``$(CC)`` 编译 .S 时如果支持 ``-Wa,-isa=$(isa-y)`` 选项就添加它，否则不添
    加。如果提供了第三个参数，当不支持第二个参数时添加第三个参数作为选项。

as-instr
    检查对应汇编器是否支持特定的指令。

cc-option
    检查 ``$(CC)`` 是否支持对应的选项，例如： ::

        ccflags-y += $(call cc-option,-march-pentium-mmx,-march=i586)

cc-option-yn
    检查 gcc 是否支持对应的选项，返回 y 表示支持，返回 n 表示不支持。例如： ::

        biarch := $(call cc-option-yn,-m32)
        asflags-$(biarch) += -a32
        ccflags-$(biarch) += -m32

cc-disable-warning
    检查 gcc 是否支持对应的警告 disable 警告，例如： ::

        KBUILD_CFLAGS += $(call cc-disable-warning,unused-but-set-variable)

    只有当支持 -Wno-unused-but-set-variable 时才添加。

gcc-min-version
    返回 y 仅当 ``$(CONFIG_GCC_VERSION)`` 大于等于提供的版本时： ::

        ccflags-$(call gcc-min-version,70100) := -foo

    clang-min-version 针对 clang 有相同的功能。

cc-cross-prefix
    检查参数对应的前缀 ``$(CC)`` 是否支持，即 ``prefix$(CC)`` 是否支持。例如： ::

        ifeq ($(CROSS_COMPILE),)
        CROSS_COMPILE := $(call cc-cross-prefix,m68k-linux-gnu-)
        endif

ld-option
    检查 ``$(LD)`` 是否支持对应的选项。例如： ::

        LDFLAGS_vmlinux += $(call ld-option,-X)

调用脚本
--------

Kbuild 提供了 ``$(CONFIG_SHELL)``， ``$(AWK)``， ``$(PERL)``， ``$(PYTHON3)`` 等变
量用来执行对应的脚本。例如： ::

    cmd_depmod = $(CONFIG_SHELL) $(srctree)/scripts/depmod.sh $(DEPMOD) $(KERNELRELEASE)

模块目录
--------

顶层 Makefile 使用 core-y，libs-y，drivers-y 来决定构建哪些模块。其中 ``$(libs-y)`` 
指出模块的 lib.a 文件的目录路径，其他两个指定对应模块的 built-in.a 的目录路径。

编译逻辑
--------

目标文件的编译

1. 开始编译 obj/path/foo.o 文件，因为全新编译，还没有该目标依赖的规则，只能使用隐式规
   则： ::

        $(obj)/%.o: $(src)/%.c FORCE
            $(call if_changed_dep,c_to_o)

2. if_changed_dep 中 target_cmd_change 检测到命令有变化（还没有保存命令），执行
   cmd_c_to_o 编译源文件

3. 成功编译出 obj/path/foo.o，并且编译器自动生成了目标依赖文件 obj/path/.foo.o.d

4. 目标依赖文件包含了该目标的具体依赖，这些依赖有源文件 src/path/foo.c 和该源文件包含
   的头文件 obj/path/foo.o: src/path/foo.c foo.h header.h

5. 目标依赖文件还定义了编译该目标使用的编译命令： ::

        cmd_obj/path/foo.o := gcc -I... -D... -W... -c -o obj/path/foo.o src/path/foo.c

第一次编译完成后，重新启动一次编译：

1. 开始编译 obj/path/foo.o 文件，因为该目标已经有了定义的依赖规则，使用该规则进行编译： ::

        obj/path/foo.o: src/path/foo.c foo.h header.h $(src)/%.c FORCE
            $(call if_changed_dep,c_to_o)

2. 因为没有文件更新，且 target_cmd_change 检测到编译命令没有变化

3. 因此不会执行编译

后面再次编译，只有在修改了下列内容才能触发重新编译：

1. 修改了源文件 src/path/foo.c，或者修改了其中的头文件；
2. 修改了对应模块的 Makefile 文件，或者顶层的 Makefile 文件，使得编译命令发生了变化；
3. 删除了 obj/path/foo.o 文件，或者删除了obj/path/.foo.o.d 文件

归档目标文件的编译

1. 如果一个目标文件 obj/path/foo.o 是一个归档目标文件，会被区别对待

2. 归档的目标文件需要在模块 Makeilfe 中定义对应的变量 obj/path/foo.o-y 声明其依赖的文
   件

3. 对每个归档文件都会定义两个规则，一个规则定义构建这个归档文件的操作命令： ::

        obj/path/foo.o: FORCE
            $(call if_changed,ar_src_file)

4. 另一个规则是为每个库文件动态生成的规则，用于定义这个库文件依赖的文件： ::

    obj/path/foo.a: obj/path/bar.o obj/path/baz.a obj/subdir/built-in.a

5. 依赖的文件可以包含一系列的 .o，一系列的 .a，以及一系列的子目录模块

6. 当这些依赖的文件依次构建后，通过 cmd_ar_src_file 创建出最终的库文件

7. 如果归档文件的依赖变量 obj/path/foo.o-y 没有定义或者为空，则表明这是一个普通的目标
   文件

8. 普通的目标文件必须存在一个对用的源文件 obj/path/foo.c，如果没有就会报错

从源文件编译库文件

1. 每一个 obj/path/foo.a 都会定义两个规则，一个规则定义构建这个库文件的操作命令： ::

        obj/path/foo.a: FORCE
            $(call if_changed,ar_src_file)

2. 另一个规则是为每个库文件动态生成的规则，用于定义这个库文件依赖的文件： ::

        obj/path/foo.a: obj/path/foo.o obj/path/bar.a obj/subdir/built-in.a

3. 库文件的依赖文件在模块 Makefile 中，用变量 obj/path/foo.a-y 进行定义

4. 依赖的文件可以包含一系列的 .o，一系列的 .a，以及一系列的子目录模块

5. 当这些依赖的文件依次构建后，通过 cmd_ar_src_file 创建出最终的库文件

子目录模块的编译

1. 每个子模块 obj/subdir/ 都会定义两个规则，第一个规则说明子模块要构建的目标： ::

        obj/subdir/built-in.a: obj/subdir ;

2. 第二个规则是去递归的调用 MAKE 程序真正编译子模块： ::

        obj/subdir:
            $(Q)$(MAKE) $(build)=obj/subdir

3. 当进入子模块编译时，子模块的 obj 变量变成了 obj/subdir，这样它可以进一步编译它的子
   模块： ::

        obj/subdir/subdir:
            $(Q)$(MAKE) $(build)=obj/subdir/subdir
