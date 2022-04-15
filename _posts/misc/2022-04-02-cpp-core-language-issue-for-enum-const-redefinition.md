---
title: "记一次 C++ 核心语言标准中一个 issue 的发现和提交经历"
date: 2022-04-02 22:10:00 +0800
categories: [Language, C++]
tags: [c++ standard]
---

> 该文章记录自己的一次发现一个 C++ 核心语言标准规定中，关于枚举量重定义的一个规则缺陷（defect）并提交的经历。所有对标准的引用以 N4901 草案为准（当时的较新版本）。

[C++ 核心语言标准 N4901 草案](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2021/n4901.pdf)

# 引言

问题本身是关于 enum 中枚举值 (enumerator) 的重复定义问题的。

例子如下：

```c++
enum {
	ee, ee
};
```

上面的代码，无论是在 gcc/clang 还是 g++/clang++ 上，编译都是不能通过的，报错如下：
```
enumtest.cpp:3:2: error: redefinition of enumerator 'A'
        A,
        ^
enumtest.cpp:2:2: note: previous definition is here
        A,
        ^
1 error generated.
```

也就是常用的两个编译器实现上，无论 C 还是 C++ 都不允许枚举值的重复定义（注意区分枚举类型和枚举值）。

C99 草案中对该行为作出了明确规定：
> From the draft of C99, Section 6.7:  
> 6.7.5.  A declaration specifies the interpretation and attributes of a set of identifiers. A definition of an identifier is a declaration for that identifier that:  
> — for an object, causes storage to be reserved for that object;  
> — for a function, includes the function body;   
> — for an **enumeration constant or typedef name, is the (only) declaration of the identifier.**  

enumeration constant 即枚举常量，也就是上述代码中的 `ee`。C 语言标准的这一规定，就避免了同一个标识符被多次用于定义枚举值。在实际的使用中这一行为也符合逻辑，因为每一个枚举值在未指定具体常数值的情况下，是递增分配整形常数值的，如果允许枚举值 enumerator 同名可能导致一个枚举值名字对应多个常数值，造成歧义。

# 问题

按理来说，C++ 在大多数情况下都可以认为是 C 的超集，C 标准明确规定不能通过编译的代码，在 C++ 中应该也不能通过。由于枚举类型定义的时候，会顺带定义其中的所有枚举值，又因为定义是一种特殊的声明，那么 C++ 标准中就必然存在一定的规则，要么阻止枚举量的重复定义，要么阻止枚举量的重复声明，使得上述代码非法。

## One-definition rule 不阻止枚举量的重复定义

出于好奇，查找了一下 C++ 关于这方面的规定，了解到 C++ 中，有一个单独列出的 One-definition rule 条目（6.3 [basic.def.odr]），其中第一条规定如下：

> No translation unit shall contain more than one definition of any variable, function, class type, enumeration type, template, default argument for a parameter (for a function in a given scope), or default template argument.

即：所有的翻译单元都不可以包含多于一个的任何变量、函数、类、**枚举类型**、模版、参数默认值或默认模版参数的定义。

这里特别注意到，**One-definition rule 限制了「枚举类型」的重复定义，但是没有限制「枚举量」的重复定义！**

当然 One-definition rule 相当于是在原来的声明/定义规则上打的一个“补丁”，直接用特殊规则来限制一些种类实体的重复定义。并不代表标准中的其他规则就不会限制重复定义的枚举值的存在（这在后续与委员会的邮件交流中也涉及到了），所以这里没有限制并不足以作为允许枚举量重复定义的充分条件。

由于定义是一种特殊的声明，虽然定义 definition 相关的规则没能阻止例子中的代码通过编译，但是仍然有可能在声明 declaration 中阻止了这样重复声明枚举量的情况出现，故继续探寻，发现：

## declaration 声明规则允许枚举量重复声明

### 两次 `ee` 声明的是同一实体
继续浏览标准中关于声明 declaration 和定义 definition 的章节，可以看到如下描述：

6.6 [basic.link] paragraph 8:
> **Two declarations** of entities **declare the same entity if**, considering declarations of unnamed types to introduce their names for linkage purposes, if any (9.2.4, 9.7.1), they **correspond** (6.4.1), **have the same target scope** that is not a function or template parameter scope, and either
> - they **appear in the same translation unit**, or....... （后续几种情况与问题无关，故没有列出）

即两个实体声明（在这里指两次枚举量定义 `ee` 和 `ee`，定义也是一种声明）如果它们满足：
* 相互「对应」（例子满足）
* 在同一个作用域（例子满足）
* 且出现在同一个翻译单元（例子满足）

则他们声明的是同一个实体。

「对应 correspond」的意思，在6.4.1 [basic.scope.scope], paragraph 4 中指出：
> **Two declarations correspond if they (re)introduce the same name**, both declare constructors, or both declare destructors, unless......  
> （后面的排除情况比较复杂，但都不适用本文中的例子，故省略）

在这里，两个枚举量声明由于引入了同一个名字，所以说它们「对应」。

也就是说，他们满足了声明同一个实体的三个条件，**两次 `ee` 声明的是同一实体**！

> 这个规则一般是服务于函数声明、变量声明或者类型声明的，即多次声明同一个函数，声明的其实都是同一个函数：
> ```c++
> // 例子：此代码是合法C++程序，能通过编译
> void foobar();// 声明
> void foobar();// 再次声明
> void foobar() {} // 再次声明，并定义，前面所有的声明都指向这里定义的实体
> int main() {
> 	foobar();
> }
> ```
> 但是在这一套规则下，我们一开始例子中的枚举量定义 `ee` 和 `ee` 也恰好符合这里的要求，即两次指向同一个实体。

两次 `ee` 声明的是同一实体为什么重要呢？因为声明还有一种「冲突」的情况如下：

### 两次 `ee` 声明不满足「可能冲突」的条件，不造成冲突

6.4.1 [basic.scope.scope]:
> **Two declarations _potentially conflict_ if they correspond and cause their shared name to denote different entities (6.6)**. The program is ill-formed if, in any scope, a name is bound to two declarations that _potentially conflict_ and one precedes the other (6.5).

当两个声明「对应」（满足）并使得他们共有的名字**指向两个不同实体时**，两个声明即「可能冲突」。

而前面一段已经说明了，两次 `ee` 声明，指向的是**同一实体**，也就是说这里「可能冲突」的规则并不适用，两次声明不冲突。

## 结论：枚举量重复定义不违反 C++ 标准！

由于 One-definition rule 并不限制枚举量的重复定义，而两次枚举量声明指向的是同一个实体，不会造成声明冲突，意味着一开始的示例代码：

```c++
enum {
	ee, ee
};
```

其中重复的 `ee`，无论是从实体定义的角度，还是从实体声明的角度，都**不违反 C++ 标准中的规则**，也就是说当前 C++ 标准没有像 C 标准一样成功阻止该有歧义的程序通过编译！

# 总结

当然，对同一个名字进行多次枚举量定义肯定在逻辑上是错误的，每个枚举量都必须对应「一个」整型常量，每一个枚举量定义又会使得枚举量对应的常量相比上一个枚举量定义增1，允许同个名字定义两次枚举量的话，这两个规则就产生矛盾了。（例如上面的例子里，ee 同时对应 0 和 1）

在这里，合理的解决方法应该是将枚举值 (enumerator、enumeration constant) 加入到 One-definition rule 中，阻止枚举值的重复定义。

我也将相关的信息提交给了 C++ 标准委员会相关人员，并经过几轮邮件来回解释，该问题已经被接受并成为 [C++ 核心语言议题 #2530](http://www.open-std.org/jtc1/sc22/wg21/docs/cwg_active.html#2530)。应该会在下一次委员会会议中讨论并可能在未来草案中修复。
```
That's a good point that I had overlooked; it seemed clear that 
two enumerators (that, in general, have different values) cannot 
declare the same entity, which in the case of an enumerator, is 
only a value. However, this wording contradicts that reasoning, 
so you've convinced me. This will be issue 2530 in revision 108 
of the core language issues list.
                                      -- William M. (Mike) Miller
```

最奇妙的是，这个疏忽虽然看起来很微不足道，但它至少从 [C++03](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2005/n1905.pdf) 之前就存在了（C++03 标准文档有版权问题，这里链接是2005年的工作稿），一直存在了20来年，期间的编译器作者也都默认按照 C 的规则去处理了，也没有人提交过相关 defect。甚至有理由怀疑是 C++ 标准里存在时间最久的 bug 之一，果然最简单的问题藏在眼皮底下反而不容易被发现 😃 。