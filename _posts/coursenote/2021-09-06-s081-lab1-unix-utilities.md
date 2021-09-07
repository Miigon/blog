---
title: "[mit6.s081] 笔记 Lab1: Unix utilities | Unix 实用工具"
date: 2021-09-07 17:00:00 +0800
categories: [Course Notes, MIT6.S081]
tags: [operating system]
---
> 这是我自学 MIT6.S081 操作系统课程的 lab 代码笔记第一篇：Unix utilities。此 lab 大致耗时：4小时。  
> 
> 课程地址：[https://pdos.csail.mit.edu/6.S081/2020/schedule.html](https://pdos.csail.mit.edu/6.S081/2020/schedule.html)  
> Lab 地址：[https://pdos.csail.mit.edu/6.S081/2020/labs/util.html](https://pdos.csail.mit.edu/6.S081/2020/labs/util.html)  
> 我的代码地址：[https://github.com/Miigon/my-xv6-labs-2020/tree/util](https://github.com/Miigon/my-xv6-labs-2020/tree/util)  
> Commits: [https://github.com/Miigon/my-xv6-labs-2020/commits/util](https://github.com/Miigon/my-xv6-labs-2020/commits/util)  
> 
> 本文中代码注释是编写博客的时候加入的，原仓库中的代码可能缺乏注释或代码不完全相同。  

# Lab 1: Unix utilities

This lab will familiarize you with xv6 and its system calls.

实现几个 unix 实用工具，熟悉 xv6 的开发环境以及系统调用。

## Boot xv6 (easy)
准备环境，编译编译器、QEMU，克隆仓库，略过。
```
$ git clone git://g.csail.mit.edu/xv6-labs-2020
$ cd xv6-labs-2020
$ git checkout util
$ make qemu
```

## sleep (easy)
> Implement the UNIX program sleep for xv6; your sleep should pause for a user-specified number of ticks. A tick is a notion of time defined by the xv6 kernel, namely the time between two interrupts from the timer chip. Your solution should be in the file user/sleep.c.

练手题，记得在 Makefile 中将 sleep 加入构建目标里。

```c
// sleep.c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h" // 必须以这个顺序 include，由于三个头文件有依赖关系

int main(int argc, char **argv) {
	if(argc < 2) {
		printf("usage: sleep <ticks>\n");
	}
	sleep(atoi(argv[1]));
	exit(0);
}
```

```makefile
UPROGS=\
	$U/_cat\
	$U/_echo\
	$U/_forktest\
	$U/_grep\
	$U/_init\
	$U/_kill\
	$U/_ln\
	$U/_ls\
	$U/_mkdir\
	$U/_rm\
	$U/_sh\
	$U/_stressfs\
	$U/_usertests\
	$U/_grind\
	$U/_wc\
	$U/_zombie\
	$U/_sleep\ .   # here !!!
```

## pingpong (easy)
> Write a program that uses UNIX system calls to "ping-pong" a byte between two processes over a pair of pipes, one for each direction. The parent should send a byte to the child; the child should print "<pid>: received ping", where <pid> is its process ID, write the byte on the pipe to the parent, and exit; the parent should read the byte from the child, print "<pid>: received pong", and exit. Your solution should be in the file user/pingpong.c.

管道练手题，使用 fork() 复制本进程创建子进程，创建两个管道，分别用于父子之间两个方向的数据传输。

```c
// pingpong.c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int main(int argc, char **argv) {
	// 创建管道会得到一个长度为 2 的 int 数组
	// 其中 0 为用于从管道读取数据的文件描述符，1 为用于向管道写入数据的文件描述符
	int pp2c[2], pc2p[2];
	pipe(pp2c); // 创建用于 父进程 -> 子进程 的管道
	pipe(pc2p); // 创建用于 子进程 -> 父进程 的管道
	
	if(fork() != 0) { // parent process
		write(pp2c[1], "!", 1); // 1. 父进程首先向发出该字节
		char buf;
		read(pc2p[0], &buf, 1); // 2. 父进程发送完成后，开始等待子进程的回复
		printf("%d: received pong\n", getpid()); // 5. 子进程收到数据，read 返回，输出 pong
		wait(0);
	} else { // child process
		char buf;
		read(pp2c[0], &buf, 1); // 3. 子进程读取管道，收到父进程发送的字节数据
		printf("%d: received ping\n", getpid());
		write(pc2p[1], &buf, 1); // 4. 子进程通过 子->父 管道，将字节送回父进程
	}
	exit(0);
}
```

注：序号只为方便理解，实际执行顺序由于两进程具体调度情况不定，不一定严格按照该顺序执行，但是结果相同。

```
$ pingpong
4: received ping
3: received pong
$
```

## primes (moderate) / (hard)
> Write a concurrent version of prime sieve using pipes. This idea is due to Doug McIlroy, inventor of Unix pipes. The picture halfway down [this page](http://swtch.com/~rsc/thread/) and the surrounding text explain how to do it. Your solution should be in the file user/primes.c.

十分好玩的一道题hhhhh，使用多进程和管道，每一个进程作为一个 stage，筛掉某个素数的所有倍数（筛法）。很巧妙的形式实现了多线程的筛法求素数。具体原理和流程可以看[课程提供的这篇文章](http://swtch.com/~rsc/thread/)。

```
主进程：生成 n ∈ [2,35] -> 子进程1：筛掉所有 2 的倍数 -> 子进程2：筛掉所有 3 的倍数 -> 子进程3：筛掉所有 5 的倍数 -> .....
```
每一个 stage 以当前数集中最小的数字作为素数输出（每个 stage 中数集中最小的数一定是一个素数，因为它没有被任何比它小的数筛掉），并筛掉输入中该素数的所有倍数（必然不是素数），然后将剩下的数传递给下一 stage。最后会形成一条子进程链，而由于每一个进程都调用了 `wait(0);` 等待其子进程，所以会在最末端也就是最后一个 stage 完成的时候，沿着链条向上依次退出各个进程。

```c
// primes.c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

// 一次 sieve 调用是一个筛子阶段，会从 pleft 获取并输出一个素数 p，筛除 p 的所有倍数
// 同时创建下一 stage 的进程以及相应输入管道 pright，然后将剩下的数传到下一 stage 处理
void sieve(int pleft[2]) { // pleft 是来自该 stage 左端进程的输入管道
	int p;
	read(pleft[0], &p, sizeof(p)); // 读第一个数，必然是素数
	if(p == -1) { // 如果是哨兵 -1，则代表所有数字处理完毕，退出程序
		exit(0);
	}
	printf("prime %d\n", p);

	int pright[2];
	pipe(pright); // 创建用于输出到下一 stage 的进程的输出管道 pright

	if(fork() == 0) {
		// 子进程 （下一个 stage）
		close(pright[1]); // 子进程只需要对输入管道 pright 进行读，而不需要写，所以关掉子进程的输入管道写文件描述符，降低进程打开的文件描述符数量
		close(pleft[0]); // 这里的 pleft 是*父进程*的输入管道，子进程用不到，关掉
		sieve(pright); // 子进程以父进程的输出管道作为输入，开始进行下一个 stage 的处理。

	} else {
		// 父进程 （当前 stage）
		close(pright[0]); // 同上，父进程只需要对子进程的输入管道进行写而不需要读，所以关掉父进程的读文件描述符
		int buf;
		while(read(pleft[0], &buf, sizeof(buf)) && buf != -1) { // 从左端的进程读入数字
			if(buf % p != 0) { // 筛掉能被该进程筛掉的数字
				write(pright[1], &buf, sizeof(buf)); // 将剩余的数字写到右端进程
			}
		}
		buf = -1;
		write(pright[1], &buf, sizeof(buf)); // 补写最后的 -1，标示输入完成。
		wait(0); // 等待该进程的子进程完成，也就是下一 stage
		exit(0);
	}
}

int main(int argc, char **argv) {
	// 主进程
	int input_pipe[2];
	pipe(input_pipe); // 准备好输入管道，输入 2 到 35 之间的所有整数。

	if(fork() == 0) {
		// 第一个 stage 的子进程
		close(input_pipe[1]); // 子进程只需要读输入管道，而不需要写，关掉子进程的管道写文件描述符
		sieve(input_pipe);
		exit(0);
	} else {
		// 主进程
		close(input_pipe[0]); // 同上
		int i;
		for(i=2;i<=35;i++){ // 生成 [2, 35]，输入管道链最左端
			write(input_pipe[1], &i, sizeof(i));
		}
		i = -1;
		write(input_pipe[1], &i, sizeof(i)); // 末尾输入 -1，用于标识输入完成
	}
	wait(0); // 等待第一个 stage 完成。注意：这里无法等待子进程的子进程，只能等待直接子进程，无法等待间接子进程。在 sieve() 中会为每个 stage 再各自执行 wait(0)，形成等待链。
	exit(0);
}
```

这一道的主要坑就是，stage 之间的管道 pleft 和 pright，要注意关闭不需要用到的文件描述符，否则跑到 n = 13 的时候就会爆掉，出现读到全是 0 的情况。
```
prime 2
prime 3
prime 5
prime 7
prime 11
prime 13
prime 0
prime 0
prime 0
prime 0
prime 0
prime 0
...
```

这里的理由是，fork 会将父进程的所有文件描述符都复制到子进程里，而 xv6 每个进程能打开的文件描述符总数只有 16 个 （见 `defs.h` 中的 `NOFILE` 和 `proc.h` 中的 `struct file *ofile[NOFILE];  // Open files`）。

由于一个管道会同时打开一个输入文件和一个输出文件，所以**一个管道就占用了 2 个文件描述符**，并且复制的**子进程还会复制父进程的描述符**，于是跑到第六七层后，就会由于最末端的子进程出现 16 个文件描述符都被占满的情况，导致新管道创建失败。

解决方法有两部分：
* 关闭管道的两个方向中不需要用到的方向的文件描述符（在具体进程中将管道变成只读/只写）  
  > 原理：每个进程从左侧的读入管道中**只需要读数据**，并且**只需要写数据**到右侧的输出管道，所以可以把左侧管道的写描述符，以及右侧管道的读描述符关闭，而不会影响程序运行 
  > 这里注意文件描述符是进程独立的，在某个进程内关闭文件描述符，不会影响到其他进程
* 子进程创建后，关闭父进程与祖父进程之间的文件描述符（因为子进程并不需要用到之前 stage 的管道）

具体的操作在上面代码中有体现。（fork 后、执行操作前，close 掉不需要用掉的文件描述符）
```
$ primes
prime 2
prime 3
prime 5
prime 7
prime 11
prime 13
prime 17
prime 19
prime 23
prime 29
prime 31
$
```

## find (moderate)
> Write a simple version of the UNIX find program: find all the files in a directory tree with a specific name. Your solution should be in the file user/find.c.

这里基本原理与 ls 相同，基本上可以从 ls.c 改造得到：

```c
// find.c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include "kernel/fs.h"

void find(char *path, char *target) {
	char buf[512], *p;
	int fd;
	struct dirent de;
	struct stat st;

	if((fd = open(path, 0)) < 0){
		fprintf(2, "find: cannot open %s\n", path);
		return;
	}

	if(fstat(fd, &st) < 0){
		fprintf(2, "find: cannot stat %s\n", path);
		close(fd);
		return;
	}

	switch(st.type){
	case T_FILE:
		// 如果文件名结尾匹配 `/target`，则视为匹配
		if(strcmp(path+strlen(path)-strlen(target), target) == 0) {
			printf("%s\n", path);
		}
		break;
	case T_DIR:
		if(strlen(path) + 1 + DIRSIZ + 1 > sizeof buf){
			printf("find: path too long\n");
			break;
		}
		strcpy(buf, path);
		p = buf+strlen(buf);
		*p++ = '/';
		while(read(fd, &de, sizeof(de)) == sizeof(de)){
			if(de.inum == 0)
				continue;
			memmove(p, de.name, DIRSIZ);
			p[DIRSIZ] = 0;
			if(stat(buf, &st) < 0){
				printf("find: cannot stat %s\n", buf);
				continue;
			}
			// 不要进入 `.` 和 `..`
			if(strcmp(buf+strlen(buf)-2, "/.") != 0 && strcmp(buf+strlen(buf)-3, "/..") != 0) {
				find(buf, target); // 递归查找
			}
		}
		break;
	}
	close(fd);
}

int main(int argc, char *argv[])
{
	if(argc < 3){
		exit(0);
	}
	char target[512];
	target[0] = '/'; // 为查找的文件名添加 / 在开头
	strcpy(target+1, argv[2]);
	find(argv[1], target);
	exit(0);
}
```

```
$ find . b
    ./b
    ./a/b
```

## xargs (moderate)
> Write a simple version of the UNIX xargs program: read lines from the standard input and run a command for each line, supplying the line as arguments to the command. Your solution should be in the file user/xargs.c.

编写 xargs 工具，从标准输入读入数据，将每一行当作参数，加入到传给 xargs 的程序名和参数后面作为额外参数，然后执行。

```c
// xargs.c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include "kernel/fs.h"

// 带参数列表，执行某个程序
void run(char *program, char **args) {
	if(fork() == 0) { // child exec
		exec(program, args);
		exit(0);
	}
	return; // parent return
}

int main(int argc, char *argv[]){
	char buf[2048]; // 读入时使用的内存池
	char *p = buf, *last_p = buf; // 当前参数的结束、开始指针
	char *argsbuf[128]; // 全部参数列表，字符串指针数组，包含 argv 传进来的参数和 stdin 读入的参数
	char **args = argsbuf; // 指向 argsbuf 中第一个从 stdin 读入的参数
	for(int i=1;i<argc;i++) {
		// 将 argv 提供的参数加入到最终的参数列表中
		*args = argv[i];
		args++;
	}
	char **pa = args; // 开始读入参数
	while(read(0, p, 1) != 0) {
		if(*p == ' ' || *p == '\n') {
			// 读入一个参数完成（以空格分隔，如 `echo hello world`，则 hello 和 world 各为一个参数）
			*p = '\0';	// 将空格替换为 \0 分割开各个参数，这样可以直接使用内存池中的字符串作为参数字符串
						// 而不用额外开辟空间
			*(pa++) = last_p;
			last_p = p+1;

			if(*p == '\n') {
				// 读入一行完成
				*pa = 0; // 参数列表末尾用 null 标识列表结束
				run(argv[1], argsbuf); // 执行最后一行指令
				pa = args; // 重置读入参数指针，准备读入下一行
			}
		}
		p++;
	}
	if(pa != args) { // 如果最后一行不是空行
		// 收尾最后一个参数
		*p = '\0';
		*(pa++) = last_p;
		// 收尾最后一行
		*pa = 0; // 参数列表末尾用 null 标识列表结束
		// 执行最后一行指令
		run(argv[1], argsbuf);
	}
	while(wait(0) != -1) {}; // 循环等待所有子进程完成，每一次 wait(0) 等待一个
	exit(0);
}
```

上程序用到了一些指针与C字符串构成的取巧用法。

# Optional challenges

额外 challenge，不是满分必要条件，会挑选有意思的或有意义的做。

## uptime (easy)

> Write an uptime program that prints the uptime in terms of ticks using the uptime system call. (easy)

```c
// uptime.c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int main() {
	printf("%d\n", uptime());
	exit(0);
}
```

## regex for find (easy)

> Support regular expressions in name matching for find. grep.c has some primitive support for regular expressions. (easy)

从 grep.c 把 match 函数抄过来：

```c
// find.c
int matchhere(char*, char*);
int matchstar(int, char*, char*);

int match(char *re, char *text) {
	if(re[0] == '^')
		return matchhere(re+1, text);
	do{  // must look at empty string
		if(matchhere(re, text))
		return 1;
	}while(*text++ != '\0');
	return 0;
}

// matchhere: search for re at beginning of text
int matchhere(char *re, char *text) {
	if(re[0] == '\0')
		return 1;
	if(re[1] == '*')
		return matchstar(re[0], re+2, text);
	if(re[0] == '$' && re[1] == '\0')
		return *text == '\0';
	if(*text!='\0' && (re[0]=='.' || re[0]==*text))
		return matchhere(re+1, text+1);
	return 0;
}

// matchstar: search for c*re at beginning of text
int matchstar(int c, char *re, char *text)
{
	do{  // a * matches zero or more instances
		if(matchhere(re, text))
			return 1;
	}while(*text!='\0' && (*text++==c || c=='.'));
	return 0;
}
```

再将匹配规则改为用 match 匹配即可：
```c
// find.c
// ....
switch(st.type){
case T_FILE:
	if(match(target, path)) {
		printf("%s\n", path);
	}
	break;
case T_DIR:
// ....
```

## improved shell
* not print a $ when processing shell commands from a file (moderate) - 完成

改进前（输出了很多多余 $）：
```
$ cat xargstest.sh | sh
$ $ $ $ $ $ hello
hello
hello
$ $ 
```
改进后：
```
$ sh xargstest.sh
hello
hello
hello
$
```

* modify the shell to support wait (easy) - 完成

```
$ wait 20
(wait 20 ticks with no output...)
$ 
```

* modify the shell to support lists of commands, separated by ";" (moderate) - 完成

```
$ echo hello; echo world; echo 2333333!
hello
world
2333333!
$ 
```

* modify the shell to support sub-shells by implementing "(" and ")" (moderate) - 跳过
* modify the shell to support tab completion (easy) - 完成

```
$ sh xar [tab][回车]
auto-completed: xargstest.sh

hello
hello
hello
$ ec [tab][空格] hello,world! [回车]
auto-completed: echo

hello,world!
```

* modify the shell to keep a history of passed shell commands (moderate) - 跳过

shell 代码过长，已经放到 GitHub: 

> 修改记录：https://github.com/Miigon/my-xv6-labs-2020/commit/5f91ae357e5dbc031e4164e13141e6096596656d#diff-c5682e6f79d8e68b805047fc80c703adb4dbb0b972fa009bdfed1ea69dddd93f
> 完整文件：https://github.com/Miigon/my-xv6-labs-2020/blob/5f91ae357e5dbc031e4164e13141e6096596656d/user/sh.c

主要点在，系统提供的 gets 不足以满足我们的需求，所以将系统的 gets 实现拷贝到 sh.c 中，然后进行改进（支持对 \t 的检测）。
然后就是自动补全的实现，使用与 ls.c 相同的目录遍历逻辑，然后将前缀匹配的文件名自动替换到当前输入缓冲区内，实现自动补全。
从文件执行 shell 脚本，由于 `cat foobar.sh | sh` 的形式，shell 收到的指令来自标准输入（无法分辨是来自文件还是来自用户输入），故加入一个参数，输入要执行的脚本文件名，然后另外打开该脚本执行，并判断不输出 `$`。