---
tags:
  - 代码审计
  - myblog
title: codeql基础语法学习
---
csdn上阅读体验更加，这个目录太长这里用着不太舒服
https://blog.csdn.net/qq_72685284/article/details/144118176

网上对于codeql的基础讲解的比较少，很多都是直接从codeql for java 或者直接拿一个靶场开始练习的。我通过对官方文档的一些翻译和加上自己的一些个人的理解和丰富合适的例子。帮助新人对codeql有一个基础的认识。因为好多英文名词并没有比较统一的翻译，所以我这里会标注一些关键的英文词的中文含义，很多意思还需要大家从英文的原文中体会其中的含义，同时方便大家结合文档进行查看。

参考文档。

https://codeql.github.com/docs/ql-language-reference/about-the-ql-language/

# declarative languages

声明式(declarative)是结果导向的，命令式(imperative)是过程导向的。它们都有自己适用的场景和局限，于是现实中的编程语言常常都有两者的身影。

declarative 案例 ： SQL, HTML ,codeql

imperative 案例 ： C, C++, Java

>QL is a declarative, object-oriented(面向对象的) query language that is optimized(优化) to enable efficient analysis of hierarchical data structures, in particular, databases representing software artifacts.
>
>The syntax(语法) of QL is similar to SQL, but the semantics(语意) of QL are based on Datalog, a declarative logic programming language often used as a query language. This makes QL primarily a logic language, and all operations in QL are logical operations. Furthermore, QL inherits recursive(递归) predicates(谓语) from Datalog, and adds support for aggregates(聚合), making even complex queries concise(简洁) and simple. 

总之就是codeql的语法和常规的编程语言不太一样，他是基于datalog的，不能拿C,java 那类语言去类比，有很多的概念和专业用语都是不太一样的。

> Object orientation(面向对象) is an important feature of QL. The benefits of object orientation are well known – it increases modularity(模块化), enables information hiding, and allows code reuse(复用). QL offers all these benefits without compromising(妥协) on its logical foundation. This is achieved by defining a simple object model where classes are modeled as predicates and inheritance as implication. The libraries made available for all supported languages make extensive use of classes and inheritance.

与sql相比，codeql的语言功能更加丰富，增加了面相对象的特性。可以自定义谓词。

# 谓词Properties

谓词用于描述构成 QL 程序的逻辑关系。

严格来说，谓词求值为一组元组。例如，考虑以下两个谓词定义：

```java
predicate isCountry(string country) {
  country = "Germany"
  or
  country = "Belgium"
  or
  country = "France"
}

predicate hasCapital(string country, string capital) {
  country = "Belgium" and capital = "Brussels"
  or
  country = "Germany" and capital = "Berlin"
  or
  country = "France" and capital = "Paris"
}

from string country, string capital
where hasCapital(country, capital) and country = "Germany"
select country, capital
```

谓词 isCountry 返回的是一组一元组 {("Belgium"),("Germany"),("France")}，而 hasCapital 是一组二元组 {("Belgium","Brussels"),("Germany","Berlin"),("France","Paris")}。这些谓词的元数分别为 1 和 2。

通常，谓词中的所有元组都具有相同数量的元素。谓词的 arity 就是元素的数量，不包括可能的结果变量。

QL 中有许多内置谓词。您可以在任何查询中使用这些谓词，而无需导入任何其他模块。除了这些内置谓词之外，您还可以定义自己的谓词：

## 谓词组成

- 谓词的关键字predicate，或者谓词的返回值。（谓词可以分为without result 和 with result）
- 谓词名词
- 谓词参数
- 谓词主体

## without result

开头那两个谓词也是 without result 的

```java
predicate isSmall(int i) {
    i in [1 .. 9]
}

from int i
where isSmall(i)
select i
```

## with result

您可以通过将关键字 predicate 替换为结果类型来定义带有结果的谓词。这引入了特殊变量 result。result的值就是我们的返回值。

这个from就是定义一个变量。

```java
int getSuccessor(int i) {
	result = i + 1 and
	i in [1 .. 9]
}

from int x
select x,getSuccessor(x)
```

可以在谓词主体中调用其他的谓词

```java
Person getAChildOf(Person p) {
  p = getAParentOf(result)
}
```

谓词还可能对其参数的每个值产生多个结果（或根本没有结果）

```java
string getANeighbor(string country) {
	country = "France" and result = "Belgium"
	or
	country = "France" and result = "Germany"
	or
	country = "Germany" and result = "Austria"
	or
	country = "Germany" and result = "Belgium"
}

select getANeighbor("France")
```

## Recursive(递归) predicates

```java
string getANeighbor(string country) {
  country = "France" and result = "Belgium"
  or
  country = "France" and result = "Germany"
  or
  country = "Germany" and result = "Austria"
  or
  country = "Germany" and result = "Belgium"
  or
  country = getANeighbor(result)
}
```

## Kinds(种类) of predicates

谓词有三种类型，即Non-member谓词、member谓词和Characteristic谓词。

非成员谓词是在类之外定义的，也就是说，它们不是任何类的成员。

```java
int getSuccessor(int i) {  // 1. Non-member predicate
  result = i + 1 and
  i in [1 .. 9]
}

class FavoriteNumbers extends int {
  FavoriteNumbers() {  // 2. Characteristic predicate
    this = 1 or  //特征谓词是类的构造函数中定义的谓词，用来描述类的成员资格（membership criteria）。它决定了一个值是否属于某个类。
    this = 4 or
    this = 9
  }

  string getName() {   // 3. Member predicate for the class `FavoriteNumbers`
    this = 1 and result = "one"
    or
    this = 4 and result = "four"
    or
    this = 9 and result = "nine"
  }
}

from FavoriteNumbers fn, string name
where fn = 4 and fn.getName() = name  // 成员谓词只能在类的实例上应用
select name
```

## Binding behavior

It must be possible to evaluate a predicate in a finite(有限的) amount of time, so the set it describes is not usually allowed to be infinite(无限的). In other words, a predicate can only contain a finite number of tuples.

这里的无限其实也不是无穷大或者无穷多。上面和下面的那些反例其实也就是多了一点判断条件。

```java
int getSuccessor(int i) {
	result = i + 1 and
	i in [1 .. 9]
}
```

```java
/*
  Compilation errors:
  ERROR: "i" is not bound to a value.
  ERROR: "result" is not bound to a value.
  ERROR: expression "i * 4" is not bound to a value.
*/
int multiplyBy4(int i) {
  result = i * 4
}

/*
  Compilation errors:
  ERROR: "str" is not bound to a value.
  ERROR: expression "str.length()" is not bound to a value.
*/
predicate shortString(string str) {
  str.length() < 10
}
```

### Binding sets

如果一定要用infinite的谓词。

在这种情况下，您可以使用 bindingset 注释指定显式绑定集。此注释适用于任何类型的谓词。

```java
bindingset[i]
int multiplyBy4(int i) {
  result = i * 4
}

from int i
where i in [1 .. 10]
select multiplyBy4(i)
```

也可以多个，这种形式的意思是。x,y至少有一个是bound的。和bindingset[x, y]不同，意味着两个都是bound的

- If `x` is bound, then `x` and `y` are bound.
- If `y` is bound, then `x` and `y` are bound.

```java
bindingset[x] bindingset[y]
predicate plusOne(int x, int y) {
  x + 1 = y
}

from int x, int y
where y = 42 and plusOne(x, y)
select x, y
```

后者可以用于两种不同类型的情况

```java
bindingset[str, len]
string truncate(string str, int len) {
  if str.length() > len
  then result = str.prefix(len)
  else result = str
}
```



# 查询Queries

```sql
from /* ... 声明变量 ... */
where /* ... logical formula ... */
select /* ... expressions ... */
```

除了“表达式”中描述的表达式之外，您还可以包括：
as 关键字，后跟名称。这为结果列提供了“标签”，并允许您在后续的选择表达式中使用它们。
order by 关键字，后跟结果列的名称，以及可选的关键字 asc 或 desc。这决定了显示结果的顺序。

from 和 where 部分是可选的。

除了“表达式”中描述的表达式之外，您还可以包括：

- as 关键字，后跟名称。这为结果列提供了“标签”，并允许您在后续的选择表达式中使用它们。
- order by 关键字，后跟结果列的名称，以及可选的关键字 asc 或 desc。这决定了显示结果的顺序。

这些和sql都差不多，不同地方在于是，这里的from可以声明变量。

例子

```sql
from int x, int y
where x = 3 and y in [0 .. 2]
select x, y, x * y as product, "product: " + product
```

## Query predicates

```java
int getProduct(int x, int y) {
    x = 3 and
    y in [0 .. 2] and
    result = x * y
}

from int x, int y
where x = 3 and y = 1 
select x, y,getProduct(x, y)
```

这个查询谓语返回以下结果

| x    | y    | result |
| :--- | :--- | :----- |
| 3    | 0    | 0      |
| 3    | 1    | 3      |
| 3    | 2    | 6      |

编写查询谓词而不是 select 子句的一个好处是，您也可以在代码的其他部分调用该谓词。相比之下，select 子句就像一个匿名谓词，因此您无法稍后调用它。

# Types

## Primitive types

有以下这些类型。

boolean，float，int，string，date。

QL 在原始类型上定义了一系列内置操作，例如，1.toString() 是整数常量 1 的字符串表示形式。有关 QL 中可用的内置操作的完整列表，请参阅 QL 语言规范中的内置部分。

https://codeql.github.com/docs/ql-language-reference/ql-language-specification/#built-ins

> 此外，QlBuiltins::BigInt 中还有一个实验性的任意精度整数原始类型。默认情况下，此类型在 CodeQL CLI 中不可用，但可以通过将 --allow-experimental=bigint 选项传 来启用它。

## Classes

### 定义一个类

1. class 关键字
2. class 名，要求首字母大写
3. 通过 extends和instanceof定义的supertypes
4. 括号括起来的类体

```java
class OneTwoThree extends int {
  OneTwoThree() { // characteristic predicate
    this = 1 or this = 2 or this = 3
  }

  string getAString() { // member predicate
    result = "One, two or three: " + this.toString()
  }

  predicate isEven() { // member predicate
    this = 2
  }
}
```

定义了一个类 OneTwoThree，它包含值 1 2 3。

OneTwoThree 继承了 int，也就是说，它是 int 的子类。QL 中的类必须始终具有至少一个supertypes。使用 extends 关键字引用的supertypes称为**base types** of the class。

### Class bodies

The body of a class can contain:
- A characteristic predicate declaration.  // 前面讲 谓词种类 的时候都提到了 #Kinds(种类) of predicates
- Any number of member predicate declarations. 
- Any number of field declarations.

#### Characteristic predicates

相当于构造方法。前面提到过

#### Member predicates

可以用这种方法调用一个member predicate

```
(OneTwoThree).getAString()
```

#### Fields

在body of a class中进行声明，例子如下。

```java
class SmallInt extends int {
  SmallInt() { this = [1 .. 10] }
}

class DivisibleInt extends SmallInt {
  SmallInt divisor;   // declaration of the field `divisor`
  DivisibleInt() { this % divisor = 0 }

  SmallInt getADivisor() { result = divisor }
}

from DivisibleInt i
select i, i.getADivisor()
```

### Concrete classes

比如上面那个 DivisibleInt 有一个  SmallInt 的 Fields

下面这些就和面向对象的语言很相似，看一下名字就知道是什么意思。不细讲。

### Abstract classes

抽象类

### Overriding member predicates

重写父类谓词

### Multiple inheritance

多继承

### Final extensions

类似于final关键字。

### Non-extending subtypes

其实就是有点像接口，下面这个会报错。instanceof 声明的超类型中的字段和方法不会成为子类的一部分

```java
class Foo extends int {
    Foo() { this in [1 .. 10] }
  
    string fooMethod() { result = "foo" }
  }
  
  class Bar instanceof Foo {
    string toString() { result = super.fooMethod() }
}

select any(Bar b).fooMethod()
```

```java
class Interface extends int {
    Interface() { this in [1 .. 10] }
    string foo() { result = "" }
  }
  
  class Foo extends int {
    Foo() { this in [1 .. 5] }
    string foo() { result = "foo" }
  }
  
  class Bar extends Interface instanceof Foo {
    override string foo() { result = "bar" }
}

select any(Bar b).foo()  // 返回 bar
// select any(Foo f).foo()   // 返回foo
```

## 

# Modules
之类就给一个例子吧，太复杂的官方也没有给例子，这就是好多语言都有的一个东西。

**OneTwoThreeLib.qll**
```java
class OneTwoThree extends int {
    OneTwoThree() {
      this = 1 or this = 2 or this = 3
    }
  }


module M {
class OneTwo extends OneTwoThree {
    OneTwo() {
	    this = 1 or this = 2
	    }
	}
}

```

**OneTwoQuery.ql**
```java
import OneTwoThreeLib

from OneTwoThree ott
where ott = 1 or ott = 2
select ott
```

在对chat的逼问之下，还是把这个调用方法给问出来了。
**OneTwoQuery2.ql**
```java
// 导入库文件
import OneTwoThreeLib

// 使用 M 模块中的 OneTwo 类

from M::OneTwo oneTwoInstance

select oneTwoInstance, "Instance of OneTwo"
```

# Signatures

这个概念其实比较抽象。我暂时不想详细的写。这里就直接给官网的例子了。但是手边暂时没有具体使用的案例。

## Predicate signatures

注意这个分号

```java
signature int operator(int lhs, int rhs);
```

## Type signatures

```java
signature class ExtendsInt extends int;

signature class CanBePrinted {
  string toString();
}
```

## Module signatures

```java
signature module MSig {
  class T;
  predicate restriction(T t);
  default string descr(T t) { result = "default" }
}

module Module implements MSig {
  newtype T = A() or B();

  predicate restriction(T t) { t = A() }
}
```

### Parameterized module signatures

```java
signature class NodeSig;

signature module EdgeSig<NodeSig Node> {
  predicate apply(Node src, Node dst);
}

module Reachability<NodeSig Node, EdgeSig<Node> Edge> {
  Node reachableFrom(Node src) {
    Edge::apply+(src, result)
  }
}
```

# Aliases

## Defining an alias

### Module aliases

```java
module ModAlias = ModuleName;
```

下面这个会弃用oldversion。

```java
deprecated module OldVersion = NewVersion;
```

### Type aliases

```java
class TypeAlias = TypeName;
```

你可以使用别名将基本类型boolean的名称缩写为Bool：

```java
class Bool = boolean;
```

在OneTwoThreeLib中使用模块M中定义的类OneTwoThreeLib.qll，你可以创建一个别名来使用更短的名称OT

```java
import OneTwoThreeLib

class OT = M::OneTwo;

...

from OT ot
select ot
```

### Predicate aliases

```java
int getSuccessor(int i) {
  result = i + 1 and
  i in [1 .. 9]
}
```

可以为这个谓语设置一个这样的别名。

```java
predicate succ = getSuccessor/1;
```

### Strong and weak aliases

有annotation 注解的是 Strong aliases。来自相同module/type/predicate的weak aliases定义之间的别名歧义是允许的，但来自不同Strong aliases定义之间的别名歧义是无效的QL。

> Every alias definition is either **strong** or **weak**. An alias definition is **strong** if and only if it is a [type alias](https://codeql.github.com/docs/ql-language-reference/aliases/#type-aliases) definition with [annotation](https://codeql.github.com/docs/ql-language-reference/annotations/#annotations) `final`. During [name resolution](https://codeql.github.com/docs/ql-language-reference/name-resolution/#name-resolution), ambiguity between aliases from **weak** alias definitions for the same module/type/predicate is allowed, but ambiguity between between aliases from distinct **strong** alias definitions is invalid QL. Likewise, for the purpose of applicative instantiation of [parameterised modules](https://codeql.github.com/docs/ql-language-reference/modules/#parameterized-modules) and :ref:`parameterised module signatures <parameterized-module-signatures>, aliases from **weak** alias definitions for instantiation arguments do not result in separate instantiations, but aliases from **strong** alias definitions for instantiation arguments do.

# Variables

## Declaring a variable

其实前面就一只在遇见了。声明，赋值，输出。

```java
from int i
where i = 10
select i
  
from int i
where i in [0 .. 9]
select i
```

## Free and bound variables

```java
"hello".indexOf("l")
min(float f | f in [-3 .. 3])
(i + 7) * 3
x.sqrt()
```

第一个没有 variables  ，值是2

第二个**bound variable**  ，值恒为3

第三个**free variables** ，i 的值影响着整体的值。表达式是否成立取决于i的值。

```java
"hello".indexOf("l") = 1
min(float f | f in [-3 .. 3]) = -3
(i + 7) * 3 instanceof int
exists(float y | x.sqrt() = y)
```

第一个永远不成立。

第二个永远成立

第三个可能成立

第四个单y为负数时永远不成立

# Expressions

## Literals

Integer literals

```
0
42
-2048
```

Float literals

```
2.0
123.456
-100.5
```

等等

## Parenthesized(括号) expressions

## Ranges

其实前面也见到一些了

`[3 .. 7]`表示的是3到7的整数

## Set literal expressions

`[2, 3, 5, 7, 11, 13, 17, 19, 23, 29]` 是一个集合参数

## Super expressions

这个直接看例子，其实就是super.  然后找父类。

```java
class A extends int {
  A() { this = 1 }
  int getANumber() { result = 2 }
}

class B extends int {
  B() { this = 1 }
  int getANumber() { result = 3 }
}

class C extends A, B {
  // Need to define `int getANumber()`; otherwise it would be ambiguous
  override int getANumber() {
    result = B.super.getANumber()
  }
}

from C c
select c, c.getANumber()
```

## Calls to predicates

调用类的方法

例如 a.getAChild() 是调用的 a 的谓词 getAChild()

## Aggregations

```
<aggregate>(<variable declarations> | <formula> | <expression>)
```

```java
// 内容超过500行的文件
count(File f | f.getTotalNumberOfLines() > 500 | f)   
// 寻找函数最多的js文件。  
max(File f | f.getExtension() = "js" | f.getBaseName() order by f.getTotalNumberOfLines(), f.getNumberOfLinesOfCode())
// 逐个字符比较 返回 De Morgan
min(string s | s = "Tarski" or s = "Dedekind" or s = "De Morgan" | s)
// 返回36，等等吧，还有很多。
sum(int i, int j | i = [0 .. 2] and j = [3 .. 5] | i * j)
```

### Evaluation of aggregates

以这个为例

```java
select sum(int i, int j |
    exists(string s | s = "hello".charAt(i)) and 
    exists(string s | s = "world!".charAt(j)) 
    | i)
```

步骤

1. **确定输入变量**：这是聚合表达式中声明的变量，包括在聚合内声明的变量和在聚合外部使用的变量。

2. **生成所有可能的元组（组合）**：这些元组是输入变量的所有可能值的组合，必须满足给定的条件公式。

3. **应用 <expression>**：对每个元组应用<expression>，并收集生成的值（可能有多个不同的值）。

4. **应用 aggregates function**：使用 aggregates function（如 sum、count 等）对第3步中生成的值进行处理，计算最终结果。

**1. 确定输入变量：**

•输入变量是 i 和 j，分别代表 "hello" 和 "world!" 中的字符位置。

**2. 生成所有可能的元组：**

•我们通过 exists(string s | s = "hello".charAt(i)) 和 exists(string s | s = "world!".charAt(j)) 来生成所有可能的 (i, j) 对，表示字符串 "hello" 和 "world!" 中字符的位置。

•"hello" 有 5 个字符（0, 1, 2, 3, 4），而 "world!" 有 6 个字符（0, 1, 2, 3, 4, 5）。

•所以所有可能的 (i, j) 对是：

```
(0, 0), (0, 1), (0, 2), (0, 3), (0, 4), (0, 5),
(1, 0), (1, 1), (1, 2), (1, 3), (1, 4), (1, 5),
(2, 0), (2, 1), (2, 2), (2, 3), (2, 4), (2, 5),
(3, 0), (3, 1), (3, 2), (3, 3), (3, 4), (3, 5),
(4, 0), (4, 1), (4, 2), (4, 3), (4, 4), (4, 5)
```

**3. 应用<expression>：**

•在这个查询中，聚合表达式是 i。我们从每个元组中选择 i 的值。

•例如，所有 30 个元组中，i 的值会分别为：

`0, 0, 0, 0, 0, 0, 1, 1, 1, 1, 1, 1, 2, 2, 2, 2, 2, 2, 3, 3, 3, 3, 3, 3, 4, 4, 4, 4, 4, 4`

**4. 应用 aggregates function：**

`0 + 0 + 0 + 0 + 0 + 0 + 1 + 1 + 1 + 1 + 1 + 1 + 2 + 2 + 2 + 2 + 2 + 2 + 3 + 3 + 3 + 3 + 3 + 3 + 4 + 4 + 4 + 4 + 4 + 4`

值为60。

如果为这种

```java
select sum(int i, int j |
    exists(string s | s = "hello".charAt(i)) and 
    exists(string s | s = "world!".charAt(j)) 
    | i+j)
```

```
0 + 0 = 0
0 + 1 = 1
0 + 2 = 2
...
4 + 5 = 9
```

值为135

按照那4步来就行。

### Omitting parts of an aggregation

其实有些地方可以适当简写。

1. **省略 <variable declarations> 和 <formula>**

```
<aggregate>(<type> v | <expression> = v | v)
简写成
<aggregate>(<expression>)
```

例如

```java
count(int i | i = "hello".indexOf("l") | i)
count("hello".indexOf("l"))
```

• "hello".indexOf("l") 直接返回所有 "l" 的索引位置（2, 3）。

• count 统计这些索引的个数，因此两种写法等价。

2. **省略 <expression>**

如果只有一个aggregation variable，则可以省略 aggregation variable，此时表达式默认为该变量本身。

```
avg(int i | i = [0 .. 3] | i)
avg(int i | i = [0 .. 3])
```

3. 在 count 中，即使有多个aggregation variable，也可以省略 <expression>，此时表达式默认为常量 1，即统计满足条件的所有元组数量。

```java
count(int i, int j | i in [1 .. 3] and j in [1 .. 3] | 1)
count(int i, int j | i in [1 .. 3] and j in [1 .. 3])
```

4. **省略 <formula>，仅保留两个竖线 ||**

```
<aggregate>(<variable declarations> | | <expression>)
```

```java
max(File f | | f.getTotalNumberOfLines())
```

这段代码的意思是：统计数据库中所有文件的最大行数。

5. **省略 <formula> 和 <expression>**

```java
count(File f | any() | 1)
count(File f | | 1)
count(File f)
```

count(File f) 直接统计数据库中的文件数量。

### Monotonic(单调) aggregates

直接看官方给的例子吧。

```java
string getPerson() { result = "Alice" or
                     result = "Bob" or
                     result = "Charles" or
                     result = "Diane"
                   }
string getFruit(string p) { p = "Alice"   and result = "Orange" or
                            p = "Alice"   and result = "Apple" or
                            p = "Bob"     and result = "Apple" or
                            p = "Charles" and result = "Apple" or
                            p = "Charles" and result = "Banana"
                          }
int getPrice(string f) { f = "Apple"  and result = 100 or
                         f = "Orange" and result = 100 or
                         f = "Orange" and result =   1
                       }

predicate nonmono(string p, int cost) {
  p = getPerson() and cost = sum(string f | f = getFruit(p) | getPrice(f))
}

language[monotonicAggregates]
predicate mono(string p, int cost) {
  p = getPerson() and cost = sum(string f | f = getFruit(p) | getPrice(f))
}

from string variant, string person, int cost
where variant = "default"  and nonmono(person, cost) or
      variant = "monotonic" and mono(person, cost)
select variant, person, cost
order by variant, person
```

| variant   | person  | cost |
| :-------- | :------ | :--- |
| default   | Alice   | 201  |
| default   | Bob     | 100  |
| default   | Charles | 100  |
| default   | Diane   | 0    |
| monotonic | Alice   | 101  |
| monotonic | Alice   | 200  |
| monotonic | Bob     | 100  |
| monotonic | Diane   | 0    |

**标准聚合** 和 **单调聚合** 在处理公式 <formula> 和表达式 <expression> 的值时有如下不同：

**标准聚合：**

•对每个由 <formula> 生成的值，计算对应的 <expression> 值，**将它们展平为一个列表**。

•然后对这个列表应用聚合函数，例如 sum、count 等。

**单调聚合：**

•对每个由 <formula> 生成的值，计算对应的 <expression> 值，**创建所有可能的组合**。

•对每种组合分别应用聚合函数。

**结果差异：**

•**标准聚合** 通常返回一个结果，表示所有值的总和、计数等。

•**单调聚合** 会返回多行结果，表示每种可能组合的聚合值。

**场景 1: 缺少 <expression> 值**

•如果 <formula> 生成的某个值没有对应的 <expression> 值：

•**标准聚合** 会忽略这个缺失值，计算其他值的结果。

•**单调聚合** 不会计算结果，因为缺失值使得无法创建完整的组合。

**场景 2: 多个 <expression> 值**

•如果 <formula> 生成的某个值有多个对应的 <expression> 值：

•**标准聚合** 将所有 <expression> 值展平成一个列表，计算一个结果。

•**单调聚合** 会生成多种组合，对每种组合分别计算结果。

#### Recursive monotonic aggregates

暂时不讲解

## Any

| Expression                          | Values                                      |
| :---------------------------------- | :------------------------------------------ |
| `any(File f)`                       | all `File`s in the database                 |
| `any(Element e | e.getName())`      | the names of all `Element`s in the database |
| `any(int i | i = [0 .. 3])`         | the integers `0`, `1`, `2`, and `3`         |
| `any(int i | i = [0 .. 3] | i * i)` | the integers `0`, `1`, `4`, and `9`         |

## Unary operations

```
-6.28
+(10 - 4)
+avg(float f | f = 3.4 or f = -9.8)
-sum(int i | i in [0 .. 9] | i * i)
```

## Binary operations

```
5 % 2
(9 + 1) / (-2)
"Q" + "L"
2 * min(float f | f in [-3 .. 3])
```

## Casts(类型转换)

```java
import java

from Type t
where t.(Class).getASupertype().hasName("List")
select t
```

## Don’t-care expressions

其实这个符号在其它语言也经常见

```java
from string s
where s = "hello".charAt(_)
select s
```

# Formulas 公式

## Comparisons 比较

这里提一下。
就是一个等号在这里代表的就是Equal to
Not equal to就是 !=
定义的话用的是from

## Type checks

类型检查，和java里面的instanceof功能差不多

`<expression> instanceof <type>`

## Range checks
检查范围
A range check is a formula that looks like:
`<expression> in <range>`

`x in [2.1 .. 10.5]` 

## Calls to predicates

谓词调用。

## Parenthesized formulas

杯括号包裹起来的 formulas


### Explicit quantifiers显示量词

#### `exists`

都很好理解。

This quantifier has the following syntax:

```
exists(<variable declarations> | <formula>)
```

You can also write `exists(<variable declarations> | <formula 1> | <formula 2>)`. This is equivalent to `exists(<variable declarations> | <formula 1> and <formula 2>)`.

This quantified formula introduces some new variables. It holds if there is at least one set of values that the variables could take to make the formula in the body true.

For example, `exists(int i | i instanceof OneTwoThree)` introduces a temporary variable of type `int` and holds if any value of that variable has type `OneTwoThree`.

#### `forall`

This quantifier has the following syntax:

```
forall(<variable declarations> | <formula 1> | <formula 2>)
```

`forall` introduces some new variables, and typically has two formulas in its body. It holds if `<formula 2>` holds for all values that `<formula 1>` holds for.

For example, `forall(int i | i instanceof OneTwoThree | i < 5)` holds if all integers that are in the class `OneTwoThree` are also less than `5`. In other words, if there is a value in `OneTwoThree` that is greater than or equal to `5`, then the formula doesn’t hold.

Note that `forall(<vars> | <formula 1> | <formula 2>)` is logically the same as `not exists(<vars> | <formula 1> | not <formula 2>)`.

#### `forex`

This quantifier has the following syntax:

```
forex(<variable declarations> | <formula 1> | <formula 2>)
```

This quantifier exists as a shorthand for:

```
forall(<vars> | <formula 1> | <formula 2>) and
exists(<vars> | <formula 1> | <formula 2>)
```

> In other words, `forex` works in a similar way to `forall`, except that it ensures that there is at least one value for which `<formula 1>` holds. To see why this is useful, note that the `forall` quantifier could hold trivially. For example, `forall(int i | i = 1 and i = 2 | i = 3)` holds: there are no integers `i` which are equal to both `1` and `2`, so the second part of the body `(i = 3)` holds for every integer for which the first part holds.
>
> Since this is often not the behavior that you want in a query, the `forex` quantifier is a useful shorthand.

### Implicit quantifiers

相当于我们上面提到的 Don’t-care expressions。

## Logical connectives

1. Negation (not)
2. Conditional formula (if…..then…else)
3. Conjunction (and)
4. Disjunction (or)
5. Implication (implies)

### `any()`

### `none()`

### `not`

……

# Annotations

有点像java里面的修饰符。

像 `abstract` `deprecated` `final`这些前面都提到过。

# Lexical syntax

## Comments

有两种注释方法。

```java
/**
 * A QLDoc comment that describes the class `Digit`.
 */
class Digit extends int {  // A short one-line comment
  Digit() {
    this in [0 .. 9]
  }
}

/*
  A standard multiline comment, perhaps to provide
  additional details, or to write a TODO comment.
*/
```

# Name resolution(解析)

## Names

处理一个name首先在当前模块的命名空间中查找名称。

如果是import语句。name resolution会更加复杂，看下面这个例子。

```java
import javascript
```

编译器首先检查库模块javascript.qll，再采取下面说到的那些步骤。如果失败，它会检查在Example.ql的模块命名空间中定义的名为javascript的explicit module。

## Qualified references

限定引用是一种模块表达式，使用 . 作为文件路径分隔符。它只能在 import 语句中使用，用于导入由相对路径定义的库模块。比如我们Example.ql有如下一个import语句。

```java
import examples.security.MyLibrary
```

**1. 当前目录查找** 编译器首先从包含当前 Example.ql 文件的目录中查找目标文件 examples/security/MyLibrary.ql

**2. 查询目录查找** 查找相对路径 examples/security/MyLibrary.qll。如果查询目录未配置或路径中未找到目标文件，继续下一步。查询目录是第一个包含qlpack.yml文件的目录。（或者，在老版本中，是一个名为queries.xml的文件。）

**3. 库路径查找 ** 查看qlpack.yml 文件中的 libraryPathDependencies 设置

**4. 查找失败** 如果还找不到编译会报错。

## Selections

```
<module_expression>::<name>
```

### Example

**CountriesLib.qll**

```java
class Countries extends string {
  Countries() {
    this = "Belgium"
    or
    this = "France"
    or
    this = "India"
  }
}

module M {
  class EuropeanCountries extends Countries {
    EuropeanCountries() {
      this = "Belgium"
      or
      this = "France"
    }
  }
}class Countries extends string {
  Countries() {
    this = "Belgium"
    or
    this = "France"
    or
    this = "India"
  }
}

module M {
  class EuropeanCountries extends Countries {
    EuropeanCountries() {
      this = "Belgium"
      or
      this = "France"
    }
  }
}
```

可以用如下方式使用

```java
import CountriesLib

from M::EuropeanCountries ec
select ec
```

```java
import CountriesLib::M

from EuropeanCountries ec
select ec
```

## Namespaces

命名空间。和其它语言比较像，不过多解释。

### Global namespaces

### Local namespaces

# QL language specification

语言规范太多了，这里就不细讲了。

https://codeql.github.com/docs/ql-language-reference/ql-language-specification/
