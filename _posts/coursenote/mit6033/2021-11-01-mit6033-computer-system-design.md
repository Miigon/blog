---
title: "MIT6.033 Computer System Design | 计算机系统设计"
date: 2021-11-01 02:19:00 +0800
categories: [Course Notes, MIT6.033]
tags: [system design, course recommendation]
---

# MIT6.033 Computer System Design (Spring 2021)
> 前置课程：
> * 6.004 Computation Structures 计算架构
> * 6.006 Introduction to Algorithms 算法导论
> 
> ps. 本课 6.033 Computer System Design 和 [6.S081 Operating System Engineering](/categories/mit6-s081/) 共同作为 6.824 Distributed Systems 的前置课。

## 课程介绍
MIT6.033 Computer System Design 是麻省理工学院计算机科学本科的必修课程，课程涵盖了计算机软件系统与硬件系统的工程设计：控制复杂度的技巧、利用客户-服务设计保证强模块性、操作系统、性能、网络、命名、安全与隐私、容错系统、原子性以及并发活动协调、错误恢复等。

MIT 6.033 包含了四个单元的技术内容：**操作系统**、**网络**、**分布式系统**、**安全**。

来自文献所提供的现实系统的研究，为系统设计学习提供了参照与对比。课程包含一个学期长的合作设计项目（design project），学生参与全面的沟通交流能力练习。

## 课程目标

使得学生能**设计自己的分布式系统来解决现实世界中的问题**。**论证自己的设计选择**也是设计分布式系统能力的一部分。

这一主要目标由其他这些目标所支撑：
* 学生将能分析并评价现有系统，以及自己的系统设计。作为该目标的一部分，学生将学会辨别现有系统中的设计选择。
* 学生将可以将课程中学习到的技术知识应用到新的系统组件中。这意味着需要学会辨别与描述：
    * 计算机系统中，如何使用常见的设计模式（例如抽象与模块化）来限制复杂度。
    * 操作系统如何使用虚拟化与抽象来确保模块性。
    * 互联网是如何为大容量、多元化应用、互相竞争的经济利益而设计
    * 如何在不可靠的网络上构建可靠、可用的分布式系统
    * 计算机系统安全的常见陷阱，以及如何应对它们

## 课程架构

### Lectures
Lectures 将教学生设计他们自己的计算机系统所需要的技术细节，并将这些细节放在更大的图景中：包括具体领域系统的图景以及作为一个系统总体的图景。

### Recitations
Recitations 给学生一个练习系统分析与口语交流技能的机会。每一次 Recitation 围绕着一个具体的关于系统的 paper 展开。通过读这些 paper，学生能够更好的体会该领域内的沟通交流是如何完成的。Recitation 是基于讨论的，学生得以练习分析、评价以及交流系统的能力

> Note: 这一部分是 MIT 学生每周必须要做的论文阅读复述与讨论环节，旨在锻炼学生的口语表达能力以及描述系统设计的能力。个人觉得这个是十分重要的技能，特别是在团队/开源社区等合作开发环境下，能够准确、清晰且简洁地将自己的系统设计、设计选择与取舍讲清楚，是十分重要的一项能力。（避免在沟通上内耗）
> 这一部分内容可以作为扩展阅读使用，重点不仅是看一些实际系统是如何设计的，还要看作者是如何把自己的系统设计选择向别人讲清楚的。

### Writing Tutorials
这些教程将教你关于这门课的沟通的理论以及实践，并帮助你为作业（尤其是 design project）做好准备。你将会流畅掌握不同的沟通流派，培养将技术概念展现给不同听众所需要的策略与技能，学习如何使用写作来培养以及加深你的技术理解。

> Note: 主要面向 MIT 学生需要团队完成的学期大作业 design project。该作业要求学生三人组队设计一个自己的系统以解决一个现实世界中的问题。现实世界中的系统都不是由一个人独立构建的，总是团队协作的成果。design project 包含报告、演示、peer review 等环节，感兴趣的可以前往 [Design Project 页面](https://web.mit.edu/6.033/2021/wwwdocs/dp.shtml) 查看。

## 课程资源
课程网站：https://web.mit.edu/6.033/2021/wwwdocs/
课本：https://ocw.mit.edu/resources/res-6-004-principles-of-computer-system-design-an-introduction-spring-2009/

> 很可惜，据我所知目前还没有中文课程翻译。

下面整理的内容已经将非 MIT 学生无法访问的 assignment 以及 design project 的内容去掉了。只留下 lecture、recitation和 writing tutorials。

LEC: lectures
REC: recitations
TUT: writing tutorials

### Operating Systems
> 操作系统如何在「单主机」的范围内施行模块化。

#### Lectures
**LEC 1:** [Enforced Modularity via Client/server Organization](https://web.mit.edu/6.033/2021/wwwdocs/lec/l01.shtml)  
**LEC 2:** [Naming](https://web.mit.edu/6.033/2021/wwwdocs/lec/l02.shtml)  
**LEC 3:** [Virtual memory](https://web.mit.edu/6.033/2021/wwwdocs/lec/l03.shtml)  
**LEC 4:** [Bounded buffers and locks](https://web.mit.edu/6.033/2021/wwwdocs/lec/l04.shtml)  
**LEC 5:** [Threads](https://web.mit.edu/6.033/2021/wwwdocs/lec/l05.shtml)  
**LEC 6:** [OS structure, Virtual Machines](https://web.mit.edu/6.033/2021/wwwdocs/lec/l06.shtml) 

#### Recitations
**REC 1:** [Introduction to 6.033 Recitations](https://web.mit.edu/6.033/2021/wwwdocs/recitations/01-intro.shtml)  
**REC 2:** [We Did Nothing Wrong](https://web.mit.edu/6.033/2021/wwwdocs/recitations/02-wrong.shtml)  
**REC 3:** [DNS](https://web.mit.edu/6.033/2021/wwwdocs/recitations/03-dns.shtml)  
**REC 4:** [UNIX](https://web.mit.edu/6.033/2021/wwwdocs/recitations/04-unix.shtml)  
**REC 5:** [UNIX](https://web.mit.edu/6.033/2021/wwwdocs/recitations/05-unix.shtml)  
**REC 6:** [DP Discussion](https://web.mit.edu/6.033/2021/wwwdocs/recitations/06-dp.shtml)  

### Networking
> 多机系统的机器之间数据与请求是如何交换的。如何放大规模。

#### Lectures
**LEC 7:** [Intro to networking and layering](https://web.mit.edu/6.033/2021/wwwdocs/lec/l07.shtml)    
**LEC 8:** [Network Layer: Routing](https://web.mit.edu/6.033/2021/wwwdocs/lec/l08.shtml)  
**LEC 9:** [BGP](https://web.mit.edu/6.033/2021/wwwdocs/lec/l09.shtml)  
**LEC 10:** [Transport Layer: TCP](https://web.mit.edu/6.033/2021/wwwdocs/lec/l10.shtml)  
**LEC 11:** [In-network Resource Management](https://web.mit.edu/6.033/2021/wwwdocs/lec/l11.shtml)  
**LEC 12:** [Application Layer](https://web.mit.edu/6.033/2021/wwwdocs/lec/l12.shtml)  
**LEC 13:** Canceled  
**EXAM**: [Midterm](https://web.mit.edu/6.033/2021/wwwdocs/assignments/exam-1.shtml)

#### Recitations
**REC 7:** [Ethernet](https://web.mit.edu/6.033/2021/wwwdocs/recitations/07-ethernet.shtml)  
**REC 8:** [Encapsulation](https://web.mit.edu/6.033/2021/wwwdocs/recitations/08-encapsulation.shtml)  
**REC 9:** [Overlay Networks (RON)](https://web.mit.edu/6.033/2021/wwwdocs/recitations/09-ron.shtml)  
**REC 10:** [Performance, Measurement, and Evaluation (RON)](https://web.mit.edu/6.033/2021/wwwdocs/recitations/10-ron.shtml)  
**REC 11:** [DCTCP](https://web.mit.edu/6.033/2021/wwwdocs/recitations/11-dctcp.shtml)  
**REC 12:** [End-to-end Arguments](https://web.mit.edu/6.033/2021/wwwdocs/recitations/12-e2e.shtml)  
**REC 13:** Canceled  


### Distributed Systems
> 通过网络连接的分布式系统中可能会遇到什么问题/错误，如何应对。

#### Lectures
**LEC 14:** [Reliability](https://web.mit.edu/6.033/2021/wwwdocs/lec/l14.shtml)  
**LEC 15:** [Transactions](https://web.mit.edu/6.033/2021/wwwdocs/lec/l15.shtml)  
**LEC 16:** [Logging](https://web.mit.edu/6.033/2021/wwwdocs/lec/l16.shtml)  
**LEC 17:** [Isolation](https://web.mit.edu/6.033/2021/wwwdocs/lec/l17.shtml)  
**LEC 18:** [Distributed Transactions](https://web.mit.edu/6.033/2021/wwwdocs/lec/l18.shtml)  
**LEC 19:** [Replication](https://web.mit.edu/6.033/2021/wwwdocs/lec/l19.shtml)  

#### Recitations
**REC 14:** [GFS](https://web.mit.edu/6.033/2021/wwwdocs/recitations/14-gfs.shtml)  
**REC 15:** [MapReduce](https://web.mit.edu/6.033/2021/wwwdocs/recitations/15-mapreduce.shtml)  
**REC 16:** [ZFS](https://web.mit.edu/6.033/2021/wwwdocs/recitations/16-zfs.shtml)  
**REC 17:** Canceled  
**REC 18:** [Consistency Guarantees](https://web.mit.edu/6.033/2021/wwwdocs/recitations/18-baseball.shtml)  
**REC 19:** [Raft](https://web.mit.edu/6.033/2021/wwwdocs/recitations/19-raft.shtml)  

### Security
#### Lectures
**LEC 20:** [Intro to Security + Authentication](https://web.mit.edu/6.033/2021/wwwdocs/lec/l20.shtml)  
**LEC 21:** [Low-level attacks](https://web.mit.edu/6.033/2021/wwwdocs/lec/l21.shtml)  
**LEC 22:** [Secure Channels](https://web.mit.edu/6.033/2021/wwwdocs/lec/l22.shtml)  
**LEC 23:** [ToR](https://web.mit.edu/6.033/2021/wwwdocs/lec/l23.shtml)  
**LEC 24:** [Network Attacks](https://web.mit.edu/6.033/2021/wwwdocs/lec/l24.shtml)  
**LEC 25:** [Wrap-up](https://web.mit.edu/6.033/2021/wwwdocs/lec/l25.shtml)  
**EXAM**: [Final exam](https://web.mit.edu/6.033/2021/wwwdocs/assignments/exam-2.shtml)

#### Recitations
**REC 20:** [Raft](https://web.mit.edu/6.033/2021/wwwdocs/recitations/20-raft.shtml)  
**REC 21:** [Meltdown](https://web.mit.edu/6.033/2021/wwwdocs/recitations/21-meltdown.shtml)  
**REC 22:** [DNSSEC](https://web.mit.edu/6.033/2021/wwwdocs/recitations/22-dnssec.shtml)  
**REC 23:** Canceled  
**REC 24:** [Mirai](https://web.mit.edu/6.033/2021/wwwdocs/recitations/24-mirai.shtml)  


### Writing Tutorials
**TUT 1:** [Intro to 6.033 Communication](https://web.mit.edu/6.033/2021/wwwdocs/tutorials/01-intro.shtml)  
**TUT 2:** [Consensus and Reasoning About Systems](https://web.mit.edu/6.033/2021/wwwdocs/tutorials/02-consensus.shtml)  
**TUT 3:** [Reading for Systems Concepts](https://web.mit.edu/6.033/2021/wwwdocs/tutorials/03-systems.shtml)  
**TUT 4:** [Collaboration and Collaborative Writing](https://web.mit.edu/6.033/2021/wwwdocs/tutorials/04-collaboration.shtml)  
**TUT 5:** [Visual Design, Figures, and Diagrams](https://web.mit.edu/6.033/2021/wwwdocs/tutorials/05-design.shtml)  
**TUT 6:** [Assembling the DPPR](https://web.mit.edu/6.033/2021/wwwdocs/tutorials/06-dppr.shtml)  
**TUT 7:** [DP Presentation](https://web.mit.edu/6.033/2021/wwwdocs/tutorials/07-presentation.shtml)  
**TUT 8:** [Responding to Feedback](https://web.mit.edu/6.033/2021/wwwdocs/tutorials/08-feedback.shtml)   
**TUT 9:** Analysis and Evaluation  
**TUT 10:** [Peer Review](https://web.mit.edu/6.033/2021/wwwdocs/tutorials/10-peerreview.shtml)  
**TUT 11:** Final DP Report  
