% UFTRACE-LIVE(1) Uftrace User Manuals
% Namhyung Kim <namhyung@gmail.com>
% May, 2016

NAME
====
uftrace-live - Trace functions in a command lively


SYNOPSIS
========
uftrace [live] [*options*] COMMAND [*command-options*]


DESCRIPTION
===========
This command runs COMMAND and prints its functions with time and thread info.  This is basically same as running `uftrace-record` and `uftrace-replay` command in turn, but it does not save data file.  This command accepts most of options that are accepted by either of record or replay command.


OPTIONS
=======
-b *SIZE*, \--buffer=*SIZE*
:   Size of internal buffer which trace data will be saved.  Default size is 128k.

\--daemon
:   (XXX: rename to 'dont-wait' or 'keep') Trace daemon process which calls `fork`(2) and then `exit`(2).  Usually uftrace stops recording when its child exited but daemon process calls `exit`(2) before doing its real job (in the child process).  So this option is used to keep tracing such daemon processes.

-F *FUNC*, \--filter=*FUNC*
:   Set filter to trace selected functions only.  This option can be used more than once.  See *FILTERS*.

-N *FUNC*, \--notrace=*FUNC*
:   Set filter not trace selected functions only.  This option can be used more than once.  See *FILTERS*.

-T *TRG*, \--trigger=*TRG*
:   Set trigger on selected functions.  This option can be used more than once.  See *TRIGGERS*.

-D *DEPTH*, \--depth=*DEPTH*
:   Set global trace limit in nesting level.

-t *TIME*, \--time-filter=*TIME*
:   Do not show small functions under the time threshold.  If some functions explicitly have 'trace' trigger, those are always traced regardless of execution time.

-A *SPEC*, \--argument=*SPEC*
:   Record function arguments.  This option can be used more than once.  See *ARGUMENTS*.

-R *SPEC*, \--retval=*SPEC*
:   Record function return value.  This option can be used more than once.  See *ARGUMENTS*.

\--force
:   Allow to run uftrace even if some problem might occur.  When `uftrace-record` finds no mcount symbol (which is generated by compiler) in the executable it quits with an error message since it there's no need to run the program.  However it's possible one is only interested functions in a library, in this case she can use this option so uftrace can keep running the program.  Also -A,\--argument and -R,\--retval option works only for binaries built with -pg so it makes uftrace exit when it tries to run other binaries.  This option ignores the warning and go tracing without the argument and/or return value.

\--flat
:   Print flat format rather than C-like format.  This is usually for debugging and testing purpose.

-L *PATH*, \--library-path=*PATH*
:   Load necessary internal libraries from this path.  This is for testing.

-k, \--kernel
:   Trace kernel functions as well as user functions.  Only kernel entry/exit functions will be traced as if -D 1 was given.

-K, \--kernel-full
:   Trace kernel functions as well as user functions.  Kernel functions will be traced in full detail (ie. without depth limit unless explicitly given)

\--no-libcall
:   Do not record library function invocations.  The uftrace traces library calls by hooking dynamic linker's resolve function in the PLT.  One can disable it with this option.

\--no-pltbind
:   Do not bind dynamic symbol address.  This option uses the LD_BIND_NOT environment variable to trace library function calls which might be missing due to concurrent (first) accesses.  It's only meaningful to use this option without the --no-libcall option.

\--disable
:   Start uftrace with tracing disabled.  This is only meaningful when used with 'trace_on' trigger.

\--demangle=*TYPE*
:   Demangle C++ symbol names.  Possible values are "full", "simple" and "no".  Default is "simple" which ignores function arguments and template parameters.

\--report
:   Show live-report before replay.

--column-view
:   Show each task in separate column.  This makes easy to distinguish functions in different tasks.

--column-offset=*DEPTH*
:   When `--column-view` option is used, this option specifies the amount of offsets between each task.  Default is 8.

--task-newline
:   Interleave a new line when task is changed.  This makes easy to distinguish functions in different tasks.

\--num-thread=*NUM*
:   Use NUM of thread to record trace data.  Default is 1/4 of online cpus (but when full kernel tracing is enabled, it'll use the same number of the cpus).

--no-comment
:   Do not show comments of returned functions.

\--libmcount-single
:   Use single thread version of libmcount for faster recording.  This is ignored if target program calls pthread_create().


FILTERS
=======
The uftrace support filtering only interested functions.  When uftrace is called it receives two types of function filter; opt-in filter with -F/--filter option and opt-out filter with -N/--notrace option.  These filters can be applied either record time or replay time.

The first one is an opt-in filter; By default, it doesn't trace anything and when it executes one of given functions it starts tracing.  Also when it returns from the given funciton, it stops again tracing.

For example, suppose a simple program which calls a(), b() and c() in turn.

    $ cat abc.c
    void c(void) {
        /* do nothing */
    }

    void b(void) {
        c();
    }

    void a(void) {
        b();
    }

    int main(void) {
        a();
        return 0;
    }

    $ gcc -o abc abc.c

Normally uftrace will trace all the functions from `main()` to `c()`.

    $ uftrace ./abc
    # DURATION    TID     FUNCTION
     138.494 us [ 1234] | __cxa_atexit();
                [ 1234] | main() {
                [ 1234] |   a() {
                [ 1234] |     b() {
       3.880 us [ 1234] |       c();
       5.475 us [ 1234] |     } /* b */
       6.448 us [ 1234] |   } /* a */
       8.631 us [ 1234] | } /* main */

But when `-F b` filter option is used, it'll not trace `main()` and `a()` but only `b()` and `c()`.

    $ uftrace -F b ./abc
    # DURATION    TID     FUNCTION
                [ 1234] | b() {
       3.880 us [ 1234] |   c();
       5.475 us [ 1234] | } /* b */

The second type is an opt-out filter; By default, it trace everything and when it executes one of given functions it stops tracing.  Also when it returns from the given funciton, it starts tracing again.

In the above example, you can omit the function b() and its children with -N option.

    $ uftrace live -N b ./abc
    # DURATION    TID     FUNCTION
     138.494 us [ 1234] | __cxa_atexit();
                [ 1234] | main() {
       6.448 us [ 1234] |   a();
       8.631 us [ 1234] | } /* main */

In addition, you can limit the print nesting level with -D option.

    $ uftrace -D 3 ./abc
    # DURATION    TID     FUNCTION
     138.494 us [ 1234] | __cxa_atexit();
                [ 1234] | main() {
                [ 1234] |   a() {
       5.475 us [ 1234] |     b();
       6.448 us [ 1234] |   } /* a */
       8.631 us [ 1234] | } /* main */

In the above example, it prints functions up to 3 depth, so leaf function c() was omitted.  Note that the -D option works with -F option.

Sometimes it's useful to see long-running functions only.  This is good because there're many tiny functions that are not interested usually.  The -t/\--time-filter option implements the time-based filter that only records functions run longer than the given threshold.  In the above example, user might want to see functions running more than 5 micro-seconds like below:

    $ uftrace live -t 5us ./abc
    # DURATION    TID     FUNCTION
     138.494 us [ 1234] | __cxa_atexit();
                [ 1234] | main() {
                [ 1234] |   a() {
       5.475 us [ 1234] |     b();
       6.448 us [ 1234] |   } /* a */
       8.631 us [ 1234] | } /* main */

The -t/\--time-filter option works for user-level functions only.

You can also set triggers to filtered functions.  See *TRIGGERS* section below for details.


TRIGGERS
========
The uftrace support triggering some actions on selected function with or without filters.  Currently supported triggers are depth (for record and replay) and backtrace (for replay only).  The BNF for the trigger is like below:

    <trigger>  :=  <symbol> "@" <actions>
    <actions>  :=  <action>  | <action> "," <actions>
    <action>   :=  "depth=" <num> | "backtrace" | "trace" | "trace_on" | "trace_off" | "recover"

The depth trigger is to change filter depth during execution of the function.  It can be use to apply different filter depths for different functions.  And the backrace trigger is to print stack backtrace at replay time.

Following example shows how trigger works.  The global filter depth is 5, but function 'b' changed it to 1 so functions below the 'b' will not shown.

    $ uftrace live -D 5 -T 'b@depth=1' ./abc
    # DURATION    TID     FUNCTION
     138.494 us [ 1234] | __cxa_atexit();
                [ 1234] | main() {
                [ 1234] |   a() {
       5.475 us [ 1234] |     b();
       6.448 us [ 1234] |   } /* a */
       8.631 us [ 1234] | } /* main */

The 'backtrace' trigger is only meaningful in replay command.  The 'traceon' and 'traceoff' (you can omit '_' between 'trace' and 'on/off') controls whether uftrace records functions or not.

The uftrace trigger only works for user-level functions for now.


ARGUMENTS
=========
The uftrace supports recording function arguments and/or return value using -A/\--argument and -R,\--retval options respectively.  The syntax is very similar to the trigger:

    <argument> :=  <symbol> "@" <specs>
    <specs>    :=  <spec>  | <spec> "," <spec>
    <spec>     :=  ( "arg" N | "retval" ) [ "/" <format> [ <size> ] ]
    <format>   :=  "i" | "u" | "x" | "s" | "c"
    <size>     :=  "8" | "16" | "32" | "64"

The -A,\--argument option takes argN where N is an index of the arguments.  The index starts from 1 and corresponds to argument passing order of the calling convention on the system.  Currently it assumes all the arguments are integer (or pointer) type, so the result might not correct if there're floating-point arguments as well.  The -R,\--retval option takes "retval" and also assumes it's an integer type.

Users can specify optional format and size of the arguments and/or return value.  Without this, the uftrace treats them as 'long int' type and print them appropriately.  The "i" format makes it signed integer type and "u" format is for unsigned type.  Both are printed as decimal while "x" format makes it printed as hexa-decimal.  The "s" format is for string type and "c" format is for character type.

    $ uftrace -A main@arg1/x -R main@retval/i32 ./abc
    # DURATION    TID     FUNCTION
     138.494 us [ 1234] | __cxa_atexit();
                [ 1234] | main(0x1) {
                [ 1234] |   a() {
                [ 1234] |     b() {
       3.880 us [ 1234] |       c();
       5.475 us [ 1234] |     } /* b */
       6.448 us [ 1234] |   } /* a */
       8.631 us [ 1234] | } = 0; /* main */

Note that these arguments and return value are recorded only if the executable was built with -pg option.  The executables built with -finstrument-functions will exit with an error message.  Also it only works for user-level functions for now.


SEE ALSO
========
`uftrace-record`(1), `uftrace-replay`(1), `uftrace-report`(1)