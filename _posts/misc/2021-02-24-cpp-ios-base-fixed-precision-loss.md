---
title: "问题分析：ios_base::fixed 导致输出精度丢失？"
date: 2021-02-24 20:08:00 +0800
categories: [Misc, Discussion]
tags: [C++, Chinese]
---

> 这篇文章是来自我在 [0xffff.one](https://0xffff.one/) 上的一个帖子 [https://0xffff.one/d/911/](https://0xffff.one/d/911/) 的回复。
>
> __原帖内容：__  
> 百度说是这行代码的作用是使用定点输出，同时输出小数点后6位(我试了好多数，仍然表示很迷)
>
> 为什么有这行代码有时候求两个数加减乘除的结果就不对，没有这行代码就对呢
> 
> 比如55.25+11.17有上面那行代码结果是66.419998，没有就是66.42；而20.5+10.5有无上面那行代码结果都正确.
> 
> 这是为什么😳 呢？

# How to research? How to approach?
我们注意到，导致输出不同的，是这样一行代码：
```c++
cout.setf(ios_base::fixed,ios_base::floatfield);
```
控制变量，这行代码干的事情就是我们的切入点。

可以看到，我们使用了 `setf`，对 floatfield 设置了一个 `fixed` 的 flag，那么这些就是我们搜索的关键词。

搜索 `setf`，我们得到：http://www.cplusplus.com/reference/ios/ios_base/setf/
> ...
The format flags of a stream __affect the way__ data is interpreted in certain input functions and __how it is written by certain output functions__. See [ios_base::fmtflags](http://www.cplusplus.com/reference/ios/ios_base/fmtflags/) for the possible values of this function's arguments.
>...

我们知道了 **format flags 是可以改变数据被显示的方式的**。

继续搜索 `fixed`：http://www.cplusplus.com/reference/ios/fixed/
> When `floatfield` is set to `fixed`, floating-point values are written using fixed-point notation: __the value is represented with exactly as many digits in the decimal part as specified by the precision field ([precision](http://www.cplusplus.com/reference/ios/ios_base/precision/))__ and with no exponent part.

precision！！！我们的问题看起来就是一个精度相关的问题！

看一下参考中关于 ios_base::precision 的部分：http://www.cplusplus.com/reference/ios/ios_base/precision/

> For the default locale:
Using the **default floating-point notation**, the precision field specifies the maximum number of **meaningful digits to display in total** counting both those before and those after the decimal point. Notice that it is ......
> 
> In both **the fixed** and scientific **notations**, the precision field specifies exactly **how many digits to display after the decimal point**, even if this includes trailing decimal zeros. The digits before the decimal point are not relevant for the precision in this case.

啊，破案了。

在默认的浮点输出模式下，precision 代表的是**精确到第几位有效数字**，而在 fixed （或scientific）的输出模式下，precision 代表的是**精确到小数点后第几位**。

# Solution

知道了这个事实，就可以很容易猜到这是同一个浮点数，输出时的 rounding 不同造成的区别，而不是由于精度丢失造成的区别。（精度丢失依然存在，只是在这里不是问题的直接原因）
```cpp
int main() {
    using namespace std;
    
    float result = 55.25f + 11.17f;
    cout << result << endl;    // 66.42
    
    cout.setf(ios_base::fixed,ios_base::floatfield);
    cout << result << endl;    // 66.419998

    return 0;
}
```
我们知道 C++ 默认的浮点输出精度是 6，但是这个 6 在不同的输出模式下有不同的含义。

在默认的浮点输出模式下，6 代表的是 **精确到6 位有效数字**，而在 fixed （或scientific）的输出模式下，6 代表的是 **精确到小数点后第 6 位**。

猜到了吗？

没错，其实对于计算机来说，由于精度不足，我们的数字是（且一直都是） 66.4199981689......，只是在默认的输出模式下，由于整数部分已经消耗掉了 6 位有效数字精度中的 2 位，只剩下 4 位有效数字给小数，因而小数点后只能精确到第四位。精确到小数点后第四位，就要看这个数字的第五位，在这个数字 66.4199__9__81689 中，第五位是个9，所以四舍五入就是 __66.4199__9 ≈ 66.42 了。

而采用固定小数点后位数的输出方式，精度的含义是，精确到小数点后 6 位。66.419998__1__689...... 的第七位是1，四舍五入舍去，留下 __66.419998__1  ≈ 66.419998 。

说到底，是因为不同的输出方式下，对 “精度”（ios_base::precision） 的理解不一样。

我们也可以很方便地验证我们的结论，只需对普通的方法设置一个 8 的精度即可：
```cpp
int main() {
    using namespace std;
    
    float result = (55.25f+11.17f);
    cout.precision(6);  // 6 位有效数字
    cout << result << endl;    // 66.42
    cout.precision(8);  // 8 位有效数字
    cout << result << endl;    // 66.419998
    
    cout.setf(ios_base::fixed,ios_base::floatfield);
    cout.precision(6);  // 小数点后 6 位
    cout << result << endl;    // 66.419998

    return 0;
}
```
至于 double？在 double 下，55.25 + 11.17 = 66.4200000000000017053025658242404460906982421875......
直到小数点后第15位才出现进度丢失，所以在两种显示方案下，无论小数点后是有 4 位精度还是 6 位精度，都会被四舍五入到 66.42。

所以这个问题总结起来是：在 float 存储时精度丢失的前提下，不同输出方案导致了输出时小数点后精确位数不同，进而导致 rounding 不同。

能够学到的：
1. 能 double 就尽量不要 <del>single</del> float
2. 如果应用场景不需要显示那么多位小数，就把 precision 设小点