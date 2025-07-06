bgcmd
=====

bgcmd is a way to run (almost) any interactive repl in background

it keeps a persistent session between each line, and it prints any output
produced by the repl



usage
-----

1. set the `$BGCMDPROMPT` env var to your repl's prompt string
2. (optional) set the `$BGCMDDIR` env var to some private directory
   (it defaults to `$HOME/.bgcmd` otherwise)
3. run `bgcmd START your-repl-here arg1 arg2 arg3` to start the repl
4. run `bgcmd input here` to pass that input to the repl


after every invocation, the output from the repl will be printed to stdout

this lets you programmatically interact with repls and use their outputs

here's a demo of a cpython session:
![python repl demo](python.png)

and here is claude playing with sqlite3 with no sqlite mcp server:

https://github.com/user-attachments/assets/7ced94bf-63d2-4b97-968b-9fbeeae975e1



faq/troubleshooting
-------------------

### why is this useful?  i could already type in a repl

your ai tools could not.  now they can drive almost any repl, as long as they
can run arbitrary shell commands.  for instance, claude code could not easily
use the [rr](https://rr-project.org) debugger (rr is an interactive program but
claude code would always wait for the whole session to time out or terminate)
but now it easily can

(rr has a gdbmi interface but it doesn't help for this)



### what if my repl doesn't print a prompt when not writing to a terminal?

check if it accepts a `-i` parameter or something to force an interactive mode



### the bg process stays around forever?

bgcmd spawns a reader process that waits for your input through a fifo.  its
pid is stored in `$BGCMDDIR/pid`.  killing this process will send an end of
file to the repl, which causes most repls to gracefully terminate

restarting bgcmd with the same `$BGCMDDIR` will also do this for you



### can ai tools interact with multiple repls at the same time?

one way to do this is to create a few mini wrappers, one for each

for instance, this is a wrapper to run rr, which inspired this project:

```sh
#!/bin/sh

export BGCMDPROMPT='(rr) '
export BGCMDDIR=/home/potato/.bgcmdrr

if [ "$1" = START ]; then
    bgcmd START rr replay
else
    bgcmd "$@"
fi
```



### why does this not spawn a pty?

because then you'd get ansi escapes mixed in, and you'd have to remove them
before being able to use the reply from the repl



### my repl ignored/dropped the input and stuff got stuck

your repl is bad and you should use something else.  however, you can manually
resend the input by just writing it to the `in` fifo in `$BGCMDDIR`



### my repl printed the prompt as part of its output and things got out of sync

you can flush the state with `cat $BGCMDDIR/out > /dev/null`

unfortunately, waiting for the prompt is the only thing that works semi
generically for arbitrary repls.  yeah it's a bit jank

(btw since you fully control the repl, you can just not cause this to happen)



### can't you do this with expect/tmux/shl/...?

probably, yeah.  but this was short to write and it's generic and easy to use

in most cases, you'd have a similar level of jank with those anyway



### it's 2025, why does this readme not say mcp at least x times?

mcp mcp mcp mcp mcp mcp mcp mcp mcp mcp mcp mcp mcp



one more demo because i'm feeling generous
------------------------------------------

this is claude driving rr to debug a nontrivial problem autonomously

[![claude](https://asciinema.org/a/726302.svg)](https://asciinema.org/a/726302)

the program aborts when it detects an incorrect value, but without reverse
debugging it's hard to determine when this value was written to the cache.
a reverse debugger makes it trivial

by default claude isn't very good at using this type of debugger, so i'm putting
together a bit of a [guide](https://gist.github.com/izabera/9157af255a4bb5c47618d268b465ce5b)
(very much WIP, not particularly complete/well structured).  without a tutorial
on reverse debugging, claude seems to be aware of the general capabilities, but
then it doesn't use them well at all, and it tends to jump around aimlessly,
making little to no progress, getting stuck, hitting irrelevant breakpoints
over and over, until it finally gets frustrated by the lack of progress and
gives up, and makes up an explanation that may or may not be based on reality

it's just like people frfr
