Recently I've spent a lot of time trying to read PARI code. This tutorial explains the best way I found for doing it, some basic debugging commands, and some ways to make using the debugger less painful.
I'll focus on using ```gdb``` to learn about PARI's source code. 

PARI/GP https://pari.math.u-bordeaux.fr/ is a computational number theory project that has been running since 1995.
It has two main parts: a large library of low-level C code for (PARI) and a frontend for the
library, a shell called ```gp.```
There is some documentation of the PARI library, but what is available is aimed at explaining what is done.
For me, the documentation is not sufficient to understand what is done, let alone *why* it's done which is what we're ultimately after.
Debugging tools like ```gdb``` make it easier to understand the code. 
They give a way to step through the library code and view variables as they change (their state) which helps to 
get a handle on what is happening.

To start, lets review the options for debugging GP programs. There's nothing I could find in the docs specifically aimed at reading PARI source code but they do mention some methods for debugging your own GP scripts:
* The debug level can be set in GP using \g. E.g. \g 3
* GPs has some basic debugging commands like breakpoint() that can be used without running a debugger
* PARI source code can be compiled to C with the scripts gp2c. One of these scripts gp2c-dbg compiles the GP code and then starts running the gdb debugger on it.
None of these options are very helpful for investigating PARI's source code. The best way I have found for doing this is by running the gdb debugger on GP itself.

To get started we need to build the PARI source code with debug symbols. 
From the main PARI directory we can do this with
```
./Configure -g
make dbg
```
this creates a new directory with .o files with debug symbols. This directory is referenced by gp.dbg the debugging version of GP.

To start debugging we can run `gdb gp.dbg`

Suppose we want to look at the PARI function that initializes number fields. We know there’s a GP function for this named bnfinit and we want to set a breakpoint in the corresponding PARI library function. We can use gdb’s info command to look for functions with similar names
`(gdb) info func bnfinit`
which gives
```
../src/basemath/buch2.c
GEN bnfinit0(GEN, long, GEN, long);
```
We can set a breakpoint here with `break` like this
`(gdb) break bnfinit0`
The shorthand `b` can also be used.

Using the commands
```
(gdb) run
(?) bnfinit(polcyclo(5))
```
we can start GP. Then we reach our bnfinit breakpoint by constructing a numberfield with the 5th cyclotomic polynomial. If everything is working we should see some information from the breakpoint. This includes
```
Breakpoint 1, bnfinit0 
```
We can look at the inputs to the function using `info args`
```
(gdb) info args
P = 0x7ffff7b9ee88
flag = 0 
data = 0x0 
prec = 4
```
Of these 4 args two of them are primitive C types so gdb will print them. The PARI object P can be examined with the call command
```
(gdb) call output(P)
x^4+x^3+x^2+x+1 
```
We can advance a few more lines by entering next (n) repeatedly. At this point it is helpful to find out where we are again. Using info line we see the current line number. Let's say we're on line 3300 then we can see the next 10 lines of PARI's source code with the list command
```
(gdb) list 3300
```
To return control of the program to GP we can use continue (c).

Some other useful gdb commands:
```
step (s) - like next but if a function is being called it will step into that function
backtrace (bt) - for tracing the sequence of function calls that led to the current frame
delete - deletes a breakpoint
info breakpoint - shows all your breakpoints
``` 
### Saving commands to gdb init

Commands you need to use often can be saved to the file ~/.gdbinit. For example we can save an alias to the command we used to print the PARI object P by putting the following in the file.
```
define ppi
    call output((GEN)$arg0)
end
```
Aliases can be used for many things like saving and loading your gdb breakpoints or GP sessions to a file.

### Installing A GUI for Visual Debugging
While a visual debugger is not required for doing debugging I think it can save a lot of time when working with a large codebase. This is especially true when the source code is organized into files thousands of lines long which is the case with PARI.

Gdbgui seems to be a good choice for a lightweight visual debugging tool. The installation instructions are here. 

Running 
```gdbgui gp.dbg```
you should see some lines from the main function in gp.c in the main window.

Visual debugging uses the PARI source files to let you move around the codebase. Clicking the *Fetch source files* button will load these files. If everything is working you should be able to open the PARI source directory.  
![gdbgui](/imgs/image.png)

Earlier we set a breakpoint using the gdb command line on the bnfinit0 function in buch2.c. We can navigate to this file by opening basemath/buch2.c in the src directory. To locate the function we can try the *jump to line* box.  
![jumpto](/imgs/image2.png)

To jump to the line where bnfinit0 is defined we can enter the line number found by entering info func bnfinit0 in the gdb prompt.

We can place a breakpoint inside the function by clicking a line number in the left gutter.  
![break](/imgs/image3.png)

To reach this breakpoint we start the program with the circle-arrow button in the top right.  
![run](/imgs/image4.png)

The program starts running. However, it immediately hits a breakpoint that gdbgui automatically inserts in the main function of GP (this behavior can be turned off in preferences).  
![firstbreak](/imgs/image5.png)

Using the play button in the top right lets the program continue. In the terminal you will see that GP has been loaded. On the gdb prompt we can enter the same GP code as above
![cmd](/imgs/image6.png)
and which hits the breakpoint in the bnfinit0 function.

Some useful features of debugging this way is the side menu which lets us easily work with local variables and breakpoints  
![cmd](/imgs/image6.png)

Another good feature is tooltips. By moving the mouse over a variable we can see its value. Moving over a function shows the signature
![tooltip](/imgs/image7.png)
