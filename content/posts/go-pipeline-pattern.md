---
title: "Go 并发模式：Pipeline"
date: 2026-04-15
tags: ["Go", "并发", "Pipeline"]
categories: ["技术"]
series: ["Go 并发"]
ShowToc: true
summary: "Go 语言的 Pipeline 并发模式详解——用 channel 串联多个处理阶段。"
---

## 什么是 Pipeline？

Pipeline 是一种并发模式，将任务拆成多个阶段（stage），每个阶段通过 **channel** 连接。

```
输入 → [Stage 1] → [Stage 2] → [Stage 3] → 输出
          ↑            ↑            ↑
       goroutine    goroutine    goroutine
```

## 基本示例

```go
func gen(nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        for _, n := range nums {
            out <- n
        }
        close(out)
    }()
    return out
}

func sq(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        for n := range in {
            out <- n * n
        }
        close(out)
    }()
    return out
}

func main() {
    // 串联两个阶段
    for n := range sq(sq(gen(2, 3))) {
        fmt.Println(n) // 16, 81
    }
}
```

## 扇出 / 扇入 (Fan-out / Fan-in)

当某个阶段计算量大时，可以启动多个 goroutine 并行处理：

```go
func merge(cs ...<-chan int) <-chan int {
    var wg sync.WaitGroup
    out := make(chan int)

    output := func(c <-chan int) {
        for n := range c {
            out <- n
        }
        wg.Done()
    }

    wg.Add(len(cs))
    for _, c := range cs {
        go output(c)
    }

    go func() {
        wg.Wait()
        close(out)
    }()
    return out
}
```

## 要点总结

1. **每个 stage 自己 close channel** — 谁生产谁关闭
2. **用 `range` 遍历 channel** — 自动处理关闭
3. **扇出要配合扇入** — 否则下游处理不过来
4. **用 `sync.WaitGroup` 协调** — 等待所有 goroutine 完成

Pipeline 模式让并发代码清晰、可组合，是 Go 并发编程的基础功。
