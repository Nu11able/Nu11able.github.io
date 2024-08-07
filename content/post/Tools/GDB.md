---
title: GDB
date: 2022-01-10
tags:
 - tools
categories:
 - tools
---

# Chapter 1

gdb `startfile`

set width w(a number)

break breakpoint `it could be line number|function name|address`

run

n `next. line`

s `step. into the function`

bt `backtrace.`

p lquote `print. the value of lquote`
- you can also use `p` to assign a specific value to an element,such as `p len_lquote=strlen(lquote)`
> We can set them to better values using the p command, since it can print the value of any expression—and that expression can include subroutine calls and assignments.

l `list. the source surrounding the current line(10 lines by default) `
- you can use "l 50" to display the source surrounding the 50th line

c `continue.`

# Chapter 2

Ctrl-d `Program exited normally`

q `quit. the gdb`

> The most usual way to start gdb is with one argument, specifying an executable program:`gdb program`
You can also start with both an executable program and a core file specified:`gdb program core`
You can, instead, specify a process ID as a second argument or use option -p, if youwant to debug a running process:`gdb program 1234|gdb -p 1234`


> `gdb --args gcc -O2 -c foo.c `This will cause gdb to debug gcc, and to set gcc’s command-line arguments (see
Section 4.3 [Arguments], page 34) to ‘-O2 -c foo.c.

gdb --silent|-quiet|-q `Do not print the introductory and copyright message`

gdb --help | gdb -h
 

gdb --cd directory `Run gdb using directory as its working directory, instead of the current directory`


> ```
> !command-string
> Invoke a standard shell to execute command-string. Note that no space is
> needed between ! and command-string. If it exists, the environment variable
> SHELL determines which shell to run. Otherwise gdb uses the default shell
> (‘/bin/sh’ on Unix systems, ‘COMMAND.COM’ on MS-DOS, etc.).
> 
> ```

> pipe [command] | shell_command
> | [command] | shell_command
> pipe -d delim command delim shell_command
> | -d delim command delim shell_command
> Executes command and sends its output to shell command. Note that no space
> is needed around |. If no command is provided, the last command executed is
> repeated.
> In case the command contains a |, the option -d delim can be used to specify an alternate delimiter string delim that separates the command from the
> shell command.
> Example:
> ```
> (gdb) p var
> $1 = {
>     black = 144,
>     red = 233,
>     green = 377,
>     blue = 610,
>     white = 987
> }
> (gdb) pipe p var|wc
> 7 19 80
> (gdb) |p var|wc -l
> 7
> (gdb) p /x var
> $4 = {
>     black = 0x90,
>     red = 0xe9,
>     green = 0x179,
>     blue = 0x262,
>     white = 0x3db
> }
> (gdb) ||grep red
> red => 0xe9,
> (gdb) | -d ! echo this contains a | char\n ! sed -e ’s/|/PIPE/’
> this contains a PIPE char
> (gdb) | -d xxx echo this contains a | char!\n xxx sed -e ’s/|/PIPE/’
> this contains a PIPE char!
> (gdb)
> ```
> The convenience variables $_shell_exitcode and $_shell_exitsignal can be used to
> examine the exit status of the last shell command launched by shell, make, pipe and |.
> ```

# GDB commands
> set print elements NUMBER-OF-ELEMENTS
> set print array-indexes
> set print elements 10 # to change the limit of array elements to print
> 