---
title: "[Js] Js变量初始化之使用默认值"
date: 2023-03-08T18:34:15+08:00
draft: true
# weight: 1
# aliases: ["/first"]
tags: ["tech", "js"]
author: "pangwa"
# author: ["Me", "You"] # multiple authors
showToc: false
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "使用默认值初始的一些技巧."
canonicalURL: "https://canonical.url/to/page"
disableHLJS: true # to disable highlightjs
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

## 关于对象引用

在使用 js 时，对一些变量初始时可能需要使用默认值初始化, 比如

```js
const DEFAULT_PREF = {
  connectionTimeout: 1000,
  idleTimeout: 10_000,
};

const newConnPref = DEFAULT_PREF;
```

这种方式有一点点小问题，比如如果需要修改`newConnPref`变量，则有可能发生一些预料之外的问题, 比如：

```js
const DEFAULT_PREF = {
  connectionTimeout: 1000,
  idleTimeout: 10_000,
};

const newConnPref = DEFAULT_PREF;
newConnPref.idleTimeout = 50_000; // 注意这里会同时改变DEFAULT_PREF 对象!
console.log(DEFAULT_PREF, newConnPref);
```

这是因为在 js 中对象的赋值是按引用传递的，在上面的代码中， `DEFAULT_PREF` 与 `newConnPref` 都指向了同一个对象， 所以在修改 `newConnPref` 对象的时候，其实修改的是 `newConnPref` 指向的对象，由于 `DEFAULT_PREF` 也同时发生了改变。

![对象引用示意](/images/2023/js-object-ref.png)

### 解决方法

在修改`newConnPref` 的时候可以使用对象复制的方式进行赋值， 比如：

```js
let newConnPref = DEFAULT_PREF;
newConnPref = {
  ...newConnPref,
  idleTimeout,
};
```

这种方式对上面的例子工作良好， 因为我们在修改 newConnPref 中的属性时，先对 newConnPref 对象做了一个复制。

## 新的问题， 多级对象

可以上面的代码还有一点点小问题，众所周知，js 中的`...`操作符进行的是对象的浅复制，这意味着，如果被复制对象有多级结构时， 使用`...`操作符复制会产生意想不到的问题， 比如:

```js
const DEFAULT_PREF = {
  connectionTimeout: 1000,
  idleTimeout: 10_000,
  auth: {
    user: "user",
    password: "password",
  },
};

let newConnPref = DEFAULT_PREF;
newConnPref = {
  ...newConnPref,
  idleTimeout: 50_000,
};

newConnPref.auth.user = "test_user";
console.log(DEFAULT_PREF, newConnPref);
```

可以看到， 虽然`DEFAULT_PREF`与`newConnPref`的 idleTimeout 不再相同， 但它们的 auth 对象也指向了同一个对象！所以`DEFAULT_PREF.auth.user`也发生了变化!

![嵌套对象引用示意](/images/2023/js-object-nested-ref.png)

## 解决思路

对于上面的问题，看来确保`newConnPref`一定要与`DEFAULT_PREF`完全不是一个对象才行，那么对对象使用深度复制是一个非常直观的方案，这里可以使用[lodash](http://lodash.com/)这个库的`_.cloneDeep`函数对对象进行深度复制， 比如:

```js
let newConnPref = _.cloneDeep(DEFAULT_PREF);
newConnPref.auth.user = "test_user";
```

由于`_.cloneDeep`会返回一个全新深度复制的对象，`newConnPref`指向的对象完全和`DEFAULT_PREF`不一样了，所以后续对 newConnPref 做任何修改也不会影响`DEFAULT_PREF`对象。

上面的方法已经工作的相当好了， 还有没有其他方法？

### 使用函数返回默认值

我们可以把上述代码改造一下， 使用一个函数返回默认值，比如：

```js
function getDefaultPref() {
  return {
    connectionTimeout: 1000,
    idleTimeout: 10_000,
    auth: {
      user: "user",
      password: "password",
    },
  };
}

const pref1 = getDefaultPref();
const pref2 = getDefaultPref();
pref1.auth.user = "user1";
pref2.auth.user = "user2";
console.log(pref1, pref2);
```

可以看到， 这种方式，`pref1`与`pref2`也完全指向不同的对象，对它们的修改并不会影响彼此。

### 比较

使用函数返回默认值对象与使用深拷贝本质上差别不大，目的都是构造对象的深度复制版本，对于使用函数返回默认的方法，js 的运行时可能进行一定的内部优化（比如写时复制技术等），而使用`_.cloneDeep`则保证了对象在第一时间深度复制，潜在性能上使用函数返回默认值有可能会在实际运行时更好一些（当然需要在频繁使用 getDefaultPref()的场景下才有区别，如果并不经常使用方法获取默认值，则区别不是特别大。）
