---
title: Process vs Threads
date: 2026-05-04
categories: [Linux-Internals]
tags: [Linux-internals]
---
# Linux Internals — Process vs Thread
---

# Process

## Definition

A process is an independent running program with its own memory and resources.

## Features

* Separate address space
* Own PID
* Own heap, stack, globals
* Heavyweight

## Creation

```c
fork();
```

## Example

```bash
firefox
vim
gcc
```

---

# Thread

## Definition

A thread is a lightweight execution unit inside a process.

## Features

* Shares process memory
* Own stack + registers
* Lightweight
* Faster creation

## Creation

```c
pthread_create();
```

---

# Process vs Thread

| Feature       | Process  | Thread        |
| ------------- | -------- | ------------- |
| Memory        | Separate | Shared        |
| Stack         | Separate | Separate      |
| Heap          | Separate | Shared        |
| Communication | IPC      | Shared memory |
| Speed         | Slow     | Fast          |
| Isolation     | Strong   | Weak          |

---

# Shared Between Threads

```text
Code Segment
Heap
Global Variables
File Descriptors
```

---

# NOT Shared Between Threads

```text
Stack
Registers
Program Counter
```

---

# Linux Internal

Both processes and threads use:

```c
task_struct
```

Linux creates threads using:

```c
clone()
```

---

# Process Creation Example

```c
#include <stdio.h>
#include <unistd.h>

int main()
{
    pid_t pid = fork();

    if (pid == 0)
    {
        printf("Child Process\n");
    }
    else
    {
        printf("Parent Process\n");
    }
    while(1);
    return 0;
}
```

---

# Thread Creation Example

```c
#include <pthread.h>
#include <stdio.h>

void* func(void* arg)
{
    printf("Thread Running\n");
    return NULL;
}

int main()
{
    pthread_t t;

    pthread_create(&t, NULL, func, NULL);
    pthread_join(t, NULL);

    return 0;
}
```

Compile:

```bash
gcc file.c -o out -pthread
```

---

# Race Condition

## Problem

Multiple threads access shared resource simultaneously.

Example:

```c
count++;
```

## Fix

Use mutex.

```c
pthread_mutex_lock(&lock);
count++;
pthread_mutex_unlock(&lock);
```

---

# Mutex

Protect critical section.

## APIs

```c
pthread_mutex_init()
pthread_mutex_lock()
pthread_mutex_unlock()
pthread_mutex_destroy()
```

---

# Critical Section

Code accessing shared resource.

Example:

```c
count++;
```

---

# Context Switch

CPU switching between:

* processes
* threads

## Process switch

Heavy:

* page tables
* TLB flush

## Thread switch

Cheaper:

* same address space

---

# Process States

```text
Running
Ready
Sleeping
Stopped
Zombie
```

---

# Zombie Process

Child exits but parent does not call:

```c
wait();
```

# Important Commands

## Check processes

```bash
ps -ef
```

## Check threads

```bash
ps -eLf
```

## View thread IDs

```bash
ls /proc/<PID>/task/
```
