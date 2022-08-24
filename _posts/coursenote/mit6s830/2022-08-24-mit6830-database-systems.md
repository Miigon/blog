---
title: "MIT6.830 Database Systems | 数据库系统"
date: 2022-08-24 18:24:00 +0800
categories: [Course Notes, MIT6.830]
tags: [system design, database, course recommendation]
---

# MIT6.830 Database Systems (Spring 2021)
> 前置课程：
> - [6.033 Computer System Design 计算机系统设计](/posts/mit6033-computer-system-design/)
> 
> 这门课与MIT 6.814是同一门课程，两者区别在于Final Project在6.814中由Lab5以及Lab6替代。虽然这门课是研究生课程，但是在MIT里，这门课大概1/3的学生是本科生。

## 课程介绍
MIT6.830 Database Systems 数据库系统课程为麻省理工学院的研究生课程，主要通过来自数据库社区的阅读材料（论文），向学生介绍数据库系统的基础，重点关注基本概念，如实现关系代数和数据模型、schema 范式化、查询优化、事务等。课程不假设学生有任何先前的数据库经验。

### 涉及话题
与数据库系统的设计有关的话题，包括：数据模型、数据库和 schema 设计、schema 范式化和完整性限制（integrity constraints）、查询处理、查询优化与开销预估、事务、恢复、并行控制、隔离与一致性、分布式/并行/多样数据库、自适应数据库、trigger系统、键值存储、对象-关系映射、流式数据库、服务化数据库。来自原创研究论文的lecture和阅读。

## 课程架构
完整的课程结构参考[课程Assignment](http://db.lcs.mit.edu/6.5830/2021/assign.php)和 [Lecture 1 PDF](http://db.lcs.mit.edu/6.5830/2021/lectures/lec1-notes.pdf)。
### Lecture
主要完成视频、PPT、PDF。

节约大家挑选课程的时间，这里将 Lecture 1 中提到的课程核心概念内容摘抄出来：

- Data modeling & layout 数据模型 & 布局
	- Systematic approach to structuring / representing data
	- Important for consistency, sharing, efficiency of access to persistent data
- Declarative Querying and Query Processing 声明式请求和请求处理
	- High level language for accessing data
	- "Data Independence"
	- Compiler that finds optimal plan for data access
	- Many low-level techniques for efficiently getting at data
- Consistency / Transactions + Concurrency Control 一致性 / 事务 + 并行控制
	- Atomicity -- Complex operations can be thought of as a single atomic operation that either completes or fails; partial state is never exposed
	- Consistency and Isolation -- Semantics of concurrent operations are well defined --equivalent to a serial execution, respecting invariants over time
	- Durability -- Completed operations persist after a failure
	
	Makes programming applications MUCH easier, since you don’t have to reason about arbitrary interleavings of concurrent code, and you know that the database will always be in a consistent state
- Distributed data processing 分布式数据处理
- 一点点其他领域的东西 A bit of many fields: systems, algorithms and data structures, languages + language design, more recently AI + learning, distributed systems….
- 一些最新的论文 This course will look in detail at these areas, as well as a number of papers current in DBMS
research, e.g., streaming, large scale data processing (Spark etc)


### Reading Assignment
Lecture前后读论文，都是与Lecture内容相关的或者Lecture围绕的论文。读论文也是这门课的主题之一。

**LEC 1:** Introduction to Databases  
**LEC 2:** [The Relational Model](http://db.lcs.mit.edu/6.5830/2021/lectures/lecture2.php)  
**LEC 3:** [Schema Design](http://db.lcs.mit.edu/6.5830/2021/lectures/lecture3.php)  
**LEC 4:** [Intro to Database Internals](http://db.lcs.mit.edu/6.5830/2021/lectures/lecture4.php)  
**LEC 5:** [Database Operators and Query Processing](http://db.lcs.mit.edu/6.5830/2021/lectures/lecture5.php)  
**LEC 6:** [Indexing and Access Methods](http://db.lcs.mit.edu/6.5830/2021/lectures/lecture6.php)  
**LEC 7:** [Database Layout for Analytic Databases](http://db.lcs.mit.edu/6.5830/2021/lectures/lecture7.php)  
**LEC 8:** [Join Algorithms](http://db.lcs.mit.edu/6.5830/2021/lectures/lecture8.php)  
**LEC 9:** [Query Optimization](http://db.lcs.mit.edu/6.5830/2021/lectures/lecture9.php)  
**LEC 10:** [Query Execution in Main Memory DBMS](http://db.lcs.mit.edu/6.5830/2021/lectures/lecture10.php)  
**LEC 11:** [Learned Indexing](http://db.lcs.mit.edu/6.5830/2021/lectures/lecture11.php)  
**LEC 12:** [Transactions And Locking](http://db.lcs.mit.edu/6.5830/2021/lectures/lecture12.php)  
**LEC 13:** [Optimistic Concurrency Control and Snapshot Isolation](http://db.lcs.mit.edu/6.5830/2021/lectures/lecture13.php)  
**LEC 14:** [Recovery](http://db.lcs.mit.edu/6.5830/2021/lectures/lecture14.php)  
**LEC 15:** [Recovery (cont.)](http://db.lcs.mit.edu/6.5830/2021/lectures/lecture15.php)  
**LEC 16:** [Distributed Databases](http://db.lcs.mit.edu/6.5830/2021/lectures/lecture16.php)  
**LEC 17:** [Distributed Transactions](http://db.lcs.mit.edu/6.5830/2021/lectures/lecture17.php)  
**LEC 18:** [Modern Transactions](http://db.lcs.mit.edu/6.5830/2021/lectures/lecture18.php)  
**LEC 19:** [Eventual Consistency](http://db.lcs.mit.edu/6.5830/2021/lectures/lecture19.php)  
**LEC 20:** [MapReduce / Hadoop](http://db.lcs.mit.edu/6.5830/2021/lectures/lecture20.php)  
**LEC 21:** [Cluster Computing](http://db.lcs.mit.edu/6.5830/2021/lectures/lecture21.php)  
**LEC 22:** [Project/Lab Discussions](http://db.lcs.mit.edu/6.5830/2021/lectures/lecture22.php)  
**LEC 23:** [Final Project Presentations](http://db.lcs.mit.edu/6.5830/2021/lectures/lecture23.php)  

这些链接都指向Reading Assignment，关于Lecture本身的信息请查看[课程Assignment](http://db.lcs.mit.edu/6.5830/2021/assign.php)。

### Lab
一个Java实现的基本教学数据库SimpleDB，以代码留空+自动化单元测试的形式，让学生接触数据库的基本机制的代码实现：存取数据、查询操作符、事务、锁、并行查询、索引等等。

**LAB 1:** [SimpleDB](https://github.com/MIT-DB-Class/simple-db-hw-2021/blob/master/lab1.md)  
**LAB 2:** [SimpleDB Operators](https://github.com/MIT-DB-Class/simple-db-hw-2021/blob/master/lab2.md)  
**LAB 3:** [Query Optimization](https://github.com/MIT-DB-Class/simple-db-hw-2021/blob/master/lab3.md)  
**LAB 4:** [SimpleDB Transactions](https://github.com/MIT-DB-Class/simple-db-hw-2021/blob/master/lab4.md)  
**LAB 5:** [B+ Tree Index](https://github.com/MIT-DB-Class/simple-db-hw-2021/blob/master/lab5.md)  
**LAB 6:** [Rollback and Recovery](https://github.com/MIT-DB-Class/simple-db-hw-2021/blob/master/lab6.md)  

另外，原课程还包含了一个团队的自选主题Final Project、以及可以替代Final Project的[Programming Contest组队比赛](https://github.com/MIT-DB-Class/programming-contest-2021)。以及三个Problem Set练习和Quiz小测，感兴趣以及有时间的话也推荐完成一下。同样的，最全的信息在[课程Assignment](http://db.lcs.mit.edu/6.5830/2021/assign.php)中。

## 课程资源
课本：
- “Readings in Database Systems” (The Red Book, 4th ed!)
- “Database Management Systems” (Ramakrishnan and Gehrke)

课程主页：<http://db.lcs.mit.edu/6.5830/2021/index.php>  
课程介绍：<http://db.lcs.mit.edu/6.5830/2021/syllabus.php>  
课程时间表（含有Reading Assignment）：<http://db.lcs.mit.edu/6.5830/2021/sched.php>  
课程Assignment（课程视频、PPT、Project、Lab）：<http://db.lcs.mit.edu/6.5830/2021/assign.php>  
部分课程公网链接（2014）：[YouTube](https://www.youtube.com/watch?v=F3XGUPll6Qs&list=PLfciLKR3SgqOxCy1TIXXyfTqKzX2enDjK) [B站搬运](https://www.bilibili.com/video/av506856895)  
