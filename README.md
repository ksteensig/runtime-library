# Run-time Library
**This is a round-robin scheduled, co-operative, M:N thread library written in C.**

* Round-robin: Each thread is served as first come first served.
* co-operative: Tasks have to yield themselves, they are not preemptive scheduled.
* M:N threads: M lightweight threads (in this library tasks) scheduled on N POSIX threads.

This library is used by creating a lot of (N) tasks and then they are scheduled on top of a specified amount (M) POSIX threads. This should reduce the overhead of context switching, as the kernel isn't entered to make the switch.

## API

The library has an entry procedure called *rmain* for _run-time main_. *rmain* takes two parameters: *os_thread_count* and *initial*. The first argument is how many POSIX threads you want to create (I usually create as many as the amount of CPU cores I have). The second argument is the initial task (entry task) where from other subtasks can be spawned.

```c
void rmain(uint64_t os_thread_count, task_t *initial)
```

To create a task there are two different functions that can be used. The first one is *taskcreate* which takes to arguments. The first argument is a function pointer to the procedure which the task should run. The second argument is a void pointer to the argument for the procedure. There can only be given one argument, so if you wish to pass multiple values to the procedure, then you have to wrap them in a struct. The procedure cannot return any values either, so the struct wrapping the parameters also has to be used for output parameters of the procedure.

The second function is *taskalloc*, which takes an additional argument. The two first arguments are the same as *taskcreate*, but the third argument is used to tell how big the stack of the procedure should be. **Bear in mind this is a static number which the system doesn't dynamically increase on run-time.** *taskcreate* _just_ calls *taskalloc* with a fixed number as the stack. So for the best experience *taskalloc* should be used.

**IMPORTANT: the argument that the procedure actually gets when run is the task itself. But the task has a member which is the *arg* given to *taskcreate* or *taskalloc*.**

```c
task_t *taskcreate(void (*fn)(void*), void *arg);
task_t *taskalloc(void (*fn)(void*), void *arg, uint stack);
```

The task has the following structure. So the argument that the procedure actually gets is an instance of this task handle. The *arg* passed to *taskcreate* or *taskalloc* is the member *startarg*.

```c
typedef struct task_s {
    context_t	 context;           // the context of the task
    task_state_t state;             // which state the task is in (waiting, running, new, etc)
    uint 	scheduler_id;           // the id of the scheduler executing the task right now
	uint 	sub_tasks_alloc;	    // how many subtasks there are allocated for
    uint    sub_task_len;           // number of sub tasks
    struct  task_s  **sub_tasks;    // array of subtask references
    uchar	*stk;                   // legacy shit for checking if task is running out of stack
    void	(*startfn)(void*);      // the function that the task should run
    void	*startarg;              // arguments for the task's function
} task_t;
```

The reason that the task handle is what's given to the procedure rather than the actual argument is because the task uses its identity to do a context switch (the tasks cooperatively scheduled). To do a context switch the procedure *taskyield* is used. The only parameter is the task which should yield.

```c
void taskyield(task_t *t);
```

To create a subtask you just use *taskcreate* or *taskalloc* and then use the procedure *subtaskadd*. It takes two arguments where the first is the task creating the subtask and the second is the subtask itself. This has to be used as the scheduler needs to know when all subtasks are done before it can safely deallocate subtasks.

```c
void subtaskadd(task_t *parent, task_t *subtask);
```

And then finally to exit a task the procedure *taskexit* is used. This only takes one argument which is the task which should exit. *taskexit* deallocates the stack of the subtasks and the task itself.

```c
void taskexit(task_t *t);
```

## Example

In this example I create a task to add two numbers, which creates a subtask to negate the first number, before adding the two numbers.

```c
typedef struct numbers_s {
    int a;
    int b;
    int res;
} numbers_t;

// negate a
void *neg_a(task_t *task) {
    numbers_t *args = (numbers_t *)task->startarg;

    args->a *= -1;

    taskexit(task);
}

// add a and b
void *add(task_t *task) {
    numbers_t *args = (numbers_t *)task->startarg;

    task_t *subtask = taskcreate(neg_a, task->startarg);
    subtaskadd(task, sub_neg_a);
    taskyield(task);

    // a has been negated at this point because neg_a has been run
    // add a and b and put into res
    args->res = args->a + args->b;

    // exit task
    taskexit(task);
}

void main() {
    number_t *ns = malloc(sizeof(numbers_t));

    ns->a = 5;
    ns->b = 2;

    task_t *initial = taskcreate(add, (void *)ns);

    rmain(2, initial);

    // should print -3
    printf("%d\n", ns->res);

    // we still have to manually free arguments
    free(ns);
}
```