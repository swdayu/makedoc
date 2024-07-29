编写规则
=========

规则的出现顺序不太重要，除了第一个非点.开始的规则的目标是默认目标。但是以点开始的规则，
如果包含一个或多个斜杠/字符可以是默认目标。另外，定义了模板规则的目标也没有默认目标的
效果。一般的，我们通常编写第一个规则用来编译 makefile 描述的整个程序或所有的程序，通
常使用 all 作为规则的目标名称。

规则语法
---------

规则的语言如下： ::

    targets: perequisites
        recipe
        ...
    targets: perequisites ; recipe
        recipe
        ...

目标和前置条件是以空格分隔的文件名，可以包含通配符，或者是 a(m) 归档文件中的第 m 个文
件。通常一个规则只有一个目标名称，但是有些情况可以有多个目标。目标是不是需要更新，看这
个目标文件是不是存在，或者目标文件比任何一个前置条件中的文件旧。

操作命令的第一行可以与前置条件同行，用分号;相隔。连续两个操作命令之间可以是空行或者行
注释。但是以 Tab 开头的空行不是空白行，而是一个空命令（empty recipe）。一个命令行中
出现的注释不是 make 注释，它会被原样传递到 shell 中处理。规则的上下文从一个规则的定义
开始，到下一个规则定义或变量的定义结束。make 文件拥有两种语法，第一种是 make 语法，第
二种是操作命令的 shell 语法。make 程序不需要懂 shell 语法，它只对操作命令作简单处理，
然后直接把内容完全交由 shell 去执行。

因为 $ 符号在 make 中用于对变量的引用，如果想把这个符号用在目标或者前置条件中，必须使
用 $$ 代替，但是如果还开启了二次扩展，在前置条件中必须使用 $$$$。

两种前置条件
------------

有两种前置条件： ::

    targets: normal-perequisites | order-only-perequisites

order-only-perequisites 用字符 | 分隔位于正常前置条件之后，这种前置条件只为了保证它
们必须在 targets 之前存在，而不参与判断是否更新 targets。如果一个文件名称在两种前置条
件中都出现了，它会被当作正常的前置条件处理。在判断一个目标是否要更新时，任何的 order
only 前置条件对应的文件都不会被检查，也即任何的 order only 前置条件都不会引起目标更新。

使用通配符
----------

使用通配符一个文件名可以匹配多个文件，make 中可以使用的通配符时*,?,和[...]。如果一个
文件名匹配了多个文件，新版本的 gnu make 会对这些文件进行排序。另一个特殊字符是 ~，当
它是文件名的第一个字符时，~/ 会被替换成 /home/usr/，~name 会被替换成 /home/name。在
不支持 home 目录的系统中，可以设置 HOME 环境变量达到相同的目的。

make 会自动处理目标和前置条件中的通配符，而操作命令中的通配符则由 shell 负责处理。在
其他情况下，必须使用 wildcard 函数来显式地请求使用通配符功能。比如变量定义里 ``objects = *.o``，
如果你在目标和前置条件中引用这个变量，这个确实会进行匹配操作，但如果用在其他地方，它就
是一个简单的字符串 ``*.o``，这时如果要进行通配符处理比如用在操作命令中，需要使用 wildcard
函数，比如： ``objects := $(wildcard *.o)``。在 makefile 任何地方使用 ``$(wildcard pattern...)``
这个操作，都会被替换成用空格分隔的排序的匹配文件名的一个列表。如果模式没有匹配任何文件，
在规则里会原样使用模式字符串，而在其他地方模式会被忽略。

如果文件名确实包含一个通配符里的字符，需要转义处理，比如 ``foo\*bar`` 表示这个文件名
确实包含一个星号字符。 ::

    objects = *.o
    foo: $(objects)
        cc -o foo $(CFLAGS) $(objects)

    如果没有.o文件，会报*.o目标无法生成，因为foo规则会扩展成：
    foo: *.o
        cc -o foo $(CFLAGS) *.o

    objects := $(patsubst %.c,%.o,$(wildcard *.c))
    foo: $(objects)
        cc -o foo $(objects)

这里.o文件会从.c文件匹配而来，而且 objects 的值是被立即展开的，由于不是在规则里面，如
果没有.c文件最后的值是空。

路径搜索
--------

变量 VPATH 定义了一个 make 应该查找的目录列表，如果目标和前置条件中的文件在当前文件夹中
找不到，会使用 VPATH 来查找这些文件。VPATH 中的目录用空白字符或者冒号:分隔，例如： ::

    VPATH = src:../headers

另外是用 vpath 指示语句为特定的模式定义一个目录搜索列表，它有三种形式： ::

    vpath pattern directories # 为pattern定义一个目录搜索列表
    vpath pattern # 清除pattern的搜索列表
    vpath # 清楚vpath定义的所有的搜索列表

vpat h的模式字符串是包含一个 % 字符的字串，% 可以匹配零个或多个字符。如果要查找的文件名
匹配多个 vpath 定义的模式，则会一个一个按顺序查找。例如： ::

    vpath %.h ../headers
    vpath %.c src:bar
    vpath % blish

    foo.o: foo.c

    如果当前文件夹找不到foo.c，而在src文件夹找到了，相当于被解析成:
    foo.o: src/foo.c

当目标文件名是通过路径搜索找到的，如果 make 不需要更新会使用这个路径，但是如果 make 确实
要更新这个目标，make 不会使用这个路径而是选择本地生成。如果你想在搜索到的目录更新目标，需
要设置 GPATH 变量，定义在哪些目录目标的更新不本地更新而是在对应目录下更新。也就是说，如果
一个目标是通过目录搜索找到的，并且这个目录出现在 GPATH 中，那么这个目标的更新将在这个展开
的目录下更新。 ::

    VPATH = src:../headers
    foo.o: foo.c defs.h hack.h
        cc -c $(CFLAGS) $< -o $@

如果前置条件中的文件是通过目录搜索找到的，那么必须谨慎的些规则的操作命令，因为你需要保证在
操作命令中操作的文件是对应目录的文件。这时 ``$<`` 以及 ``$^`` 等自动变量就派上了用场，这
个变量会被解析成真正搜索到的对应目录的文件。

库文件的目录搜索，如果前置条件中用 -lname 的形式使用了一个 name 的库文件名，make 会一次查
找库文件 libname.so 和 libname.a，如果当前文件搜不到，会进行目录搜索。变量 .LIBPATTERNS
定了搜索库文件的模式，它的默认值是 lib%.so lib%.a。如果将这个值设为空，会关掉这个库文件目
录搜索的功能。

库文件的目录搜索的顺序是：vpath 目录，VPATH 目录，然后是 /lib，/usr/lib，以及 prefix/lib
（通常为 /usr/local/lib，在 MS 上 prefix 是 DJGPP 的安装根目录）。例如： ::

    foo: foo.c -lcurses
        cc $^ -o $@

如果没有对应的 so 文件，而存在 /usr/lib/libcurses.a，那么造作命令会被解析成： ::

    cc foo.c /usr/lib/libcurses.a -o foo。


伪目标
-------

伪目标（Phony Targets）对应的目标名称不是一个真正的文件，它的目的仅仅是执行一段操作命令，
并不生成目标文件。 ::

    .PHONY: clean
    clean:
        rm *.o temp

clean 生命成伪目标的作用是，即时存在一个真实的 clean 文件，clean 目标还是会被更新，触发
执行对应的操作命令。使用伪目标，可以避免目标名称与真实存在的文件名冲突，另外定义的伪目标，
make 不会继续查找尝试生成这个目标的隐式规则，可以提高 make 的执行效率。

.PHONY 的前置条件会被原样解析，因此不能使用模板字符串。为了声明一个总被执行的模板目标，可
以使用 force target。

伪目标还用来沟通递归调用中的 make。在这种情况下，makefile 文件通常包含一个变量，列出需要
构建的子目录列表。一种简单的做法是，定义一个规则，对子目录一个个遍历： ::

    SUBDIRS = foo bar baz
    subdirs:
        for dir in $(SUBDIRS); do \
        $(MAKE) -C $$dir; \
        done

但这个方法有几个问题，第一个是，子级 make 检测到的错误被忽略了导致出错了还会继续执行后续的
目录。第二个问题是，不能利用 make 并行构建的能力，每个子目录得顺序执行。 ::

    SUBDIRS = foo bar baz
    .PHONY: subdirs $(SUBDIRS)
    subdirs: $(SUBDIRS)
    $(SUBDIRS):
        $(MAKE) -C $@
    foo: baz

可以将子目录定义成伪目标，因为子目录是一直存在的，必须定义成伪目标才能让子目录的规则被执行。
另外，foo:baz 定义了一个子目录的依赖顺序，必须先构建 baz，才能构建 foo。这样的子目录依赖
定义，在并行构建中是非常重要的。

伪目标不应该作为真实目标文件的前置条件，否则 make 更新这个目标文件时，伪目标的操作命令每次
都会执行。相当于使用伪目标或 FORCE 目标作为前置条件的目标，每次都会被更新，除非将这些前置
条件声明成 order only 类型的前置条件。另外，如果 makefile 文件名被声明成伪目标，make 不
会尝试去检查这个 makefile 文件是否有更新的版本。伪目标可以作为伪目标的前置条件，这个伪目标
相当于是一个子程序： ::

    .PHONY: cleanall cleanobj cleandiff
    cleanall: cleanobj cleandiff
        rm program
    cleanobj:
        rm *.o
    cleandiff:
        rm *.diff

伪目标可以有前置条件，例如如下的一个 Makefile 描述了多个程序。Phony 的特性不会被继承，伪
目标的前置条件不是伪目标，除非显式地将它们声明为伪目标。 ::

    all: prog1 prog2 prog3
    .PHONY: all
    prog1: prog1.o utils.o
        cc -o prog1 prog1.o utils.o
    prog2: prog2.o
        cc -o prog2 prog2.o
    prog3: prog3.o sort.o utils.o
        cc -o prog3 prog3.o sort.o utils.o


没有前置条件或操作命令的规则
----------------------------

当一个规则没有前置条件或者没有操作命令，并且目标文件是一个不存在的文件时，make 当执行这个
规则时，每次都会更新这个目标。这意味着所有依赖于这个目标的目标，都会每次更新。 ::

    clean: FORCE
        rm $(objects)
    FORCE:

可以看到，使用 FORCE 更使用 .PHONY 的效果差不多，但是有些版本的 make 可能不支持 .PHONY，
只能 FORCE 这种形式。

空命令规则
----------

空命令（Empty Recipes）规则是只包含空白操作命令的规则： ::

    target: ;

    或者
    target:
        # tab

定义空命令规则，可以避免 make 去查找隐式规则来尝试更新目标。另外空规则可以保证一些目标可以
正常执行，如果一个目标不存在，空规则可以告诉 make 这个目标不需要构建不要报无法构建目标的错
误，make 就不会去构建这个目标但是认为这个目标是需要更新的（out of date）。

空目标文件记录事件
------------------

空目标文件是一个真实存在的文件，但它的内容一般为空或者说它有什么内容不重要。空目标文件的目
的，是用来记录它的操作命令最近一次被执行的时间（用touch命令更新这个目标文件）。这个空目标
文件一般都会有前置条件，这样相当于只要任何一个前置条件中的文件发生变化时，都会记录一次。比
如： ::

    print: foo.c bar.c
        lpr -p $? # 自动变量$?解析成发生了变化的文件
        touch print

多个目标的规则
--------------

当一个显式规则拥有多个目标时，如果是用:分隔符定义的规则，相当于这些目标拥有相同的前置目标和
相同的操作命令。通常在操作命令中用 ``$@`` 代表当前执行的是哪个目标。但是，你不能根据当前是
哪个目标来修改使用的前置条件，这个功能需要使用静态模板规则才能达到。

上面的多个目标是独立的，但如果使用&:分隔符来定义规则，那么这些目标变成成组的目标（grouped
targets），它们不再独立，相当于这个规则是用来生成所有这些目标的，也即这条规则会生成多个目标
文件。也就是说，只要这些目标中的任何一个目标不存在，或者任何一个目标需要更新，都会执行规则的
操作命令更新所有这些目标文件。 ::

    foo bar biz &: baz boz
        echo $^ > foo
        echo $^ > bar
        echo $^ > biz

在组目标中，变量$@会被设成触发规则执行的那个特定的目标。另外，组目标规则必须要有操作命令。组
目标中的目标也可以出现在没有操作命令的多个独立目标的目标列表中。

一个目标应该只有一个与之关联的操作命令，如果一个组目标中的目标还出现在一个有操作命令的单独目
标规则的目标中或另一个组目标的目标中，会产生一个警告，并且后面的操作命令会把前面的替换掉，而
且会将这个目标从前面的规则中移除。

如果想要一个目标在多个组目标列表中存在，可以使用 &:: 分隔的规则，这样每个 &:: 都是独立的，
这个目标被更新的次数，处决于每个 &:: 是否被更新。

一个目标拥有多个规则
--------------------

一个目标可以拥有多个规则，这样多个规则中的前置条件会拼接成一个完整的前置条件列表。但是一个目
标一般是有一个操作命令与之对应，如果多个规则定义了多个操作命令，make 只会只用最后一个操作命
令，并且打印错误信息作为提示，但是对于用点号.开头的目标不会报错。

少数情况下，如果想让一个目标执行不同部分定义的多个规则中的操作命令，你可以使用双冒号 :: 来定
义这些规则。 ::

    objects = foo.o bar.o
    foo.o: defs.h
    bar.o: defs.h test.h
    $(objects): config.h

如果一个目标存在对应的真实文件，但是 make 找不到更新这个文件的操作命令，make 会尝试从隐式规
则中去查找。

静态模板规则
-------------

静态模板规则可以根据目标名构造不同的前置条件名。其中 target-pattern 和 prereq-patterns
定义了怎么构造每个目标的前置条件。目标名称匹配 tareget-pattern 的那一部分成为目标名称的主干
名称（stem），这个主干名称会替换到 prereq-pattern 中形成不同的前置条件。 ::

    targets ...: target-pattern: prereq-patterns ...
        recipe
        ...

    objects = foo.o bar.o
    all: $(objects)
    $(objects): %.o : %.c
        $(CC) -c $(CFLAGS) $< -o $@

每个目标都必须与 target-pattern 匹配，否则会报警告错误。如果有一个文件列表，只有一些满足匹配
要求，你可以使用 filter 函数： ::

    files = foo.elc bar.o lose.o
    $(filter %.o,$(files)): %.o: %.c
        $(CC) -c $(CFLAGS) $< -o $@
    $(filter %.elc,%(files)): %.elc: %.el
        emacs -f batch-byte-compile %<

    另一个例子，$*会替换成目标的主干名称：
    bigoutput littleoutput: %output: test.g
        generate test.g -$* > $@

双冒号规则
-----------

如果一个目标名称处在在多个规则中，这些规则必须是相同的类型，都是普通规则，或者都是双冒号规则。
如果是双冒号规则，相同目标名称的多个规则是相互独立的，每个规则会独立更新，只要目标不存在，或
者对应的前置条件更新。另外比较特别的是，如果双冒号规则如果没有前置条件，目标总是会更新，即使
这个目标文件已经存在。

双冒号规则必须有操作命令，如果没有命令，make 会查找隐式规则尝试生成目标文件。模板规则中的双
冒号规则意义不同，因为模板规则每条规则总是独立的，模板规则使用双冒号来表明这个规则是最终规则。

自动生成前置条件
----------------

当编译程序时，目标文件不仅依赖于源文件，还依赖于源文件包含的头文件。简单情况下，源文件包含的
头文件可以手动写到前置条件中，但是对于大型程序来说，这时不现实的。为了避免这个问题，现代 C 编
译器可以辅助生成对应的规则，例如： ::

    cc -M main.c 可以输出规则 main.o: main.c defs.h，假如main.c只包含了defs.h的话。

    %.d: %.c
        @set -e; rm -f $@; \
        $(CC) -M $(CPPFLAGS) $< > $@.$$$$; \
        sed 's,\($*\)\.o[ :]*,\1.o $@ : ,g' < $@.$$$$ > $@; \
        rm -f $@.$$$$

命令 ``set -e`` 设置任何命令执行出错，就会退出。使用 GNU C 编译器，你可能需要使用 -MM 选
项，这个选项会忽略掉系统头文件。然后 sed 命令的目的是将 main.o: main.c defs.h 的行转换成
main.o main.d: main.c defs.h。然后你只要将包含这些依赖关系的 makefile 包含就行，比如： ::

    souces = foo.c bar.c
    include $(souces:.c=.d) # 这里相当于 include foo.d bar.d

由于 .d 文件包含有规则定义，所以这里的 include 一般要放在 makefile 的第一个即默认目标之后。

更新归档文件
-------------

归档文件（archive file）是包含很多命名子文件或成员文件的文件，它使用 ar 程序进行维护，主要
的目的是用作链接库时的子程序。

归档文件中的成员可以作为规则的目标或前置条件使用，但不能用在规则的操作命令中。使用以下形式来
指定归档文件中的一个成员文件：archive(member)。

大多数程序并不支持对成员文件的操作，除了ar或其他特别设计的程序。因此为了更新成员文件，大概率
需要使用ar命令。例如： ::

    foolib(back.o): hack.o
        ar cr foolib hack.o

这里 ar 通过拷贝 hack.o，来创建一个成员文件 back.o 到归档文件 foolib 中。实际上，几乎所有
的归档成员文件都是这样更新的，都有一个隐式规则来做这件事。可以指定多个成员文件，比如： ::

    foolib(hack.o kludge.o)
    等价于
    foolib(hack.o) foolib(kludge.o)

还可以使用通配符，比如 ``foolib(*.o)``，它会展开成匹配到的所有成员文件，像： ::

    foolib(hack.o) foolib(kludge.o)

make 会自动寻找隐式规则来更新成员文件，例如 make 'foo.a(bar.o)' 时，这里只需要有一个 bar.c
存在，即使没有指定 makefile 文件，make 也会自动使用隐式规则创建这个成员文件。它执行的所有操
作等价于： ::

    cc -c bar.c -o bar.o
    ar r foo.a bar.o
    rm -f bar.o

归档文件中的成员名称不能包含目录名称，但是你可以这样写以帮助 make 将对应目录下的文件拷贝到归
档文件中，比如下面的例子 make 将 dir/file.o 拷贝到 foo.a 文件中的 file.o 成员中。
make 'foo.a(dir/file.o)' 会让 make 自动执行 ar r foo.a dir/file.o。对于这种用法，自动变
量 %D 和 %F 有它的用处。

作为程序库使用的归档文件，通常包含一个特殊的成员文件叫做 __.SYMDEF，它包含了所有其他成员文件
对应的外部符号名称目录。当更新一个成员文件时，也需要更新 __.SYMDEF 文件来汇总所有其他成员文
件属性。这个操作通过 ranlib 命令来完成： ::

    ranlib archivefile

正常你要将这个命令添加到更新归档文件的规则中，并将所有成员文件作为依赖条件，例如： ::

    libfoo.a: libfoo.a(x.o y.o ...)
        ranlib libfoo.a

这里你可以不提供更新成员文件如果更新的规则，因为 make 可以查找隐式规则来完成。特别的，对于
GNU ar 程序，不需要手动提供 ranlib 命令，因为 GNU ar 会自动更新 __.SYMDEF 成员。

**不兼容并行执行**

更新归档文件的隐式规则，不能与并行执行兼容。这些规则将每个目标文件添加到归档文件中，当并行执
行开启时，多个 ar 程序将同时更新同一个归档文件，这是不被允许的。如果你想要在并行执行时能够构
建归档文件，你可以提供你自己的版本以替换隐式规则，例如： ::

    (%): % ;
    %.a: ; $(AR) %(ARFLAGS) $@ $?

第一行改变隐式规则更新单独的目标文件到不做任何事情，第二行则是用一个命令在构建归档文件时去更
新所有的不是最新的成员文件（$?）。当然，这里你需要声明归档文件所依赖的所有成员文件： ::

    libfoo.a: libfoo.a(x.o y.o ...)

或者对特定的归档文件直接显式的添加一个规则： ::

    libfoo.a: libfoo.a(x.o y.o ...)
        $(AR) $(ARFLAGS) $@ $?

**归档文件的后缀规则**

你可以编写一种特定的后缀规则，来处理归档文件。后缀规则在 GNU make 中已经过时了，因为模板规则
是更通用的机制。但是后缀规则仍然保留以兼容其他 make。

编写一个归档文件的后缀规则，只要编写一个归档文件后缀名 .a 的后缀规则。例如从 C 源文件更新程序
库归档文件： ::

    .c.a:
        $(CC) $(CFLAGS) $(CPPFLAGS) -c %< -o $*.o
        $(AR) r $@ $*.o
        $(RM) $*.o

    等价于模板规则：
    (%.o): %.c
        $(CC) $(CFLAGS) $(CPPFLAGS) -c %< -o $*.o
        $(AR) r $@ $*.o
        $(RM) $*.o

实际上，这也是 make 做的，它会将对应的后缀规则转换成模板规则。而且，为了解决 .a 后缀还可能作
为其他文件类型的问题，make 会将 .x.a 形式的后缀规则转换成两个模板规则： ::

    (%.o): %.x 以及 %.a: %.x。

规则命令
---------

规则的命令包含一个或多个需要执行的命令行命令，这些命令行命令会一个一个按顺序交给命令行执行。用
户可以使用不同的命令行，除非特殊指定否则 make 默认使用 /bin/sh。

Make 文件有两套语法系统，一套是 make 语法，另一套是规则命令中使用的 shell 语法。make 程序不
需要懂得 shell 语法，它仅仅对规则命令做自己范围内的转换，然后直接把内容完全交由 shell 去执行。

命令的语法
----------

操作命令的每行必须以 Tab 键开始（或者 .RECIPEPREFIX 变量定义的字符），除了出现在前置条件同行
的第一个操作命令外。任何以Tab键开始的内容，并且处于规则上下文中，都被 make 解析成规则命令的一
部分。规则上下文，从规则定义开始，到另一个规则定义或变量定义结束。

在规则上下文中，可以出现不以 Tab 开头的空白行和注释行，这些内容会被忽略。而以 Tab 开头的任何内
容都会传递给 shell 去处理，比如以 Tab 开头的空行是一个空命令行（empty recipe）。另外，操作命
令的注释不是 make 的内容，这些注释与操作命令一起直接传递给 shell 处理。

**命令的分行**

操作命令也可以通过反斜杠和换行的组合进行续航（\newline)，这样一个逻辑行可以被分成多个物理行。但
是这个逻辑行传递给 shell 的时候，反斜杠和换行并不会被 make 移除掉，它们被原样保留交给 shell 处
理。另外，如果续航的第一个字符是 Tab 键的话，这个 Tab 键会被 make 移除，也只有这个字符会被移除，
其余字符都会被保留。例如： ::

    all:
        @echo no\
    space
        @echo no\
        space
        @echo one \
        space
        @echo one\
        space

**命令中使用变量**

规则命令中的展开是延时展开，实在第二阶段这个规则真正被执行的时候进行展开，也就是不需要更新的目标中
的命令是不会展开的。变量和函数的展开有相同的语法，也都使用 $ 字符作为引用符号。由于 make 和 shell
都使用 $ 作为引用符号，你需要清晰的分辨这个变量引用到底是 make 语法环境下的还是 shell 环境下的： ::

    LIST = one two three
    all:
        for i in $(LIST); do \
            echo $$i; \
        done

很明显，$(LIST) 是 make 语法下的，它被扩展成 one two three。而 ``$$i`` 是 shell 环境下，它首
先会被 make 转换成 ``$i`` 然后传给 shell 处理。

命令回显
---------

make 会在命令执行之前打印这条命令，如果在命令之前加一个 @ 字符，命令就不会被打印。另外，如果 make
设置了 -n 或者 --just-print 选项，所有的命令都只会打印，不会被执行，即使是使用 @ 开头的命令。

还有 -s 和 --silent 选项可以抑制所有的打印，相当于所有的命令都加上了 @ 一样。而且 make 还有一个
特殊的 .SILENT 目标没有前置条件的规则，提供相同的效果。

命令执行
--------

规则的每一行命令都在一个全新的 shell 中执行，除非这条规则使了.ONESHELL。这意味着 shell 变量的设
置，以及想 cd 这样设置 shell 本地进程上下文的 shell 命令，影响不到下一行命令的执行。如果想影响下
一行命令，要么将这些命令定义到一行，或者将规则定义成 .ONESHELL。 ::

    foo: bar/lose
        cd $(<D) && gobble $(<F) > ../$@

    .ONESHELL:
    foo: bar/lose
        cd $(<D)
        gobble $(<F) > ../$@

如果使用了 .ONESHELL，只有命令的第一行才会检查特殊的前缀字符（@，-，或者+），后续的特殊前缀字符都
会原样保留传递给 shell。如果你想让第一行的特殊前缀字符也保留，你可以加一行注释作为第一行或者变通处
理不让前缀字符出现在开始位置： ::

    .ONESHELL
    SHELL = /usr/bin/perl
    .SHELLFLAGS = -e
    show:
        # make sure "@" is not the first character one the first line
        @f = qw(a b c);
        print "@f\n";

    .ONESHELL
    SHELL = /usr/bin/perl
    .SHELLFLAGS = -e
    show:
        my @f = qw(a b c);
        print "@f\n";

但是对于 POXIS shell，每一行的前缀字符还是会被去除，因为 POSIX shell 这些特殊的前缀字符对于 POXIS
shell 是非法的。例如下面的命令还是会正常执行： ::

    .ONESHELL:
    foo: bar/lose
        @cd $(@D)
        @gobble $(@F) > ../$@

但是 .ONESHELL 执行的命令还是有一点区别在于，除最后一行命令，其他命令行的失败都不会被 make 检测到。
可以设置 .SHELLFLAGS 变量，加上 -e 选项，这样命令运行出现任何错误到会导致 shell 返回失败。

make 程序使用哪个 shell 由 SHELL 变量决定，如果这个变量没有在你的 makefile 文件中设置，make 默认
使用 /bin/sh。传递给 shell 的参数则由变量 .SHELLFLAGS 提供。.SHELLFLAGS 的默认值通常是 -c，或者
POSIX shell 中为 -ec。

不像其他变量，SHELL 变量不会拿环境变量中的值进行设置。另外 SHELL 的值不会被 export 到 make 调用的
命令行环境中，相反的命令行 shell 环境继承的当前用户的环境。如果你想覆盖 shell 环境中的值，可以手动
export SHELL 的值，强制把它传递到 shell 的命令行环境中。

然而在 MS 系统中，SHELL 环境变量中的值会被使用，因为 MS 系统多数用户不会设置这个变量值，如果设置大
概率是为了使用 make。在 MS-DOS 上，如果 SHELL 的值不适合 make，你可以设置 MAKESHELL 这个变量，如
果 MAKESHELL 设置了会优先使用。

在 MS 平台上选择 shell 是一个比其他系统复杂的事情。在 MS-DOS 上，如果 SHELL 没有设置，COMSPEC 的
值会被使用，这个变量总是会被设置，另外由于 MS 的原生 shell，command.com 有很多限制大多数 make 用
户都会安装一个自己喜欢的替换它。这样，make 会检查 SHELL 设置的到底是 Unix 风格的还是 DOS 风格的
shell，然后根据对应的风格做不同的 shell 处理。

而且，MS 系统上如果 shell 设置成了 Unix 风格的 shell，make 还会检查这个 shell 是否确实存在。make
会在以下地方寻找这个 shell：

1. SHELL 明确指定的地方，比如 SHELL=/bin/sh，make 会在当前驱动盘上找 /bin 目录；
2. 当前目录；
3. 按顺序一次查找 PATH 环境变量中的目录；

在查找文件的时候，make 会先查找指定的名字，比如上面的 sh，如果找不到会接着找对应后缀名的文件，比如
.exe，.com，.bat，.btm，.sh，等等。注意的是，这种 shell 额外的查找只适用于在 makefile 中设置的
shell，如果它在环境变量或者命令行中设置，必须确切的设置这个 shell 的全路径，跟 Unix 平台一样。

这样的效果是，如果你在MS上安装了 sh.exe，并且可以在 PATH 中可以找到，Unix 风格的 make 就可以无修
改的直接可以在 MS 平台上只执行。

并行执行
---------

正常情况下，make 会依次执行一个命令行命令，等待这个命令执行完，然后执行下一个。但是，如果在 make 命
令行加上 -j 或者 -jobs 选项，make 会同时执行多条命令。在 makefile 中，你可以选择某些或所有的目标，
不进行这种并行执行。注意的是，-j 选项在 MS-DOS 上没有效果，会被忽略。如果 -j 选项后面跟了一个整数，
这个值是同时执行的命令的条数。如果不是一个整数，那么同时执行的命令条数是没有限制的。需要注意，make
的递归调用对并行执行有些需要注意的问题。

如果一条指令失败，错误不会被忽略，构建同一个目标剩下的命令将不会被继续中。如果 make 没有设置 -k 或者
--keep-going 选项，make 会中断执行。如果一个 make 中断了执行，它会等所有的子进程都执行完毕，然后才
真正退出。

如果系统当前负载重，我们可能想让 make 执行的并行任务少一点。可以使用 -l 或者 --max-load 指定一个浮
点数，比如 -l 2.5。当系统平均负载高于 2.5 的时候，不允许 make 启动多于一个任务。如果 -l 不带参数，
相当于移除前面设置的负载的限制。更确切的，make 在启动新的任务之前，先查看是否有至少一个任务在执行，如
果是的话它检查系统当前平均负载，如果等于大于 -l 选项设定的值，make 会等负载下降到限制之下，或者所有
的任务都执行完。

**关闭并行执行**

如果 makefile 定义了所有目标之间的完整而精确的依赖关系，不管并行执行有没有打开，make 都会正确的构建
目标。但是，有些时候一些目标或全部的目标不能以并行方式执行，而且不能用可行的前置条件依赖方式正确描述。
这种情况下，有几种方式来关闭并行执行。

如果 .NOTPARALLEL 出现在了 makefile 中并且不带任何前置条件，那么当前的 make 实例会顺序执行，而不管
并行执行是否设置。如果设置了前置条件，那么前置条件列表中的目标会顺序执行，但是只有执行这些目标才会顺序
执行。.NOTPARALLEL 不应该包含操作命令。 ::

    all: one two three
    one two three: ; @sleep 1; echo $@
    .NOTPARALLEL:

    all: base notparallel
    base: one two three
    notparallel: one two three
    one two three: ; @sleep 1; echo $@
    .NOTPARALLEL: notparallel

这里如果执行 make -j base，目标 one two three 会并行执行，但是 make -j notparallel 会顺序执行，而
make -j all 会并行执行，因为 base 可以并行执行。

还可以 .WAIT 更精细地控制目标的并行执行，.WAIT 后面的目标必须等前面的目标都完成后才会执行。下面例子，
如果打开了并行执行，one 和 two 会并行执行，然后等这两个目标都执行完之后才会执行 three。另外，.WAIT 前
置条件不会出现在任何自动变量中。 ::

    all: one two .WAIT three
    one two three: ; @sleep 1; echo $@

**并行执行的输出**

为了并行执行的打印输出混乱，可以使用 -O 或者 --output-sync 选项，该选项让 make 保存正在执行的所有命令
的输出，只有当命令执行完之后才一次性输出。如果还有多个递归的 make 在执行，它们会沟通一次只让一个 make
进行输出打印。同步打印输出有多个颗粒度可以选择，比如 -Oline 或者 --output-sync=recurse。不管选择哪种
模式，总构建时间是相同的，不同的指示输出的方式。

* none - 不同步输出，有输出就立即输出
* line - 按行独立输出
* target - 按目标独立输出，这时 -O 不带参数的默认值
* recurse - 整个递归调用的 make 独立输出

如果工作目录的打印打开了（--print-directory），进入和离开目标的消息对会被打印。如果不需要这种打印，可以
添加 --no-print-directory 选项到 MAKEFLAGS 中。

**并行执行的输入**

两个进程不能同时对同一个设备进行输入。为了确保一次只有一个命令进程从标准输入进行读取，处理当前运行的进程
其他进行的读取都会报 Broken pipe 异常。因为如果使用并行执行，你不能够依赖任何从标准输入进行读取的命令。

错误处理
---------

每个 shell 调用返回时，make 都会检查它的退出状态。如果返回成功（返回0），make 继续再新的 shell 中执行
下一行命令，知道规则的最后一行命令。如果返回失败，make 会中断当前规则的执行。如果一行命令执行失败时无所
谓的，可以在命令之前加一个 - 字符，可以让 make 忽略这一行命令执行的任何错误。如果使用 -i 或者 --ignore-errors，
make 会忽略所有命令的错误。另外一个使用了 .IGNORE 的规则也有相同的效果。如果一个错误由于 - 或 -i 被忽
略掉，make 将这些错误像返回成功一样处理，但是错误消息还是会打印，并且说明这个错误被忽略掉了。

当错误发生并且不能忽略时，表示当前目标不能被正确创建，make 不会继续执行这个规则后续命令。正常情况下，make
就立即中断操作。但是如果添加了 -k 或者 --keep-going 选项，make 会继续执行目标未执行完的操作以及依赖的
操作，直到最后 make 才错误返回。正常的操作时，make 一旦遇到错误就会立即返回。而使用 -k 的目的是，让 make
尽可能多的去更新目标，以便一次尽可能多的发现问题，这样你可能在下次执行之前把它们全部修复。这也是为什么
Emacs 的 compile 命令默认加上了 -k 选项的原因。

通常情况下，如果操作命令失败了而且他操作了目标文件，一般这个目标文件就被破坏了不能使用，或者至少这个目标
文件没有被完整的更新。但是目标文件的时间戳有表明它是最新的，因此 make 下一次不会更新它。为了避免这个问题，
当命令返回错误时也应当删除目标文件，make 会做这个工作，如果我们给目标加上 .DELETE_ON_ERROR。另外 make
在收到 fatal signal 终止执行时（比如CTRL-C），make 根据目标文件上一次的时间戳，可能删掉可能被不完整更
新的文件。

但是有些时候你不想删除文件，比如文件更新时以一种原子的方式执行的，或者这个文件仅仅用来记录时间戳它的内容
不重要，或者这个文件必须总是存在来避免一些问题。这时候，你需要使用 .PRECIOUS 作为目标的前置条件来避免
make 删除文件。

虽然 make 做了最大的努力来做错误的清理，但是保不住程序的 bug 和不预期异常，导致生成的破坏的目标文件没有
被清除掉，导致各种不预期的构建问题。你必须有自己的防御机制，比如只在临时文件夹中更新目标，直到最后才把临
时文件夹更名为目标文件夹。

Make 的递归调用
----------------

Make 的递归调用，是在 makefile 文件中将 make 作为命令进行调用。比如大型系统，包含很多子目录，每个子目录
都包含自己的 makefile 文件。这样你需要调用 make 去构建子目录： ::

    subsystem:
        cd subdir && $(MAKE)
    或者：
    subsystem:
        $(MAKE) -C subdir

当 make 启动时（在处理完 -C 之后）会设置好 CURDIR 变量，这个值当 make 或子 make 启动后就不会被 make
更新了，即使 makefile 包含了其他目录文件。

递归调用 make 时，需要使用 MAKE 变量。如果在顶层 make 使用了一个特殊的 make 程序，当递归调用时也会使用
这个特殊的 make。

通过 MAKEFLAGS，make 的选项会被传递给子 make。但是，对于 -t（--touch），-n（--just-print），-q（--question）
这些不真正执行操作命令的选项，会被以+字符开头的命令行和使用了 MAKE 变量的命令行忽略。因为使用MAKE变量，
是要真正执行这条命令去进行递归构建。

如果使用了多级递归调用，可以使用 -w 或者 --print-directory 更好的显示出 make 的目录结构。这样，当 make
进入和离开一个目录都会打印出目录信息。如果你使用了 -C 选项，-w 选项会自动开启。但是，如果使用了 -s 或者
--no-print-directory，-w 选项会被关闭。

**传递变量到子 make**

顶层 make 中的变量可以通过环境变量传递给子 make，这些变量会被定义在子 make 中，但是它们不会覆盖子 make
中的变量，除非使用了 -e 选项。

传递一个变量（或者export），make 就会将这个变量和它的值添加到规则命令运行时的每一行命令的环境中。子 make
也就使用这个环境来初始化它的变量列表。

除了显式的传递一个变量，make 还会自动传递那些原本就定义的环境变量，和定义在命令行中的只包含字母、数字、
和下划线的变量。特殊的，环境变量 SHELL 不会被传递，而是从调用 make 的值传递到子 make，你可以使用 export
明确 make 做到这一点。变量 MAKEFLAGS 总会被传递，只要你设置了 MAKEFILES 的值，这个值就会被传递，除非被
声明成了 unexport。make 会自动传递命令行定义的变量，方式是将它们添加到 MAKEFLAGS 变量中。一些 make 默
认定义的值，不会正常传递，因为子 make 也默认定义它们自己的值。

通过 export 和 unexport 可以显式的将变量传递给子 make，或防止一个变量的值传递给子 make: ::

    export variable ...
    unexport variable ...

这些名称可以被扩展，因此可以是变量引用和函数调用。为了方便，还可以和变量操作结合。你可以看到 make 中 export
的使用和 shell 中使用方式一样。 ::

    export variable = value
    variable = value
    export variable
    export variable := value
    variable := value
    export variable
    export variable += value
    variable += value
    export variable

可以单独使用 export 来传递所有的变量，它告诉 make 任何没有声明成 export 和 unexport 的变量都进行传递。
老版本 GNU make 默认使用这种行为，即默认传递所有变量的值，如果你想默认使用这种老版本行为，你可以在你的
makefile 中添加 .EXPORT_ALL_VARIABLES，而声明一个单独的 export。这里传递所有的变量，也只会传递那些只
包含数字字母和下划线的变量，其他变量还得使用 export 对单独的变量进行声明。相同的，单独使用 unexport 告
诉 make 不去传递变量，因为这时 make 默认行为，只有在前面定义了单独的 export（比如包含了一个定义了单独
export 的 makefile），你才会用到单独定义的 unexport。注意不能使用单独的 export 和 unexport 让一部分
变量都传递，一部分变量都不传递，因为最后一个定义的 export 或 unexport 会决定整个 make 的执行行为。

将变量的值添加到环境中，需要对它进行展开，如果展开有副作用，比如展开有 info 或者 eval 或者类似的函数，
那么这个副作用在每一次命令执行时都会出现。为了避免这类问题，不应该传递这些异常的值，最后的方法是不去使用
默认传递所有变量这个特性，而是按需要去传递。

一个特殊的变量 MAKELEVEL 在传递到下一层 make 的时候会发生变化，来记录 make 的递归调用深度。顶层 make，
它的值是 0，然后随着调用深度一次加 1。另外，使用 MAKEFILES 可以让子 make 使用额外的一些 makefile 文件。
它的值，是一个 makefile 文件的用空白分隔的列表。这个值如果定义在了外层 make 中，他会通过环境传递到子
make 中，然后子 make 会读取其他 makefile 之前，自动加载这些 makefile 文件。

**传递选项到子 make**

make 的命令行选项会自动传递到子 make，比如 make -ks 那么会传递的 MAKEFLAGS 的值是 ks。也即每个子 make
都会在环境中得到这个值，并且值当作命令行参数进行解析。不像其他环境变量，MAKEFLAGS 在环境中的值优先于
makefile 中的值。

但是一些特殊的选项，比如 -C、-f、-o、以及 -W 这些不会添加到 MAKEFLAGS 中，这些选项不会被传递到子 make。
选项 -j 比较特别，如果设置了数字参数 N 并且当前的系统支持并行执行，那么当前的 make 和所有的子 make 会进
行沟通，使得所有执行的并行任务数量不超过 N。

如果你不想传递其他的命令选项（除了 make 已经有的命令行选项），你可以这样： ::

    subsystem:
        cd subdir $$ $(MAKE) MAKEFLAGS=

其实 MAKEFLAGS 是 MAKEOVERRIDES 变量的引用，如果你不想正常的传递命令行参数，你可以想下面将 MAKEOVERRIDES
设为空，当然通常这不是很有用。但是，有些系统有对环境变量大小的限制，在 MAKEFLAGS 中添加大量的内容可能会
超过这个限制，导致出现 'arg list too loog' 的错误（在 POSIX.2 严格模式下，改变 MAKEOVERRIDES 不会影
响 MAKEFLAGS，如果在 makefile 中声明了 .POSIX 目标。你可能不需要关心这个）。 ::

    MAKEOVERRIDES =

由于历史原因，还有一个相似的变量 MFLAGS，它跟 MAKEFLAGS 类似，但是不包含命令行中定义的选项，并且总是以
``-`` 字符开头除非内容为空。而 MAKEFLAGS 以 ``-`` 字符开头除非它不包含单字符选项，例如 --warn-undefined-variables。
MAKEFLAGS 以一组单字符选项开头，然后跟随一个空格，然后是带参数的选项或者常字符选项。如果一个选项既有单
字符和长字符版本那么会选用单字符版本。传统 MFLAGS 的用法是，在递归调用 make 的命令行显式的使用它。但除
非为了与旧版本兼容，现在使用 MAKEFLAGS 就行。 ::

    subsystem:
        cd subdir $$ $(MAKE) $(MFLAGS)

在 makefile 中，你还可以添加额外的选项到 MAKEFLAGS 中，但是不能对 MFLAGS 这样用，因为 MFLAGS 仅仅用
于兼容，make 不会自动去解析你设给这个变量的值。

如果在 GNU make 的基础上，你还想使用其他实现的 make，也就不想将 GNU make 版本的选项添加到 MAKEFLAGS
中，这样 GNU make 特有的选项你可以添加到 GNUMAKEFLAGS 中。这样 GNU make 的结果还是一样，因为 make 会
在 MAKEFLAGS 之前解析 GNUMAKEFLAGS 这个变量。最好是将不影响行为的选项比如 --no-print-directory、
--output-sync 等添加到 GNUMAKEFLAGS 中。当然如果你只是用 GNU make，简单使用 MAKEFLAGS 就行。

定义封装的命令组
----------------

封装的命令组（Canned Recipes）： ::

    define run-yacc =
    yacc $(firstword $^)
    mv y.tab.c $@
    endef

    foo.c: foo.y
        $(run-yacc)

    define frobnicate =
    @echo "frobnicating target $@"
    frob-step-1 $< -o $@-step-1
    frob-step-2 $@-step-1 -o $@
    endef

    frob.out: frob.in
        @$(frobnicate) # 所有行都不会被回显
