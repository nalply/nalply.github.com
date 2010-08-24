---
layout: bordered
Title: Disguise Linux Process
---

This is about a hack to disguise a process. It does not need root
privileges or a root kit. How a process is listed in `ps` and `w` is changed,
also data in the proc filesystem.


##Rename process names

The `prctl()` option `PR_SET_NAME` sets the current process' name. The system
call is used by daemons to give them useful names. Only the current process can
be renamed, however. With the `LD_PRELOAD` environment variable and gcc's
`__attribute ((constructor))` we can set the victim's name before execution
enters `main()`.

`LD_PRELOAD` is used by [reverse engineers][re] to trick compiled executables
into executing arbitrary code. Functions in libraries and the command itself
can be hijacked. It's poor man's [monkey patching][mp] in C. It works like
this: One compiles code to be injected to a dynamic library `patch.so` and
invokes the command prepended with `LD_PRELOAD=./hack.so`.

`__attribute ((constructor))` is similar to C++ static initialization. In fact,
I also could have written the hack in C++. I use this feature because I don't
want hijack functions. I want code to be executed before anything else.

Combined these three features

- `prctl(PS_SET_NAME, ...)`
- `LD_PRELOAD`
- `__attribute ((constructor))`

can force any name to any process a user starts.


##Trash cmdline

This is not good enough. `ps -f` shows all arguments. `cmdline` in the proc
filesystem also blabs the arguments.

How to trash cmdline?

`program_invocation_name` seems to be essentially a pointer to data in 
`cmdline` and `argv`. I don't know how these things relate exactly to each other
but in my experiments, if I change either `argv[0]` or `program_invocation_name`
(`cmdline` is not writable), the other two also change. That's very neat to
trick `ps`&nbsp;`-f`.

I also discovered that the memory for the arguments seems to immediately follow 
`argv[0]`. `cmdline` also strongly hints this: the arguments are separated by
a ASCII NUL character. What if I overwrite the arguments by writing beyond
the end of `argv[0]`? 

This is *undefined behavior* and dangerous ground but it seems to "work". In
experiments however I trashed a local variable when I exceeded the memory
range of `cmdline`. So I must never write more than the original `cmdline`
contains. I get the length by reading the file from the proc filesystem and
use the length as an upper bound.

This is nice and changes the behavior of `w`, `ps -f` and `cmdline`. This also
causes the victim to complain about bad arguments. What happened? I overwrote
the arguments before the victim had a chance to evaluate them...!

\*smile\*

What if I somehow could change the arguments *after* the parsing of the victim
process? What if I somehow could wait one second and let the victim do its
thing? Bingo! Use a thread (POSIX pthreads) and wait one second.

Most processes evaluate arguments as soon as they are started and never need
them again.

Let's recapitulate again: With these features

- `program_invocation_name`
- File length of `/proc/self/cmdline` == memory size of cmdline
- Use a thread to overwrite cmdline later

one can trash a process' cmdline (but not enlarge).


##Compiling and invocation

I wrote a small C file `rptitle.c` (about 60 lines) and compiled it with

    gcc -Wall -fPIC -c rptitle.c -o rptitle.o
    gcc -shared -o rptitle rptitle.o

With

    LD_PRELOAD=./rptitle.so RPTITLE='changedcommand args' command \
        -options args
    
I change the process to be listed as `changedcommand args`.


##References

- [prctl man page][prctl]
- [Reverse Engineering use of LD_PRELOAD][re]
- [gcc Function Attributes][gccfa]
- [program_invocation_name man page][pin]
- [Monkey Patching][mp]
- [POSIX pthreads][pt]


##Future improvements

`exe` in the proc filesystem still links to the original executable. I have
an idea about this... I will write about this in a later post.


[prctl]: http://www.kernel.org/doc/man-pages/online/pages/man2/prctl.2.html
[re]:    http://securityvulns.com/articles/reveng/
[pin]:   http://www.kernel.org/doc/man-pages/online/pages/man3/program_invocation_name.3.html
[mp]:    http://en.wikipedia.org/wiki/Monkey_patch
[pt]:    http://www.ibm.com/developerworks/linux/library/l-posix1.html
[gccfa]: http://gcc.gnu.org/onlinedocs/gcc/Function-Attributes.html
