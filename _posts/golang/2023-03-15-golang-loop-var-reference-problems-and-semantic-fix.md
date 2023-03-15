---
title: "随笔：Golang 循环变量引用问题以及官方语义修复"
date: 2023-03-15 17:05:00 +0800
categories: [Language, Golang]
tags: [discussion]
---

这篇文章谈一个已经在 Golang 中存在多年的，几乎每一个新手都要被坑一遍的设计：引用捕获了循环变量，且逃逸出循环迭代范围而造成的逻辑错误。

以及重点是讨论了目前对这个常见问题的解决办法的探索（静态分析存在的不足，以及2022年10月 Golang 官方提出的直接更改 for 循环变量语义，从语言设计上根本地消除这个问题的 proposal）

# background
这个问题是一个 Golang 从很早版本就一直存在的设计选择造成的。

简单地讲就是 for 循环中，由于 func 捕获，或者显式/隐式的取引用，对循环变量产生了引用并且这个引用逃逸出了当前循环迭代（iteration）的生命周期范围。而由于 Golang 一开始决定将将循环变量（i、k、v）的生命周期定义为整个循环，而不是每个迭代都有新一份的循环变量，导致了每一轮迭代产生的引用实际上都指向同一个值，而不是指向每一轮各自对应的值。

常见的场景有以下：
- function literal captures loop variable  

```go
arr := []int{1,2,3,4,5}
for _, v := range arr {
	go func() { fmt.Println(v) }() // v is a implicit reference
}
// prints 5 5 5 5 5
```

- implicit reference due to receiver mismatch（更难发现）  

```go
type MyInt int

func (mi *MyInt) Show() {
	fmt.Println(*mi)
}

func main() {
	ms := []MyInt{1, 2, 3, 4, 5}
	for _, m := range ms {
		go m.Show()
		// implicitly converted to `go (&m).Show()`
		// thus creating a reference to loop variable.
		// but you would never know this without more context.
	}
	time.Sleep(100 * time.Millisecond)
}
// prints 5 5 5 5 5
```

- appending `&v` to a slice  

```go
arr := []int{1,2,3,4,5}
for _, v := range arr {
	arr2 = append(arr2, &v) // all new elements &v are the same.
}

// arr2 == {v_arr, v_arr, v_arr, v_arr, v_arr}
// *v_arr == 5
```

golang 的循环变量是 per loop 的，而不是 per iteration 的。如果对循环变量产生了引用（比如闭包 capture，或者取指针），不同次迭代取到的指针都是同一个。

如果这个指针/引用被逃逸出了一次迭代的范围内（比如 append 到了一个数组里，或者被go/defer后面的闭包capture了），因为所有 iteration 里取到的指针都是同一个，指向的对象也都会是同一个（最后一轮iteration的结果）。

# workaround
```go
 	for _, a := range alarms {
+		a := a
 		go a.Monitor(b)
 	}
```
一个 workaround 是加一句创建同名新变量 shadow 掉原来的循环变量，强制拷贝变量，把 per loop 的循环变量变成 per iteration的。

问题是很多时候很难知道某个循环是否需要写这么一行拷贝，导致很容易因为遗漏而产生bug。另一个极端是有的开发者因为担心遗漏，选择过度矫正，把所有的循环都写上这一句拷贝，使得代码可读性降低。

# vet? static analysis?
go vet 或其他 static analysis 方案虽然能帮助找到很明显的错误场景，但是由于静态分析并不能完全100%理解程序逻辑，在没有 proof 某个循环变量指针一定会超出 iteration 范围的前提下，会出现 false positive。

> 并不能单用逃逸分析，根据引用是否逃逸来判定是否会出问题。
> 
> 例子：循环体和 goroutine 之间可能使用了 waitgroup 进行了同步，从而使得虽然循环变量引用逃逸到了 goroutine 中，但是每一个 goroutine 的执行时机实际上都不会超过对应 iteration 的生命周期

而 go vet 中的 loopclosure 则是采取了保守的方案，只有十足把握是错误情况才报告，会有false negative。

静态分析的问题是分析无法透过一些运行时功能，比如 interface 方法，比如 reflection。只能理解相对简单的代码。

# sematics fix
问题的本质是 golang 设计之初，决定将循环变量设定为 per loop 的而不是 per iteration 的。想要根除这个问题，需要在语义层面修复。即将循环变量设定为 per iteration。

Russ Cox（rsc）在2022年10月的时候，重新提出了这个话题：
<https://github.com/golang/go/discussions/56010>

> ### A decade of experience shows the cost of the current semantics
> ...... Since then, I suspect every Go programmer in the world has made this mistake in one program or another. I certainly have done it repeatedly over the past decade, despite being the one who argued for the current semantics and then implemented them. (Sorry!)
> The current cures for this problem are worse than the disease.

rsc 提到了，在Golang进入Go1版本之前就已经讨论过这个问题，当时的结论是：虽然很烦但是问题没有大到要改。但是过去这个 decade 已经展示出来当前语义设计的后果。rsc 本人也常常被这个问题坑到。

更重要的是，目前对这个问题的解决方法，比问题本身还糟糕。

```go
 	for _, informer := range c.informerMap {
+		informer := informer
 		go informer.Run(stopCh)
 	}
```

```go
 	for _, a := range alarms {
+		a := a
 		go a.Monitor(b)
 	}
```

光看这两份代码，都有上述提到的 workaround。但实际上一个是真正的 bugfix，另一个是没有作用的。在没有上下文的前提下，没有任何办法区分。实际上其中一个是 interface 类型，创建拷贝变量并没有任何效果。另一个则是 struct 类型调用了 pointer receiver 方法，是真正的 bugfix。

并且，还有一些代码，不论上下文是什么，添加的拷贝都是没必要的拷贝（没有任何隐式引用循环变量的可能）：
```go
 	for _, scheme := range artifact.Schemes {
+		scheme := scheme
 		Runtime.artifactByScheme[scheme.ID] = id
 		Runtime.schemesByID[scheme.ID] = scheme
 	}
```

像这样的迷惑性和二义性，恰恰违反了 Go 可读的设计目标。

> This kind of confusion and ambiguity is the exact opposite of the readability we are aiming for in Go.

之前 Golang 社区尝试通过文档和工具的方式，尝试防止用户因为这个语义而写出 bug 代码。但是实际实践中已经证明了，这种方式并不是非常有效，即使是语言的老手也经常不经意写出问题代码。

## the official fix
在 <https://github.com/golang/go/discussions/56010> 中提到的，从根本上彻底解决这个问题的方案是，将循环变量（三段式循环以及range循环）改为 per iteration。概念上等同于，在每个 iteration 开始时，将原本 per loop 的循环变量拷贝一份，并且在每个 iteration 结束时拷贝回去。

这样，每一轮 iteration 取到的地址都会是不同的地址。

而这个 sematic change 会通过 go.mod 来判断是否启用，旧的项目的行为照旧完全不变。

## effects of migrating to new semantics

### Google's Go Tests

> 目前（2023-03）这个提案还没有被正式确定引入 Golang 中。这里提到的结果都是 Golang 团队测试/实验的结果。

> 原文：Changing the semantics is usually a no-op, and when it’s not, it fixes buggy code far more often than it breaks correct code

大多数情况下，这个语义变更没有影响。在有影响的情况下，常常产生的影响都是修复了有bug的代码，而不是让更多代码出问题。

他们（rsc）测试了 Google 内所有 Go 测试的一个子集。在变更语义后的新失败率大约是1/2000，但是几乎所有失败的测试都是之前没有发现的真实的bug。而原本正确的代码被这个更改影响坏的比率是1/50000。

10w 个测试（包含 130w 个 for loop） 中，只有 58 个测试出现了失败。其中 36 个（62%）测试是由于和 t.Parallel 错误的交互而导致的不正确的无效测试，而在 for 循环变量语义更改后反而更正了这些测试了（指的是：测试失败的原因，是原本错误的测试在语义更改后变得正确了，然后测试从无效变成了有效，并且帮助找到了代码里确实存在的 bug，所以报告了失败）。

第二大常见的错误是每一轮迭代将 &v append 给了一个 slice，从而产生一个有 N 个相同指针的 slice。

在这 10w 个测试的 58 个失败的测试中，只找到了 2 个是真的依赖了 per-loop 循环变量的语义，并且真的因为语义变更而导致失败的：
> One involved a handler registered using once.Do that needed access to the current iteration’s values on each invocation. The other involved low-level code running in a context when allocation is disallowed, and the variable escaped the loop (but not the function), so that the old semantics did not allocate while the new semantics did.

两个都非常地容易修复。

### perspective: csharp's migration to per-iteration loop vars

C# 团队中的 [@jaredpar](https://github.com/jaredpar) （负责处理 customer feedback 的主要人物）提供了视角：

C#5 的时候也做过类似的更改，将 foreach 的循环变量从 per-loop 改为 per-iteration。当时由于 C# 没有类似 go.mod 的版本指定机制，所以唯一的选项就是要么无条件地改掉并且 break 一些东西，要么永远忍受现状。

循环变量的生命周期问题，在语言引入 lambda 表达式之后变成了一个痛点（闭包捕获）。随着语言对 lambda 表达式的使用越来越广泛，问题也越来越明显。严重到 C# 团队决定，无差别地全盘修改是值得的。相比给每个新的用户都解释一遍这个非常 tricky 的行为，相比之下给（但愿）数量较少的受影响的客户解释显得更容易一些。

最终的结果是：受这个更改所影响的客户的数量，比想象中的少。并且对于那些被影响到的客户，相应都是积极的，并且都接受了所提议的代码修复。

C# 作出这个更改已经10年了，jaredpar 的原话：”I'm honestly struggling to remember the last time I worked with a customer hitting this.“ （this 指更改语义造成的代码 break）

# more

本文摘选以及翻译/总结自 Golang 官方仓库 Discussion 中对于该话题的帖子 <https://github.com/golang/go/discussions/56010> 。完整的讨论请前往原链接。（原讨论帖 17 天内收到了 241条回复，由于社区反馈已经比较充足，新回复基本上也是旧回复的重复，原讨论帖已被 lock）

以下是原讨论帖的大致内容：
- 原 discussion 下方的评论几乎一致地支持这个新语义变更的发生。即使是多年的 Golang 语言使用者也承认会被这个语义坑到。一致同意 per-loop 的循环变量语义是 Golang 的一个 foot gun，是问题和 bug 的一个持续来源。
- 主要的讨论点在于CI/工具链和现有代码/依赖如何平滑迁移到新的语义上，以及是否有依赖旧语义才能正确工作的合法代码会被break（不多。只找到了极其稀有的 case 是依赖 per-loop 循环变量的旧行为的，而且这个 case 本身代码就不是特别清晰）。
- 另外一个问题就是 go generate 必须生成新旧版本都能正确编译的代码，不过同样的，依赖旧语义的代码极其稀有，大多数代码生成器不会被 break。
- 一部分用户对把三段式 for loop （`for i := 1; i < n; i++`）中的循环变量也修改成 per iteration 这个提议提出了质疑，主要理由是会和其他语言（主要是C以及一众类C语言）的行为不一致，其他语言背景的用户迁移过来也会被坑到。（C# 迁移到 per-iteration 循环变量作用域的时候就只迁移了 foreach，而没更改三段式 for loop 的循环变量作用域）
- 一些 practical 的问题：如何在用户升级的时候告知用户这一变动？如何检测升级前后是否会 break 用户的具体代码？这个变更应该是在 minor 版本发布还是在 major 版本（Go2）中发布？