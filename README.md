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