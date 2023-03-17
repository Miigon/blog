---
title: "git的前世，和BitKeeper"
date: 2022-04-02 22:10:00 +0800
categories: [Misc, Unix]
tags: [git]
---

很多人应该都知道git的开发，最早是用来管理linux的内核源码的。在git之前，linux用的是一个叫做BitKeeper的商业软件进行源码管理和patch merge。

最早最早，Linux采用的其实是和unix一样的邮件互发patch的方式来共享更改，随着内核开发者群体的不断增长，这样的贡献管理方式所需要的人力劳动已经开始明显影响内核的开发进展了。

BitKeeper（BK）软件的作者/公司CEO--Larry McVoy向Linus提出，如果使用一个源代码管理软件，可以解决Linux日益严重的项目管理问题，使得整个流程再一次顺畅起来。

Larry是Linux的极早期贡献者（1992年，Linux 0.97阶段），他本身是很支持自由软件的发展的，所以Larry希望，和开源社区达成一个协议，让开源社区可以免费使用他公司的商业软件BK。

2002年，在Larry成功说服Linus相信BK能做他所需要做的一切后，Linus决定试一试BK，从此就是一条不归路。

> Linus使用BK作为源代码管理工具的决定也在社区中掀起了风波，原因很简单，Linux是历史上最重要的开源软件之一，而这么重要的开源项目，居然由一个商业软件做源码管理。Linus表示自己不是开源软件狂热信奉者，在开源方案比商业方案好的时候自己会选择开源方案，而在商业方案比开源方案好的时候，自己也不会抗拒。这在当时激怒了许多社区中的开源软件狂热者。
> 
> 详细阅读：https://www.linuxjournal.com/content/git-origin-story

> BitKeeper enabled a work method and patch flow which naturally supported the kernel's development model.

BK的设计，与Linux内核的协作开发流十分的契合，解决了Linux开发中愈发严重的scale问题和内核开发者一直以来的痛，内核社区的开发效率得到明显提升（这跟Larry本身是内核社区的一员并且一直和开源社区紧密合作有脱不开的干系）。

但是问题在于，BK终究是一个商业软件，背后有一个商业公司和团队要养，这就导致了Larry在授权BK给开源社区的时候，加入了严格的限制。并且由于必须考虑BK的商业利益，Larry需要在「给开源社区使用BK」和「不能让社区出现可能对BK造成威胁的东西」之间做出艰难平衡。许可协议常常出于一些Larry所「认为存在」的威胁为理由而收严（Larry需要保证开源社区不会开发出会让BK式微的替代品）。以至于最后协议被戏称为“别惹毛Larry协议”（"don't piss off Larry license."）。

最后一根稻草，是协议中一条不允许社区对BK进行反编译的条例，被一个为OSDL（Open Source Development Labs）工作的开发者（Andrew Tridgell，samba和rsync的作者）公然高调违反（社区对这一事情有很两极化的看法，详情可以读[这篇文章的评论区](https://lwn.net/Articles/130746/)）。Larry一气之下决定撤销BK对开源社区的授权。

时间是2005年，这一下没了BK，内核社区需要一个替代品作为源代码管理软件，于是有了[这个邮件列表](https://lkml.org/lkml/2005/4/7/121)里Linus表示决定bite the bullet，找BK的替代品，并且表明自己会尝试一段时间不用BK会是怎么样子（原话“至今为止感觉是晦暗荒芜的”）

> So I just wanted to say that I'm personally very happy with BK, and with   
Larry. It didn't work out, but it sure as hell made a big difference to   
kernel development. And we'll work out the temporary problem of having to   
figure out a set of tools to allow us to continue to do the things that BK   
allowed us to do.

Linus在邮件列表里告诉大家不要去恨BK的公司BitMover，并认可了Larry将商业软件BK和内核开发结合的努力，虽然最后没能促成，但Larry和BK对内核开发的影响是巨大的。后续的历史也证实了BK的影响深远。

在[讨论了现有源代码管理工具](https://lkml.iu.edu/hypermail/linux/kernel/0504.0/1826.html)之后，Linus在[一封邮件](https://lkml.iu.edu/hypermail/linux/kernel/0504.0/2022.html)里首次提到了git。一开始的git的目标是一个分布式的代码存档工具，可以commit、pull、push，但是没有考虑merge，因为一开始的定位并不是一个完整的SCM源代码管理工具，而只是一个在不同的开发者之间同步代码历史的方式。

git的设计和BK有很多相似之处：如分布式（没有一个“中心仓库”，所有人的副本的地位平等。相比SVN的集中式，Linus在邮件列表中多次提到自己不喜欢集中式的源码管理方式），如[不实现cherry-pick](https://lkml.org/lkml/2005/4/7/150)以保证历史不变性。文中也提到了可以将git看成更像一个「内容寻址文件系统」（即git的hash object系统，相比键寻址的传统文件系统）。但也有其独特之处，比如相比一个源代码管理系统，git的设计更像是一个低层文件系统；相比保存patch，git保存了完整的内容并动态按需生成patch，这使得git达到了当时其他尝试替代BK的开源替代品（arch, darcs, monotone...）不可企及的速度。“速度”是git的关键，Linux内核代码之庞大和开发者社区之活跃，使得SCM的速度变得至关重要。

> https://lkml.org/lkml/2005/4/7/150
> 插曲：不实现cherry-pick这个话题也蛮有意思的，一开始BK没有cherry-pick这件事Linus也是十分讨厌的，Larry说服了他历史只增加而不改变能使复制更简单。
> 
> 而第二个使得Linus决定说cherry-pick是错误的的理由是一个非技术理由：cherry-picking隐含了一个「“食物链上游”的用户可以修改其“下游”的人所做的工作」的逻辑。因为进行cherry-pick的理由往往是想要修复“别人的错误”，即你不同意的东西。
> 
> 问题是，这种做法往往导致整个系统里面出现错误的气氛和心理（results in the wrong dynamics and psychology in the system）。首先它假设了“食物链上下游”的存在，Linus认为这是错误的。内核开发正逐步变成一个“网络”，而Linus认为自己相比是“顶部的人”，更不过仅仅是"网络里比较中心的人"而已，而且Linus认为就应该是这样。内核的开发就应该是一个随意的网络而不是一个层级的结构。
> 
> 另外一个是，支持cherry-picking相当于隐式地将质量控制的职责放到了上游（“我会从你的提交树里面挑出好的东西”），而不支持cherry-picking则使得双方都有义务来保证提交树的干净。至于开发树的干净如何保证，Linus提到不同的人有不同的做法，有人有很多颗树，然后用自己开发的工具合并；有人在开发树上尽情开发，当满意后，把开发树丢掉，重新在主树上提交。（这有点像现在git分支的雏形）
> 
> 这里可以看出Linus对社区治理的态度。

git吸收了很多BK的设计：分布式的仓库，提交，同步，多树（“分支”），合并等等。

Linus从1991年以来，第一次暂停了参与Linux的开发，转而投入了git这个新工具的开发上：经过Linus短短几天的开发，git在2005年4月8日迎来了[第一个commit](https://github.com/git/git/commit/e83c5163316f89bfbde7d9ab23ca2e25604af290)（commit message: "git 第一版，地狱而来的内容管理器"），并实现了自己托管自己的源代码。几周内，git便已经有托管linux完整源代码的能力了。

	// 很Linus风的第一版README
	GIT - the stupid content tracker
	
	"git" can mean anything, depending on your mood.
	
	 - random three-letter combination that is pronounceable, and not
	   actually used by any common UNIX command.  The fact that it is a
	   mispronounciation of "get" may or may not be relevant.
	 - stupid. contemptible and despicable. simple. Take your pick from the
	   dictionary of slang.
	 - "global information tracker": you're in a good mood, and it actually
	   works for you. Angels sing, and a light suddenly fills the room. 
	 - "goddamn idiotic truckload of sh*t": when it breaks
	
	This is a stupid (but extremely fast) directory content manager.  It
	doesn't do a whole lot, but what it _does_ do is track directory
	contents efficiently.

> 看git的第一个提交可以看到，git的早期代码极其简单，只有不到一千行的c代码，git的核心设计分为两部分：内容索引的对象数据库，以及当前目录结构缓存，简单有效且快速。第一个git源代码的提交也是通过git自身完成的。

从Linux-2.6.12-rc2开始，[Linux内核正式转移到了git上托管](https://github.com/torvalds/linux/commit/1da177e4c3f41524e886b7f1b8a0c1fc7321cac2)，后续的历史也证明了基于git协作的简单有效，BK最终还是迎来了它一直以来最担心的东西：一个会威胁到他生存的开源替代品。不过此时BK已经放弃开源社区了，愤世嫉俗地看，也可以说BK在linux社区已经吸收了足够的利益，并决定已经可以离开了罢了。

从此git席卷世界的时代开始了。

> Author: Miigon（禁止转载）  
> Sources:   
> 1. https://lwn.net/Articles/130746/
> 2. https://lkml.org/lkml/2005/4/7/121
> 3. https://lkml.iu.edu/hypermail/linux/kernel/0504.0/2022.html
> 4. https://www.linuxjournal.com/content/git-origin-story
> 5. https://github.com/git/git
> 6. https://github.com/torvalds/linux
> 7. https://en.wikipedia.org/wiki/BitKeeper
> 
> Sources中的邮件列表链接有远比这篇文章更详细的讨论，已经是众多历史爱好者的考古现场了，本文只是对这些资料的粗糙概括，**强烈建议对这段辛酸史感兴趣的朋友读一下Sources中的1，以及2中邮件列表所有Linus Torvalds的邮件**。
> 
> 后记：bitKeeper在2016年选择了以Apache协议开源，并目目前已不再继续维护(来源请求)。截止2020年3月，仅仅GitHub上就已经有超过1.28亿公有git repo。