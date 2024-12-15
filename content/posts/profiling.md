---
categories:
  - profiling
tags:
  - Go
title: "Profiling Hidden Bottlenecks: Learning Through Deliberate Code Regressions"
slug: "profiling-deliberate-code-regressions"
date: 2024-12-15T15:44:13+05:30
draft: false
---

Reducing unknown-unknowns is a unique challenge—after all, how can you tackle something you don’t even know exists? It’s like looking for a shadow in the dark. You can start by reading up on new topics, but that only takes you so far. The information is limited by what’s already known and documented. Profiling, on the other hand, is a hands-on way to dive into the fine details of your system, bringing hidden problems to light.

But let’s be honest—profiling can be intimidating. It’s often like opening Pandora’s box, with more complexity than you bargained for. After spending way too much time profiling legacy code and failing to understand what each code invocation does, I came up with a better structure (at least for me) for learning how to profile: **deliberately introducing performance regressions into code.**

## The Counterintuitive Approach

It sounds counterintuitive, but by intentionally degrading your system, you can learn how these changes manifest in your profiling data. Think of it as an experimental framework—an active way to uncover those minute, unseen issues before they spiral out of control. It’s a bold approach, but when the unknowns feel overwhelming, boldness is sometimes the best way forward.

I decided to test this idea on code I wrote for a technical hiring coding challenge. The task was to implement a key-value store with persistence—a seemingly straightforward challenge. I went with a barebones approach, and naturally, the first solution that popped into my head was to persist the entire hashmap to disk with every SET operation.

Deep down, I knew this approach was sketchy at best. Saving the entire file to disk on every operation? Talk about a performance bottleneck! Under pressure, I scrapped that idea and implemented a binlog-style persistence method, confident enough to defend it during the evaluation.

Recently, while exploring the idea of deliberate code regressions, I decided to revisit this old code. I reopened my `gokvstore` repo and rewrote my binlog approach back to the original full-save-to-disk approach—just to see how it would affect the profiling data.

And guess what? It completely challenged my assumptions.

## Revisiting the Full-Save Approach

This exercise helped me dig deeper into the actual performance impact of my design choices, forcing me to rethink what "shady" really means in terms of code efficiency. Here's the simplest implementation of the `SET` operation:

```go
func Set(key, value string) {
    mutex.Lock()
    hashmap[key] = value
    saveToFile(hashmap)
    mutex.Unlock()
}
```

The idea is basic: lock the mutex on the hashmap, set the value, save the data to disk, and unlock. The devil, however, is in the details. Here’s my `saveToFile` implementation:

```go
func saveToFile(store map[string]string) {
    lines := []string{}
    for key, value := range store {
        lines = append(lines, key+" "+value)
    }
    os.WriteFile("store.txt", []byte(strings.Join(lines, "\n")), 0644)
}
```

### The Bottleneck

At first glance, the bottleneck seems obvious: the `os.WriteFile` call. Writing an entire hashmap to disk on every `SET` is painfully slow, especially as the size of the store grows. Before jumping into optimizations, I outlined a few potential strategies:

- **Buffered Writes**: Write data in chunks rather than all at once.
- **Append Mode**: Only append the new key-value pair to the file.
- **Compression**: Use compression to reduce the size of data written to disk.
- **Asynchronous Writes**: Offload file writes to a separate thread to avoid blocking the main flow.

These are tempting, but first, **let’s profile** this code and identify where the real time is being spent.

## Profiling the Application

I used Go’s standard `pprof` tool, exposing profiling data on port `6060`, and integrated it with **Grafana Phlare** to visualize flamegraphs. To simulate a real-world workload, I wrote a load test program with these parameters:

| Parameter         | Value             |
| ----------------- | ----------------- |
| Worker Goroutines | 100               |
| Operations        | 10,000 per worker |
| Key Length        | 6 characters      |
| Value Length      | 10 characters     |

![Flamegraph-4](/images/profiling-f4.png)

The profiling results were insightful. Between `13:38:30` and `13:41:15`, the application consumed a total of **3.60 minutes of compute time**. Here’s the breakdown:

- **2.46 minutes**: `main.HandleClient` (processing `SET` requests)
- **1.02 minutes**: `runtime.gcBGMarkWorker` (garbage collection)

Notably, **28%** of the time was spent on garbage collection—an indicator of inefficiencies in memory management.

### The SET Operation Breakdown

Focusing on the `SET` operation:

![Flamegraph-5](/images/profiling-f5.png)

- Total time: **2.45 minutes**
- Time spent in `saveToFile`: **2.44 minutes**
  - Of that, `os.WriteFile`: **15.7 seconds**

Wait, what? The actual file-writing step accounted for less than 11% of the time. The real culprits were **string concatenation and slice growth**:

- `runtime.concatstring3`: **1.40 minutes** (string concatenation)
- `strings.Join`: **12 seconds**
- `runtime.growslice`: **17 seconds**

## Identifying Hidden Bottlenecks

The bottleneck lay in how I built the `lines` slice:

```go
lines = append(lines, key+" "+value)
```

Every time `key + " " + value` is concatenated, Go creates a new string, triggering memory allocation and garbage collection. Similarly, the slice growth, while efficient in amortized terms, added up due to frequent appends.

## Lessons Learned

This exercise highlighted several takeaways:

1. **Immutable Strings Are Expensive**: String concatenation in Go creates new objects, leading to significant overhead in loops.
2. **Profiling Reveals Unknowns**: Without profiling, I would have blamed the file-writing step, missing the real inefficiencies.
3. **Challenge Assumptions**: Profiling challenges what you think you know, surfacing "unknown-knowns" and true "unknown-unknowns."

## The Power of Profiling

Profiling is a powerful tool for uncovering hidden inefficiencies and challenging assumptions. By deliberately introducing regressions, you can better understand how changes manifest in your system.

Here’s the flamegraph for my binlog approach with the same load test parameters:

![Flamegraph-7](/images/profiling-f7.png)

This framework is a reminder that you can never fully predict how code will behave until you look under the hood. Profiling is essential for reducing unknowns and becoming a better engineer.

## References

1. [Go pprof Documentation](https://pkg.go.dev/net/http/pprof)
2. [String Concatenation Performance in Go](https://teivah.medium.com/string-concatenation-performance-in-go-7dd7db322fb3)
3. [Notes from the Architect - Red Hat Developers Blog](https://developers.redhat.com/blog)
4. [Using Pyroscope with Grafana](https://grafana.com/docs/grafana/latest/datasources/pyroscope/)

---

_PS: As of March 2023, Grafana has acquired Pyroscope, halting Phlare's development. RIP Phlare._
