# runtime-library
This is a round-robin scheduled, co-operative, M:N thread library written in C.

* Round-robin: Each thread is served as first come first served.
* co-operative: Tasks have to yield themselves, they are not preemptive scheduled.
* M:N threads: M lightweight threads (in this library tasks) scheduled on N POSIX threads.

This library is used by creating a lot of (N) tasks and then they are scheduled on top of a specified amount (M) POSIX threads. This should reduce the overhead of context switching, as the kernel isn't entered to make the switch.

## API

The library has an entry procedure called *rmain* for _run-time main_. *rmain* takes two parameters: *os_thread_count* and *initial*. The first argument is how many POSIX threads you want to create (I usually create as many as the amount of CPU cores I have). The second argument is the initial task (entry task) where from other subtasks can be spawned.

```c
void rmain(uint64_t os_thread_count, task_t *initial)
```


```c

```

```c

```