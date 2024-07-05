---
title: RegreSSHion? Technical analysis of CVE-2024-6387
excerpt: How RegreSSHion vulnerability works and are you affected by it?
mode: immersive
header:
  theme: dark
article_header:
  type: overlay
  theme: dark
  background_color: false
  background_image: 
    gradient: 'linear-gradient(to right, orange, red)'
aside:
  toc: true
author: Mehul Singh
show_author_profile: true
mermaid: true
key: regresshion-03-07-2024
tags: 
  - SSH
  - CVE
  - c
  - memory corruption
  - race condition
  - remote code execution
  - aysnc-signal-unsafe
  - glibc
---

## What is RegreSSHion vulnerability?

On July 1, Qualys Inc published a [security advisory](https://www.qualys.com/2024/07/01/cve-2024-6387/regresshion.txt) stating a new vulnerability in Glibc-based SSH implementation named OpenSSH, which affects 8.5p1 <= OpenSSH < 9.8p1 and OpenSSH <= 4.4p1 versions. According to the researchers, a race condition affects the vulnerable version where, upon winning this race condition the user can execute any privileged code, remotely.

This was made possible due to GNU's C library, glibc, not locking memory altering modules such as `malloc()` in the vulnerable versions, which runs with full privilege, and outside any kind of sandbox and reaching that unsafe state before the code completes its execution. The name of this CVE is due to it being an actual regressed SSH state as it was due to CVE-2006-5051, which also allowed race condition in signal handler to provide remote code execution.

An overview of race condition across different vulnerable versions:
1. OpenSSH 3.4p1 (Debian 3.0r6): Exploiting this version required interrupting a call to `free()` with _SIGALRM_, leaving the heap in an inconsistent state and exploiting this state in another call to free(). It took the researchers approximately 10,000 tries, or about one week on average, to obtain a remote root shell in this version.
2. OpenSSH 4.2p1 (Ubuntu 6.06.1): Exploitation of this involved interrupting a call to `pam_start()` with _SIGALRM_, leading to an inconsistent state exploited in a call to pam_end(). It takes about 1-2 days to obtain a remote root shell
3. OpenSSH 9.2p1 (Debian 12.5.0): Exploitating this involves interrupting a call to malloc() with _SIGALRM_, resulting in an inconsistent heap state exploited in another malloc() call. It took them about 6-8 hours on average to obtain a remote root shell

## How did it came to be?

We will try to understand how they did it, but from a red teaming mindset; to understand, in order to break. And even if you don't understand C programming, don't worry. Just understanding the general flow through function and variable names, should be enough to understand why this vulnerability exists for starter.

### Timeline

On October 16 2020, the following piece of code got removed from log.c file as seen in the [OpenSSH#752250c](https://github.com/openssh/openssh-portable/commit/752250caabda3dd24635503c4cd689b32a650794) commit. And this triggered a chain code execution which, if timed right, can allow a person to execute remote code without prior authentication.

```c
172 void 
173 sigdie(const char *fmt,...)
174 {
175 #ifdef DO_LOG_SAFE_IN_SIGHAND
176      va_list args;
177
178      va_start(args, fmt);
179      do_log(SYSLOG_LEVEL_FATAL, fmt, args);
180      va_end(args);
181  #endif
182      _exit(1);
183  }
```

The researchers in Qualys actually exploited the older version of OpenSSH (prior to 4.4p1) by exploiting its calling of `grace_alarm_handler()` function which waited for the _LoginGraceTime_ before freeing the buffer by calling `free()` function, which is a well known unsafe asynchronous signal function. With this information, they tried to find two things or conditions which must be met to exploit the same race condition in the latest versions of OpenSSH as well:
1. Is there a similar function being called here which is async-signal-unsafe such as `malloc()` and `free()`?
2. If there is, is it behind a lock?

Interestingly enough, just after they started working on it, [Bugzilla #3690](https://bugzilla.mindrot.org/show_bug.cgi?id=3690) thread got created which raised an extremely related issue, which, got marked as a duplicate by [Bugzilla #3598](https://bugzilla.mindrot.org/show_bug.cgi?id=3598)

### Execution Chain in OpenSSH


In the latest versions of OpenSSH, (the last vulnerable version) the `grace_alarm_handler()` function in `sshd.c` [ultimately] calls a very interesting function called `syslog()`. Below is the complete chain of codes:

```c
 353 grace_alarm_handler(int sig)
 354 {
 ... 
 364     /* Log error and exit. */
 365     sigdie("Timeout before authentication for %s port %d",
 366         ssh_remote_ipaddr(the_active_state),
 367         ssh_remote_port(the_active_state));
 368 }
```

This `sigdie()` function is a macro defined in `log.h` header file where macro expansion to `sshsigdie()` function takes place. This `sshsigdie()` function is defined in `log.c` file:

```c
451 sshsigdie(const char *file, const char *func, int line, int showfunc,
452     LogLevel level, const char *suffix, const char *fmt, ...)
453 {
...
457     sshlogv(file, func, line, showfunc, SYSLOG_LEVEL_FATAL, suffix, fmt, args);
460     _exit(1);
461 }
```

This `sshlogv` is also defined in `log.c` through macro in `log.h`:

```c
464 sshlogv(const char *file, const char *func, int line, int showfunc,
465     LogLevel level, const char *suffix, const char *fmt, va_list args)
466 {
... 
493     do_log(level, forced, suffix, fmt2, args);
494 }
```

And finally, this `do_log()` is what calls glibc's native syslog() (line 419) in `log.c`:

```c
336 static void do_log(LogLevel level, int force, const char *suffix,
337       const char *fmt, va_list args)
338 {
...
413 #if defined(HAVE_OPENLOG_R) && defined(SYSLOG_DATA_INIT)
...
417 #else
...
419         syslog(pri, "%.500s", fmtbuf);
...
421 #endif
...
424 }
```

* Going back to our question 1, does this `syslog()` call any async-signal-unsafe? Yes! But *only if* the very first call to `syslog()` is made inside the _SIGALRM_ handler, then syslog allocates a file structure and an internal read buffer through malloc

* What about the second question? Is it locked? No! Based on commit [#3f6bb8a](https://sourceware.org/git/?p=glibc.git;a=commit;h=3f6bb8a32e5f5efd78ac08c41e623651cc242a89), [#a15d53e2](https://sourceware.org/git/?p=glibc.git;a=commit;h=a15d53e2de4c7d83bda251469d92a3c7b49a90db) and [#](https://sourceware.org/git/?p=glibc.git;a=commit;h=905a7725e9157ea522d8ab97b4c8b96aeb23df54) on glibc, we notice that a 'feature' was added to bypass locking when single-threading our application.

This guarantees that race conditions can be achieved, barring the features like NX which further attempts to prevent execution of codes in certain regions and ASLR which randomizes the address space in attept to make memory addres predictions harder. But even then, in systems like i386, some libraries like glibc is always mapped either at address 0xb7200000 or at address 0xb7400000, making their predictions right 50% of the time (more than 50% technically)

### Exploiting Malloc

Now that we know that we can run aysnc-signal-unsafe functions, the next thing is to find a control flow in `malloc()` which, when interrupted by _SIGALRM_ at the right time, leaves the heap in a volatile state. If this gets possible, another signal can be made from the _SIGALRM_ handler in sshd with user defined payload to be executed in a privileged state (something like a piggyback attack).

The researchers found numerous possible entries which might trigger this, and they went ahead with exploiting the function `_int_malloc()` which makes use of relative sizes and not absolute addresses, and this makes it better suited for amd64 exploits due to them having better ASLR capabilities. But even with this approach, their exploit worked only on i386 systems for the time being. 

From here on out the exploit gets real technical as it deals with controlling and corrupting the heap memory with various precise function calls. The exact steps taken are better explained by the researchers themselves from here on out, and can be accessed from the security advisory linked in the introduction of this post. 



## Should everyone be concerned? 

*TL;DR: No\**

From their report, although this attack can provide remote code execution, it takes thousands of attempts which can be logged and easily detected by modern solutions. And this by default doesn't affect some systems such as OpenBSD which already uses async-signal-safe functions such as `syslog_r()`.

### Mitigations and Precautions

This vulnerability was mitigated on June 6, 2024, through commit [#81c1099](https://github.com/openssh/openssh-portable/commit/81c1099d22b81ebfd20a334ce986c4f753b0db29) which was a part of a much bigger commit aiming at a diffense-in-depth approach which could be seen from another recent commit [#03e3de4]() "Start the process of splitting sshd into separate binaries". And since this might introduce difficulties in backporting, you can simply remove or comment out the async-signal-unsafe code from previously mentioned `sshsigdie()` function.

And in the worse case scenarios where sshd cannot be updated or recompiled, you can just set the _LoginGraceTime_ to 0 in the configuration file. This will make the code vulnerable to DoS attacks but protect against possible remote code executions as mentioned in their advisory.


## Further reading

1. [Delivering Signals for Fun and Profit](https://lcamtuf.coredump.cx/signals.txt)
2. [JPEG COM Marker Processing Vulnerability](https://www.openwall.com/articles/JPEG-COM-Marker-Vulnerability#exploit)
3. [ASLRn’t: How memory alignment broke library ASLR](https://zolutal.github.io/aslrnt/)
