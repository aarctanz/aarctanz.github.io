---
title: The anatomy of lightweight threads
description: Comparison of lightweight threads of elixir and golang
pubDate: 2026-03-03
heroImage: /go-elixir.png
---

> A thread is the smallest unit of execution within a process that can run independently while sharing the same memory and resources.

Let's talk about threads, not the native linux threads, but lightweight one. Virtual threads managed by language runtime engines, goroutines and Elixir BEAM processes.


A **goroutine** is a lightweight thread of execution that is managed by the Go runtime running in the same address space. To start a goroutine, we simply add `go` prefix to any function call
```go
go f(x,y)
```

The Go scheduler uses an M:N model: many goroutines are multiplexed over a smaller pool of OS threads. This makes it cheap(2KB) to spin up thousands or even millions of goroutines in the same process. Thread pools and OS scheduling is handled by runtime itself.
___

Elixir processes, running on the BEAM virtual machine, are
extremely lightweight, isolated, and concurrent units of execution with their own heap and garbage collector managed internally by the VM. The easiest way to create a function is with the `spawn` function.

```erlang
pid = spawn(fn ->
  IO.puts("Hello from process #{inspect(self())}")
end)
```

Instead of sharing memory, these processes are isolated by design. You can create thousands of them, and because they are isolated, a crash in one process does not directly corrupt others.

---

## Communication and synchronization

### Go: shared memory + channels

In Go, goroutines share the same address space. That means they can:

- Read and write to shared variables
- Use synchronization primitives to protect shared data from race condition:
    - Mutexes (`sync.Mutex`)
    - Atomic operations (`sync/atomic`)
    - WaitGroups, Cond, etc.
- Communicate via channels:

```go
ch := make(chan int)
go func() {   
	ch <- 42 
}() 
value := <-ch
```

Channels encourage a “share memory by communicating” style, but Go does not forbid shared mutable state. If you do share state directly, you must synchronize access to avoid data races.

### Elixir: message passing only

In Elixir (and Erlang), each process has its own heap and cannot access another process’s memory. The only way to communicate is by sending messages:

```erlang
send(pid, {:hello, self()}) 
receive do   
	{:hello, from} ->    
	IO.puts("Got hello from #{inspect(from)}") 
end
```

There are no mutexes or locks at the language level, because they aren’t needed – processes don’t share memory. Elixir doesn't allows shared mutable state making reasoning with concurrency simpler.

---

## Error handling and fault tolerance

### Go

In Go:

- A serious runtime error (like a nil pointer dereference) causes a panic.
- A panic, if not recovered, will bring down the whole program.
- You can recover from panics inside a goroutine using `defer` + `recover`, but you have to design that yourself.

Go gives you powerful building blocks (goroutines, channels, errors, panics), but it does not prescribe a fault‑tolerance strategy. Typically, resilience is handled at the process / container level (e.g., restart in Kubernetes).

## Elixir

Elixir leans heavily into fault tolerance:

- Processes are cheap and isolated.
- If a process crashes, it doesn’t take down the whole VM.
- Supervisors monitor processes and restart them according to configurable strategies.

This is the “let it crash” philosophy: write straightforward code, allow failures, and rely on supervision trees to keep the system healthy. It’s a very different mindset from trying to prevent all crashes inside a single long‑lived process.

---

## When to use which model

You can think of the two as optimized for slightly different goals:

- **Goroutines (Go)**
    
    - Great for high‑performance services where you want:
        - Simple syntax for concurrency  
        - Efficient use of CPU 
        - Direct access to shared memory when needed 
    - You keep more manual control over data structures and error handling.

- **Elixir processes (BEAM)**
    - Great for highly available systems where you want:
        - Strong isolation between units of work   
        - Built‑in supervision and restarts  
        - Pure message‑passing concurrency  
    - You trade away shared memory to gain safety and fault tolerance.

---
