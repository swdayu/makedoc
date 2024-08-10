隐式规则
=========

一些标准的构建目标的方式是通用的，比如使用 C 编译器把 C 源文件编译成目标文件。隐式规则的
目的就是将这些通用规则标准化，不需要你再次详细描述规则的细节。根据需要，make 可以应用一
连串隐式规则来构建目标。内建的隐式规则使用了一些变量，通过修改这些变量，可以变量隐式规则
构建行为。例如 CFLAGS。另外，你可以通过模板规则，定义自己的隐式规则。另外一种过时的方式，
是使用后缀规则来定义隐式规则，但是它的功能有限，留着的目的仅用于版本兼容，使用模板规则更
通用更清晰。

每条隐式规则，都有一个目标模板和前置条件模板，而且可能多个隐式规则有着相同的目标模板。例
如，有多种构建 .o 文件的规则，比如 .c 文件构建，.p 文件通过 Pascal 编译器构建，等等。
哪个目标会被真正构建，要看对应的前置条件是否存在并且能够被构建。例如当需要构建 foo.o 时，
如果你有文件 foo.c，make 就会使用 C 编译器构建，如果有文件 foo.p 则会使用 Pascal 编译
器来构建。

一般的，make 会对每个没有操作命令的规则或双冒号规则的目标，搜索对应的隐式规则来构建这个
目标。如果一个文件仅出现在前置条件中，也即对于这个目标没有对应的显式规则，也会搜索隐式规
则。注意显式的前置条件，并不会影响 make 对隐式条件的搜寻，比如： ::

    foo.o: foo.p

虽然显式的描述了 foo.o 依赖的是 foo.p 文件，但是如果同时存在有 foo.c 文件，make 会选择
使用 foo.c 来构建 foo.o，因为在隐式规则中，.c 的隐式规则定义在 .p 的隐式规则之前。

如果你不想让一个没有操作命令的目标，触发隐式规则的搜寻，你可以定义一个空命令的目标规则，
通过使用分号（;）或者定义一行空指令。

内置的隐式规则
--------------

下面是 make 预定义的隐式规则列表，这些规则总是可用的，除非在 makefile 中显式地将它们覆
盖或取消掉了。使用选项 -r 或者 --no-builtin-rules 可以取消掉所有预定义的规则。

这里仅仅列出基于 POSIX 操作系统的默认可以规则，在其他操作系统比如 VMS、WIindows、OS/2
等等上，可能有不同的默认规则。可以使用 make -p 在没有 makefile 的目录下，查看你的 GNU
make 在你的系统上支持的完整的默认规则和内置变量。

大多数预定义内置规则是通过后缀名定义的，默认的后缀名列表是：.out .a .ln .o .c .cc .C
.cpp .p .f .F .m .r .y .l .ym .lm .s .S .mod .sym .def .h .info .dvi .tex .texinfo
.texi .txinfo .w .ch .web .sh .elc .el。下面所有那些依赖这个表中对应后缀名的规则，实
际上都是后缀规则。这个列表被定义在 .SUFFIXES 这个特殊目标的前置条件中，你可以修改这个变
量来开关隐式规则对对应后缀名文件的支持。 ::

    %.o: %.c
        $(CC) $(CPPFLAGS) $(CFLAGS) -c # CPP - C preprocessor

    %.o: %.cc
    %.o: %.cpp
    %.o: %.C
        $(CXX) $(CPPFLAGS) $(CXXFLAGS) -c

    %.o: %.p
        %(PC) %(PFLAGS) -c

    %.o: %.f
        %(FC) %(FFLAGS) -c
    %.o: %.F
        %(FC) $(FFLAGS) $(CPPFLAGS) -c
    %.o: %.r
        %(FC) $(FFLAGS) $(RFLAGS) -c
    %.f: %.F
        $(FC) $(CPPFLAGS) $(FFLAGS) -F
    %.f: %.r
        $(FC) $(FFLAGS) $(RFLAGS) -F

    %.sym: %.def
        $(M2C) %(M@FLAGS) $(DEFFLAGS)
    %.o: %.mod
        $(M2C) $(M@FLAGS) $(MODFLAGS)

    %.o: %.s
        $(AS) $(ASFLAGS)
    %.s: %.S
        $(CPP) $(CPPFLAGS)

    %: %.o
        $(CC) $(LDFLAGS) %.o $(LOADLIBES) $(LDLIBS)

对于多个目标文件，它也会做正确的事，比如： ::

    x: y.o z.o

当 x.c y.c z.c 存在时，make 会执行： ::

    cc -c x.c -o x.o
    cc -c y.c -o y.o
    cc -c z.c -o z.o
    cc x.o y.o z.o -o x
    rm -f x.o y.o z.o

当可执行文件没有对应名称的目标文件时，你需要写一个显式的规则来链接程序。 ::

    %.c: %.y
        $(YACC) $(YFLAGS)

    %.c: %.l
        $(LEX) $(LFLAGS)
    %.r: %.l
        $(LEX) $(LFLAGS)

这里如果你只想生成.r文件，你可以临时将.c后缀名从.SUFFIXES特殊目标中去掉，例如： ::

    .SUFFIXES:
    .SUFFIXES: .o .r .f .l ...

    %.ln: %.c
        $(LINT) $(LINTFLAGS) $(CPPFLAGS) -i

    %.dvi: %.tex
        $(TEX)
    %.tex: %.web
        $(WEAVE)
    %.tex: %.w
    %.tex: %.ch
        %(CWEAVE)
    %.p: %.web
        $(TANGLE)
    %.c: %.w
    %.c: %.ch
        $(CTANGLE)

    %.dvi: %.texinfo
    %.dvi: %.texi
    %.dvi: %.txinfo
        $(TEXI2DVI) $(TEXI2DVI_FLAGS)
    %.info: %.texinfo
    %.info: %.texi
    %.info: %.txinfo
        $(MAKEINFO) $(MAKEINFO_FLAGS)

    %: %,v
    %: RCS/%,v
        $(CO) $(COFLAGS)

    %: %.n
    %: SCCS/%.n
        $(GET) $(GFLAGS)

注意，内置的隐式规则中的操作命令，实际上使用了形如 COMPILE.c，LINK.p，以及 PREPROCESS.S
等变量，它们的值如上面所列的规则所示。也即 make 编译 .x 源文件时使用 COMPILE.x，相同的
链接 .x 文件生成可执行文件时使用 LINK.x，对 .x 进行预处理时使用 PREPROCESS.x。

另外每个生成目标文件的规则，都是用了 OUTPUT_OPPTION 变量，make 根据编译时选项，将这个
变量定义成 -o $@ 或者空。你需要使用 -o 选项将输出文件输出到正确的目录下当源文件在不同目
录时，例如当你使用了 VPATH 的时候。但是，一些系统上的编译器不支持 -o 选项，当你使用这种
系统并且使用 VPATH 时，一些编译会把文件输出到错误的地方，一种可能的规避方式是将 OUTPUT_OPTION
变量设置成 ``; mv $*.o $@``。

你可以根据需要，在 makefile 或者命令行或者环境变量中，修改隐式规则中使用的变量。你也可
以全部取消这些变量的定义，使用选项 -R 或者 --no-builtin-variables。

下面列出的最常用到的预定义变量的默认值，没有指定的表示默认值为空： ::

    AR - ar, archive-maintaining program
    AS - as, program for compiling assembly files
    CC - cc, program for compiling C programs
    CXX - g++, program for compiling C++ program
    CPP - $(CC) -E, program for running the C preprocessor, with results to stdout
    FC - f77, program compiling or preproccessing Fortran and Ratfor programs
    M2C - m2c, program to use to compile Modula-2 source code
    PC - pc, program for compiling Pascal programs
    CO - co, program for extracting a file from RCS
    GET - get, program for extracting a file from SCCS
    LEX - lex, program to use to turn Lex grammars into source code
    YACC - yacc, program to use to turn Yacc grammars into source code
    LINT - lint, program to use to run lint on source code
    MAKEINFO - makeinfo, program to convert a Texinfo source file into an Info file
    TEX - tex, program to make TEX DVI files from TEX source
    TEXI2DVI - texi2dvi, program to make TEX DVI files from Texinfo source
    WEAVE - weave, program to translate Web into TEX
    CWEAVE - cweave, program to translate C Web into TEX
    TANGLE - tangle, program to translate Web into Pascal
    CTANGLE - ctangle, program to translate C Web into C
    RM - rm -f, command to remove a file
    ARFLAGS - rv, flags to give the archive-maintaining program
    ASFLAGS - , extra flags to give to the assembler
    CFLAGS - , extra flags to give to the C compiler
    CXXFLAGS - , extra flags to give to the C++ compiler
    COFLAGS - , extra flags to give to the RCS co program
    CPPFLAGS - , extra flags to give to the C preprocessor and programs that use it
    FFLAGS - , extra flags to give to the Fortran compiler
    GFLAGS - , extra flags to give to the SCCS get program
    LDFLAGS - , extra flags to give to compilers when they are supposed to invoke the linker, such as -L
    LDLIBS - , library flags or names given to compilers when they are supposed to invoke the linker, such as -lfoo
    LOADLIBES - , deprecated but still supported, same as LDLIBS
    LFLAGS - , extra flags to give to Lex
    YFLAGS - , extra flags to give to Yacc
    PFLAGS - , extra flags to give to the Pascal compiler
    RFLAGS - , extra flags to give to the Fortran compiler for Ratfor programs
    LINTFLAGS - , extra flags to give to lint

隐式规则链
-----------

有时候，一个文件能够通过一系列隐式规则，按顺序来构建。例如 .o 可以由 .y 构建出 .c 再构
建出 .o。这种序列称为隐式规则链。当 n.c 存在或者描述在 makefile 中，那么 make 不会进行
特别的搜索，它先看 n.c 是否需要更新（例如有 n.y 且有修改），然后再生成 n.o。然而，如果
n.c 不存在或者没被提到，由于 make 知道怎样从 n.y 构建 n.o，因此会自动进行构建，产生一个
中间文件 n.c。中间文件于正常文件的两点区别是：第一，正常文件不存在时，会马上被 make 生
成，但中间文件只有当他的依赖条件真正有变化时才生成；第二，中间文件再使用后，会自动被 make
删除。你可以将一个文件名声明到 .INTERMEDIATE 这个特殊目标的前置条件中，将这个文件显式地
标记为中间文件。

但是，出现在目标或前置条件中的文件，不能标记成为中间文件。因此你可以将中间文件添加到某个
目标的前置条件中，以避免 make 自动删除这个中间文件。但是这会让 make 在搜索隐式规则时做
多余的工作。另外，可以将中间文件加到特殊目标 .NOTINTERMEDIATE 的前置条件中达到相同的目
的。相同的，如果将模板目标添加到 .NOTINTERMEDIATE，这个模板规则生成的目标文件都不会被当
成是中间文件。你还可以完全取消所有的中间文件，只要将目标 .NOTINTERMEDIATE 的前置条件设
置为空。

如果你不想让 make 仅仅因为文件不存在而创建这个文件，并且不自动将文件删除，你可以将这个
文件声明为辅助文件，只要将文件名添加到特殊目标 .SECONDARY 的前置条件中。将文件声明成辅
助文件，也同时将这个文件设置成了中间文件。

隐式规则链可以包含更多的规则，比如 foo.o 可以依次调用 RCS、Yacc、cc 由 RCS/foo.y,v 构
建，其中产生了 foo.y 和 foo.c 连个中间文件。但是，一个隐式规则不能在一条规则链中出现多
次，这样避免了 make 在搜寻隐式规则是产生无限循环。

一些特殊的隐式规则可以优化规则链的使用。例如，从 foo.c 生成可执行文件，可以通过编译和链
接这一规则链构建。但实际上，有一条特殊的规则可以用一条 cc 命令来完成整个工作。这条优化的
规则会被优先使用，因为它定义在对应规则的之前。

另外，出于性能考虑，make 在构建隐式规则的前置条件时，不会使用非最终的匹配任何文件的规则，
匹配任何文件的规则时形如 %: 的规则。

定义模板规则
-------------

你可以通过模板规则定义隐式规则，模板规则跟一般的规则很像，除了它的目标名称包含了一个 %
字符。这种目标是一个模板，其中的%字符可以匹配任何非空的字符串，而前置条件中使用 % 字符表
明了它的名字与目标名称的关系。例如模板规则 %.o: %.c 说明了怎样从任何以 .c 后缀名的文件
stem.c 构建 stem.o。实际匹配 % 的字符串称为词干（stem）。注意，模板规则中 % 的替换，发
生在所有变量引用和函数调用的展开之后。

模板规则能够执行，其目标模板必须匹配相应的目标文件名，并且它所有的前置条件必须都是存在的
或者可以被构建的文件。一个目标文件可能与多个模板规则匹配，使用那个模板规则根据 make 认为
哪个规则最匹配来决定。

一个模板规则可以有多个目标模板，不同于普通规则，这些模板目标总是被当成是组目标
（grouped targets），不管它是通过 : 或者 &: 定义的，也即整个规则是用来生成所有这些目标
文件的。在普通规则下，组目标的任何目标不存在，或者有任何前置条件有更新，即比任何一个目标
更新，都会执行规则生成所有这些目标。但在模板规则中，如果一个目标失效或者不存在，但是当前
并不需要构建这个文件，这种情况不会让组目标中的其他目标失效，从而可能触发模板规则的执行。
但这个例外可能在未来的 GNU make 版本中移除，因此不要依赖这个行为。如果 make 检测到了这
种情况的发生，会生成一个 pattern recipe did not update peer target 的警告。但是 make
并不能检测到所有的情况，因此需要你自己确保模板规则中的操作命令在运行时会更新它所有的目标。

模板规则的一些例子： ::

    %.o: %.c
        $(CC) -c $(CFLAGS) $(CPPFLAGS) $< -o $@

    % :: RCS/%,v # 双冒号表示这个规则是最终规则，也即它的前置条件不能是中间文件
        $(CO) $(COFLAGS) $<

    %.tab.c %.tab.h: %.y
        bison -d $<

这个规则告诉 make，bison -d x.y 可以生成 x.tab.c 和 x.tab.h 两个文件。如果 foo 依赖
于 parse.tab.o 和 scan.o，并且 scan.o 依赖于 parse.tab.h，当 parse.y 被修改后，这个
规则只需要执行一次，parse.tab.o 和 scan.o 的依赖条件都会被满足。

**自动变量**

非常重要的是，自动变量可以使用的作用域，自动变量的值只有在操作命令中才会被设置。如果用在
目标或依赖条件中，这些变量的值为空。但是使用二次展开这个特性，自动变量可以使用在依赖条件
中。二次展开发生在 make 真正构建目标时。

下面是 make 预定义的自动变量列表：

$@
    当前规则的目标名称，或触发当前规则执行的目标名称，如果目标归档成员文件 $@ 表示的是
    归档文件名
$%
    归档文件的成员名，如果不是归档文件则为空，例如目标是 foo.a(bar.o)，那么 $@ 是 foo.a，
    $% 是 bar.o
$<
    第一个前置条件名
$?
    所有比目标更新的前置条件，如果目标不存在则为所有的前置条件
$^
    所有的前置条件，会去掉前置条件中重复的文件名，并且 $^ 中不会包含 order-only 的前置
    条件
$+
    与 $^ 相同，但是不会去除重复文件名，这在链接命令中有用因为特定顺序出现的重复库文件
    是有意义的
$|
    所有的 order-only 的前置条件
$*
    隐式规则匹配的词干，比如 dir/a.foo.b 匹配 a.%.b，那么词干是 dir/foo；在静态模板规
    则中，词干是 % 匹配的字符串；在显式规则中是没有词干的，但是 GNU make 为了兼容其他
    make 实现，会把去掉后缀名的目标名称当作 $*，比如目标名称是 foo.c 那么 $* 是foo，但
    是一般情况，只应在隐式规则和静态模板规则中使用 $*；并且，这个后缀必须是可以识别的后
    缀，而且如果目标名称不带后缀，$* 为空

另外还有这些变量的变种，用来获取对应的目录名和文件名，这些变种实在原来变量的基础上添加 D
或 F 字符。函数 dir 和 notdir 有类似的效果，但不同的是 D 变种不会在目录名后添加一个斜
杠字符，而 dir 函数则总是带一个斜杠字符。

$(@D)
    如果 $@ 是 dir/foo.o，那么 $(@D) 是 dir，如果 $@ 不包含目录，那么 $(@D) 表示当前
    目录即一个点（.）字符
$(@F)
    如果 $@ 是 dir/foo.o，那么 $(@F) 是 foo.o，等价于 $(notdir $@)
$(%D) $(%F)
    归档成员名称的目录和文件名，仅当是归档文件 archive(member) 并且归档文件成员 member
    包含目录时才有用
$(<D) $(<F)
    第一个前置条件的目录和文件名
$(?D) $(?F)
    所有比目标更新的前置条件名称，这些名称的目录列表和文件名列表
$(^D) $(^F)
    去除重复之后的所有前置条件名称，这些名称的目录列表和文件名列表
$(+D) $(+F)
    所有前置条件名称，这些名称的目录列表和文件名列表
$(\*D) $(\*F)
    词干中的目录和文件名

**模板匹配**

当目标模板中不包含目录时，对文件进行匹配，首先会将文件的目录名移除，再与目标模板的前缀和
后缀进行比较，匹配成功之后，目录会加回来作为词干。例如，文件 src/eat 匹配 e%t，对应的词
干是 src/a，如果依赖的前置条件是 c%r，以同样的目录处理方式将词干匹配进去，依赖条件被扩
展成 src/car。

显式规则总是优先于隐式规则，并且对于当前的目标不需要另外隐式规则协助的规则总是优先于其他
需要依赖隐式规则的规则。一个目标可能同时匹配多个模板规则，这种情况下，make 会优先选择词
干最短的哪个（也即匹配最精确的），如果词干长度一样，会选择最先找到的那个模板规则。例如： ::

    %.o: %.c
        $(CC) -c $(CFLAGS) $(CPPFLAGS) $< -o $@
    %.o: %.f
        $(COMPILE.F) $(OUTPUT_OPTION) $<
    lib/%.o: lib/%.c
        $(CC) -fPIC -c $(CFLAGS) $(CPPFLAGS) $< -o $@

当编译 bar.o 时，如果 bar.c 和 bar.f 都存在，make 会优先执行第一条规则编译 C 文件；当
编译 lib/bar.o 时，会选择第三个规则，因为第一个规则匹配的词干是 lib/bar，而第三个规则
匹配的词干是 bar，第三个规则匹配的词干更短。

匹配一切的模板规则
------------------

如果模板规则的目标只包含一个 % 字符时，它可以匹配任何文件名。这种规则非常有用，但是非常
耗时，因为 make 必须对所有出现的目标名称和前置条件，都尝试这种规则。

为了执行效率，在定义一个匹配一切的模板规则时，你必须使用下面两种方式之一来尽量减少 make
的耗时。第一种方式是将匹配一切的模板规则，定义成最终规则，即使用双冒号进行定义，最终规则
的前置条件不能是中间文件，也即这些前置条件的生成不能依赖于其他隐式规则。

如果没被定义成最终规则，那么这条匹配一切的规则就不能应用于隐式规则的前置条件，而且也不能
应用于已知类型的文件。已知类型的文件，是指可以与其他隐式规则（不是匹配一切的规则）的目标
匹配的文件。

因此第二种方式是将所有已知类型的文件，显式的声明一个哑模板规则，避免这些已知的文件总是去
与匹配一切的模板规则进行匹配。哑模板规则是没有前置条件也没有操作命令的模板规则，而且它们
只用在这一种情况下，其他情况 make 会忽略这些哑模板规则。实际上，make 以及预定义了那些常
用后缀名对应的哑模板规则。例如： ::

    %.p :

声明了 .p 是已知类型的文件，当构建 foo.p 时，不会去与匹配一切的模板规则进行匹配，从而避
免去尝试构建形于 foo.p.o 和 foo.p.c 等中间文件的时间浪费。

最终默认规则
------------

你可以通过写一个没有前置条件的最终的匹配一切的规则，来定义一个最终默认规则。这个特殊的规
则，可以匹配任何目标，也就是说这个特殊规则的操作命令，会应用到那些没有自己的规则，也没有
匹配其他隐式规则的目标和前置条件上。比如下面的例子，它会对所有不存在也不能自动构建的文件，
都生成一个空文件： ::

    % ::
        touch $@

类似的，可以给 .DEFAULT 目标写一个规则，称为默认规则，这个规则的操作命令会应用到那些没
有规则的或者没有操作命令的目标上。如果定义一个空的 .DEFAULT:，前面储存在 .DEFAULT 中的
操作命令会被 make 清除，就像没有定义默认规则一样。

如果你不想一个目标运行任何的操作命令，也不想让它去执行匹配一切的模板规则中的命令，也不想
让它去执行默认规则中的命令，可以给这个目标顶一个空操作指令规则。

重定义隐式规则
--------------

你可以重定义一个新的模板规则覆盖原来的，这个重定义的模板规则有着相同的目标和前置条件，但
是操作命令不同。如果一个新的模板规则定义了，内置的隐式规则以及后面你自己定义的隐式规则将
被覆盖。

如果重定义的模板规则不带操作命令，那么这个模板规则相当于被显式的取消了。例如下面的模板规
则将运行汇编器的模板规则取消掉了： ::

    %.o: %.s

旧的后缀规则
------------

后缀规则时旧的定义隐式规则的方法，它有双后缀和单后缀两种形式。双后缀例如 .c.o: 等价于模
板规则 %.o: %.c，单后缀例如 .c: 等价于模板规则 %: %.c。

后缀规则不能有任何前置条件，否则被当成时普通的规则，例如 .c.o: foo.h 被当成时一个普通规
则，它的目标名称时 .c.o。另外没有操作命令的后缀规则也是一个普通规则，不会像模板规则那样
具有取消规则的作用。

默认的已知后缀定义在 .SUFFIXES 的前置条件列表中，你可以添加新的后缀到这个列表中，例如： ::

    .SUFFIXES: .hack .win

这里添加了两个像的后缀。如果你不想用默认定义的后置，可以先清掉所有后缀再添加你自己的： ::

    .SUFFFIXES:
    .SUFFFIXES: .c .o .h

另外，添加 -r 或者 --no-builtin-rules 会清掉所有内置的隐式规则，也会清除掉后缀的默认列
表。变量 SUFFIXES 保存了 make 在读取任何 makefile 之前定义的默认后缀列表，你可以通过
.SUFFIXES 修改后缀列表，但这不会影响 SUFFIXES 变量的值。

搜索隐式规则
------------

下面是 make 为目标 t 搜索隐式规则的步骤，以下情况都会执行这个流程：

1. 没有操作命令的双冒号规则
2. 每个没有操作命令的普通规则的目标
3. 任何不是规则目标的前置条件

对于归档成员目标 archive(member)，下面的流程会执行两次，第一次是用完整的目标名称 t，第
二次是如果第一次没找到执行的规则，用 (member) 作为目标名称 t。

1. 将目标名称分成目录部分 d，和余下部分 n，例如 src/foo.o 被分成 src/ 和 foo.o

2. 将所有匹配 t 或 n 的模板规则放到一个列表中，如果目标目标包含目标使用 t 进行匹配，否
   则使用 n 进行匹配

3. 如果列表中有任意一个不是匹配一切的规则，或者 t 是一个隐式规则的前置条件，则将所有的
   非最终的匹配一切的模板规则移除

4. 移除所有没有操作命令的规则

5. 对于每个在列表中的模板规则

   a. 找到匹配词干 s，它是 t 或 n 的子字符串
   b. 通过用 s 替换前置条件中的 %，计算出前置条件名称，如果目标模板没有包含目录，将 d
      添加到计算出的前置条件名称之前
   c. 检测是否所有的前置条件都存在，或者应该存在，如果一个文件作为目标出现或是目标 t 的
      显式前置条件，则称这个文件应该存在
   d. 如都存在或者应该存在，或者这条规则没有前置条件，则应用这个规则

6. 如果当还没有找到可应用的模板规则，那么重新对列表中的每个模板规则：

   a. 如果是最终规则，忽略掉继续下一个；
   b. 如前一样计算前置条件的名称；
   c. 对每个不存在的前置条件，递归地执行这整个算法流程，看是否可以通过某个隐式条件生成
   d. 如果所有的前置条件存在，或者应该存在，或者能被隐式规则创建，则应用这个规则

7. 修改应该存在的定义，将作为目标出现或是任何目标的显式前置条件都当作存在，继续执行步骤
   5 和 6，这一步骤仅仅是 GNU make 为了兼容老版本执行的，不推荐依赖这个行为

8. 如果没有应用的隐式规则，如果定义了 .DEFAULT 默认规则，则默认规则的操作命令当作目标
   t 的操作命令，否则目标t没有定义的操作命令可执行

当一个可应用的模板规则被找到后，对应的操作命令会被执行，然后这个模板规则的所有目标名称会
被保存到 make 的数据库中，并且被标记为已更新的状态。
