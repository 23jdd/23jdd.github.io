+++
date = '2026-06-23T20:08:44+08:00'
draft = false
title = '关于我'
+++
# Hey, I'm 23jdd 👋

> I build things that are fast, useful, and a little bit nerdy.

你好，我是 **23jdd**，一名喜欢把想法写成代码的开发者。

我主要使用 **Go**，也在持续探索 **Rust**、后端系统、开发者工具和终端交互。比起只让程序“跑起来”，我更在意它是否足够清晰、可靠，以及使用起来是否真的舒服。

```go
type Developer struct {
    Focus      []string
    Learning   []string
    Building   string
    Philosophy string
}

me := Developer{
    Focus:      []string{"Go", "Backend", "Developer Tools"},
    Learning:   []string{"Rust", "Systems", "Better Design"},
    Building:   "small tools with real value",
    Philosophy: "Stay curious. Ship things.",
}
```

## What I do

我喜欢研究那些藏在应用背后的东西：请求如何流动，任务如何调度，数据如何存储，系统如何在出错时继续保持可靠。

我的兴趣方向包括：

- 用 Go 构建后端服务、并发程序和基础设施工具；
- 学习 Rust，并探索类型系统、内存安全和系统编程；
- 设计 API、网络协议、消息系统与存储组件；
- 制作 CLI、TUI 和其他面向开发者的工具；
- 阅读源码，把模糊的概念变成可以运行的实现。

我享受从 `mkdir project` 开始，逐渐看到一个想法拥有结构、界面和生命的过程。

## What I'm building

最近，我在构建 **ged**：一个使用 Go 和 Star 编写的终端代码编辑器。

它从一个简单的文本缓冲区开始，逐步拥有了文件树、读写模式、模糊搜索、命令行、语法高亮，以及基于 LSP 的智能补全、诊断、文档和自动导包。

这个项目对我来说不只是一款编辑器，也是一次完整的工程练习：

```text
keyboard input
      ↓
editor state ─── filesystem
      │
      ├────────── shell commands
      │
      └────────── language server
      ↓
terminal UI
```

我喜欢这样的项目——它既足够小，让人可以理解每一个部分；又足够复杂，迫使我认真思考架构、交互和边界条件。

## How I think about code

### Make it work

先让想法变成可以运行的东西。真实的程序比停留在脑中的完美设计更有价值。

### Make it clear

代码首先是写给人看的。好的抽象应该减少理解成本，而不是只让实现看起来更“高级”。

### Make it reliable

错误处理、测试和边界条件不是最后才补上的装饰，它们本身就是功能的一部分。

### Keep learning

技术会变化，工具会变化，今天熟悉的答案也可能过时。保持好奇，比记住所有答案更重要。

## Beyond the code

写代码之外，我也喜欢记录构建过程。

我会在这里分享：

- 项目从想法到实现的过程；
- Go 与 Rust 的学习笔记；
- 后端、网络和系统设计中的实践；
- 开发工具的设计与使用体验；
- 那些看起来微小、实际上很有意思的技术细节。

这里不会只有“正确答案”。也会有失败的方案、被推翻的设计，以及我为什么改变想法。

因为真正的成长，往往发生在代码第一次无法工作的时候。

## Current status

```text
🟢 Building useful things
🟡 Learning Rust deeply
🔵 Exploring systems and developer tools
🟣 Open to interesting ideas and collaboration
```

## Let's connect

如果你也喜欢 Go、Rust、后端系统、开源项目或终端工具，欢迎来聊。

- GitHub: [github.com/23jdd](https://github.com/23jdd)
- Email: `your-email@example.com` <!-- Replace with your public email -->
- Blog: `your-blog.example.com` <!-- Replace with your blog URL -->

你也可以只留下一句：

> “Hey, I saw what you built.”

这通常就是一次有趣交流的开始。

---

<div align="center">

### Build. Break. Learn. Repeat.

**Thanks for stopping by.**

</div>


### 文章推荐
- [深入理解 HyperLogLog：从抛硬币到UV统计](https://23jdd.github.io/uv)
- [为什么 `go install .` 会提示 `Use -buildvcs=false](https://23jdd.github.io/buildvcs)
- [ged：一个终端代码编辑器](https://23jdd.github.io/ged)
---

