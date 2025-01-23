---
title: Debugging Asynchronous Programming in AiiDA
date: '2025-01-25'
categories:
    - post
tags:
    - aiida 
    - aysnc
---

# Debugging Asynchronous Programming in AiiDA

Asynchronous allows program to scale better by switching tasks run on CPU while waiting on I/O, network operations, or other tasks that don’t require constant processing. 
However, debugging async code can be tricky in the context of **AiiDA**, where the event loop may be configured in unorthodox ways.

In this blog post, we will:

1. Introduce asynchronous programming in AiiDA.  
2. Explore why debugging async code is difficult.  
3. Discuss standard ways to debug Python async code.  
4. Show how these approaches fit (or sometimes do not fit) in AiiDA’s setup.  
5. Walk through a specific case study of using `aiomonitor` to find a coroutine stuck in the event loop.  
6. Look at possible avenues for improving async programming within AiiDA.

---

## 1. Async Programming in AiiDA

As part of its architecture, AiiDA employs **Plumpy** (a Python workflow library) under the hood. 
Plumpy relies on asynchronous mechanisms (Futures, event loops, coroutines) for handling complex workflows with potentially long-lived tasks.

Most Python libraries that deal with concurrency use the `asyncio` module. 
In AiiDA, we often launch tasks, orchestrate them across multiple processes, and schedule them on event loops that could be nested or re-entered through libraries like `nest-asyncio`. 
The combination of concurrency, external runners, and re-entrant event loops helps AiiDA remain responsive, but it can also create non-standard debugging situations.

---

## 2. Why Is Debugging Async Code Hard?

Debugging asynchronous code is tricky because:

1. **Execution Flow**: In async code, the flow of execution doesn’t follow a strict top-to-bottom, call-stack pattern like synchronous code. Coroutines can yield control back to the event loop at different times, making it harder to pinpoint where problems occur.

2. **Breakpoint Problems**: Traditional Python debugging (like `pdb.set_trace()`) often relies on a breakpoint that suspends all execution. But in async code, placing a breakpoint in the middle of a coroutine may not truly pause the entire event loop. Other coroutines can still be running in parallel, leading to inconsistent states when you inspect variables.

3. **Thread & Event Loop**: Python’s `asyncio` typically runs in a single thread, but AiiDA’s environment might spin up dedicated runners on separate threads or even processes. Since the main thread is not always the one directly running the event loop, standard debugging tools can be less effective.

4. **`nest-asyncio`**: AiiDA uses [nest-asyncio](https://pypi.org/project/nest-asyncio/) in some places to allow the event loop to be re-entered. This allows AiiDA to run asynchronous code in an interactive environment like an IPython shell. However, it can lead to edge cases that don’t arise in a purely “vanilla” `asyncio` setup.

5. **RPC-Like Communication**: AiiDA’s `verdi daemon` or its internal runners can communicate with each other (and with the client) via remote procedure calls (RPC). Debugging these interactions can be complicated when you have multiple processes or threads exchanging async messages.

---

## 3. Standard Ways to Debug Async Python Code

Despite these complexities, there are still some regular tools for debugging async code in Python:

1. **`print` and Logging**:  
   - The simplest solution can be the most effective. Insert logging statements (`logging.debug`, `logging.info`) or `print` statements to trace the execution flow.  
   - You can set up more verbose logging or increase the logging level to capture more details about what is happening inside coroutines.

2. **`asyncio` Debug Mode**:  
   - `loop.set_debug(True)` can be used to enable extra debugging features in `asyncio`. It often logs warnings about long-running tasks, late callbacks, or exceptions that aren’t caught.  
   - In standard Python code, you might do something like:
     ```python
     import asyncio

     loop = asyncio.get_event_loop()
     loop.set_debug(True)

     # The rest of your async code...
     loop.run_until_complete(some_coroutine())
     ```
   - This can already provide insights into where the event loop might be getting stuck.

3. **Tracers and Profilers**:  
   - Tools such as [Yappi](https://github.com/sumerc/yappi) or [PyInstrument](https://github.com/joerick/pyinstrument) can sometimes give you an overview of where the code is spending time.

4. **`pdb`/`ipdb`**:  
   - You can still use `pdb`, but you have to be mindful: hitting a breakpoint in a coroutine does not always guarantee the rest of the event loop will pause nicely.  
   - In some scenarios, wrapping coroutines in synchronous “test harnesses” (i.e., running them via `asyncio.run(...)`) can let you debug them in a more controlled environment.

---

## 4. Debugging Async Code in AiiDA

### Regular Tools vs. AiiDA’s Setup

Because AiiDA can run the event loop in a dedicated runner or separate interpreter, not all standard techniques map cleanly. For example:

- If you try `loop.set_debug(True)` in a shell script or an AiiDA plugin, you might not control the loop if it’s managed elsewhere by the AiiDA daemon.  
- If you add breakpoints in the code run by the daemon, you might not have access to the daemon’s console to interact with `pdb`.  
- Logging messages may go to different loggers or be handled by the daemon’s logging configuration.

### Where Do My Debug Messages Go?

A key question is: *Which logger am I using, and where are logs being handled?* In AiiDA, you often have logs going to:

- The daemon’s own logs.  
- The user’s command line (for `verdi` commands).  
- Possibly other logging handlers configured by your environment.

When debugging, double-check your `logging` setup and make sure you’re either writing to `stderr` or to a file that you can monitor.

### A Real-World Stuck Coroutine

One known challenge is diagnosing a situation where **Plumpy** or the AiiDA RPC communication gets “stuck.” 
This often manifests as your code simply hanging, with no apparent errors or timeouts. 
Tracing this can be *extremely* frustrating without a real-time window into the event loop’s state.

---

## 5. `aiomonitor` Is Your Friend: A Case Study

One particularly useful tool in such scenarios is [`aiomonitor`](https://github.com/aio-libs/aiomonitor). `aiomonitor` allows you to attach a small Telnet-based console to your running event loop so you can:

- Inspect running tasks and coroutines.
- Trigger debugging or introspection on them.
- See how many tasks are currently in the loop.

### Installation and Usage

```bash
pip install aiomonitor
```

In a typical script (or if you have direct control over the loop), you can wrap the event loop with an `aiomonitor` context:

```python
import asyncio
from aiomonitor import Monitor

async def main():
    # Your async code here
    while True:
        await asyncio.sleep(1)

if __name__ == '__main__':
    loop = asyncio.get_event_loop()

    # Start aiomonitor on port 50101 by default
    with Monitor(loop=loop, host='127.0.0.1', port=50101):
        loop.run_until_complete(main())
```

Now, if your event loop seems stuck, you can open a separate terminal and run:
```bash
telnet 127.0.0.1 50101
```
From there, you can type commands like `tasks` to see a list of running coroutines. If you notice one coroutine that isn’t making progress, you can delve deeper, potentially finding out if there’s a deadlock, an indefinite `await`, or some blocking I/O.

### Applying This in AiiDA

In AiiDA, using `aiomonitor` often requires hooking into the same event loop the daemon or your workflows are using. 
This can be done by customizing the daemon’s run script or by creating a test script that reproduces the issue and manually starts the loop with the monitor. 
In some advanced use cases, you could monkey-patch the runner to wrap it in `aiomonitor`, though that requires deeper familiarity with AiiDA’s internal architecture.

---

## 6. Future Improvements for Async Programming in AiiDA

Debugging async code in AiiDA is still evolving. Some potential areas for improvement include:

1. **Better Logging & Tracing**:  
   - Introducing more trace logs around critical sections (like scheduling, cancellations, and state transitions of coroutines) could help developers pinpoint issues faster.

2. **Built-In Async Debugging Tools**:  
   - Perhaps AiiDA could ship with optional built-in integration for `aiomonitor` or equivalent. This would let users easily attach a monitor to the AiiDA daemon without hacking the internals.

3. **Documentation & Recipes**:  
   - Sharing debugging “recipes” (e.g., how to attach an event loop debugger in a plugin vs. in the daemon) would help new developers.

4. **Test Harnesses for Async**:  
   - Providing a robust local runner that can be launched in a fully controlled environment (with or without `nest-asyncio`) would let developers more easily replicate issues in their own environment.

5. **Enhanced Timeout & Deadlock Detection**:  
   - Building on top of `asyncio` debug mode, AiiDA could detect tasks that exceed certain time limits, automatically log stack traces, or even attempt to break out of deadlocks.

---

## Conclusion

Asynchronous programming enables AiiDA to manage complex, distributed workflows efficiently—but it also complicates debugging. 
From standard Python debugging approaches (`pdb`, logging, trace tools) to more advanced techniques like `aiomonitor`, there are ways to tackle the challenges. 
While some approaches work seamlessly, others need adaptation for AiiDA’s nest-asyncio and multi-process environment.

By continuing to refine our logging strategies, adopting specialized debugging tools, and sharing best practices across the community, we can make async in AiiDA more transparent and robust for everyone. 
As the AiiDA framework grows, expect to see more built-in support for async debugging—making your scientific workflows both powerful and more transparent to debug!

