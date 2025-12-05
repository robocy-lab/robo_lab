# **But what's async anyway?**
## Intro
Imagine a simple program whose job is to periodically send
requests to server1 and server2. The delay before the next
request is determined by the previous response; if you stop
sending, your computer will blow up. How could it be
implemented? Naively, like this:
```python
def send_to_server(addr):
    time_before_next_request = 0
    while True:
        time.sleep(time_before_next_request)
        resp = http.request("GET", addr)
        time_before_next_request = int(resp.data)

def main(addr1, addr2):
    send_to_server(addr1)
    send_to_server(addr2)
```

Looks fine and your computer is safe, isn't it?
Not at all, because first `send_to_server` will
single-handedly occupy whole thread of execution forever and you'll
never reach second call! It's definitely
not something great, and we must find new solution. You could
think about using several threads of execution
(and it'll definetly work), but I can show you a way of reaching the
same goal within **single** thread.

If you know some python, you might immediatly come with
such beautiful solution:
```python
async def send_to_server(addr):
    time_before_next_request = 0
    while True:
        await asyncio.sleep(time_before_next_request)
        resp = http.request("GET", addr)
        time_before_next_request = int(resp.data)

async def main(addr1, addr2):
    join = asyncio.gather(
        send_to_server(addr1),
        send_to_server(addr2)
    )
    await join

asyncio.run(main())
```
> Remark: Notice that this code is still not truly asynchronous,
because http.request blocks the thread. We’ll fix that in a moment.
For now, let's say that it is our implementation

If you are looking at this masterpiece of a syntax construction
and already regret that you've opened this post, stop.
It will become clearer in a moment. And even if you've written
something like that, you might not fully understand what does it
mean and how is it implemented, so let's talk about it!

## Building intuition
So, let's firstly think, what we want from our code. Obvious goal is
"One function should be able to work while other is sleeping".
But our functions have variables with same exact name! Won't this
variable be overwritten? Or even argument `addr`! To avoid this
kind of stuff, we can save state of a function each time,
when we think, that other functions have time to be executed (we
mark this places with `await` keyword):
```python
{
    function_name: "send_to_server",
    next_line: 4,
    variables: {
        addr: <some_server_addr>,
        time_before_next_request: <some_value>,        
    },
}
```
Okay, our function's states are safe, but how will we know, when
some function should be called? We have finite amount of tasks,
that should be continued at some point and each of those have same
priority (if you miss request to any of the servers, your computer
won't live long), so that sounds like a loop! Also, we have some
"system state", not only state for each function, so let's call
this state `runtime`
```python
class Runtime:
    tasks_ready_to_continue: List[(int, func_state)] # some id + state
    timers: List[(time.time, (int, func_state))] # when should end + which task in

    ... # <some functions, that we will use later and which are trivial>
```

And now, as we have `Runtime`, lets try to design our own loop!
> Remark: basically, when we use `await`, function
saves it's state and "jumps" to loop
```python
while True:
    # this loop is often called **event loop**!

    # continue all tasks that we can right now!
    # (but only one at a time, because single thread)
    while runtime.tasks_ready_to_continue:
        task = runtime.tasks_ready_to_continue.pop()
        runtime.continue_executing(task)
    

    # but what if there are no tasks?
    closest_timer = min(runtime.timers)
    time_left  = closest_timer[0] - time.now()
    
    time.sleep(max(0, time_left)) # thread can sleep without hesitation

    timers = []
    while runtime.timers:
        timer = runtime.timers.pop()
        if time.now() >= timer[0]:
            runtime.tasks_ready_to_continue.append(timer[1])
            continue
        timers.append(timer)
    runtime.timers = timers       
```
This solves the ‘sleep’ part, but we still have a problem:
the HTTP request itself is blocking. While `http.request` is
running, the whole thread is stuck and no other coroutine can
make progress

So, we should add await to http request like this, right?
```python
async def send_to_server(addr):
    time_before_next_request = 0
    while True:
        await asyncio.sleep(time_before_next_request)
        resp = await aiohttp.request("GET", addr)
        time_before_next_request = int(resp.data)
```

But our loop (for now!) has no way to work with non-timer events.
In Linux, exists api called `epoll`
(MacOS has `kqueue` and Windows has `IOCP`, that does
practically the same), you can think about it as a structure
that can store some amount of
files (socket that we use in http request is a file too!) and
answer a question "Which of these files are ready to be read from
or written to?". Function answering to that question is
called `epoll_wait`. You don't need to understand how `epoll` is
implemented, just remember what it's function is.


Now, with that in mind, let's add support for non-timer operations! 

```python
while True:
    while runtime.tasks_ready_to_continue:
        task = runtime.tasks_ready_to_continue.pop()
        runtime.continue_executing(task)
    
    closest_timer = min(runtime.timers)
    time_left     = closest_timer[0] - time.now()

    # when we can read from file, epoll returns some "event" that
    # includes previously specified value. let's assume that we've
    # specified (task_id, state) for each non-timer operation
    events = []

    # also, let's assume that epoll already has necessary files
    # epoll wait returns number of files that can be interacted with
    n = epoll_wait(epoll=runtime.epoll,
                   events_list=events,
                   timeout=max(0, time_left))

    if n == 0:
        # timer was earlier than any file was ready
        timers = []
        while runtime.timers:
            timer = runtime.timers.pop()
            if time.now() >= timers[0]:
                runtime.tasks_ready_to_continue.append(timers[1])
                continue
            timers.append(timer)
        runtime.timers = timers
        continue # as no files need any interaction with

    for ev in events:
        runtime.tasks_ready_to_continue.insert(0, ev.data)
        epoll_remove(ev) # as we already got what we wanted
```
And that's final version of our `event loop`! It handles multiple
tasks at once, can work with timers and keep your computer safe in
case of very specific and unrealistic problem, that almost certainly
won't ever happen in your life! But there are some cases when async is really needed:
- Web servers, that get many requests at once. Threads are good, but
threads that can handle multiple requests at any time are cooler!
- Database access in your app (they're relatively slow too)
- Background processing of external program 
- Many other things

If you've read carefully, you might have noticed a problem: `await`
can be used only in `async` function, because any other function
doesn't have state, that saved somewhere. That means you can’t
await an async function from a normal function. Once some function
needs await, it usually has to become async too, and this can
propagate through your codebase. This phenomenon is called
`function colouring`, which exists across
multiple languages and is a design choice, not a bug. Wonder, how
can you deal with it? Accept.

## Conclusion
Obviously, our implementation of event loop is not ideal, it lacks
proper error handling, not very optimal, doesn't include implementation
of real-world state machine for functions etc., but I hope the process
of building it gave you some intuition about how async in python, Rust,
JavaScript ans C# is implemented (this model is called stackless
coroutines). `async` makes compiler store functions' state somewhere, ans `await` saves it for caller and jumps to 
`event loop` where it waits for next event.

And also, that's not only way of
implementing concurrent execution, but that's a theme for another post :)

Thanks for reading!


## Read more about it
[epoll](https://man7.org/linux/man-pages/man7/epoll.7.html)  
[asyncio](https://docs.python.org/3/library/asyncio.html)  
[non-blocking io](https://eklitzke.org/blocking-io-nonblocking-io-and-epoll)
