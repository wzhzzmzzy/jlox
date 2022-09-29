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

