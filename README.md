## memleax

`memleax` detects memory leak of a *running* process.


## why write this

Once I met a memory leak problem in production environment.
The process began memory leak after long-time running. If using current
memory debuggers, such as `Valgrind`, I have to restart the target process
and wait for several days again. Besides it impacts the target process
much for several days, which is not good for production environment.

So I need a tool to debug a running process without restarting it.
I searched the internet but didn't find one. Then I wrote such a tool myself.

Hope this is useful for others.


## how it works

`memleax` attachs a running process, hooks memory allocate/free APIs,
records all memory blocks, and reports the blocks which live longer
than 5 seconds (you can change this time by -e option) in real time.

It is very *convenient* to use, and suitable for production environment.
There is no need to recompile the program or restart the target process.
You run `memleax` to monitor the target process, wait for the real-time memory
leak report, and then kill it (e.g. by Ctrl-C) to stop monitoring.

NOTE: Since `memleax` does not run along with the whole life of target
process, it assumes the long-live memory blocks are memory leak.
Downside is you have to set the expire threshold by -e option according
to your scenarios; while the upside is the memory allocation for process
initialization is skipped, besides of the convenience.


## performace impact

Because target progress's each memory allocation/free API invokes a TRAP, the
performance impact depends on how often the target program calls memory
APIs.
For example, it impacts lightly to nginx with HTTP, while heavily with HTTPS,
because OpenSSL calls malloc seriously.

Although performance impact is worthy of consideration, since `memleax` is
run to attach the target progress only when you certain it is in memory leak,
and stopped after real-time memory leak report, so it is not need to attach
the target progress for long time.


## difference with Valgrind

+ `Valgrind` starts target process, while `memleax` attachs a running process;

+ `Valgrind` gives memory leak report on quiting, while `memleax` assumes
that long-living memory blocks are leaks, so it reports in real time;

+ `Valgrind` runs target process on its virtual CPU, which makes it slow.
While `memleax` hooks memory APIs, which *maybe* less slow, if the target process
call memory APIs not often.

+ `Valgrind` debugs kinds of memory bugs, while `memleax` is lightweight and
only detects memory leak.

In summary, I think `Valgrind` is more powerful, while `memleax` is more
convenient and suitable for production environment.


## environment

+ GNU/Linux-x86_64, tested on CentOS 7.2 and Ubuntu 16.04
+ FreeBSD-amd64, tested on FreeBSD 10.3


## requirement

+ libunwind
+ libdwarf, if you do not have this, set `--disable-libdwarf` to `configure` to
  disable it, as a result you will not see file name and line number in backtrace.
+ libelf, on GNU/Linux, while FreeBSD has libelf already.


## usage

### start

To debug a running process, run:

    $ memleax [options] <target-pid>

then `memleax` begins to monitor the target process, and report memory leak in real time.

### wait and check the report

The memory blocks live longer than the threshold, are showed as:

    CallStack[3]: memory expires with 101 bytes, backtrace:
        0x00007fd322bd8220  malloc()+0
        0x000000000040084e  foo()+14  foo.c:12
        0x0000000000400875  bar()+37  xxxxx.c:20
        0x0000000000400acb  main()+364  test.c:80

`CallStack[3]` is the ID of CallStack where memory leak happens.

The backtrace is showed only on the first time, while it only shows the
ID and counter if expiring again:

    CallStack[3]: memory expires with 101 bytes, 2 times again

If the expired memory block is freed later, it shows:

    CallStack[6]: expired-memory frees after 10 seconds

If there are too many expired-memory-blocks are freed on one CallStack,
this CallStack will not be showed again:

    Warning: too many expired-free at CallStack[6]. will not show this CallStack later

When you think you have found the answer, stop the debug.

### stop

`memleax` stops monitoring and quits on:

* you stop it, by Ctrl-C or kill,
* the target process quits, or
* too many leaks on one CallStack.

After quiting, it also gives statistics for the CallStacks with memory leak:

    CallStack[3]: may-leak=20 (2020 bytes)
        expired=20 (2020 bytes), free_expired=0 (0 bytes)
        alloc=20 (2020 bytes), free=0 (0 bytes)
        freed memory live time: min=0 max=0 average=0
        un-freed memory live time: max=20
        0x00007fd322bd8220  malloc()+0
        0x000000000040084e  foo()+14  foo.c:12
        0x0000000000400875  bar()+37  xxxxx.c:20
        0x0000000000400acb  main()+364  test.c:80