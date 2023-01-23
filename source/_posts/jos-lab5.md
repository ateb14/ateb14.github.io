---
title: jos-lab5
date: 2022-11-30 00:43:13
tags:
- operating system
- jos
categories:
- operating system
---

# LAB5, FILE SYSTEM, SPAWN AND SHELL

## Challenge

I finish the challenge of adding more features to the shell:

- File creation: `touch <name>`

Create the file with option O_CREAT.

- Command-line history.

Automatically record the command line history in a file called HISTORY. In order to pass the tests, this function is deactivated by a macro called `COM_HISTORY`.

`user/sh.c`:

```c
void
runcmd(char* s)
{
	char *argv[MAXARGS], *t, argv0buf[BUFSIZ];
	int argc, c, i, r, p[2], fd, pipe_child;

	pipe_child = 0;
	gettoken(s, 0);

	// record the command-line history
#if COM_HISTORY == 1
	record_history(s);
#endif
...
```



- O_APPEND option for opening the file.

The `open()` function will modify the offset in the file descriptor to the size of the file by looking up the `stat` information.

## Exercise1 & Question1

Simply set the FL_IOPL_3 bits in eflags of the newly created environment.

*Do you have to do anything else to ensure that this I/O privilege setting is saved and restored properly when you subsequently switch from one environment to another? Why?*

No. Because the trap frame will store and restore the eflags when there is a context switch.

## Exercise2

We should check the bitmap after the block is loaded into the main memory, for it's possible for the block being loaded to be a part of the bitmap.

## Exercise10

The pipe mechanism implemented in the user library is not compatible with a change I made in Lab4 to finish the sfork challenge, where the system call `sys_getenvid` is called every time the environment gets access to `thisenv`, which fails the pipe race check in `_pipeisclosed` forever due to the fact that the `env_runs` count will increase after a system call. To fix it, I use a local variable to store `thisenv`, and replace all  `thisenv` with this variable.

`lib/pipe.c`:

```c
static int
_pipeisclosed(struct Fd *fd, struct Pipe *p)
{
	int n, nn, ret;

	while (1) {
		struct Env *myenv = thisenv;
		n = myenv->env_runs;
		ret = pageref(fd) == pageref(p);
		nn = myenv->env_runs;
		if (n == nn)
			return ret;
		if (n != nn && ret == 1)
			cprintf("pipe race avoided %d %d %d \n", n, nn, ret);
	}
}
```

