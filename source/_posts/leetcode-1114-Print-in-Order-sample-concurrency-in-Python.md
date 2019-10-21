---
title: 'leetcode 1114 Print in Order, sample concurrency in Python'
date: 2019-10-21 16:30:15
tags:
  - concurrency
  - Python
  - multithreading
categories:
  - Coding
  - Python
---

# leetcode 1114 Print in Order, sample concurrency in Python

## Problem

> Suppose we have a class:
>
> ```
> public class Foo {
>   public void first() { print("first"); }
>   public void second() { print("second"); }
>   public void third() { print("third"); }
> }
> ```
>
> The same instance of `Foo` will be passed to three different threads. Thread A will call `first()`, thread B will call `second()`, and thread C will call `third()`. Design a mechanism and modify the program to ensure that `second()` is executed after `first()`, and `third()` is executed after `second()`.



<!--more-->

Print in order is a simple concurrency programming problem, especially in Python. But it's my first concurrency program in Python, so record in here.



## Solution

I simply use **two locks** to achieve synchoronization.

```python
import threading
class Foo:
    lock1 = threading.Lock()
    lock2 = threading.Lock()
    def __init__(self):
        self.lock1.acquire()
        self.lock2.acquire()
        pass


    def first(self, printFirst: 'Callable[[], None]') -> None:
        
        # printFirst() outputs "first". Do not change or remove this line.
        printFirst()
        self.lock1.release()


    def second(self, printSecond: 'Callable[[], None]') -> None:
        self.lock1.acquire()
        # printSecond() outputs "second". Do not change or remove this line.
        printSecond()
        self.lock1.release()
        self.lock2.release()


    def third(self, printThird: 'Callable[[], None]') -> None:
        self.lock2.acquire()
        # printThird() outputs "third". Do not change or remove this line.
        printThird()
        self.lock2.release()
```

