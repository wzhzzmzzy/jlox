# Representing Code


# [Context-Free Grammars](https://craftinginterpreters.com/representing-code.html#context-free-grammars)


## Lexical grammar 和 Syntactic grammar 的区别

Lexical 由 Scanner 处理，将字符串处理为 Lexeme 或者 Token。而在 Syntactic grammar 中，Parser 会将其处理为 Expression。


## Rules for grammar

在 CFG 中，我们需要创建 _Productions_ 作为规则，将词素组织起来，所有的文本都是这些 _Productions_ 的 _Derivations_。_Productions 有两种元素构成，称为 terminal 和 nonterminal：terminal 可以看作是字面量，是一个独立的词素；nonterminal 则是对另一条规则的引用，用于规则与规则的组合，当然也支持递归。下面是一个 Production 的例子：_

	_breakfast → protein ( "with" breakfast "on the side" )?_
	_          | bread ;_
	_protein   → "really"+ "crispy" "bacon"_
	_          | "sausage"_
	_          | ( "scrambled" | "poached" | "fried" ) "eggs" ;_
	_bread     → "toast" | "biscuits" | "English muffin" ;_

上面的 CFG 生成式语法类似于正则，他们（这样的，或是EBNF或是别的）有助于明确语法的形式化设计，也便于与其他语言设计者进行交流。


# [A Grammar for Lox expressions](https://craftinginterpreters.com/representing-code.html#a-grammar-for-lox-expressions)

_下面是 Lox 的表达式语法：_

	_expression     → literal_
	_               | unary_
	_               | binary_
	_               | grouping ;_
	_literal        → NUMBER | STRING | "true" | "false" | "nil" ;_
	_grouping       → "(" expression ")" ;_
	_unary          → ( "-" | "!" ) expression ;_
	_binary         → expression operator expression ;_
	_operator       → "==" | "!=" | "<" | "<=" | ">" | ">="_
	_               | "+"  | "-"  | "*" | "/" ;_


# [_Implementing Syntax Trees_](https://craftinginterpreters.com/representing-code.html#implementing-syntax-trees)

目前我们的表达式类型一共有四种，对于每一种表达式，在不同阶段都存在不同的操作。在 Java 中，我们增加一种表达式，只需要增加一个 Class，但增加一种操作，就需要在每个 Class 中增加一个处理方法。而在 FP 语言当中，我们可以定义函数，在函数内部判断表达式类型，然后做出不同的操作，坏处就是增加操作时需要修改每一个函数。

这里介绍一种 Visitor 模式，通过这种模式，可以在 Java 这种 OOP 语言中做到类似 FP 的按操作扩展：

	interface Visitor {
	  visitLiteralExpr(Literal expr);
	  visitLiteralExpr(Literal expr);
	  visitUnaryExpr(Unary expr);
	}
	abstract class Expr {
	  abstract accept(Visitor visitor);
	}
	static class Literal extends Expr {
	  @Override
	  accept(Visitor visitor) {
	    return visitor.visitLiteralExpr(this);
	  }
	}

这是一种非常常见的设计模式，可以让类型变得易于扩展。

# ◉ Parsing Expressions


# [Ambiguity and the Parsing Game](https://craftinginterpreters.com/parsing-expressions.html#ambiguity-and-the-parsing-game)

前文介绍了 CFG 是如何生成字符串的，对于编译器来说，这个过程则是相反的，也就是对一个字符串，找到一个能够和他匹配的 CFG。这里可能会发生歧义问题，也就是一个字符串可能可以解析成多种 AST。此处就需要引入两个方法——Precedence 和 Associativity。

- 优先级（Precedence）决定了多个运算符之间的求值顺序，例如 ‘/’ > ‘-’。
- 结合性（Associativity）决定了单个运算符的多个操作数的求值顺序，例如 ‘-’ 是左结合，就会先求左边的值，赋值是右结合，会先求右边的值。

下一个问题是如何在 CFG 内使用这两个方法。以 C 语言的运算符优先级为例，我们将其拆分为以下这些运算符：

	expression     → ... // 表达式
	equality       → ... // == !=
	comparison     → ... // > >= < <=
	term           → ... // - +
	factor         → ... // / *
	unary          → ... // ! -
	primary        → NUMBER | STRING | "true" | "false" | "nil" ｜ "(" expression ")" ;

为了让 equality 是最低优先级的，我们可以让 express 直接指向它：

	expression     → equality

然后，我们从下往上补全：

	unary          → ( "!" | "-" ) unary
	               | primary ;
	factor         → factor ( "/" | "*" ) unary
	               | unary ;

这里的 factor 用了左递归写法，用来表示乘除法的左结合完全没有问题。只是，左递归的解析性能显然较差，所以我们替换为迭代：

	factor         → unary ( ( "/" | "*" ) unary )* ;

最终结果就是下面这样：

	expression     → equality ;
	equality       → comparison ( ( "!=" | "==" ) comparison )* ;
	comparison     → term ( ( ">" | ">=" | "<" | "<=" ) term )* ;
	term           → factor ( ( "-" | "+" ) factor )* ;
	factor         → unary ( ( "/" | "*" ) unary )* ;
	unary          → ( "!" | "-" ) unary
	               | primary ;
	primary        → NUMBER | STRING | "true" | "false" | "nil"
	               | "(" expression ")" ;

通过这种写法，可以完整的表述左结合和运算符之间的优先级。


# [Recursive Descent Parsing](https://craftinginterpreters.com/parsing-expressions.html#recursive-descent-parsing)

递归下降是一种 top-down parser，与前文 CFG 语法的对应关系大概如下：

Terminal - if 语句，解析指定的 Token

Nonterminal - 调用对应规则的函数

| - if / switch 语句，处理内部的 Terminal / Nonterminal

* or + - while / for 循环

? - if 语句

# [Syntax Errors](https://craftinginterpreters.com/parsing-expressions.html#syntax-errors)

Parser 的工作有两项：解析有效的 Token 序列，生成对应语法树；解析无效的 Token 序列，监测任何异常并告知用户。在监测异常部分，一个 Decent Parser 需要做到的点是：

1. 检测到错误并反馈给用户
2. 避免出现崩溃和无限循环的情况
3. 快速检测
4. 报告出尽可能多的错误
5. 最小化级联错误，避免单个异常导致一连串的报错

处理异常并恢复的常见方法称为 panic mode。当 Parser 遭遇错误时，会陷入 panic，在 Java 中就是抛出了 Exception。我们可以在预设的同步点（例如语句）捕获异常，抛弃直到下一个语句之前的所有 Token，就可以让 Parser 继续工作了。

# [Statements](https://craftinginterpreters.com/statements-and-state.html#statements)

这里我们添加两种语句：表达式语句、Print 语句。由于 Lox 这门动态类型的脚本语言非常简单，所以程序可以被看作一个语句列表。以下是新的 CFG 定义：

    program        → statement* EOF ;
    statement      → exprStmt
    | printStmt ;
    exprStmt       → expression ";" ;
    printStmt      → "print" expression ";" ;
    至此，Lox 的语句定义就初步完成了。

# [Executing statements](https://craftinginterpreters.com/statements-and-state.html#executing-statements)

修改一下 Interpreter 和 Parser，让 Parser 解析 Token 后按上面的优先级返回 PrintStatement 或者 ExpresstionStatement，然后由 Interpreter 执行。由于 Statement 不需要返回值，所以 Interpreter 将返回 Void。

# [Global Variables](https://craftinginterpreters.com/statements-and-state.html#global-variables)

处理全局变量时，我们需要添加两个新的结构：

1.	变量声明语句：将一个变量名与一个值进行绑定
2.	变量表达式：绑定完成之后，使用变量名作为表达式时，会自动查找对应值并返回

# [Variable Syntax](https://craftinginterpreters.com/statements-and-state.html#variable-syntax)

声明语句和其他语句之间有一些区别，比如说，在 Lox 的语法当中，变量不可以在 Control Flow 语句内部使用，因为这会导致对变量的作用域有所困惑。因此，为了区分声明和其他语句，我们添加一条新的 CFG 定义：

    program        → declaration* EOF ;
    declaration    → varDecl
    | statement ;
    statement      → exprStmt
    | printStmt ;

目前的声明语句只包括了变量，之后会添加函数和类的声明。任何允许使用声明语句的地方都可以使用其他语句，所以 declaration 以 statement 兜底。然后我们可以添加 varDecl 的定义：

    varDecl        → "var" IDENTIFIER ( "=" expression )? ";" ;

为了访问变量的值，添加一类新的 primary expression：IDENTIFIER。

    primary        → "true" | "false" | "nil"
    | NUMBER | STRING
    | "(" expression ")"
    | IDENTIFIER ;

declaration 对于 Parser 来说，永远是优先级最高的解析方式，所以可以在这个位置处理程序异常，报错并丢弃异常代码行，让解释器继续向下运行。

# [Environment](https://craftinginterpreters.com/statements-and-state.html#environments)

为了存储变量名和值的映射，我们需要一个数据结构，一般它被称为 Environment。

当我们编写变量寻址逻辑时，我们需要回答两个问题：

1.	如果用户重复使用 var 关键字声明变量，我们应该如何处理？是报错阻止用户，还是允许这种行为？简单起见，我们可以允许这种行为。
2.	如果用户访问了一个未定义的变量，我们应该如何处理？这里，我们可以返回一个默认值空，或是返回一个SyntaxError，或是返回一个RuntimeError。默认值空看上去不是什么好主意，在语法检查时抛出SyntaxError会让声明相互引用的递归函数变得困难，所以我们选择在访问时抛出RuntimeError。

# [Assignment syntax](https://craftinginterpreters.com/statements-and-state.html#assignment-syntax)

为了赋值，我们需要添加一类表达式。

    expression     → assignment ;
    assignment     → IDENTIFIER "=" assignment
    | equality ;

当我们解析赋值语句时，我们将等号两边区分为左值和右值，处理两者的逻辑有着一些差别，首先就体现在赋值语句的 AST 定义上：

    "Assign   : Token name, Expr value",

当我们解析赋值语句时，我们将左值和右值都当作一个普通表达式。如果我们发现左值是一个变量，那我们就可以生成一个赋值语句，否则将其直接返回。

与其他二元运算符不同，赋值运算符是右结合的，所以我们并不使用循环来反复解析语句，而是使用了递归来解析右值当中的赋值语句。

# [Scope](https://craftinginterpreters.com/statements-and-state.html#scope)

作用域有两种，一种被称为词法作用域（Lexical scope）或者静态作用域（static scope），这种情况只需静态阅读代码即可知道该变量在何处声明；另一种则是动态作用域（dynamic scope），需要在运行时才能清楚地知道变量具体的值是什么。词法作用域是经典的，不需要过多解释。动态作用域一般常见于对象的方法或属性上，例如：

    class Saxophone {
        play() {
            print "Careless Whisper";
        }
    }

    class GolfClub {
        play() {
            print "Fore!";
        }
    }

    fun playIt(thing) {
        thing.play();
    }

这里没法提前知道 thing 指的是 GolfClub 还是 Saxophone，只有在调用时才能知道。

Scope 和 envirnment 是一体两面，前者是概念上的名词，后者是实现机制。代码工作时，语法树节点会改变环境，例如我们可以使用大括号来分隔内外变量的作用域。

# [Nesting and shadowing](https://craftinginterpreters.com/statements-and-state.html#nesting-and-shadowing)

内部作用域声明了和外部作用域相同的名字的变量，这种情况下，外部的变量应该被隐藏起来，但不受内部变量的赋值和操作影响到。这种情况被称为 shadowing。而内部作用域当然也可以访问外部的变量，当没有触发隐藏机制时，内外作用域应该互相嵌套。

实现这种机制的方式是将环境链接到一起，访问时，从内向外寻址。内层是局部作用域，越外层作用域越大，直到顶层。

# [Block syntax and semantics](https://craftinginterpreters.com/statements-and-state.html#block-syntax-and-semantics)

为了实现环境嵌套的效果，需要让语言支持 block 语法。
    statement      → exprStmt
                    | printStmt
                    | block ;

    block          → "{" declaration* "}" ;
    
对于 block 来说，他就是一系列语句的集合。

# [Conditional Execution](https://craftinginterpreters.com/control-flow.html#conditional-execution)

通常来说，控制流大致分为两种：
–	条件或分支控制流用于不执行某些代码；
–	循环控制流用于多次执行相同的代码。

简单起见，我们不实现三元条件运算符，只是添加一个 if - else 关键字。

    statement      → exprStmt
                    | ifStmt
                    | printStmt
                    | block ;
    ifStmt         → "if" "(" expression ")" statement
                    ( "else" statement )? ;

通常，if 语句由一个条件表达式，两个语句组成。

# [Logical Operators](https://craftinginterpreters.com/control-flow.html#logical-operators)

我们需要特殊处理逻辑运算符，因为他们具有特殊的短路机制：如果通过左侧的值就可以知道整个表达式的值，我们就不会继续计算后面的数据了。我们需要为逻辑或、逻辑且两个运算符添加专门的语法描述：

    expression     → assignment ;
    assignment     → IDENTIFIER "=" assignment
                    | logic_or ;
    logic_or       → logic_and ( "or" logic_and )* ;
    logic_and      → equality ( "and" equality )* ;

我们并没有直接在 logic_or 情况下使用 equality 或者 assignment，而是使用了 logic_and。

# [While Loops](https://craftinginterpreters.com/control-flow.html#while-loops)

    statement      → exprStmt
                    | ifStmt
                    | printStmt
                    | whileStmt
                    | block ;
    whileStmt      → "while" "(" expression ")" statement ;

while 循环的实现非常容易，只需要借助 Java 的 while 直接调用即可。

# [For Loops](https://craftinginterpreters.com/control-flow.html#for-loops)

    statement      → exprStmt
                    | forStmt
                    | ifStmt
                    | printStmt
                    | whileStmt
                    | block ;
    forStmt        → "for" "(" ( varDecl | exprStmt | ";" )
                    expression? ";"
                    expression? ")" statement ;

for 循环是完全可以像 while 循环一样使用的，所以括号里的所有表达式都可以为空。

1.	第一个子句的工作是初始化，通常是个表达式，也允许声明变量。
2.	第二个子句是条件，与 while 一致。
3.	最后一个子句是累加，在每次循环迭代结束时做一些副作用工作。

for 语句是至今为止最复杂的语句，不过没有使用到任何非已有的工具。我们可以将其看作一个 while 语句的语法糖，所以，处理它的工序被称为脱糖（desugaring）。具体的流程是，将子句拆分开之后，将 for 循环体和 increment 组成一个 block，将条件和循环 block 组成 while 语句，将 while 和初始化子句组成 body。

# [Function Calls](https://craftinginterpreters.com/functions.html#function-calls)

函数调用的语法一般是：average(1, 2);，但 callee 事实上可以是任何返回了函数的表达式，所以也可以是这样：getCallback()()。所以我们对函数调用的语法定义如下：

    unary          → ( "!" | "-" ) unary | call ;
    call           → primary ( "(" arguments? ")" )* ;
    arguments      → expression ( "," expression )* ;

一个表达式后面可以跟着0个或多个调用括号和参数列表，如果是0个，那就是一个普通表达式。

为了保存函数出错时调用函数的位置，我们存储调用表达式时，将闭合括号一起存储到表达式当中：

    "Call     : Expr callee, Token paren, List<Expr> arguments",
# [Maximum argument counts](https://craftinginterpreters.com/functions.html#maximum-argument-counts)

许多语言会限制函数调用参数列表的长度，例如 C 语言要求支持至少 127 个参数，Java 要求支持不多于 255 个参数。为了与后续的字节码解释器保持一致，我们这边也添加 255 个参数的数量限制。

# [Interpreting function calls](https://craftinginterpreters.com/functions.html#interpreting-function-calls)

可能比较令人困惑的一点是，我们在实现函数之前实现了调用模块。不过不太重要。在调用函数时，我们首先计算 callee，然后依次计算参数列表中每个参数的值，然后将这些准备工作存放到一个专门的 LoxCallable 对象当中。

当然，我们还需要对 callee 的类型做检查，并及时抛出一个 RuntimeError。

# [Native Functions](https://craftinginterpreters.com/functions.html#native-functions)

原生方法对程序语言来说非常重要，用户大量与操作系统交互的操作都依赖语言提供的原生方法。这里，我们实现方法主要依赖 Java 内置的一些原生函数。例如调用系统时钟时，会使用System.currentTimeMillis。

# [Function Declarations](https://craftinginterpreters.com/functions.html#function-declarations)

这里，我们给函数定义添加语法说明：

    declaration    → funDecl
                    | varDecl
                    | statement ;
    funDecl        → "fun" function ;
    function       → IDENTIFIER "(" parameters? ")" block ;
    parameters     → IDENTIFIER ( "," IDENTIFIER )* ;

有了这个，我们就可以声明函数了。

对于函数来说，核心是两个部分，参数列表和代码块。实现函数调用时的操作其实非常简单，就是按照参数列表的定义顺序，将表达式一一求值，然后在新的envirnment中定义一遍，而后在新的envirnment中执行代码块。

# [Return Statements](https://craftinginterpreters.com/functions.html#return-statements)

目前来看，我们的函数还不能带回任何数据，所以我们需要添加返回语句。

statement      → exprStmt
                | forStmt
                | ifStmt
                | printStmt
                | returnStmt
                | whileStmt
                | block ;
returnStmt     → "return" expression? ";" ;

同时，事实上任何一个 Lox function 都需要返回一些东西，即使它并没有返回语句。也就是默认返回 nil 值。当我们实现 Return 语句时，我们会发现，他的特性很像是异常：他会一下子打破函数调用过程当中的所有调用链，并结束函数调用 LoxFunction.call，使其返回结果。因此，我们使用 throw 来实现它。

# [Local Functions and Closures](https://craftinginterpreters.com/functions.html#local-functions-and-closures)

我们的函数目前还存在一个漏洞，也就是不支持闭包。前面实现的函数作用域，直接关联了 global 作用域，但目前我们支持在任何一个位置的局部作用域声明函数，这会导致局部函数的闭包和递归无法正常工作。要修复这个问题也很简单，添加一个闭包参数，将定义时的环境传入即可。