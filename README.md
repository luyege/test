# Homework 2 Debugging and Fixing - CSE 320 - Fall 2020
#### Professor Eugene Stark

### **Due Date: Friday 10/2/2020 @ 11:59pm**

# Introduction

In this assignment you are tasked with updating an old piece of
software, making sure it compiles, and that it works properly
in your VM environment.

Maintaining old code is a chore and an often hated part of software
engineering. It is definitely one of the aspects which are seldom
discussed or thought about by aspiring computer science students.
However, it is prevalent throughout industry and a worthwhile skill to
learn.  Of course, this homework will not give you a remotely
realistic experience in maintaining legacy code or code left behind by
previous engineers but it still provides a small taste of what the
experience may be like.  You are to take on the role of an engineer
whose supervisor has asked you to correct all the errors in the
program, plus add additional functionality.

By completing this homework you should become more familiar
with the C programming language and develop an understanding of:

- How to use tools such as `gdb` and `valgrind` for debugging C code.
- Modifying existing C code.
- C memory management and pointers.
- Working with files and the C standard I/O library.

## The Existing Program

Your goal will be to debug and extend an old program called `finddup`,
which was posted to Usenet in early 1991.
It was written by Bill Davidsen of GE Corporate R&D center,
and it was intended to run on a generic Unix system.
The version I am handing out is very close to the original version,
except that I have made a few changes for this assignment.
First of all, I rearranged the source tree and re-wrote the `Makefile`
to conform to what we are using for the other assignments in this course.
I also introduced a few bugs here and there to make things more interesting
and educational for you :wink:.
Aside from these changes and the introduced bugs, which only involve a few
lines, the code is identical to the original, functioning version.

The purpose of the `finddup` program is to read an input file containing a
list of filenames, determine which of the files in the list have content
that duplicates that of other files in the list, and print out a report
of the results.  The program employs the following strategy to try to
make the detection of duplicate content efficient:  First, the length of each
file is obtained, and files of different lengths are regarded as non-duplicates.
Then, for files of the same length, the content is read and a certain kind of
"checksum" (or hash) is computed.  Files having different hashes are regarded
as non-duplicates.  Finally, for files having the same length and same hash,
a full byte-by-byte comparison of the contents is performed.

The `finddup` program is pretty simple, though there are a few technical details
involved in getting file metadata (such as length).  With a little bit of effort,
I would expect that you should be able to understand all of it.
When you make modifications to the program, limit the changes you make to the
minimum necessary to achieve the specified objectives.  Don't rewrite the program;
assume that it is essentially correct and just fix a few compilation errors and
bugs as described below.  You will likely find it helpful to use `git` for this (I did).
Make exploratory changes first on a side branch (*i.e.* not the master branch),
then when you think you have understood the proper changes that need to be made,
go back and apply those changes to the master branch.  Using `git` will help you
to back up if you make changes that mess something up.

### Getting Started - Obtain the Base Code

Fetch base code for `hw2` as you did for the previous assignments.
You can find it at this link:
[https://gitlab02.cs.stonybrook.edu/cse320/hw2](https://gitlab02.cs.stonybrook.edu/cse320/hw2).

Once again, to avoid a merge conflict with respect to the file `.gitlab-ci.yml`,
use the following command to merge the commits:

<pre>
  git merge -m "Merging HW2_CODE" HW2_CODE/master --strategy-option=theirs
</pre>

  > :nerd: I hope that by now you would have read some `git` documentation to find
  > out what the `--strategy-option=theirs` does, but in case you didn't :angry:
  > I will say that merging in `git` applies a "strategy" (the default strategy
  > is called "recursive", I believe) and `--strategy-option` allows an option
  > to be passed to the strategy to modify its behavior.  In this case, `theirs`
  > means that whenever a conflict is found, the version of the file from
  > the branch being merged (in this case `HW2_CODE/master`) is to be used in place
  > of the version from the currently checked-out branch.  An alternative to
  > `theirs` is `ours`, which makes the opposite choice.  If you don't specify
  > one of these options, `git` will leave conflict indications in the file itself
  > and it will be necessary for you to edit the file and choose the code you want
  > to use for each of the indicated conflicts.

Here is the structure of the base code:

<pre>
.
├── .gitlab-ci.yml
└── hw2
    ├── doc
    │   └── finddup.1C
    ├── hw2.sublime-project
    ├── include
    │   └── .gitkeep
    ├── Makefile
    ├── src
    │   ├── crc32.c
    │   ├── finddup.c
    │   ├── getopt.c
    │   └── main.c
    └── tests
        ├── hw2_tests.c
        └── rsrc
            ├── binary_test.err
            ├── binary_test_names
            ├── binary_test.out
            ├── hard_links_test.err
            ├── hard_links_test_names
            ├── hard_links_test.out
            ├── larger_test.err
            ├── larger_test_names
            ├── larger_test.out
            ├── quick_test.err
            ├── quick_test_names
            ├── quick_test.out
            ├── test_tree
            │   ├── binary1
            │   ├── binary2
            │   ├── empty
            │   ├── empty1
            │   ├── file1
            │   ├── file1.dup
            │   ├── file1.lnk
            │   ├── file2
            │   ├── file2.dup1
            │   ├── file2.dup2
            │   ├── file2.lnk
            │   ├── subdir1
            │   │   └── file1
            │   └── subdir2
            │       └── file2
            ├── valgrind_leak_test.err
            ├── valgrind_leak_test.out
            ├── valgrind_uninitialized_test.err
            └── valgrind_uninitialized_test.out
</pre>

The `doc` directory included with the assignment basecode contains the
original documentation file (a Unix-style `man` page) that was distributed
with the program.  You can format and read the `man` page using the following
command:

<pre>
nroff -man doc/finddup.1C | less
</pre>

  > :nerd:  Since time immemorial, Unix `man` pages have been written in
  > a typesetting language called `roff` (short for "run-off").
  > The `nroff` program processes `roff` source and produces text formatted
  > for reading in a terminal window.

The source files for the `finddup` application are in the `src` directory.
This program had no header files, but if you feel that it is appropriate to add any,
you should put them in the `include` directory.
The source files are as follows:

- `main.c` -- A file that I added to satisfy the requirement that the `main` function
  needs to be in file by itself.  This function just calls `finddup_main` and you should
  not modify this file at all.
- `finddup.c` -- This is the original code for the `finddup` program.  The original
  `main` has been changed to `finddup_main`.
- `getopt.c` -- This is the source code for a version of the `getopt` function that
  was originally distributed with the `finddup` source and is called by it.  This particular
  implementation of `getopt` came from AT&T.  Once you get the program working, you will
  be replacing this function by the modern GNU `getopt` function from the C library.
- `crc32.c` -- This is an implemention of the "CRC-32" (32-bit cyclic redundancy check)
  algorithm which I downloaded from RosettaCode.  As part of the assignment, you will
  be replacing the somewhat dubious algorithm in the original program with this one,
  which (in contrast to the original) I was able to recognize as a *bona fide* CRC-32
  implementation.  **Note: This file is marked "DO NOT MODIFY", so don't modify it.**

There is one preprocessor symbol on which the code is conditionalized, and that is `DEBUG`.
If this symbol is defined during compilation, then the `-d` command-line option is enabled
and debugging printout code is incorporated.

The `tests` directory contains C source code (in file `hw2_tests.c`) for some
Criterion tests I have supplied.  These are by no means to be considered complete or exhaustive.
The subdirectory `tests/rsrc` contains files that are used by the tests.
The subdirectory `tests/rsrc/test_tree` is a sample tree of files and directories that is
designed to exercise various cases in the program logic.  The other files are input files
and reference output files that are used when the tests are run.

Before you begin work on this assignment, you should read the rest of this
document.  In addition, we additionally advise you to read the
[Debugging Document](DebuggingRef.md).

# Part 1: Debugging and Fixing

The command line arguments and expected operation of the program are described
by the following "Usage" message, which is printed within `finddup_main()` in `finddup.c`:

```
Calling sequence:

  finddup [options] list

where list is a list of files to check, such as generated
by "find . -type f -print > file"

Options:
  -l - don't list hard links
  -d - debug (must compile with DEBUG)
```

  > :anguished: What is now the function `finddup_main()` was simply `main()` in the original code.
  > I have changed it so that `main()` now resides in a separate file `main.c` and
  > simply calls `finddup_main()`.  This is to make the structure conform to what is
  > needed in order to be able to use Criterion tests with the program.  **Do not make
  > any modifications to `main.c`.**

There are a few options that the program understands:

- If `-h` or an unknown option is specified, then a help message is printed;

- Normally the program makes a special report on duplicates that are actually
"hard links" to the same underlying file, but if `-l` is specified then this
is not done;

- If `-d` is specified (and if the program has been compiled for debugging),
then debugging information is output.  Specifying `-d` more than once produces
additional debugging output.

You are to complete the following steps:

1. Clean up the code; fixing any compilation issues, so that it compiles
   without error using the compiler options that have been set for you in
   the `Makefile`.

    > :nerd: When you look at the code, you will likely notice very quickly that
    > (except for `main.c`, which I added) it is written in the "K+R" dialect of C
    > (the ANSI standard either didn't exist yet or was newly introduced when the program was written).
    > In K+R C, function definitions gave the names of their
    > formal parameters within parentheses after the function name, but the types of the formal
    > parameters were declared between the parameter list and the function body,
    > as in the following example taken from `getopt.c`:
    >
    > ```c
    > int
    > att_getopt(argc, argv, opts)
    > int	argc;
    > char	**argv, *opts;
    > {
	>   ...
	> }
    > ```
    >
	> Also note that in K+R C it was not required to explicitly declare the return type of
	> a function.  If a return type was not specified, it was presumed to be `int`.

    > :nerd: Another thing you might find it helpful to know is that the source code to
	> the `finddup` program was formatted using `TAB` characters (ASCII 0x9), as was
	> traditional at the time.  The "tab stops" (look up information on typewriters if you
    > don't know what these are) were assumed to be set every eight characters
    > (*i.e.* at columns 9, 17, 25, ...).  When a `TAB` character was encountered in printing
	> a line, the character position was advanced to the next tab stop.
	> The reason I am telling you this is that if you don't have the editor you are using
	> set to understand `TAB` characters with the proper 8-character tab stops, the code
	> will appear to be strangely and incorrectly indented.  This is not how it was intended;
	> what you have to do is adjust your editor's formatting preferences so that it correctly
	> interprets the `TAB` characters.

    As part of your code clean-up, you should provide ANSI C function prototypes where required.
    For functions defined in one `.c` file and used in another, their function prototypes should
    be placed in a `.h` file that is included both in the `.c` file where the function is defined
    and in the `.c` files where the function is used.  For functions used only in the `.c` file
    in which they are defined, if a function prototype is required (it will be, if any use
    of the function occurs before its definition) then the function prototype should be put in
    that same `.c` file and the function should be declared `static`.
    It is not necessary to re-write the existing function definitions into ANSI C style; just
    add any required function prototypes.

    > :nerd: As you clean up the code, you will notice that return types have not been specified
    > for some functions that sometimes return a value and that some functions that have been
    > declared to return a value actually do not.  You should select a prototype for each such
    > function that is appropriate to the way the function was intended to work.  If the function
    > does not actually return any value, use return type `void`.

    Use `git` to keep track of the changes you make and the reasons for them, so that you can
    later review what you have done and also so that you can revert any changes you made that
    don't turn out to be a good idea in the end.

2. Fix bugs.

    Run the program, exercising the various options, and look for cases in which the program
    crashes or otherwise misbehaves in an obvious way.  We are only interested in obvious
    misbehavior here; don't agonize over program behavior that might just have been the choice
    of the original author.  You should use the provided Criterion tests to help point the way,
	though they are not exhaustive.

3. Use `valgrind` to identify any memory leaks or other memory access errors.
   Fix any errors you find.

    Run `valgrind` using a command of the following form:

    <pre>
      $ valgrind --leak-check=full --show-leak-kinds=all [FINDDUP PROGRAM AND ARGS]
    </pre>

    Note that the bugs that are present will all manifest themselves in some way
    either as incorrect output, program crashes or as memory errors that can be
	detected by `valgrind`.  It is not necessary to go hunting for obscure issues
	with the program output.
    Also, do not make gratuitous changes to the program output, as this will
    interfere with our ability to test your code.

   > :scream:  Note that we are not considering memory that is "still reachable"
   > to be a memory leak.  This corresponds to memory that is in use when
   > the program exits and can still be reached by following pointers from variables
   > in the program.  Although some people consider it to be untidy for a program
   > to exit with "still reachable" memory, it doesn't cause any particular problem.

   > :scream: You are **NOT** allowed to share or post on PIAZZA
   > solutions to the bugs in this program, as this defeats the point of
   > the assignment. You may provide small hints in the right direction,
   > but nothing more.

# Part 2: Changes to the Program

## Eliminate filename length limitation

The original program uses fixed-size arrays to read filenames.  Such arrays are used in
several places: `main()`, `get_crc()`, `getfn()`, and `fullcmp()`.
This leads to the arbitrary limitation of `MAXFN` characters on filename lengths.
A more modern way to do things would be to arrange for filename storage to be dynamically
allocated as the filename is read.  The function `getline(3)` is designed for this
purpose.  Read the manual page on this function and rewrite the places where filenames
are read so that this function is used to dynamically allocate storage for the
filenames.  Note the storage returned by `getline()` will eventually need to be freed
in order to avoid storage leaks.  Find appropriate places to do this and add the
necessary calls to `free()`.

## Ignore special files

The original `finddup` program expects to see only the names of regular files in
the input file.  If the input file happens to contain the name of something that
is not a regular file, such as a directory or a symbolic link, then the program
will attempt to treat it as if it were a file, which will generally produce
meaningless results.  What you should do is to modify the program so that it ignores
anything in the input that is not the name of a regular file.
To do this, you will have to read the manual pages for the `stat(2)` and `lstat(2)`
system calls.

## Replace the "CRC-32" implementation

The original `finddup.c` contains a function `get_crc()`, which is supposed to compute
a CRC-32 checksum of the contents of a file.  The algorithm that is used doesn't look
to me much like a true CRC-32 calculation, and when I went hunting around
(*e.g.* starting by searching `Cyclic Redundancy Check` on Google and Wikipedia)
I was in fact unable to identify it as a known variant of a CRC-32 algorithm.
So, I decided it would be a good idea to replace this implementation with a standard
one.  I obtained the code in `src/crc32.c` from
[https://rosettacode.org/wiki/CRC-32#Library](RosettaCode).
What you are to do is, **without changing the code in `src/crc32.c`**, to rewrite
the `get_crc()` function in `finddup.c` so that it makes use of this new code instead
of the dubious code it originally contained.
Verify that the result really does produce correct CRC-32 checksums.
For example, you may use the Linux command `crc32` to compute the CRC-32 checksum of
a file and compare the results with that computed by your updated version of `get_crc()`.

## Replace the AT&T `getopt()` Code

As originally distributed, the `finddup` program contained a source code file `getopt.c`,
which is identified as being an implementation of `getopt()` obtained from AT&T.
I have renamed this function `att_getopt()`, so that it doesn't clash with the modern
GNU `getopt()` function that is in the C library.
What you are to do is rewrite the code in `finddup_main()` so that it uses the modern
version of `getopt()`; in particular, use the function `getopt_long()` which will allow the
program to accept long options such as `--help` as well as the traditional single-character
options such as `-h`.

In particular, your updated program should understand the following options:

   - `--help` as equivalent to `-h`
   - `--no-links` as equivalent to `-l`
   - `--debug` as equivalent to `-d`

Your version of `--debug` should accept an optional integer argument specifying the
"debug level".  The debug level should be treated equivalently to specifying the
short option `-d` that many times; thus, `--debug 2` should be equivalent to `-d -d`.

You will probably need to read the Linux "man page" on the `getopt` package.
This can be accessed via the command `man 3 getopt`.  If you need further information,
search for "GNU getopt documentation" on the Web.

> :scream: You MUST use the `getopt_long()` function to process the command line
> arguments passed to the program.  Your program should be able to handle cases where
> the (non-positional) flags are passed IN ANY order.  Make sure that you test the
> program with prefixes of the long option names, as well as the full names.

Note that the default behavior of GNU `getopt()` is to permute the program arguments
during processing, so that once the processing of option arguments has completed,
any non-option arguments will be left at the end of the argument list, even if those
non-option arguments originally occurred before some option arguments.
The `finddup` program always expects exactly one non-option argument, which is the
name of the file containing the list of files to be analyzed for duplicates.
However, you should look carefully at the way the option processing was performed in
the original code, because there might be some incompatibility between that code
and the use of the GNU functions.

# Part 3: Testing the Program

For this assignment, you have been provided with a basic set of
Criterion tests to help you debug the program.  We encourage you
to write your own as well as it can help to quickly test inputs to and
outputs from functions in isolation.

In the `tests/hw2_tests.c` file, there are seven test examples.
You can run these with the following command:

<pre>
    $ bin/finddup_tests
</pre>

To obtain more information about each test run, you can supply the
additional option `--verbose=1`.

The tests have been constructed so that they will point you at most of the
problems with the program.
Each test has one or more assertions to make sure that the code functions
properly.  If there was a problem before an assertion, such as a "segfault",
the test will print the error to the screen and continue to run the
rest of the tests.
Two of the tests use `valgrind` to verify that no memory errors are found.
If errors are found, then you can look at the log file that is left behind by
the test code.
Alternatively, you can better control the information that `valgrind` provides
if you run it manually.

The tests included in the base code are not true "unit tests", because they all
run the program as a black box using `system()`.
You should be able to follow the pattern to construct some additional tests of
your own, and you might find this helpful while working on the program.
For this program, there are a few functions (for example, `comp()`, `get_crc()`,
and `fullcmp()`) that are amenable to individual testing using true unit tests.
You are encouraged to try to write some of these tests so that you learn how
to do it, and it is possible that some of the tests we use for grading will be
true unit tests like this.  Note that in the next homework assignment unit tests
will likely be very helpful to you and you will be required to write some of your own.
Criterion documentation for writing your own tests can be found
[here](http://criterion.readthedocs.io/en/master/).

  > :scream: Be sure that you test non-default program options to make sure that
  > the program does not crash when they are used.

# Hand-in Instructions

Ensure that all files you expect to be on your remote repository are committed
and pushed prior to submission.

This homework's tag is: `hw2`

<pre>
$ git submit hw2
</pre>
