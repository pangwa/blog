---
title: "[ts] ts 进阶之范型中使用keyof约束"
date: 2023-03-30T21:16:02+08:00
draft: true
# weight: 1
# aliases: ["/first"]
tags: ["tech", "typescript"]
author: "pangwa"
# author: ["Me", "You"] # multiple authors
showToc: false
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "Desc Text."
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

## 场景描述

js 中通常会有一些辅助函数处理对数组中对象的转换工作, 有这么一个函数我们叫它`transformProp`, 它接收三个参数,

1. 一个包含了对象的数组
2. 一个属性名 `prop`
3. 一个转换函数, 它对数组中每一个对象的`prop`属性做一个转换, 并返回转换后的对象数组

下面是此函数的一个简单的 js 实现:

```js
function transformProp(objs, prop, mapper) {
  return objs.map((v) => {
    return {
      ...v,
      [prop]: mapper(v[prop]),
    };
  });
}
```

## 使用 typescript 开发一个安全的版本

首先我们以一个初步定义了第一个参数类型和 mapper 类型的 ts 函数开始

```ts
function transformProp<T, RP>(
  objs: T[],
  prop: string, // TODO: 修复prop 类型
  mapper: (v: any) => RP // TODO: v的类型?
): any[] {
  // 返回值的类型?
  return objs.map((v) => {
    return {
      ...v,
      [prop]: mapper(v[prop]),
    };
  });
}
```

这个函数的实现和 js 版本一致, 只不过在函数声明中增加了一些类型标注, 第一个参数接受一个元素类型为`T`的数组, 第二个参数我们暂时定义它为`string`类型, 第三个参数我们指定为一个函数类型, 但是函数的参数暂时给了`any`类型, 它的返回值类型为`RP`. `transformPop`函数的返回值类型也暂时写为`any`.

在 ts 版本, 我们将以下面的类型作为测试数据,验证类型定义的正确性:

```ts
interface Person {
  id: number;
  name: string;
  age: number;
}

const students: Person[] = [
  { id: 1, name: "alice", age: 18 },
  { id: 2, name: "bob", age: 15 },
];

const student_suffix = transformProp(students, "name", (n) => n + " cool");
const student_name_length = transformProp(students, "name", (n) => n.length);
const student_name_length = transformProp(students, "name2", (n) => n.length); //TODO: 这里应该报错!

const student_name_length2 = transformProp(
  students,
  "name",
  (n: number) => n + 1
); // TODO: 这里也应该报错, 因为mapper函数的参数应当与 name字段的类型相同

const std1 = student_suffix[0];
const b = std1.name; // TODO: b的类型为string
const age = std1.age; // age的类型应当为number
const c = student_name_length[0].name; // TODO: c的类型为number
```

我们第一个版本的 ts 代码中, 由于使用`string`和`any` 放宽了类型约束,导致`transformProp`函数在接收到不存在的属性值时并不能检测到错误, 如何让编译器强制约束`prop`参数必须为`T`的属性呢?

### 使用`keyof`约束`prop`参数为`T`的属性

在 ts 中`keyof T` 表示一个值为类型`T`的一个属性, 我们可以为`transformProp`函数增加一个新的类型参数`K` 作为`prop`的类型,并要求`K`满足`keyof T`约束.

```ts
function transformProp<T, K extends keyof T, RP>(
  objs: T[],
  prop: K,
  mapper: (v: any) => RP // TODO: v的类型?
);
```

而`mapper`函数要接收一个`T`类型中`K`字段的参数,所以它的类型就是 `T[K]`, 更正后的函数为:

```ts
function transformProp<T, K extends keyof T, RP>(
  objs: T[],
  prop: K,
  mapper: (v: T[K]) => RP
);
```

完善增加了`prop`与`mapper`类型后, 下面的两个函数调用的类型问题都可以被编译器检查出来:

```ts
const student_name_length = transformProp(students, "name2", (n) => n.length); //TODO: 这里应该报错!

const student_name_length2 = transformProp(
  students,
  "name",
  (n: number) => n + 1
); // TODO: 这里也应该报错, 因为mapper函数的参数应当与 name字段的类型相同
```

### 类型使用`Omit`和类型交叉(`intersection`)完善返回值类型

对于`transformProp`这个函数, 它的返回值类型应当是什么呢? 可以总结如下两点

1. 除了`prop`指定的属性外(即`K`), 其他属性与`T`类型一致
2. 它的`prop`指定的属性(即`K`)的类型应当与`mapper`的返回值类型`RP`一致.

所以我们可以将返回值类型写为: `Omit<T, K> & Record<K, RP>`, 这里我们使用`Omit`辅助类型从`T`中去除`K`属性, 然后再使用交叉类型`&`为此类型与一个包含属性名`K`,属性值类型为`RP`的记录类型合并.

完善后的`transformProp`函数如下:

```ts
function transformProp<T, K extends keyof T, RP>(
  objs: T[],
  prop: K,
  mapper: (v: T[K]) => RP
): (Omit<T, K> & Record<K, RP>)[] {
  return objs.map((v) => {
    return {
      ...v,
      [prop]: mapper(v[prop]),
    };
  });
}
```

此函数可以对上面所列举的几种代码都能检测出相应的错误, 同时对于函数的返回值类型也能精确的推断出来.
