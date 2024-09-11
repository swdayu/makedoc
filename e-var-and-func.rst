变量和函数
==========

使用变量
---------

变量是 makefile 中定义的一个名称，用来代表一串字符，这串字符也即变量的值。这些值会被替
换到变量使用的地方，比如目标、前置条件、造作命令、和 makefile 的其他部分。变量是动态作
用域，即它的值是在展开那个时间点对应的那个变量的值。

变量的名称可以使用任何字符，除了 ``:#=`` 和空白字符。但是，必须谨慎使用除了字母、数字、
已经下划线之外的其他字符，因为在一些 shell 中这些名称不能通过环境传递给子 make。用点字
符开头和使用大写字母的变量，在 make 中一些特殊意义的名称，这些名称时 make 保留的，即使
现在没有使用也可能使用在未来版本的 make 中。变量名称时大小写敏感的。

一些使用单个或几个标点符号的名称，是一些特使用途的变量，称为自动变量。

基本定义
--------

用 $ 和小括号或大括号来引用变量，比如 $(foo) 或者 ${foo}，因此 $ 是一个特殊的字符，如
果要使用它必须转义用 $$ 表示单个 $ 字符。变量的引用可以使用在 makefile 中的任何上下文
中。另外如果 $ 字符后面不是 $、(、以及 {，它将把后面的单个字符当作变量的名称，比如 $f
引用的是变量 f，但是这种写法经常让人困惑，比如 $foo 它并不是引用 foo 而是 f 然后跟随一
个字符串 oo。因此除非为了显著提高可读性（比如引用自动变量的场景），一般应该使用括号形式
的变量引用。

有两种形式的变量，它们的区别在于怎么赋值时处理对应的值，以及后面这个变量怎样使用和展开。

**递归展开**

这种变量使用 = 或者 define 或者 define = 进行定义，给这些变量的赋值会一字不差保留原样，
如果包含了对其他变量的引用，这些引用只有当该变量在真正使用时才会进行扩展。这种只有当使用
时才进行的扩展，称为递归扩展。比如： ::

    foo = $(bar)
    bar = $(ugh)
    ugh = Huh?
    all: ; echo $(foo) # 最终$(foo)会扩展成Huh?

递归展开的变量，可以在使用变量之后定义： ::

    CFLAGS = $(include_dirs) -O
    include_dirs = -Ifoo -Ibar

但是递归展开的变量，不能变量之后添加内如，因为这会进入无限递归展开陷入死循环： ::

    # $(CFLAGS) 扩展成 $(CFLAGS) -O，然后扩展成 $(CFLAGS) $(CFLAGS) -O
    CFLAGS = $(CFLAGS) -O
    # $(CFLAGS) 扩展成 -O $(CFLAGS)，然后扩展成 -O -O $(CFLAGS)，虽然也是循环但是是线性的，可以通过条件控制终止这种循环
    CFLAGS = -o $(CFLAGS)

另一个缺点是，变量定义中的函数调用，会在变量每次展开时都会被执行一次，这会导致 make 运
行变慢，更严重的它会导致 wildcard 和 shell 函数出现不预期的结果，因为你很难控制这些函
数什么什么时候调用，以及调用多少次。

变量还有一种没有定义时的条件赋值 ?=，这种变量定义只有在变量没有定义时才生效： ::

    FOO ?= bar
    # 等价于
    ifeq ($(origin FOO), undefined)
    FOO = bar
    endif

注意一个被设成空的变量仍然是定义的，因此 ?= 操作不会设置这种变量。

**简单展开**

为了避免递归展开的问题，还有一种类型的变量成为简单展开变量。简单扩展变量使用 := 或 ::=
或 define := 或者 define ::= 进行定义。两种形式的定义含义是相同的，但是只有 := 形式在
POSIX 标准中有描述，::= 的支持仅在 POSIX 问题 8 中引入。

简单展开变量的值，是在变量定义的地方直接展开，没有任何延时。只要展开完毕，这个变量的值就
不会被再次展开，在使用这个变量的地方，简单拷贝这个值就行。因此： ::

    x := foo
    y := $(x) bar # 会被展开成 y:= foo bar
    x := later    # x的值是latter

一个赋值的例子： ::

    ifeq (0,${MAKELEVEL})
    whoami := $(shell whoami)
    host-type := $(shell arch)
    MAKE := ${MAKE} host-type=$(host-type} whoami=$(whoami}
    endif

    ${subdirs}:
        ${MAKE} -C $@ all

简单扩展的变量，让 makefile 程序更可被预测，因为这种变量跟大多数的程序语言一样。通过简
单扩展变量，你可以使用变量自己的值重定义自己的内容，并且能更高效的使用函数扩展。你还可以
控制变量的前置空白字符，由于变量引用的前置空白会被从输入中丢弃掉，你可以通过变量引用的保
护来精确引入一个前置空白： ::

    nullstring :=
    space := $(nullstring) # end of the line

由于变量内容尾部的空白不会被忽略，因此你可以在内容的尾部直接加空白字符，但是这样非常难以
阅读。你最好的尾部同时加一条注释来表明空白的存在，但是如果你不想要尾部空白的话，你不要随
意的在尾部空出部分内容然后加注释，因为这些空白最终会包含到变量的内容中。比如下面的例子
dir 的内容是 /foo/bar 加上四个空格，大概率不是你想要的，因为 $(dir)/file 展开后是包含
空格的。 ::

    dir := /foo/bar    # directory to put the frobs in

还有一种立即展开的变量赋值，它使用 :::= 或 deinfe :::= 进行定义。它除了在定义时就进行
展开外，它还会在使用时再次展开，因为要再次展开所有的 $ 会被转换成 $$。 ::

    var = first
    OUT :::= $(var) # 此时的值是first
    var = second

    var = one$$two
    OUT :::= $(var) # 首先扩展后的值是one$two，但是会存在再次展开，值被转换成one$$two
    OUT += $(var) # 此时的值是one$$two $(var)，因为OUT此时是一个递归展开变量，不会对$(var)进行展开
    var = three$$four
    $(OUT) # 此时的值是one$two three$four

它与简单扩展变量的区别是，在变量赋值之后它是一个递归扩展变量，它在使用时不是简单地拷贝内
容，而是会进行扩展。因此使用 += 操作时，它不会被立即扩展。

这种风格的赋值跟传统的 BSD make 中 := 的行为一样，你可以看到它与 GNU make 中 := 不一
样的地方。GNU make 为了兼容性在 Issue 8 中将 :::= 添加到 POSIX 标准中的。

设置变量
--------

变量的值可以来源于很多地方：

1. 你可以在运行 make 时覆盖一个变量在 makefile 中定义的值
2. 在 makefile 中定义，可以使用赋值或者多行赋值
3. 在 let 函数或者 foreach 函数中，使用一个短生命周期的值
4. 来源于环境中的变量，被 make 使用变成一个 make 中的变量
5. 自动变量在每个规则中会自动得到它们对应含义的值
6. 一些在隐式规则中使用的变量，拥有常量初始值

**环境变量**

make 中的变量可以来源于 make 运行的环境，make 识别到的每个环境变量会在 make 启动时被转
换成相同名字和值的 make 变量。但是，环境变量的值会被 make 命令行变量和 makefile 中定义
的变量的赋值覆盖。如果使用了-e 选项，环境变量的值会覆盖 makefile 中定义的变量赋值。

在每个命令行命令的前面，可以单独设置只影响这一条命令的一组环境变量。比如： ::

    x='once upon' y='a time' bash -c 'echo $x $y'

通过 export 可以将 make 变量传递到环境中，变成环境变量。注意环境变量是每一条运行的 shell
命令都可以访问到的变量。当 make 运行的是另一个 mak e时（递归调用的子 make），make 不仅
将原本定义的环境变量以及 export 的环境变量传递给子 make，还把命令行定义的变量以及命令行
参数也传递给子 make（通过 MAKEFLAGS）。另外，MAKELEVEL 和 MAKEFILES 两个特殊的变量也
会传递给子 make。特别地，make 不会传递环境变量中的 SHELL 变量，SHELL 的值也不会被 make
自动 export 到环境中，子 make 继承当前 make 的用户环境，子 make 使用哪个 shell 根据自
己的一套规则来判断。

**命令行变量**

make 命令行参数中包含 = 的参数相当于定义了一个变量，比如 v=x 定义了一个值为 x 的变量 v。
命令行变量的值会覆盖 makefile 中定义的变量赋值，而且会覆盖环境变量的值。 ::

    make CFLAGS='-g -O'
    CFLAGS = -g
    foo.o: foo.c
        cc -c $(CFLAGS) foo.c

**override 变量**

在 makefile 中定义的变量，如果你想这个变量不会被命令行变量的值覆盖，可以添加 override
指示，比如： ::

    override variable = value
    override variable := value
    override variable += more text # 会使用override版本或者命令行版本最后才是普通变量版本添加内容

声明成 override 的变量赋值拥有最高的优先级，比其他任何的赋值优先级都高，除了后面另一个
override 赋值。后面没有声明成 override 的所有对这个变量的赋值，以及对这个变量的内容添
加都会被忽略掉。

使用 override，你可以在用户提供的定制的命令行参数的基础上，经过修改添加得到新的值。比如
你想让 CFLAGS 总是添加上 -g 的选项： ::

    override CFLAGS += -g

**makefile 变量**

在 makefile 中设置变量，可以使用 =、?=、!=、:=、::=、:::=，+= 或者 define 版本的赋值
操作。赋值操作后面的内容都变成变量的值，除了赋值操作符后面的前置空白字符会被忽略（define
版本不会忽略任何字符）。

变量的名字可以包含变量的引用和函数调用，它们会在所在行被读取时立即展开。变量的值没有长度
限制，最大长度取决于你使用机器的物理内存。但是当 make 变量被传递给 shell 使用时，可能会
被这个 shell 定义的变量长度限制。

大多数没有设置的变量的值都是空字符串，也即你引用一个未定义的变量时，它的值是空字符串。一
些有内置初始值的变量，比如一些隐式规则中的变量，它们有特定的值，但是你可以通过正常的方式
设置它们。还有一类特殊的变量，它们会在每个规则中自动被设置，这一类变量称为自动变量。

如果你想只有当一个变量未定义时才设置，可以使用 ?= 来代替 = 赋值操作符。如果你想定义 :=
版本的 ?=，可以显式使用下面类似的条件语句定义。 ::

    FOO ?= bar
    # 相当于
    ifeq ($(origin FOO), undefined)
    FOO = bar
    endif

另外，还有一个 shell 赋值操作符 !=，make 会将右边的值立即展开并将展开的内容传递给 shell，
然后处理 shell 执行的输出结果当作变量的值。如果输出结果最后一个字符是一个换行，会把这个
换行移除掉，而其他的换行都会被替换成空格。最后的结果就当作这个变量的值，而且这个变量是递
归扩展的，也即如果最后的结果包含有变量引用，不会对其进行扩展。因此，如果 shell 的输出包
含 $ 字符，而且你想让它们解析成 make 的变量引用或者函数调用，你必须将所有的 $ 都转换成
$$。 ::

    hash != printf '\043'
    file_list != find . -name '*.c'

但是，你也是使用简单扩展变量，显式的调用 shell，函数 shell 刚刚执行完的 shell 脚本结果
保存在特殊的变量 .SHELLSTATUS 中： ::

    hash := $(shell printf '\043')
    var := $(shell find . -name "*.c")

**添加变量内容**

通常如果一个变量已经定义，对这个变量添加更多的内容是非常有用的。你可以使用 += 赋值操作符
添加内容： ::

    objects += another.o

如果 objects 变量有值（不是未定义的或者定义成空字符串的变量），它会读取变量值，然后添加
一个空格，再添加 += 操作符后面的内容，操作符后面的前置空白会被忽略。如果这个变量没有值，
不会添加空格，此时 ?= 相当于是 =。如果这个变量是递归扩展的，+= 操作夫只会添加内容，而不
i 会对最后的内容进行扩展；如果是简单扩展变量，会对最后的内容进行扩展。因此： ::

    variable := value
    variable += more
    # 完全等价于
    variable := value
    varialbe := $(variable) more

而对于 = 或者 :::= 的递归展开变量， ::

    variable = value
    variable += more
    # 粗略的等价如下，毕竟 += 并没有定义一个叫 temp 的变量
    temp = value
    variable = $(temp) more

另外也不能确切的等价于： ::

    variable = value
    variable := $(variable) more

因为 += 对于递归展开的变量，不会对最后的内容进行展开，只有在被引用时才展开。可以对比下面
两个例子的不同： ::

    CFLAGS = $(includes) -O
    CFLAGS += -pg
    includes = -Iinc

    CFLAGS = $(includes) -O
    CFLAGS := $(CFLAGS) -pg
    includes = -Iinc

定义多行变量
------------

可以使用 define 定义变量，是变量的值包含多行内容，define 操作的不寻常语法是它允许换行符
出现在变量值中。这样，可以很方便的定义包装的命令组，还可以方便地定义 makefile 片段让
eval 函数执行。define 定义的变量的值包含在 define 行和 endef 行之间，除了 endef 行之
前的那个最后的换行符被移除掉外，其他内容都是变量值的内容。如果你想让变量的值最后包含一个
换行符，你必须在最后添加一个空行，特殊的如果变量值是一个换行符，你需要包含两个空行： ::

    define newline


    endef

你可以嵌套定义 define，但是每个 define 都要正确的用 endef 结束，否则 make 会报错。注
意用 Tab 字符开头的行会被解析成规则的操作命令，不在 make 语法的内容范围内，因此这些行上
出现的 define 或者 endef 都不会是 make 的操作指示指令。

用 define 包装命令组的例子： ::

    define two-lines
    echo foo
    echo $(bar)
    endef

当使用时，上面 two-lines 的定义类似于： ::

    two-lines = echo foo; echo $(bar)

因为用 ; 分隔的两条命令相当于两条单独的命令，但不同的是 define 定义的多行命令是在多个独
立的 shell 进程中执行的。除了 define 赋值的定义语法差别，其他的都和普通的变量赋值定义一
样。

取消变量定义
------------

如果要清除一个变量，一般将变量值置为空即可。对于一个空值变量或者未定义的变量，这个变量的
扩展的结果都是一样的，都是空字符串。但是，如果使用 flavor 和 origin 函数，它们的结果是
不同的。这种情况下，你需要使用 undefine 将一个变量设成未定义的，也即就像这个变量从来都
没有被设置过一样。例如： ::

    foo := foo
    bar = bar
    undefine foo
    undefine bar
    $(info $(origin foo))
    $(info $(flavor bar))

这里两次 info 函数调用都会打印 "undefined"。如果你想 undefine 一个命令行定义的变量，需
要使用 override： ::

    override undefine CFLAGS

高级特性
---------

替换引用是 $(var:a=b) 或者 ${var:a=b} 形式的引用，它先引用变量的值，然后将值内容中出现
的每一个 a 都替换成 b，这里的 a 必须出现再空白字符后面或者一个字符串的尾部。例如： ::

    foo := a.o b.o l.a c.o
    bar := $(foo: .o=.c) # 它的值是a.c b.c l.a c.c

替换引用是 patsubst 表达式函数的省略写法，具体的 $(var:a=b) 等价于：$(patsubst %a,%b,var)。
另一种使用%形式的替换引用，可以使用 patsubst 函数的全部功能，这种情况等价于 $(patsubst a,b,$(var))，
例如：bar := $(foo:%.o=%.c)。

另一种高级用法是计算变量的名字，这个功能再成熟的 makefile 程序中非常有用。变量的名字可
以是另一个变量的引用，也成为嵌套的变量引用，而且嵌套的深度没有限制。例如： ::

    x = y
    y = z
    a := $($(x)) # 展开为$(y)，继续展开为z

    x = $(y)
    y = z
    z = Hello
    a := $($(x)) # 展开为$($(y))，继续展开为$(z)，最后展开为Hello

    x = v1
    v2 := Hello
    y = $(subst 1,2,$(x))
    z = y
    a := $($($(z))) # 依次展开$($(y)) $($(subst 1,2,$(x))) $($(subst 1,2,v1)) $(v2) Hello

    a_dirs := dira dirb
    1_dirs := dir1 dir2
    a_files := filea fileb
    1_files := file1 file2
    ifeq "$(use_a)" "yes"
    a1 := a
    else
    a1 := 1
    endif
    ifeq "$(use_dirs)" "yes"
    df := dirs
    else
    df := files
    endif
    dirs := $($(a1)_$(df))

    a_objects := a.o b.o c.o
    1_objects := 1.o 2.o 3.o
    sources := $($(a1)_objects:.o=.c)

    dir = foo
    $(dir)_sources := $(wildcard $(dir)/*.c)
    define $(dir)_print =
    lpr $($(dir)_sources)
    endef

计算变量名称的唯一限制是，它不能用来定制一个函数的名字。因为函数的名字的识别是在嵌套引用
展开前进行识别的。例如： ::

    ifdef do_sort
    func := sort
    else
    func := strip
    endif
    bar := a d b g q c
    foo := $($(func) $(bar))

这里，foo 的值是变量 'sort a d b g q c' 或者变量 'strip a d b g q c' 的值，而不是用
'a d b g q c' 作为参数调用 sort 或 strip 函数的返回值。这个限制可能会在未来的版本中移
除。

基于目标的变量
---------------

make 变量通常情况下是全局的，也就是说不管这个变量在那个地方使用，它们都是同一个。一种例
外是使用 let 或 foreach 定义的变量，或者自动变量。另一种例外是，基于规则目标的变量，这
些变量赋值出现在目标依赖的前置条件中： ::

    target ...: variable-assignment

这个功能允许你为同一个变量定义多个不同的值，基于不同目标决定使用多个值中的哪一个，也即
make 当前正在构建那个目标，就使用那个目标对应的值。基于目标的变量赋值可以使用所有的前置
操作执行命令，包括 export、unexport、override、或者 private，它们只应用对应的行为到当
前单独的这个目标变量实例之上。如果规则中的目标是一个列表，则会为每个目标创建一个变量实例。
注意，基于目标的变量完全区别于任何普通的全局变量，这两个变量都不必是相同的类型（递归扩展
的还是简单扩展的）。

特别地，当一个目标变量定义之后，这个变量对这个目标所有前置条件所列的依赖目标都是生效的，
并且对所有依赖目标的所有前置条件也还是生效的，除非对应的前置条件自己用 override定义了自
己的版本。例如： ::

    prog: CFLAGS = -g
    prog: prog.o foo.o bar.o

当构建 prog 目标时，CFLAGS 被设置成 -g，在构建 prog.o、foo.o、bar.o时，CFLAGS 也是 -g，
另外构建这些目标的依赖目标时，CFLAGS 还是 -g。

注意每个前置条件，在每个 make 调用中最多只会被构建一次。如果一个相同的文件，被多个目标
依赖，而且这多个目标都定义了自定版本的不同值的目标变量，那么只有第一次触发构建对应前置条
件的目标的值会被使用。当构建其他目标时，因为这个前置条件已经构建了，不会再构建。

基于模板的变量
--------------

除了目标变量，GNU make 还支持基于模板的变量，它会为任何匹配对应模板的目标都定义一个特定
的变量实例： ::

    pattern ...: variable-assignment

如果一个目标匹配多个模板，会优先使用匹配长度更长的那个模板，也即匹配更精确的那个模板。例
如： ::

    %.o: %.c
        $(CC) -c $(CFLAGS) $(CPPFLAGS) $< -o $@
    lib/%.o: CFLAGS := -fPIC -g
    %.o: CFLAGS := -g
    all: foo.o lib/bar.o

当编译 lib/bar.o 目标时，会使用第一个目标变量 CFLAGS 来编译，即使第二个目标变量也是匹
配的。如果多个匹配的长度一样，那么先出现的那个模板优先。当编译目标时，会先搜寻基于目标的
变量，然后搜寻基于模板的变量，然后再搜寻该目标作为依赖条件的上一级目标的变量（父目标）。

可以看到，依赖条件中的子目标继承了父目标定义的变量。但有时候，你并不想将目标变量继承给
子目标，这时可以使用 make 提供的 private 提示符。虽然这个提示符可以使用在任何变量赋值操
作符上，但对基于目标和模板的变量更有意义。任何声明了 private 提示符的目标变量，他的作用
域仅限于当前的目标，不会被依赖条件中的子目标继承。如果对普通全局变量声明了 private 提示
符，这个变量仅能用在全局环境，不会被任何目标继承，也就不能用到任何操作命令中。 ::

    EXTRA_CFLAGS =
    prog: private EXTRA_CFLAGS = -L/usr/local/lib
    prog: a.o b.o

因为声明了 private，编译 ao.o 或 b.o 目标时，不会继承 prog 的目标变量，只有定义为空的
全局变量是可以见的。

使用函数
--------

函数可以用来处理文本或对文件进行转换，函数调用类似于变量引用，它可以出现在任何变量引用可
以出现的地方，而且使用相同的规则进行展开。一个函数调用的语法如下： ::

    $(function arguments)或者${function arguments}

这里 function 是函数的名字，make 提供了一些内置函数，你也可以使用 call 函数定义自己的
函数。函数的参数与函数名字之间用一个或多个空格或 tab 字符进行分隔，如果参数的个书超过一
个，参数使用逗号进行分隔，注意逗号两侧的空白是参数值的一部分。每个参数除了一些特殊的情况，
都会在函数调用之前先展开，参数的展开顺序按参数出现顺序依次进行。一些函数调用的例子： ::

    $(subst a,b,$(x))
    $(subst a,b,%{x})

当使用 make 中的特殊字符作为函数的参数时，因为 GNU make 并不支持反斜杠或其他转义字符，
需要将特殊字符放到变量中已到达隐藏的目的，因为函数参数先进行分割，然后才进行扩展的。需要
隐藏的特殊字符包括：

1. 逗号
2. 第一个参数的前置空白
3. 不进行括号匹配的括号或大括号

::

    comma := ,
    empty :=
    space := $(empty) $(empty)
    foo := a b c
    bar := $(subst $(space),$(comma),$(foo)) # 结果为a,b,c

预定义函数
----------

$(subst from,to,text)
    文本中的每一处 from 都被替换成 to，例如 $(subst ee,EE,feet on the street) 的结
    果是'fEEt on the strEEt'。

$(patsubst pattern,replacement,text)
    在 text 中查找空白字符分隔的词，并且这个词可以匹配 pattern，然后将这个词用 replacement
    替换。pattern 和 replacement 中可以出现一个%字符，如果有多个只有第一个当作通配符，
    后面的%字符不被特殊对待。返回的结果，空白会被一个空格代替，并且会忽略首尾空白。% 前
    面的一个或多个反斜杠字符 \ 会首先从模式中以转义字符的形式移除，然后才进行匹配。比如
    the\%weird\\%pattern\\ 实际对应的字符串是 the%weird\%pattern\\，其中第二个 % 是
    通配符。最后两个反斜杠没有当作转义字符处理，是因为它们没有出现在%之前。$(patsubst %.c,%.o,x.c.c bar.c)
    的结果是 x.c.o bar.o，另外变量的代换引用其实是 patsubst 的缩略形式，比如 $(var:suffix=replacement)
    相当于 $(patsubst %suffix,%replacement,$(var))，例如： ::

        objects = foo.o bar.o baz.o
        $(objects:.o=.c) # 相当于$(patsubst %.o,%.c,$(objects))

$(strip string)
    去除字符串的首尾空白，并且中间的空白被替换为一个空格。例如 $(strip a b c ) 的结果
    是 'a b c'。

$(findstring find,in)
    在字符串 in 中查找 find 字符串，如果存在返回 find 字符串，否则返回空。例如： ::

        $(findstring a,a b c)   # 结果是'a'
        $(findstring a,b c)     # 结果是空

$(filter pattern...,text)
    保留文本中以空白分隔的词，只要这个词与模式列表中的任意一个模式匹配。例如： ::

        sources := foo.c bar.c baz.s ugh.h
        foo: $(sources)
        $(CC) $(filter %.c %.s,$(sources)) -o foo

    这里只有 foo.c bar.c baz.s 保留了下来。

$(filter-out pattern...,text)
    移除文本中以空白分隔的词，只要这个词与模式列表中的任意一个模式匹配。例如： ::

        objects = main1.o foo.o main2.o bar.o
        mains = main1.o main2.o
        $(filter-out $(mains),$(objects))

    这里只有 foo.o bar.o 保留了下来。

$(sort list)
    对 list 中以空白分隔的词进行按字母排序，并且去掉重复的词。例如 $(sort foo bar lose)
    的结果是 'bar foo lose'。

$(word n,text)
    返回文本中以空白分隔的第 n 个词，第一个词从 1 开始。例如： ::

        $(word 2, foo bar baz) # 结果是 bar。

$(wordlist s,e,text)
    返回文本中以空白分隔的从第 s 个词到第 e 个词，包括第 e 个词。s 从 1 开始，如果 s
    大于 e 返回空。例如： ::

        $(wordlist 2, 3, foo bar baz) # 结果是'bar baz'。

$(words text)
    返回文本中以空白分隔的词的个数。因此 text 的最后一个词是： ::

        $(word $(words text),text)

$(firstword names...)
    相当于 $(word 1,text)。

$(lastword names...)
    相当于 $(word $(words text),text)，但更简洁更高效。

$(dir names...)
    返回所有文件名称的目录部分，最后一个字符是斜杠字符 /，如果文件名称不包含目录返回 ./。

$(notdir names...)
    返回所有文件名称的非目录部分。

$(suffix names...)
    返回所有文件名称的后缀名，如果没有后缀名则为空。例如： ::

        $(suffix src/foo.c src-1.0/bar.c hacks) # 结果是 '.c .c'

$(basename names...)
    返回所有文件名称除后缀名的部分，例如 $(basename src/foo.c src-1.0/bar hacks) 的
    结果是 'src/foo src-1.0/bar hacks'。

$(addsuffix suffix,names...)
    给所有文件名添加后缀。

$(addprefix prefix,names...)
    给所有文件名称添加前缀。

$(join list1,list2)
    将两个列表相同位置的文件名称合并到一起。例如： ::

        $(join a b c,.c .o) # 结果是 'a.c b.o c'

$(wildcard pattern...)
    在当前目录下与模式相匹配的文件名列表。可以出现多个模式，
    其结果是按顺序每个模式匹配的文件的列表。

$(realpath names...)
    获取所有文件名的真实路径名，不包含 . 或者 .. 或者重复的 /，并且会解析符号链接找到真
    正的文件。如果发生错误返回空字符串，具体可以查看 realpath(3) 文档。

$(abspath names...)
    如果 realpath 的不同是，不会解析符号链接，并且不要求对应的文件或者目录真的存在。可
    以使用 wildcard 函数来测试文件是否存在。

$(if condition,then-part[,else-part])
    这类函数的特点是，不是所有的参数都会展开，只有那些需要展开的参数才会展开。第一个参数
    首先去掉首尾空白然后展开，如果不是空字符串则条件为真，如果是空字符串条件为假。如果为
    真，第二个参数会被执行当作函数的结果。如果为假，第三个参数会被执行当作函数的结果，如
    果没有第三个参数那么函数的结果为空字符串。其中第二个和第三个参数，只有一个会被执行。

$(or condition1[,condition2[,condition3...]])
    每个参数会按顺序执行，如果有一个参数的值不为空字符串，把该参数的值当作函数返回值返回，
    如果所有的参数都为空字符串则函数返回空。

$(and condition1[,condition2[,condition3...]])
    每个参数会按顺序执行，如果有一个参数的值为空字符串，则返回空字符串，如果所有的参数的
    值都不为空，函数返回最后一个参数的值作为结果。

$(intcmp lhs,rhs[,lt-part[,eq-part[,gt-part]]])
    十进制整数的比较，第一个和第二个参数先被展开，当作整数比较的左操作数和右操作数。如果
    没有第三个参数，那么函数当两个整数不相等时返回空，相等时返回这个相等的整数。如果小于
    返回第三个参数的值，如果等于返回第四个参数的值，如果大于返回第五个参数的值。如果小于
    没有第三个参数，那么函数返回空。如果等于没有第三个参数返回这个整数值，如果没有第四个
    参数返回空。如果大于没有第三个参数返回空，没有第四个参数返回空，没有第五个参数返回第
    四个参数。因此 $(intcmp 9,7,hello) 和 $(intcmp 9,7,hello,world,) 会返回空，而
    $(intcmp 9,7,hello,world) 会返回 world。

$(let var [var ...],[list],text)
    首先 var 和 list 会被展开，text 不会被展开。然后被展开的 list 的值会被依次赋值给
    var。如果 list 的值个数更多，那么剩余的值都会被赋最后一个 var。这里的变量的赋值，是
    简单展开赋值。等变量都赋值后，text 才会被展开形成最后的值返回。例如： ::

        reverse = $(let first rest,$1,$(if $(rest),$(call reverse,$(rest)) )$(first))
        all: ; @echo $(call reverse,d c b a)
        # 相当于:
        $(let first rest,d c b a,$(if c b a,$(call reverse,c b a) )d)
        $(call reverse,c b a) d
        $(let first rest,c b a,$(if b a,$(call reverse,b a) )c) d
        $(call reverse,b a) c d
        $(let first rest,b a,$(if a,$(call reverse,a) )b) c d
        $(call reverse,a) b c d
        $(let first rest,a,$(if ,$(call reverse,) )a) b c d
        a b c d

$(foreach var,list,text)
    首先 var 和 list 会被展开，text 不会被展开。然后对于展开的 list 中的每一个空白分隔
    的值，依次赋给由 var 展开的变量名，然后 text 每次用新的 var 值进行展开。最后返回 text
    每次展开的值，中间用空格隔开。 ::

        dirs := a b c d
        files := $(foreach dir,$(dirs),$(wildcard $(dir)/*))

    相当于列出这四个目录下的所有文件： ``$(wildcard a/* b/* c/* d/*)``。当 text 太复
    杂时，可以赋值给一个变量，例如： ::

        find_files = $(wildcard $(dir)/*)
        files := $(foreach dir,$(dirs),$(find_files))

    像 let 函数一样，foreach 中出现的变量不会影响全局作用域中的变量。要特别注意当 text
    覆盖变量使用时，该变量中引用的临时变量要与 foreach 中定义的临时变量名称要匹配。

$(file op filename[,text])
    允许 make 从一个文件中读取值，或者将值写入文件中。写入一个不存在的文件时会先创建这个
    文件，写入时函数的返回值为空。读取时返回读取的文件内容，但是最后一个换行字符会被移除，
    如果读取的文件不存在返回空。文件模式 op 可以是>表示覆盖文件内容，>> 表示写入到文件
    尾，< 读取文件内容。在 op 和文件名之间可以存在可选的空白。text 参数只有当写入文件时
    有效，如果读取文件提供了这个参数将报错。如果 text 不以换行符结尾（包括空字符串），会
    添加一个换行符，如果不提供这个参数，将不会写入内容。 ::

        program: $(OBJECTS)
        $(file >$@.in,$^)
        $(CMD) $(CMDFLAGS) @$@.in # 这些额外的参数在文件中用空格分隔
        @rm $@.in
        program: $(OBJECTS)
        $(file >$@.in) $(foreach O,$^,$(file >>$@.in,$O))
        $(CMD) $(CMDFLAGS) @$@.in # 这些额外的参数在文件中空换行分隔
        @rm $@.in

    使用这个函数，可以将长内容保存到文件中，如上面的例子，并且很多命令也都支持通过 @ 前
    缀从一个文件中接受命令参数。

$(call variable,param,param,...)
    该函数可以用于创建一个新的参数化的函数，新定义的函数对应的复杂的表达式可以使用一个变
    量来表达，然后使用 call 函数每次可以使用不同的参数值，对这复杂的表达式进行展开。在
    函数中可以使用 $(0)、$(1)、$(2)等等分别引用参数 variablbe,param,... 的值。如果
    variable 时一个内置函数的名字，会优先调用这个函数，即使有一个相同名字的变量存在。参
    数 param 会首先展开，然后赋值给临时变量 $(1)、$(2) 等等。该函数可以嵌套调用，但是
    每个 call 都拥有自己版本的 $(0)、$(1) 等临时变量。使用 call 的一些例子： ::

        reverse = $(2) $(1)
        foo = $(call reverse,a,b) # 结果是 b a
        pathsearch = $(firstword $(wildcard $(addsuffix /$(1),$(subst :, ,$(PATH)))))
        LS := $(call pathsearch,ls) # 结果是 /bin/ls 或者类似的
        map = $(foreach a,$(2),$(call $(1),$(a)))
        o = $(call map,origin,o map MAKE) # 结果是 file file default

    另外特别注意参数的前后空白，像大多数函数一样，call 函数的第二个和后面的参数如果包含
    空白，会被原样保留传递给临时参数，这可能导致与你预期不一样的效果。

$(value variable)
    不展开的情况下，读取一个变量的值，注意提供的参数时变量的名字，不是引用。 ::

        FOO = $PATH
        all:
        @echo $(FOO) # 因为$P未定义，因此打印 ATH
        @echo $(value FOO) # 打印 $PATH，也就是当前的环境变量 $PATH 的值

    该函数通常与 eval 一起使用。

$(eval string)
    参数的值首先会被展开，然后被当作 makefile 语法内容被执行，该函数返回空。注意该函数
    的参数被展开了两次，依次是作为参数时，第二次是被执行时。因此你可能需要提供两个层次的
    $ 字符，这种情况下使用 value 函数可能有用。 ::

        PROGRAMS = server client
        server_OBJS = server.o server_priv.o server_access.o
        server_LIBS = priv protocol
        client_OBJS = client.o client_api.o client_mem.o
        client_LIBS = protocol
        .PHONY: all
        all: @(PROGRAMS)
        define PROGRAM_template =
        $(1): $$($(1)_OBJS) $$($(1)_LIBS:%=-l%)
        ALL_OBJS += $$($(1)_OBJS)
        endef
        $(foreach prog,$(PROGRAMS),$(eval $(call PROGRAM_template,$(prog))))
        $(PROGRAMS):
            $(LINK.o) $^ $(LDLIBS) -o $@
        clean:
            rm -f $(ALL_OBJS) $(PROGRAMS)

$(origin variable)
    不操作变量的值，仅仅返回变量本身的来源： ::

        'undefined' - 未定义变量
        'default' - 预定义变量，例如CC
        'environment' - 从环境变量中继承的变量
        'environment override' - 从环境变量中继承的变量，并且覆盖了makefile中的变量的值
        'file' - makefile中变量中定义的变量
        'override' - makefile中定义的override提示符声明的变量
        'automatic' - 自由变量

        ifdef bletch
        ifeq "$(origin bletch)" "environment"
        bletch = barf, gag, etc.
        endif
        endif
        ifneq "$(findstring environment,$(origin bletch))" ""
        bletch = barf, gag, etc.
        endif

$(flavor variable)
    不操作变量的值，仅仅返回变量自身的类型信息： ::

        'undefined' - 未定义变量
        'recursive' - 递归展开变量
        'simple' - 简单展开变量

$(error text...)
    产生一个致命错误信息，然后 make 会退出。例如： ::

        ifdef ERROR1
        $(error error is $(ERROR1))
        endif
        ERR = $(error found an error!)
        .PHONY: err
        err: ; $(ERR)

$(warning text...)
    产生一个警告信息，make 会继续执行，函数的返回值是空。

$(info text...)
    打印一条信息，函数返回值为空。

$(shell cmd)
    该函数与其他函数都不一样（除了 wildcard），它可以与 make 之外的世界沟通。它执行一
    条 shell 命令，并将执行结果当作函数返回，make 会将执行结果中的最后一个换行符去掉，
    并且将其他的换行符都换成一个空格。另外!=赋值操作符也提供了类似的执行 shell 命令的能
    力，当 shell 函数或者 != 赋值执行完毕后，shell 的退出状态会保存在 .SHELLSTATUS变
    量中。 ::

        contents := $(shell cat foo) # foo文件的内容，换行被替换成空格
        files := $(shell echo *.c) # 相当于$(wildcard *.c)
        export PATH = $(shell echo /usr/local/bin:$$PATH)
        all: ; @echo $$PATH

    这里新的 PATH 会导出到环境中，然后 all 的每个命令行都会继承这个环境，会打印新的 PATH
    的值。

$(guile code)
    只有支持 GNU Guile 作为内置扩展语言时才支持，可以查看 .FEATURES 变量是否支持 guile。
    该函数的参数首先被扩展，然后传递给 GNU Guile 解释器，执行的结果会被转换成字符串当作
    函数结果返回。
