---
title: "原理分析：使用 dd 跳过开头若干字节快速拷贝文件"
date: 2021-02-18 13:20:00 +0800
categories: [Misc, Discussion]
tags: [tools, Chinese]
---

> 这篇文章是来自我在 [0xffff.one](https://0xffff.one/) 上的一个帖子 [https://0xffff.one/d/900](https://0xffff.one/d/900) 的回复。
>
> __原帖内容：__  
> 在折腾一个超大的备份文件，需要把它的前 41 个字节删除掉，没有 WinHEX，想着用 dd 命令来实现
> 
> 一开始这么干，发现速度奇慢，5分钟过去才复制40MB...
> ```
> dd if=input.bak bs=1 skip=41 > result.bak  
> ```
> google 一波发现一老哥的操作，配合 dd 和 cat 实现快速拷贝的功能，有些佩服。  
> ```
> {  dd bs=41 skip=1 count=0; cat; } < input.bak > result.bak  
> ```
> 原帖：  
> https://unix.stackexchange.com/questions/6852/best-way-to-remove-bytes-from-the-start-of-a-file


# 干了啥

看一下一开始的指令：

```
dd if=input.bak bs=1 skip=41 > result.bak
```
为什么这个指令会慢呢？首先一点背景知识：

　　计算机中每一**次**向硬盘读取和写入数据，无论读多小的数据量，都至少需要花一段常数时间（称为overhead）。
（就像你去超市买鸡蛋一样，无论你一次只买一个，还是一千个，你都至少要花从家走到超市，再从超市走回家的时间。）

　　就好比你现在要100个鸡蛋，但是你去超市一趟只买一颗鸡蛋的话，你就要来回跑100次一样。用 dd 拷文件也是同样的道理，**如果一次只跑去给硬盘要一个字节，一个文件就要来回跑特别多次，花费的时间就会特别长。**

　　为了解决这个问题，dd 在读文件的时候，会将文件切分成大小固定的一小块一小块 (block)，每次向硬盘要数据就一次性要一个“块”的大小（默认 512 个字节），也就是说，每次费那么大功夫跑过去，那就干脆多要一点数据，同样大小的文件不就可以少跑很多次了吗？

# 所以为啥慢？

　　前面提到的一次拿一个分块，这个分块的具体大小，是可以通过`bs=`参数进行人工调节的（bs = block size, 块大小）。

　　我们一开始的指令的问题就在于，我们这个指令里有一个参数`bs=1`，也就是告诉 dd，我想要每 1byte 就当作一个 block，这样 dd 实际上就又变回了跑一次读一个字节的傻 dd 了，所以这个指令最后就慢得出奇啦。（0.26MB/s)

# 大不就完事了？

_路人甲：那么既然是因为块大小 bs 设太小了，那我们改大不就行了吗？_

没错，理论上是这样的。划重点：理论上

如果我们只是想要单纯的把文件`a.txt`拷贝一份到文件`b.txt`，那我们的确可以直接把 bs 改大就行了：
```bash
# 块大小：512Bytes，速度93MB/s
dd if=a.txt of=b.txt bs=512

# 块大小：4MB，速度1138MB/s
dd if=a.txt of=b.txt bs=4m
```
仅仅把每次读的数据块大小改大，就得到 12 倍的速度提升！

然鹅，为什么我们这个例子里不行呢？
```
dd if=input.bak bs=1 skip=41 > result.bak       # 为啥不把 bs 直接改大？？？
```
因为我们除了拷贝文件外，还有另外一个要求：跳过前41个字节！

那么我们用什么方法实现跳过41个字节呢？我们一开始的指令使用了 `skip=41` 来实现这一要求。

好，刹住，大问题来了：**skip 的单位是 block**！也就是说`skip=41`不是指跳过41个字节，是指跳过41个 block 😅 （只是我们用`bs=1`让每个块都刚好是 1 字节而已）

也就是说，我们一开始的指令里的 `bs=1 skip=41` 其实是在讲，我们想要跳过 `41个块  x  每个块 1 个字节  =  共41个字节`

但是因为 **skip 只能跳整数个 block**，这就意味着，我们如果想把每个 block 大小改大，最多也就是`bs=41 skip=1`，跳过 `1个块  x  每个块 41 个字节  =  共41个字节`。因为再大的话，skip 就凑不成整数个块了。

那么矛盾就来了：
* 我们要把块大小 bs 改大，才能提高拷贝速度
* 但是为了实现跳过前 41 个字节，块大小最大也只能是 41 Bytes

_路人甲：啊这......_

# 曲线救国

这时候我们进退两难，就需要用曲线救国思路，借助我们万能的朋友——管道 来解决这个问题了。
看我们的解决方案：
```
{ dd bs=41 skip=1 count=0; dd bs=4m; } < input.bak > result.bak
```
看起来很复杂， 其实分开来看很简单，这里用到了一个 shell 的特性，花括号 `{ }`。
在花括号里的指令，会视为一个整体，里面每一条指令都会从输入管道读入数据后执行，但与每条指令分开执行不同的是，在花括号里的指令，需要排队分蛋糕一样读输入，也就是说 **花括号内的指令按顺序依次从输入管道读取同一个输入流**，每个指令读走了多少，下一个指令能读的就少掉多少。

我们最后的方案中，花括号里有两条通过分号隔开的指令，`dd bs=41 skip=1 count=0` 还有 `dd bs=4m`。

* 第一个指令是我们的老朋友—— dd，这里是让它从输入流直接跳掉1个41字节大小的块，然后`count=0`表示读 0 个 block，也就是跳完啥都不用读了，直接退出，把剩下的输入交给下个指令去处理。

* 第二个指令还是我们的 dd，但是因为第一个 dd 已经负责跳过了前41个字节了，第二个 dd 不需要考虑跳过字节，也就不需要怕 bs 设置太大啦！于是第二个 dd 放飞自我，可以直接用 4MB 的块大小（`bs=4`）去拷贝，一次搬 4MB，那叫一个快啊！

**第一个小 block size 的 dd 实现跳过，然后用第二个大 block size 的 dd 来快速搬数据**，双d齐心，其利断金！

_路人甲：妙哉!_

最后我们得到了这个d上加d的无敌大指令：`{ dd bs=41 skip=1 count=0; dd bs=4m; }`，我们只需要把文件输入到这个大指令里：`< input.bak`，再告诉它输出到哪里：`> result.bak`，回车一摁，哗啦一下，拷完啦！（1107MB/s，比最初的方法快4000+倍）

# cat? 喵喵喵？
```
# 我们的指令
{ dd bs=41 skip=1 count=0; dd bs=4m; } < input.bak > result.bak
# StackExchange 答案里的指令
{ dd bs=41 skip=1 count=0; cat; } < input.bak > result.bak
```
StackExchange 上原文用到的第二个负责搬砖的指令是 `cat` ，实际上和我们的`dd bs=4m`是差不多的，都是从输入流读数据，然后原封不动送到输出流，并且都是用较大的分块读取。

不同点在于 cat 会自动帮你选择合适的块大小，所以用 cat 的话我们什么参数都不用写，而用 dd 我们需要手动告诉它我们要 4MB 大小的分块。

# 结尾

最后给大家推荐一下这个小站吧：[0xffff.one](https://0xffff.one/)

这个小站是华南师范大学一位师兄建立的，出于改变学校内计算机科学讨论氛围欠缺的现状，打造一个能专注思考，能碰撞思维，能沉淀思想的交流社区。

和站长的认识也是算一个很难得的机会，交流到深夜，感到思维方式与看法相近，有一种很难得的归属感，在深大没曾体验过的兴奋。不知是因为是个人的圈子还是大的环境，总觉得深大的计算机科学也缺少这一种研究、沉淀的氛围，浮躁与急于求成风气比较浓。

有效的讨论我认为是加深对学科理解、促进思考与发展的重要因素，在平等的表达与交流中思维碰撞，往往能够产生超出讨论双方意料之外的结果。这样 1 + 1 > 2，双方都能有所收获，有所长进。

在这个社区中，每个人都是老师，也都是学生。

> 是故弟子不必不如师，师不必贤于弟子

我想把这个小站也推荐给深大的同学们。能够有更多人参与到研究和讨论中来，并有所收获就是最大的期望。

> If you want to go fast, go alone.  
> If you want to go far, go together.  

[0xffff.one](https://0xffff.one/)