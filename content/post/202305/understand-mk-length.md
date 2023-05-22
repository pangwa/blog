---
title: "理解The Little Schemer中的mk-length函数"
date: 2023-05-11T15:29:35+08:00
draft: true
# weight: 1
# aliases: ["/first"]
tags: ["tech", "lisp"]
author: "pangwa"
# author: ["Me", "You"] # multiple authors
showToc: false
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "有关lisp中递归的模式."
canonicalURL: "https://canonical.url/to/page"
disableShare: false
disableHLJS: false
hideSummary: false
searchHidden: true
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
editPost:
    URL: "https://github.com/pangwa/blog/blob/main/content"
    Text: "提交建议" # edit text
    appendFilePath: true # to append file path to Edit link
---

## 起

在*The Little Schemer*这本书的最后提出了一个有趣的问题：如何在不使用*define*的情况下，只使用*lambda*实现一个递归的计算数组长度的函数。

限制了使用*define*，意味着有几个限制：

1. 无法直接命名函数
2. 函数也无法通过函数名也引用自己

## 方法

在书中，给出来了一个实现， 以下为书中的代码：

```lisp
  ((lambda (mk-length)
     (mk-length mk-length))
   (lambda (mk-length)
     (lambda (l)
       (cond
         ((null? l) 0 )
         (else (add1
                ((mk-length mk-length)
                 (cdr l))))))))
```

这串代码看起来不好理解，这样是如何实现了一个递归计算列表长度的函数呢？

我们来逐步分析一下。

## 理解 (lambda (mk-length) (mk-length mk-length))

初见这个 lambda 函数的定义有些让人摸不到头脑，其实这个函数的定义非常简单，它的作用就是接受一个函数作为参数，然后调用这个函数并将这个函数自身作为参数传递给它。
但为什么要引入这么一个函数, 它的作用是什么呢？

我们回过来看题目要求，要求我们实现一个递归的函数，但是又不能使用*define*，那么我们就需要在函数内部引用自己，这个函数就是用来实现这个功能的。

即使我们不能使用*define*，但是我们可以使用*lambda*，我们可以将一个函数赋值给一个变量，然后在函数内部引用这个变量，这样就可以实现递归了。

假如对于以下函数，它是一个递归函数，这里我们先不考虑它的终止条件，只考虑它的递归部分。

```lisp
(define (my-rec)
 (my-rec))
```

我们需要将它改造成一个不使用*define*的函数。

```lisp
((lambda (mk-rec)
  (mk-rec mk-rec))
  (lambda (mk-rec2)
    ((mk-rec2 mk-rec2))))
```

这里引入了两个 lambda 函数， 第一个 lambda 函数的作用刚才已经提过，就是为了解决不能使用名字引用自己的问题，所以我们就可以将 mk-rec 作为参数传递给它，然后在函数内部调用 mk-rec。由第一个 lamda 函数的定义得知，这个参数中的*mk-rec*为一个接受一个函数作为参数的函数，所以我们需要定义一个函数，这个函数接受一个函数作为参数，然后调用这个函数并将这个函数自身作为参数传递给它，这样就可以实现递归了。

它的参数就是第二个 lambda 函数， 为了便于说明， 我们将其接收的参数为*mk-rec2*, 仔细想一下， 这里的*mk-rec2*在实际执行时它的值是什么呢？

对！ _mk-rec2_ 在执行时的值就是第二个 lambda 自己！ 这样就实现了不使用*define*来实现自己调用自己的方法！

注意在*mk-rec2*函数中，递归调用自己的方法为：((mk-rec2 mk-rec2))， 这里的*mk-rec2*是一个含有一个参数的函数，所以我们需要将它作为函数调用后返回一个不包含参数的函数，然后再调用此函数， 才能实现递归调用。

## 增加中止条件参数

下一步我们需要解决的是如何为这个递归函数添加终止条件， 在纯函数中， 如果要为函数增加终止条件，则需要为函数增加一个参数（多个参数和一个参数有本质区别吗？），通过这个参数的值来决定函数的执行是否需要终止。

为了给函数增加一个参数，意味着我们需要*mk-rec2* 返回一个接收中止条件参数的函数！ 即`(mk-rec2 mk-rec2)` 不再返回一个不接收参数的函数，而需要接收一个参数，那么*mk-rec2*将调整为:

```lisp
(lambda (mk-rec2)
(lambda (stop-condition)
  (if stop-condition
      0
      ((mk-rec2 mk-rec2) stop-condition))))
```

这里， 我们将此匿名函数调整为返回一个接收一个参数的函数，这个参数就是中止条件，如果中止条件为真，则返回 0，否则调用自身并将中止条件作为参数传递给它。
这里也应当注意，我们在调用自身时，需要将中止条件作为参数传递给它，这样才能实现递归调用, 即 `((mk-rec2 mk-rec2) stop-condition)`。

合并在一起即为:

```lisp
((lambda (mk-rec)
  (mk-rec mk-rec))
  (lambda (mk-rec2)
    (lambda (stop-condition)
      (if stop-condition
          0
          ((mk-rec2 mk-rec2) stop-condition)))))
```

## 理解`mk-length`函数

再来看`mk-length`函数的实现:

```lisp
  ((lambda (mk-length)
     (mk-length mk-length))
   (lambda (mk-length)
     (lambda (l)
       (cond
         ((null? l) 0 )
         (else (add1
                ((mk-length mk-length)
                 (cdr l))))))))
```

可见，其整体框架和`mk-rec`函数基本类似， 只是第二个内部匿名函数 `mk-length`增加了对列表的处理， 其接收的参数为一个列表， 判断停止条件为列表是否为空，如果为空则返回 0， 否则调用自身并将列表的尾部作为参数传递给它。
即将`mk-rec2`中的

```lisp
  (if stop-condition 0
   ((mk-rec2 mk-rec2) stop-condition))

```

调整为

```lisp
      (cond
         ((null? l) 0 )
         (else (add1
                ((mk-length mk-length)
                 (cdr l)))))
```

区别的地方在于 在 else 语句中 通过 add1 对结果进行了处理， 对于函数调用的参数使用了 `(cdr l)`， 而不是传递了原始参数。

## 总结

`mk-length`函数本质上还是递归函数的思路，只不过它限制了使用*define*，阻止了直接使用名字调用自己， 通过引入一个新的 lambda 函数来最终实现了调用自己的目的。

当然 lisp 的括号也在一定程度上阻碍了对代码的理解，何不试试用其他语言实现`mk-length` ;-)
