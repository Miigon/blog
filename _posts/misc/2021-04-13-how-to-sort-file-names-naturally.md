---
title: "Natural Sort: How to sort file names naturally"
date: 2021-04-13 13:33:00 +0800
categories: [Algorithm, Sorting]
tags: [javascript]
---

# What's the problem?

When a programmer is given the task of sorting file names in a list, it might be tempting to sort using something like `std::sort()`. The problem with that is: `std::sort()` sorts alphabetically. Suppose we have a list of file like this:

```
chapter1.txt
chapter10.txt
chapter2.txt
chapter3p1.txt
chapter3p2.txt
chapter3p10.txt
chapter11.txt
chapter20.txt
```

If we sort it alphabetically, aka, using `std::sort()` (or `Array.prototype.sort()` for JavaScript), what would happen?

```
chapter1.txt
chapter10.txt
chapter11.txt
chapter2.txt       # we expect chapter 2 to be in front of chapter10
chapter20.txt
chapter3p1.txt
chapter3p10.txt
chapter3p2.txt
```

We see that `chapter10` and `chapter11` have been placed before `chapter2`. This make sense for an alphabetically sorted list.
Both `10` and `11` is smaller than `2` since the first digit `1` is smaller than `2`. However, it does not follow our intuition.
As a human, we expect `2` to be bigger than `10`, not the other way around.

# Natural sort

This is where the algorithm of [natural sort](https://en.wikipedia.org/wiki/Natural_sort_order) (sometimes called alphanumerical sort) comes in handy. 

> Natural sort order is an ordering of strings in alphabetical order, except that multi-digit numbers are treated atomically, i.e., as if they were a single character. Natural sort order has been promoted as being more human-friendly ("natural") than the machine-oriented pure alphabetical order.

Pseudo-code for natural sort:
```
# Preprocessing
For every file name in list:
    processedName = []
    Scan for every character c in file name:
        If consecutive digits are found:
            Merge consecutive numerical characters into a single number
            processedName.Append(number)
        Else:
            # non-numerical character
            processedName.Append(c)
    list[current] = processedName  # use the new name as name for comparison

Sort list:
    Comparing file name x and y:
        For every cooresponding [xi in x] and [yi in y]:
            If xi == yi:
                Continue
            If both xi and yi are numbers:
                If xi < yi: x is smaller
                Else: y is smaller
            Else If both xi and yi are strings:
                If xi < yi: x is smaller
                Else: y is smaller
            Else: # One of xi or yi is a number and the other is string
                If xi is number: x is smaller
                Else: y is smaller

# the list is now sorted naturally (or alphanumerically)!
```

After applying the algorithm, the list should look like this:

```
chapter1.txt
chapter2.txt
chapter3p1.txt
chapter3p2.txt
chapter3p10.txt
chapter10.txt
chapter11.txt
chapter20.txt
```
The number rigit after `chapter` is sorted correctly as well as the second number after `chapter3p`.

# External resources

This blog only described a simplified version of natural sort. Some file manager might ignore leading and trailing spaces when sorting. Multiple consecutive spaces might be considered as one single space as well. However the overall algorithm did not change and all of these features are quite trivial to implement.

Checkout these resources:
https://blog.codinghorror.com/sorting-for-humans-natural-sort-order/
https://rosettacode.org/wiki/Natural_sorting

# Implementation in JavaScript
```js
// natural sort algorithm in JavaScript by Miigon.
// 2021-03-30
// 
// GitHub: https://github.com/miigon/

function natSort(arr){
    return arr.map(v=>{ // split string into number/ascii substrings
        let processedName = []
        let str = v
        for(let i=0;i<str.length;i++) {
            let isNum = Number.isInteger(Number(str[i]));
            let j;
            for(j=i+1;j<str.length;j++) {
                if(Number.isInteger(Number(str[j]))!=isNum) {
                    break;
                }
            }
            processedName.push(isNum ? Number(str.slice(i,j)) : str.slice(i,j));
            i=j-1;
        }
        // console.log(processedName);
        return processedName;

    }).sort((a,b) => {
        let len = Math.min(a.length,b.length);
        for(let i=0;i<len;i++) {
            if(a[i]!=b[i]) {
                let isNumA = Number.isInteger(a[i]);
                let isNumB = Number.isInteger(b[i]);
                if(isNumA && isNumB) {
                    return a[i]-b[i];
                } else if(isNumA) {
                    return -1;
                } else if(isNumB) {
                    return 1;
                } else {
                    return a[i]<b[i] ? -1 : 1 ;
                }
            }
        }
        // in case of one string being a prefix of the other
        return a.length - b.length;
    }).map(v => v.join(''));
}

let a = ['a2','a1b10z','b1a2','a1b10','a33','7','a3','a22','a1b2','abbbb','a1b1','aaaaa','a10','a1','10'];

console.log(natSort(a).join('\n'))
/*
    7
    10
    a1
    a1b1
    a1b2
    a1b10
    a1b10z
    a2
    a3
    a10
    a22
    a33
    aaaaa
    abbbb
    b1a2
*/
```
