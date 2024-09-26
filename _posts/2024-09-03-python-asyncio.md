---
layout: post
title:      "Python3 Concurrency: asyncio module/async/await"
date:       2024-09-03 09:00:00-0400
author:     zxy
math: true
categories: Coding Python
tags: Python
---


### 基本概念

- **`coroutine`**: 协程可以在函数到达 `return` 之前暂停执行，并将控制权间接传递给另一个协程。

- **`asyncio`**: 处理Python coroutine（协程）的包

  - async IO is a single-threaded, single-process design: it uses **cooperative multitasking**
  - perfect for IO-bound tasks (network, disk, database)
  - ![overview](/assets/img/in-post/2024-09-05-asyncio.png)


- **`async`/`await`**: 用来定义coroutine的Python keywords

  - `await`: 将函数控制权交还给事件循环，并且可以执行其他操作
    - 只能在coroutine中使用await，否则报错`SyntaxError` 
    - 只能await await able的对象，
      - 在 Python 中，可等待对象可以是：
        - **协程**：由 `async def` 函数创建。
        - **具有 `__await__()` 方法的对象**：该方法应返回一个迭代器。

  - `async def`:  introduces either a **native coroutine** or an **asynchronous generator**
    - It may use `await`, `return`, or `yield` in coroutine, but all of these are optional. Declaring `async def noop(): pass` is valid
      - Using `await` and/or `return` creates a coroutine function
      - use `yield` in an `async def` block. This creates an [asynchronous generator](https://www.python.org/dev/peps/pep-0525/)， which you iterate over with `async for`.

  - 主要用法 ,大多数程序会包含小型、模块化的协程，以及一个用于将这些小协程串联在一起的包装函数。

  ```python
  #!/usr/bin/env python3
  # countasync.py
  
  import asyncio
  
  async def count():
      print("One")
      await asyncio.sleep(1)
      print("Two")
  
  async def main():
      await asyncio.gather(count(), count(), count())
  
  if __name__ == "__main__":
      import time
      s = time.perf_counter()
      asyncio.run(main())
      elapsed = time.perf_counter() - s
      print(f"{__file__} executed in {elapsed:0.2f} seconds.")
  ```

- **`yield`**：遇到yield，函数执行会在当前代码处停止，并且会在下一次调用该函数时继续执行

  - pause the function and resume later

  - `yield`的主要用法是作为generator

    ```python
    # generator function
    def odds(start, end):
    	for odd in range(start, end+1, 2):
    		yield odd
    
    # ===================
    g = odds(3, 15)
    next(g) # return 3
    next(g) # return 5
    # Final Iteration: return StopIteration Exception
    # ===================
    g1 = odds(7, 13)
    list(g) # return [7, 9, 11, 13]
    # ===================
    g2 = odds(3, 21)
    for x in g2:
      print(x)
    ```


### 使用await/async构建生产者-消费者对列例子

- the key is to `await q.join()`, which blocks until all items in the queue have been received and processed, and then to cancel the consumer tasks, which would otherwise hang up and wait endlessly for additional queue items to appear

```python
#!/usr/bin/env python3
# asyncq.py

import asyncio
import itertools as it
import os
import random
import time

async def makeitem(size: int = 5) -> str:
    return os.urandom(size).hex()

async def randsleep(caller=None) -> None:
    i = random.randint(0, 10)
    if caller:
        print(f"{caller} sleeping for {i} seconds.")
    await asyncio.sleep(i)

async def produce(name: int, q: asyncio.Queue) -> None:
    n = random.randint(0, 10)
    for _ in it.repeat(None, n):  # Synchronous loop for each single producer
        await randsleep(caller=f"Producer {name}")
        i = await makeitem()
        t = time.perf_counter()
        await q.put((i, t))
        print(f"Producer {name} added <{i}> to queue.")

async def consume(name: int, q: asyncio.Queue) -> None:
    while True:
        await randsleep(caller=f"Consumer {name}")
        i, t = await q.get()
        now = time.perf_counter()
        print(f"Consumer {name} got element <{i}>"
              f" in {now-t:0.5f} seconds.")
        q.task_done()

async def main(nprod: int, ncon: int):
    q = asyncio.Queue()
    producers = [asyncio.create_task(produce(n, q)) for n in range(nprod)]
    consumers = [asyncio.create_task(consume(n, q)) for n in range(ncon)]
    await asyncio.gather(*producers)
    await q.join()  # Implicitly awaits consumers, too
    for c in consumers:
        c.cancel()

if __name__ == "__main__":
    import argparse
    random.seed(444)
    parser = argparse.ArgumentParser()
    parser.add_argument("-p", "--nprod", type=int, default=5)
    parser.add_argument("-c", "--ncon", type=int, default=10)
    ns = parser.parse_args()
    start = time.perf_counter()
    asyncio.run(main(**ns.__dict__))
    elapsed = time.perf_counter() - start
    print(f"Program completed in {elapsed:0.5f} seconds.")
```

## Reference

1. https://realpython.com/async-io-python/#the-asyncio-package-and-asyncawait
