---
layout: post
title:  "Debugging Elixir NIFs"
date:   2019-07-03 11:01:29 -0600
---
Elixir has various interop capabilities with native code, with NIFs ("naitve implemented functions") being the closest to calling a native function directly. This is usually done in order to achieve increased performance when handling computationally heavy instructions. A great example of NIFs in action is the [Matrex][matrex-url] library for matrix math!

NIFs are very powerful, but come at a cost. A NIF is called directly by the BEAM, so a crash will take down the entire virtual machine, skipping over the built in fault tolerant devices. This is usually considered a _bad_ thing, akin to getting run over by a vehicle.

When developing a NIF, it is important to properly debug and examine the code to ensure safety throughout the whole system. This is when the goto tools come into play: `gdb`, `lddb`, ect. However, the resulting NIF binaries are loaded by an external process and cannot be executed standalone. In order to launch our application under the debugger of choice, the `beam.smp` process needs to be wrapped.

A look around the erlang mailing list leads to a wonderful [example][erlang-mailing-url].
```
➜  ~ which erl
/usr/local/bin/erl
```
Opening the file with your editor of choice, modify the last few lines to the startup script more flexibility.
For this example, LLDB will be used.

```bash
if [ ! -z "$USE_LLDB" ]; then
  lldb -- $BINDIR/erlexec ${1+"$@"}
else
  exec $BINDIR/erlexec ${1+"$@"}
fi
```

Now LLDB can be used to start making things explode, but for "good" reason. Let's cause a segfault!

```c
int *num = NULL;
*num = 77;
```
Extremely naive code, but it should get the job done. [Don't dereference null pointers kids.][null-youtube]
Launching our app with our new flag `USE_LLDB` brings us into lldb.

`USE_LLDB=1 mix run`
`(lldb) run`
```
top reason = exec
    frame #0: 0x000000001069c000 dyld`_dyld_start
```
Oops, looks like we hit a snag. It might seem alarming at first, but a quick search reveals that lldb stops on re-exec whereas gdb does not. It's fine to continue running our application.
```
(lldb) continue
Process 92683 resuming
Process 92683 stopped
* thread #6, name = '1_scheduler', stop reason = EXC_BAD_INSTRUCTION (code=EXC_I386_INVOP, subcode=0x0)
    frame #0: 0x0000000010799f64 demo.so`segfault(env=<unavailable>, argc=<unavailable>, argv=<unavailable>) at demo.c:7:10 [opt]
   4    {
   5   
   6        int *foo = NULL;
-> 7        *foo = 77;
   8        return enif_make_atom(env, "ok");
   9    }
   10
```

And just like that we have an lldb terminal telling me why I'm a bad developer!



[matrex-url]: https://github.com/versilov/matrex
[rustler-url]: https://github.com/rusterlium/rustler
[erlang-mailing-url]: https://groups.google.com/d/msg/erlang-programming/Gz2zbQjC144/qF7sSVv3ts8J
[null-youtube]: https://www.youtube.com/watch?v=bLHL75H_VEM
