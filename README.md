# Debugging Zcash

This guide is designed to give beginners of C++ development and/or people new
to the zcash code base an overview of the tools available for debugging
issues as well as giving hints where issues may trip you up. This page
borrows from the excellent [debugging bitcoin](https://github.com/fjahr/debugging_bitcoin/blob/master/README.md)
and I'd like to gratefully acknowledge the author of that document. Also,
I'd like to thank @str4d and @daira for their helpful suggestions.

## Table of contents

* [Getting ready to debug](#getting-ready-to-debug)
* [Removing preprocessor mystery](#removing-preprocessor-mystery)
* [Debugging](#debugging)
    * [Running your own bitcoind](#running-your-own-bitcoind)
        * [Logging](#logging-from-own-bitcoind)
        * [Debugging](#debugging-from-own-bitcoind)
    * [Running unit tests](#running-unit-tests)
        * [Logging from unit tests](#logging-from-unit-tests)
        * [Logging from implementation code](#logging-from-unit-tests-code)
        * [Debugging](#debugging-from-unit-tests)
    * [Running functional tests](#running-functional-tests)
        * [Logging from functional tests](#logging-from-functional-tests)
        * [Logging from C++ code](#logging-from-functional-tests-code)
        * [Debugging](#debugging-from-functional-tests)
* [More Tools for Segfaults](#more-tools-for-segfaults)
   * [Core dumps](#core-dumps)
   * [Valgrind](#valgrind)
* [Further resources](#further-resources)


## Getting ready to debug

Debugging involves a lot of compiling, so you definitely want to
speed it up as much as possible. I recommend looking at the
[general productivity notes](https://github.com/bitcoin/bitcoin/blob/master/doc/productivity.md#general) in
the bitcoin core docs and install `ccache` and optimize your configuration.

It's important to disable optimizations using the `-O0` flag, otherwise
debugging will be impossible as symbol names will not be recognizable, many
local variables and function arguments are optimized away (not visible),
and the execution flow while single-stepping is very strange.

```
make clean
CONFIGURE_FLAGS='CXXFLAGS=-O0' zcutil/build.sh -j8
```

If you're interested in debugging one or a few files, you can skip the `make clean`
and instead `touch` just those files; it's okay for the executable to comprise
object files with different levels of optimization. Another shortcut to avoid
running `build.sh` is to directly edit `src/Makefile` and change the `O3`
to `O0` in the `CXXFLAGS` definition. 

Execution speed is 4 to 5 times as slow with `-O0`, so don't do any performance
evaluation using this build, although it's generally reliable to compare two
different algorithms that are both compiled using `-O0`.

## Removing preprocessor mystery

This isn't directly related to debugging, but it's sometimes useful to see
the output of the preprocessor, that is, the code with all the header files
included and all the macros expanded. It's often unclear which header file
is defining a particular symbol, for example (it may be defined differently
in different header files).

Here's a hack to do that.
First, touch the file in question, and see which object file it generates (for
the sake of clarity, some extraneous output is not shown):

```
$ touch src/rpc/mining.cpp
$ (cd src; make zcashd)
  CXX      rpc/libbitcoin_server_a-mining.o
  AR       libbitcoin_server.a
  CXXLD    zcashd
$ 
```
(I tend to run this kind of stuff directly within `src/`; the parenthesis start a subshell, so your main shell's current working directory doesn't change.) Now, recompile just that object file, but with the verbose flag, `V=1`:

```
$ touch src/rpc/mining.cpp
$ (cd src; make V=1 rpc/libbitcoin_server_a-mining.o)
/home/larry/zcash/depends/x86_64-pc-linux-gnu/share/../native/bin/ccache /home/larry/zcash/depends/x86_64-pc-linux-gnu/native/bin/clang++ (...) -c -o rpc/libbitcoin_server_a-mining.o `test -f 'rpc/mining.cpp' || echo './'`rpc/mining.cpp
$ 
```
(I've left off most of the large number of messy options for clarity.) Now, copy and paste this long compile command, but replace:
```
-c -o rpc/libbitcoin_server_a-mining.o
```
with
```
-E -o cpp.out
```
(or you can leave off the `-o cpp.out` and redirect standard output to a file.)
```
$ (cd src; /home/larry/zcash/depends/x86_64-pc-linux-gnu/share/../native/bin/ccache /home/larry/zcash/depends/x86_64-pc-linux-gnu/native/bin/clang++ (...) -E -o cpp.out `test -f 'rpc/mining.cpp' || echo './'`rpc/mining.cpp)
$ 
```
Now you can view `src/cpp.out` in an editor, and see everything expanded. It's hundreds of thousands of lines, but you can search for things.

Another useful option along similar lines is `-H`, which shows you the include tree (which header files include which header files; each additional dot is another level of include). This output appears on standard error (which you can redirect to a file):
```
$ (cd src; /home/larry/zcash/depends/x86_64-pc-linux-gnu/share/../native/bin/ccache /home/larry/zcash/depends/x86_64-pc-linux-gnu/native/bin/clang++ (...) -H -E -o cpp.out `test -f 'rpc/mining.cpp' || echo './'`rpc/mining.cpp)
. ./amount.h
.. ./serialize.h
... ./compat/endian.h
.... ./config/bitcoin-config.h
.... /home/larry/zcash/depends/x86_64-pc-linux-gnu/native/bin/../include/c++/v1/stdint.h
..... /home/larry/zcash/depends/x86_64-pc-linux-gnu/native/bin/../include/c++/v1/__config
...... /usr/include/features.h
....... /usr/include/stdc-predef.h
(a few thousand more lines; this is one reason compiling is slow!)
$ 
```
You can use `-H` with either the `-c` or the `-E` flag.

## Debugging

This guide will use [`gdb`](https://sourceware.org/gdb/current/onlinedocs/gdb/)
instead of `lldb` (even though zcash is now compiled using
`clang`), since `gdb` is what I know, and it seems to work. But you should give `lldb`
a try, it may be better in some ways.
The two tools are [very similar](https://lldb.llvm.org/use/map.html).

You probably should set up a `$HOME/.gdbinit` file; here's a copy of mine:
```
set print pretty
set height 0
set print elements unlimited
set history save on
set logging on

define lps
    set $p = logFastReached
    while ($p)
        printf "enabled=%d ", $p->enabled
        printf "category=%s\t", $p->category
        printf "line=%s\n", $p->line
        set $p = $p->next
    end
end

define freeze
    set scheduler-locking on
end

document freeze
    prevent other threads from running while stepping
end

define pbf
    printf "nSrcPos:%d nReadPos:%d nReadLimit:%d nRewind:%d\n", $arg0.nSrcPos, $arg0.nReadPos, $arg0.nReadLimit, $arg0.nRewind
end

document pbf
    CBufferedFile -- print its fields
end
```

The `set history save on` causes all of gdb's output to be appended to `gdb.txt`.
It's useful to have this file open in an editor because it's much easier to
interpret large data structures in an editor (you can find matching brackets,
for example). You also can define [gdb macros](https://sourceware.org/gdb/current/onlinedocs/gdb/Define.html#Define)
which are little functions containing `gdb` commands, great for printing
the elements of a linked-list (as `lps` does) or other containers.
The `document` entries provide a help string:
```
(gdb) help pbf
    CBufferedFile -- print its fields
(gdb) 
```
The documentation convention is that the arguments are listed before the double-dash
(in this case, a `CBufferedFile` reference) and the description follows. The macro
references its arguments using `$arg0`, `$arg1`, and so on. Variables in `gdb`
start with a dollar-sign.

You can list the user-defined macros like this:
```
(gdb) help user-defined
User-defined commands.
The commands in this class are those defined by the user.
Use the "define" command to define a command.

List of commands:

freeze --     prevent other threads from running while stepping
lps -- User-defined
pbf --     CBufferedFile -- print its fields

Type "help" followed by command name for full documentation.
Type "apropos word" to search for commands related to "word".
Command name abbreviations are allowed if unambiguous.
(gdb) 
```
You can cause `gdb` to re-read your `.gdbinit` by typing:
`(gdb) source ~/.gdbinit` (useful when you're trying to get
a macro to work).

### Starting `zcashd` from the debugger

It's useful to turn off the metrics display (the Zcash logo and the heart), otherwise
it's difficult to read the debugger output:
```
gdb --args src/zcashd -showmetrics=0

#### Logging from own `bitcoind`

In general you can use to your `std::out` but this will not appear in any logs.
It is rather recommended to use `LogPrintf`. Insert into your code a line similar
to the following example.

```
LogPrintf("@@@");
```

You can then grep for the result in your `debug.log` file:
```
$ cat ~/Library/Application\ Support/Bitcoin/regtest/debug.log | grep @@@
```
This example shows the path of the regtest environment `debug.log` file. Remember
to change this if you are logging from testnet or another environment.

If you would like to log from inside files that validate consensus rules
(see [`src/Makefile.am`](https://github.com/bitcoin/bitcoin/blob/master/src/Makefile.am#L407))
then you will errors from missing header files and when you have added those
the linker will complain. You can make this work, of course, but I would
recommend you use a debugger in that context instead.

#### Debugging from your own `bitcoind`

You can start your bitcoind using a debugging tool like `lldb` in order to debug
the code:
```
$ lldb src/bitcoind
```

Within the `lldb` console you can set breakpoints before actually running starting
the `bitcoind` process using the `run` command. You can then interact with the `bitcoind`
process like you normally would using your `bitcoin-cli`.

### 
sts

You are running unit tests which are located at `src/test/` and use the BOOST library
test framework. These are executed by a seperate executable that is also compiled with 
make. It is located at `src/test/test_bitcoin`.

So helpful tips about execution (this uses the Boost library).

Run just one test file:
`src/test/test_bitcoin --log_level=all --run_test=getarg_tests`

Run just one test:
`src/test/test_bitcoin --log_level=all --run_test=*/the_one_test`

#### Logging from unit tests

Logging from the tests you need to use the BOOST framework provided functions to
achieve seeing the logging output. Several methods are available, but simplest
is probably adding `BOOST_TEST_MESSAGE` in your code:

```
BOOST_TEST_MESSAGE("@@@");
```

#### Logging from unit tests code

To have log statements from your code appear in the unit test output you will have
to print to `stderr` directly:

```
fprintf(stderr, "from the code");
```


#### Debugging Unit Tests

You can start the `test_bitcoin` binary using `lldb` just like bitcoind. This allows you
to set breakpoints anywhere in your unit test or the code and then execute the tests
any way you want using the `run` keyword followed by the arguments you would normally
pass to `test_bitcoin` when calling it directly.

```
$ lldb src/test/test_bitcoin
(lldb) target create "src/test/test_bitcoin"
Traceback (most recent call last):
  File "<input>", line 1, in <module>
  File "/usr/local/Cellar/python@2/2.7.16/Frameworks/Python.framework/Versions/2.7/lib/python2.7/copy.py", line 52, in <module>
    import weakref
  File "/usr/local/Cellar/python@2/2.7.16/Frameworks/Python.framework/Versions/2.7/lib/python2.7/weakref.py", line 14, in <module>
    from _weakref import (
ImportError: cannot import name _remove_dead_weakref
Current executable set to 'src/test/test_bitcoin' (x86_64).
(lldb) run --log_level=all --run_test=*/lthash_tests
```


### Running functional tests

You are running tests located in `test/functional/` which are written in `python`.

#### Logging from functional tests

for log level debug run functional tests with `--loglevel=debug`.
```
self.log.info("foo")
self.log.debug("bar")
```

Use `--tracerpc` to see the log outputs from the RPCs of the different nodes running in the functional
test in std::out.

If it doesn't fail, make it fail and use this:
```
TestFramework (ERROR): Hint: Call /Users/FJ/projects/clones/bitcoin/test/functional/combine_logs.py '/var/folders/9z/n7rz_6cj3bq__11k5kcrsvvm0000gn/T/bitcoin_func_test_epkcr926' to consolidate all logs
```

You can even assert on logging messages using `with self.nodes[0].assert_debug_log(["lkagllksa"]):`.
However, don't change indentation of test lines. Just insert somewhere indented between def run_test
and the body of it.

#### Logging from functional tests code

Using `LogPrintf`, as seen before, you will be able to see the output in the combined logs.

#### Debugging from functional tests

##### 1. Compile Bitcoin for debugging

In case you have not done it yet, compile bitcoin for debugging (change other config flags as you need them).

```
$ make clean
$ ./configure CXXFLAGS="-O0 -ggdb3"
$ make -j "$(($(sysctl -n hw.physicalcpu)+1))"
```

##### 2. Ensure you have `lldb` installed

Your should see something like this:
```
$ lldb -v
lldb-1001.0.13.3
```

##### 3. Halt functional test

You could halt the test with a sleep as well but much cleaner (although a little confusing maybe) is to use another
debugger within python: `pdb`.

Add the following line before the functional test causing the error that you want to debug:
```
import pdb; pdb.set_trace()
```
Then run your test. You will need to call the test directly and not run it through the `test_runner.py` because
then you could not get access to the `pdb` console.
```
$ ./test/functional/example_test.py
```

##### 4. Attach `lldb` to running process

Start `lldb` with the running bitcoind process (might not work if you have other bitcoind processes running). Just running `lldb` instead of `PATH=/usr/bin /usr/bin/lldb` might work for you, too, but lots of people seem to run into problems when lldb tries to use their system python in that case.
```
$ PATH=/usr/bin /usr/bin/lldb -p $(pgrep bitcoind)
```

##### 5. Set your breakpoints in `lldb`

Set you breakpoint with `b`, then you have to enter `continue` since `lldb` is setting a stop to the process as well.
```
(lldb) b createwallet
Breakpoint 1: [...]
(lldb) continue
Process XXXXX resuming
```

##### 6. Let test continue

You can now let you test continue so that the process is actually running into your breakpoint.

```
(Pdb) continue
```

You should now see something like this in you `lldb` and can start debugging:
```
Process XXXXX stopped
* thread #10, name = 'bitcoin-httpworker.3', stop reason = breakpoint 1.1
    frame #0: 0x00000001038c8e43 bitcoind`createwallet(request=0x0000700004d55a10) at rpcwallet.cpp:2642:9
   2639	static UniValue createwallet(const JSONRPCRequest& request)
   2640	{
   2641	    RPCHelpMan{
-> 2642	        "createwallet",
   2643	        "\nCreates and loads a new wallet.\n",
   2644	        {
   2645	            {"wallet_name", RPCArg::Type::STR, RPCArg::Optional::NO, "The name for the new wallet. If this is a path, the wallet will be created at the path location."},
Target 0: (bitcoind) stopped.
(lldb) 
```




## More Tools for Segfaults

### Core dumps

A core dump is a full dump of the working memory of your computer. When activated it will
be created in `/cores` on your computer in case a segfault happens.

To activate core dumps on macOS you have to run
```
$ ulimit -c unlimited
```
then you have to run the process where you are observing the segfault in the same terminal.
You can then inspect them in your `/cores` directory.

You should always make sure to not generate more core dumps than you need to and clean up
your `/cores` directory when you have solved your issue. Core dumps are huge files and
can clutter up your disc space very quickly.

### `valgrind`

#### Install `valgrind`
On newer MacOS  versions this is harder than you might think because you
can only install it through homebrew up to version 10.13. If you currently
have an up-to-date MacOS Mojave installed you will have to use the
following script to make it work.

```
$ brew install --HEAD https://raw.githubusercontent.com/sowson/valgrind/master/valgrind.rb
```

#### Run `bitcoind` with `valgrind`

Using `valgrind` works similar to `lldb`. You start the executable where the segfault might
happen using `valgrind`, when an error is observed you will be able to inspect it.

For example:
```
$ sudo valgrind src/bitcoind -regtest
```

Now you can interact with your `bitcoind` normally using `bitcoin-cli`, for example, and
trigger the error by hand using the right parameters.





## Further resources

### Getting help
- [Bitcoin Stackexchange](https://bitcoin.stackexchange.com) should be your first point of contact if you are truely stuck
- [IRC channels](https://en.bitcoin.it/wiki/IRC_channels) are another possibility to find help for specific topics

### READMEs
- Bitcoin core READMEs have lots of information of course. You may find some overlaps with this guide but they should still
be very helpful.
   - https://github.com/bitcoin/bitcoin/blob/master/src/test/README.md
   - https://github.com/bitcoin/bitcoin/blob/master/test/README.md
   
### Other resources
https://medium.com/provoost-on-crypto/debugging-bitcoin-core-functional-tests-cc0aa6e7fd3e
https://bitcoin.stackexchange.com/questions/76521/debugging-bitcoin-unit-tests

### `lldb`
[Using `lldb` as a standalone debugger](https://developer.apple.com/library/archive/documentation/IDEs/Conceptual/gdb_to_lldb_transition_guide/document/lldb-terminal-workflow-tutorial.html)

[LLVM Tutorial on `lldb`](https://lldb.llvm.org/use/tutorial.html)

### BOOST
- BOOST Introduction: http://archive.is/dRBGf
- BOOST LIB: https://www.boost.org/doc/libs/1_63_0/libs/test/doc/html/boost_test/test_output/test_tools_support_for_logging/checkpoints.html
 
