# How fast is Procfs

The proc filesystem (procfs) is a virtual filesystem in the Linux kernel that
provides information about processes. It is one of the interfaces which follows
the “Everything is a file” paradigm. The procfs file system was designed a long
time ago, when an average system ran no more than a few hundred processes. At
that time it was not a problem to open one or several proc files per each
process to get some information. In our days, a system can have hundreds of
thousands of processes or even more. In this context, the idea to perform
per-process open doesn’t look so brilliant. In this article, we are going to
explore all aspects of procfs, and find which ones can be optimized.

The idea to optimize the proc file system came when we found that CRIU, the
software to checkpoint/restore processes, spends a significant amount of time
reading procfs files. We have seen how a similar problem had been solved for
sockets, so we decided to implement something like sock-diag for processes. We
understood how hard it is to change a very mature interface in the kernel, but
the biggest surprise was how many kernel developers support the idea of a new
interface. They don’t know how a new interface should look like, but they all
know that the existing solution doesn’t work well in critical situations. It
often like this: a server becomes laggy, vmstat shows lots of swap used, “ps
ax” needs 10+ seconds to show something, and top never comes up with any
output. This article doesn’t propose any specific interface; instead it tries
to give more details about the problem, and suggest ways to fix it.

In procfs, each running process is represented by a directory /proc/<pid>. Each
such directory contains dozens of files with information about a process,
divided into groups. Let’s have a quick tour (note that $$ is a shell built-in
variable, expanding to the current process’ PID):

```
$ ls -F /proc/$$
attr/            exe@        mounts         projid_map    status
autogroup        fd/         mountstats     root@         syscall
auxv             fdinfo/     net/           sched         task/
cgroup           gid_map     ns/            schedstat     timers
clear_refs       io          numa_maps      sessionid     timerslack_ns
cmdline          limits      oom_adj        setgroups     uid_map
comm             loginuid    oom_score      smaps         wchan
coredump_filter  map_files/  oom_score_adj  smaps_rollup
cpuset           maps        pagemap        stack
cwd@             mem         patch_state    stat
environ          mountinfo   personality    statm
```

All these files have different and unique formats. Most are ASCII text, easily
readable by a human. Or not.

```
$ cat /proc/$$/stat
24293 (bash) S 21811 24293 24293 34854 24876 4210688 6325 19702 0 10 15 7 33 35 20 0 1 0 47892016 135487488 3388 18446744073709551615 94447405350912 94447406416132 140729719486816 0 0 0 65536 3670020 1266777851 1 0 0 17 2 0 0 0 0 0 94447408516528 94447408563556 94447429677056 140729719494655 140729719494660 140729719494660 140729719496686 0
```

To understand this set of numbers, one needs to read proc(5) man page or kernel
documentation. For example, the field number 2 is the file name of the
executable, in parentheses, and the field number 19 is a process’ nice value.

Some of /proc/<pid> files have more human readable formats:

```
$ cat /proc/$$/status | head -n 5
Name:	bash
Umask:	0002
State:	S (sleeping)
Tgid:	24293
Ngid:	0
```

Now, how often do users look at these files directly? How much time it takes
for the kernel to encode its internal binary data into text? What is the
overhead of such pseudo file system? How good is this interface for developers
of monitoring tools? How much time do these tool need to parse text data? How
often do people have problems caused by bad performance of this interface in
critical situations? These are the questions that comes to mind.

It would not be incorrect to say that users tend to use tools like ps or top
rather than inspect raw files from procfs. To answer other questions, we need
to do a few experiments. First of all, let’s find out where the kernel spends
the time to generate these files.

We need to understand what has to be done to get information about all
processes. For that, we have to read a content of the /proc/ directory and find
all directories which names are decimal numbers. For each such directory, we
need to open a file, read its content and close it.

This results in three system calls have to be made, one of them creating a file
descriptor, an operation that requires the kernel to allocate some internal
objects. The open() and close() system calls don’t provide us with any
information, so we can say that the time spent calling those is the overhead
from this interface.

Here is the first experiment: call open() and close() for each process, without
reading file’s content. In here we open and close every /proc/*/stat file:

```
$ time ./task_proc_all --noread stat
tasks: 50290

real	0m0.177s
user	0m0.012s
sys	0m0.162s
```

Same for /proc/*/loginuid:

```
$ time ./task_proc_all --noread loginuid
tasks: 50289

real	0m0.176s
user	0m0.026s
sys	0m0.145
```

It doesn’t matter which file is opened, as its content is generated from the
read system call, which we don’t use yet.


Now, let’s take look at the corresponding perf output, to profile kernel
functions.

```
-   92.18%     0.00%  task_proc_all    [unknown]
   - 0x8000
      - 64.01% __GI___libc_open
         - 50.71% entry_SYSCALL_64_fastpath
            - do_sys_open
               - 48.63% do_filp_open
                  - path_openat
                     - 19.60% link_path_walk
                        - 14.23% walk_component
                           - 13.87% lookup_fast
                              - 7.55% pid_revalidate
                                   4.13% get_pid_task
                                 + 1.58% security_task_to_inode
                                   1.10% task_dump_owner
                                3.63% __d_lookup_rcu
                        + 3.42% security_inode_permission
                     + 14.76% proc_pident_lookup
                     + 4.39% d_alloc_parallel
                     + 2.93% get_empty_filp
                     + 2.43% lookup_fast
                     + 0.98% do_dentry_open
           2.07% syscall_return_via_sysret
           1.60% 0xfffffe000008a01b
           0.97% kmem_cache_alloc
           0.61% 0xfffffe000008a01e
      - 16.45% __getdents64
         - 15.11% entry_SYSCALL_64_fastpath
              sys_getdents
              iterate_dir
            - proc_pid_readdir
               - 7.18% proc_fill_cache
                  + 3.53% d_lookup
                    1.59% filldir
               + 6.82% next_tgid
               + 0.61% snprintf
      - 9.89% __close
         + 4.03% entry_SYSCALL_64_fastpath
           0.98% syscall_return_via_sysret
           0.85% 0xfffffe000008a01b
           0.61% 0xfffffe000008a01e
        1.10% syscall_return_via_sysret
```

Here we can see that the kernel spent roughly 75% of the time to create and
destroy file descriptors, and about 16% to list processes.

Now we know the time it takes to open() and close() one file per each process,
but we can’t yet say how significant the overhead is is in the whole process,
as we lack data for comparison. Let’s see try to read those files, choosing the
“most popular” ones. It’s easy to find out which ones -- run ps or top under
strace, see what it does. It appears that both these tools read at least
/proc/{PID}/stat and /proc/{PID}/status for each process.


Let’s perform open/read/close on /proc/*/status. It is one of the bigger files
having a fixed number of fields.

```
$ time ./task_proc_all status
tasks: 50283

real	0m0.455s
user	0m0.033s
sys	0m0.417s
```
```
-   93.84%     0.00%  task_proc_all    [unknown]                   [k] 0x0000000000008000
   - 0x8000
      - 61.20% read
         - 53.06% entry_SYSCALL_64_fastpath
            - sys_read
               - 52.80% vfs_read
                  - 52.22% __vfs_read
                     - seq_read
                        - 50.43% proc_single_show
                           - 50.38% proc_pid_status
                              - 11.34% task_mem
                                 + seq_printf
                              + 6.99% seq_printf
                              - 5.77% seq_put_decimal_ull
                                   1.94% strlen
                                 + 1.42% num_to_str
                              - 5.73% cpuset_task_status_allowed
                                 + seq_printf
                              - 5.37% render_cap_t
                                 + 5.31% seq_printf
                              - 5.25% render_sigset_t
                                   0.84% seq_putc
                                0.73% __task_pid_nr_ns
                              + 0.63% __lock_task_sighand
                                0.53% hugetlb_report_usage
                        + 0.68% _copy_to_user
           1.10% number
           1.05% seq_put_decimal_ull
           0.84% vsnprintf
           0.79% format_decode
           0.73% syscall_return_via_sysret
           0.52% 0xfffffe000003201b
      + 20.95% __GI___libc_open
      + 6.44% __getdents64
      + 4.10% __close
```

We can see that only about 60% of the time was spent in read() syscalls. If we
look at its detailed profile, we can find about 45% of the time was spent in
functions like seq_printf, seq_put_decimal_ull, that do binary to text
transformation. Apparently encoding binary data into a text format is not so
cheap. Do we really need a human-readable interface to retrieve this data from
the kernel? How often do we want to see the raw /proc contents? Why do tools
like top and ps have to decode text data back into a binary format?

Perhaps you would like to know how fast could be an interface that
uses a binary format rather than text for output data;
does not require a minimum of three system calls and one file descriptor per process.


To much surprise, the first such interface appeared in the Linux kernel back in 2004.

```
[0/2][ANNOUNCE] nproc: netlink access to /proc information (https://lwn.net/Articles/99600/)
```

nproc is an attempt to address the current problems with /proc. In
short, it exposes the same information via netlink (implemented for a
small subset).

Unfortunately, the community didn’t show much interest in this work. The most
recent attempt to introduce something like this has been made only two years
ago. 

```
[PATCH 0/15] task_diag: add a new interface to get information about processes (https://lwn.net/Articles/683371/)
```

Let's see what it's all about. The task-diag interface is based on the following principles:
* Transactional: write a request, read back a response
* Netlink message format (binary and extendable; same as used by sock_diag)
* Ability to specify a set of processes to get info about
* Optimal grouping of attributes (any attribute in a group can't affect a response time)

This interface was presented at a few conferences. It was (experimentally)
integrated into pstools and CRIU, and (to some degree) into the perf tool. All
three experiments shown that performance is always better.

The kernel community expressed some interest in this work. The primary debate
was about what transport should be used to transfer data between kernel and
userspace. The initial idea to use netlink sockets was declined. Partly it was
due to known unresolved issues in netlink code base, and partly due to a belief
that the netlink interface was designed only for a network subsystem. It was
suggested instead to use a transactional file in procfs, what means that a user
opens a file, writes a request into a file descriptor, and then reads a
response from it. As usual, there were people who didn’t like this approach
either. Alas, the solution accceptable by everyone has not been found yet. 

Now let’s compare task_diag with procfs.

The task-diag code has a test tool, which can be used for our experiments. In
the first experiment, we will get process PIDs and credentials. Here is an
example of data that a test program gets for each process:

```
$ ./task_diag_all one  -c -p $$
pid  2305 tgid  2305 ppid  2299 sid  2305 pgid  2305 comm bash
uid: 1000 1000 1000 1000
gid: 1000 1000 1000 1000
CapInh: 0000000000000000
CapPrm: 0000000000000000
CapEff: 0000000000000000
CapBnd: 0000003fffffffff
```

Now let’s run it for the same set of processes what we used earlier with procfs.

```
$ time ./task_diag_all all  -c

real	0m0.048s
user	0m0.001s
sys	0m0.046s
```

It only takes 0.05 seconds to get data enough to show a process tree. In case
of procfs, we need 0.177 seconds to merely open and close one file per process,
without even reading any data from it!


Here is a perf output:

```
-   82.24%     0.00%  task_diag_all  [kernel.vmlinux]            [k] entry_SYSCALL_64_fastpath
   - entry_SYSCALL_64_fastpath
      - 81.84% sys_read
           vfs_read
           __vfs_read
           proc_reg_read
           task_diag_read
         - taskdiag_dumpit
            + 33.84% next_tgid
              13.06% __task_pid_nr_ns
            + 6.63% ptrace_may_access
            + 5.68% from_kuid_munged
            - 4.19% __get_task_comm
                 2.90% strncpy
                 1.29% _raw_spin_lock
              3.03% __nla_reserve
              1.73% nla_reserve
            + 1.30% skb_copy_datagram_iter
            + 1.21% from_kgid_munged
              1.12% strncpy   
```

Nothing stands out, except for the fact that there are no visible
time-consuming functions to be optimized.

Now let’s see how many system calls are needed to get information about all processes:                        

```
 $ perf trace -s ./task_diag_all all -c  -q

 Summary of events:

 task_diag_all (54326), 185 events, 95.4%

   syscall            calls    total       min       avg       max      stddev
                               (msec)    (msec)    (msec)    (msec)        (%)
   --------------- -------- --------- --------- --------- ---------     ------
   read                  49    40.209     0.002     0.821     4.126      9.50%
   mmap                  11     0.051     0.003     0.005     0.007      9.94%
   mprotect               8     0.047     0.003     0.006     0.009     10.42%
   openat                 5     0.042     0.005     0.008     0.020     34.86%
   munmap                 1     0.014     0.014     0.014     0.014      0.00%
   fstat                  4     0.006     0.001     0.002     0.002     10.47%
   access                 1     0.006     0.006     0.006     0.006      0.00%
   close                  4     0.004     0.001     0.001     0.001      2.11%
   write                  1     0.003     0.003     0.003     0.003      0.00%
   rt_sigaction           2     0.003     0.001     0.001     0.002     15.43%
   brk                    1     0.002     0.002     0.002     0.002      0.00%
   prlimit64              1     0.001     0.001     0.001     0.001      0.00%
   arch_prctl             1     0.001     0.001     0.001     0.001      0.00%
   rt_sigprocmask         1     0.001     0.001     0.001     0.001      0.00%
   set_robust_list        1     0.001     0.001     0.001     0.001      0.00%
   set_tid_address        1     0.001     0.001     0.001     0.001      0.00%
```

In case of procfs, we need more than 150000 system calls to get this
information, task_diag requires a bit more than 50.

Let’s look at the real workload. For example, we want to show a process tree
with command lines for each process. For that, we need to know command line,
pid and parent pid for each process.

In case of task_diag, the test program sends a request to get general
information plus command lines for all processes and then reads requested
information:

```
$ time ./task_diag_all all  --cmdline -q


real	0m0.096s
user	0m0.006s
sys	0m0.090s
```

In case of procfs, we need to read two files /proc/pid/status and /proc/pid/cmdline.

```
$ time ./task_proc_all status
tasks: 50278

real	0m0.463s
user	0m0.030s
sys	0m0.427s
```

```
$ time ./task_proc_all cmdline
tasks: 50281

real	0m0.270s
user	0m0.028s
sys	0m0.237s
```

Here we can see that task_diag is about 7 times faster than procfs (0.096 vs
0.27 + 0.46). Usually, the increase of performance even for a few percents is a
good result. Here the effect is much more significant.

Another thing to mention is a number of kernel allocations. This factor is
critical when a system is under memory pressure. Let’s compare how many kernel
allocations happen for procfs and task-diag.

```
$ perf trace --event 'kmem:*alloc*'  ./task_proc_all status 2>&1 | grep kmem | wc -l
58184
$ perf trace --event 'kmem:*alloc*'  ./task_diag_all all  -q 2>&1 | grep kmem | wc -l
188
```

Let’s see how many allocations are required to run a trivial process:

```
$ perf trace --event 'kmem:*alloc*'  true 2>&1  | grep kmem | wc -l
94
```

Procfs requires 600 times more in-kernel memory allocations than the task diag
interface. It is just another point why procfs works so slow in critical
situations, and there is a room for optimization.

We hope that this article will attract more developers interested in improving
this part of the kernel.

Many thanks to David Ahern, Andy Lutomirski, Stephen Hemminger, Oleg Nesterov,
W. Trevor King, Arnd Bergmann, Eric W. Biederman, and other people who help to
develop and improve the task diag interface.

# Links

* [Test tools: task_diag_all, test_proc_all](https://github.com/avagin/linux-task-diag/tree/v4.16-task-diag-20180427/tools/testing/selftests/task_diag)
* [LWN: task_diag and statx()](https://lwn.net/Articles/685791/)
* [Time to rethink /proc](https://www.slideshare.net/KirKolyshkin/time-to-rethink-proc)
* [Speeding up ps and top](https://www.slideshare.net/kolyshkin/speeding-up-ps-and-top)
* [Netlink engine issues, and ways to fix those](https://blog.linuxplumbersconf.org/2016/ocw/system/presentations/4599/original/Netlink-issues.pdf)

# Git
* [github.com/avagin/linux-task-diag](https://github.com/avagin/linux-task-diag/releases/tag/v4.19-task-diag)

# Authors
* [Andrei Vagin](github.com/avagin/linux-task-diag)
* [Kir Kolyshkin](https://github.com/kolyshkin)
