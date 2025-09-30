---
title: "初试Zig"
date: 2025-09-30T11:07:10+08:00
draft: true
# weight: 1
# aliases: ["/first"]
tags: ["tech"]
author: "pangwa"
# author: ["Me", "You"] # multiple authors
showToc: false
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "一个初学者的视角"
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
听说Zig有一段时间了, 最近终于有机会尝试了一下, 这里记录一些初步的印象.

这次体验的方式是直接复刻一个Rust的Merkle tree实现到Zig, Rust版本的代码在[这里](https://github.com/literallymarvellous/merkle-tree-rs).

之所以选择这个例子, 主要是因为其代码量始终, 逻辑清晰, 依赖少, 同时也能测试大数据的场景. 同时Rust版本的设计相对清晰, 在复刻时也采用了类似的设计.

### 好的方面
1. 语法简洁, 学习曲线平缓, 很容易上手, 很接近C/Go的风格. 有defer, errdefer等语法糖, 比较便利.
2. 构建系统很棒
3. 编译速度很快, 远快于Rust
4. 编译产物小巧, 完整复刻的版本编译后的ReleaseFast的产物在Mac下只有160k左右, Linux下800k左右, 都动态链接了C库.
5. 性能非常好, 几乎没有做特别多性能优化, 近百万条数据的情况下, 性能就比Rust版本快了2.x倍, 内存占用略高(实在是不想手动优化内存占用了).
6. 支持文件内单元测试, 非常方便, 内存泄漏也可以在单测中检测出来
7. comptime 很好, 但这次没有用到.

### 不足的方面
1. 生态略欠缺, 缺少一些常用的库, 但可以和C库互通, 这点在很大程度上能够弥补生态的不足.
2. 版本不兼容变化较多, 极难找到相关文档, 通常需要去翻语言的源代码找到参考代码. 版本升级也导致有些库不能跟上最新的变化, 需要自己动手修改.
3. LLM不友好, LLM的知识版本较旧,很难写可以用在最新版本编译器的代码.
4. 对于需要动态分配内存的场景, 需要手动传入allocator, allocator无处不在, 有时候内存管理略显繁琐. 特写是手动写出有较多内存分配的代码会相对有挑战. 好在有单测,可以在测试时检测出内存泄漏的问题.

### 一些细节
1. 默认内存分配器的性能有点拉, 正式环境还是要换成c分配器, 对于大量小对象的场景, 性能提升明显, 对于Merkle tree这种大量小对象的场景, 性能提升有20倍以上的提升.

### 总结
整体来讲这次复刻的体验挺好, 由于编译速度较快可以很好的进行迭代, 语法也比较容易上手, 但库的一些语法变化着实浪费了一些时间去找相关代码. Zig的性能表现非常亮眼, 感觉它更加适合系统编程, 有些偏底层.

### 附
复刻的代码在[这里](https://github.com/pangwa/merkle-tree-zig). 欢迎提交PR ;-) .
