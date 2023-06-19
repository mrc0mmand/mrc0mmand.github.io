---
layout: post
title:  "Collecting code coverage with gcov"
date:   2023-06-13
tags:   C coverage gcov systemd
---

Having a code coverage report at hand when working on a more complex project is quite handy, and becomes more important
as the project grows. While trying to generate such reports for [systemd](https://github.com/systemd/systemd) I've
encountered several challenges that are, in my opinion, worth documenting (especially since I've been burned by a couple
of them repeatedly). Thanks to the coverage reports we've identified several quite undertested areas in systemd, some of
them plagued with nasty issues, which further emphasizes the usefulness of such reports.

## Building with gcov

Doing a coverage-aware build is usually pretty straightforward, as it is a matter of adding the `--coverage` compile and
link option to your build process (i.e. to `$CFLAGS` and `$LDFLAGS`) - `--coverage` expands to `-fprofile-arcs
-ftest-coverage` when compiling and to `-lgcov` when linking the code. Also, even though `gcov` was originally a
GCC-only thing, clang nowadays ships a GCC-compatible implementation as well, so you're not limited to GCC.

If you use one of the available build systems, enabling coverage might be as simple as flipping a switch - for example
with [meson](https://mesonbuild.com/) all you need to do is to add `-Db_coverage=true` to the `meson setup` call and
meson handles everything behind the scenes automagically.

After building the project with the necessary options, you should end up with a bunch of `.gcno` files alongside the
object files[^1], for example:

```bash
$ ls
bar.c  cov_test.c  foo.c  meson.build  shared.h
$ meson setup build -Db_coverage=true
...
lcov: LCOV version 1.14
genhtml: LCOV version 1.14
$ ninja -C build
...
$ find build -name "*.o" -o -name "*.gcno"
build/bar.p/bar.c.gcno
build/bar.p/bar.c.o
build/cov_test.p/bar.c.gcno
build/cov_test.p/cov_test.c.gcno
build/cov_test.p/cov_test.c.o
build/cov_test.p/foo.c.gcno
build/cov_test.p/foo.c.o
```

The `.gcno` files (*notes* files in `gcov` jargon) are crucial as they contain information necessary to reconstruct
basic block graphs and to assign source line numbers to blocks.

## Collecting and inspecting the coverage

After running tests with the coverage-aware build, some of the `.gcno` files should now be accompanied by newly created
`.gcda` files - these *data* files contain the actual coverage information:

```bash
$ find build -name "*.gcno" -o -name "*.gcda"
build/cov_test.p/cov_test.c.gcno
build/cov_test.p/foo.c.gcno
build/bar.p/bar.c.gcno

$ build/cov_test

$ find build -name "*.gcno" -o -name "*.gcda"
build/cov_test.p/cov_test.c.gcno
build/cov_test.p/foo.c.gcno
build/cov_test.p/foo.c.gcda
build/cov_test.p/cov_test.c.gcda
build/bar.p/bar.c.gcno
```

We can collect the coverage data using the lcov(1) utility. But before collecting the actual coverage, we first have to
capture the *initial* coverage, which basically sets the hit counter for every source line to zero. You can skip this
step and everything would still work in terms of reported coverage, but if you have parts of code that your test
coverage misses completely, they might not show up in the final coverage report hiding code that's in a dire need of
some coverage, i.e. hide the exact thing we're trying to uncover.

Using the code from previous examples, where only code from `cov_test.c` is run, files `foo.c` and `bar.c` are
completely uncovered, and `bar.c` is a completely separate build unit, we'll get the following:

```bash
$ lcov --capture --directory build --output-file coverage.info
Capturing coverage data from build
Found gcov version: 12.2.1
Using intermediate gcov format
Scanning build for .gcda files ...
Found 2 data files in build
Processing cov_test.p/foo.c.gcda
Processing cov_test.p/cov_test.c.gcda
Finished .info-file creation

$ lcov --list coverage.info
Reading tracefile coverage.info
               |Lines       |Functions  |Branches
Filename       |Rate     Num|Rate    Num|Rate     Num
=====================================================
[/home/fsumsal/tmp/cov-test/]
cov_test.c     |57.1%      7|50.0%     2|    -      0
foo.c          | 0.0%      3| 0.0%     1|    -      0
=====================================================
         Total:|40.0%     10|33.3%     3|    -      0
```

Notice that `bar.c` is missing completely from the report. Now let's do the same thing, but do an initial capture as
well, and then merge both captures into a final one:

```bash
$ lcov --initial --capture --directory build --output-file base.info
...
$ lcov --capture --directory build --output-file coverage.info
...
$ lcov --add-tracefile base.info --add-tracefile coverage.info --output-file coverage-final.info
...
$ lcov --list coverage-final.info
Reading tracefile coverage-final.info
               |Lines       |Functions  |Branches
Filename       |Rate     Num|Rate    Num|Rate     Num
=====================================================
[/home/fsumsal/tmp/cov-test/]
bar.c          | 0.0%      5| 0.0%     2|    -      0
cov_test.c     |57.1%      7|50.0%     2|    -      0
foo.c          | 0.0%      3| 0.0%     1|    -      0
=====================================================
         Total:|26.7%     15|20.0%     5|    -      0
```

Now the report contains all files from our project as expected.

To get more detail on what's covered and what's not, you can use the genhtml(1) utility to generate an HTML report
for the whole source tree:

```bash
$ genhtml --show-details --output-directory coverage_html coverage-final.info
Reading data file coverage-final.info
Found 3 entries.
Found common filename prefix "/home/fsumsal/tmp"
Writing .css and .png files.
Generating output.
Processing file cov-test/bar.c
Processing file cov-test/foo.c
Processing file cov-test/cov_test.c
Writing directory view page.
Overall coverage rate:
  lines......: 26.7% (4 of 15 lines)
  functions..: 20.0% (1 of 5 functions)

$ firefox coverage_html/index.html
```

## Potential issues

There's a couple of potential issues that might not be completely obvious from the start, but may bite you later when
you begin with the actual testing:

- All the extra counting of line hits takes a toll on the performance, which in some cases might cause timeouts
  throughout your test suite - you'll need to account for that.

- The extra code to make the coverage work + the `.gcno` and `.gcda` files take additional space, so you need to adjust
  the size of test images if that's relevant for your project (for example, in case of systemd only the `.gcno` files
  take additional ~100 MiB of space).

- You need to carry the build directory hierarchy[^1] with the `.gcno` files to the testbed/SUT (if you don't run the
  tests locally), otherwise gcov won't be able to generate the coverage. You really only need the `.gcno` files (but
  with the original directory layout intact), other files from the build directory can be left behind. More on the build
  directory shenanigans below.

## Where is my coverage?

So, you've got everything set up, built correctly, coverage collection went smooth as well, but after checking the
reports you notice that lines that definitely should have been covered by tests are still glowing red. There's actually
quite a number of reasons why the coverage might be missing, some of them are quite obvious with a helpful error
message, and some of them are silent and *slightly* frustrating to figure out. Let's start with the most obvious one.

### Inaccessible build directory

`gcov` requires both read and write access to the build directory, where it reads the notes files (`.gcno`) and also
writes the coverage counters (`.gcda`), so having correct permissions throughout the whole build directory is crucial.
For unit tests this is usually trivial, but some more involved tests, like integration tests that make use of multiple
users, might not be happy. If you're lucky and the `stderr` from the test doesn't end up in `/dev/null` you might notice
`gcov` complaining:

```
profiling: /.../build/cov_test.p/cov_test.c.gcda: cannot open: Permission denied
```

The solution might be as easy as `chmod/chown`-ing the build directory with the correct mode/user, or making use of
POSIX ACLs[^2] to allow anyone to read/write to the build directory:

```
$ setfacl --recursive --modify=d:o:rwX --modify=o:rwX build
```

(good enough for the coverage run, not recommended otherwise). Apart from file permissions, SELinux (or any other LSM)
might be involved as well, requiring additional tweaks.

Things might get slightly more complicated if the test code is running in a systemd unit that makes use of some
sandboxing directives[^3], most notably `ProtectSystem=` and `ProtectHome=`, or directives that imply them (like
`DynamicUser=yes`).  These directives either mount parts of the file system read-only or overmount them with a temporary
file system, effectively making the build directory inaccessible in case it's present in the affected tree.

In this case you have basically two options:

1. Temporarily disable sandboxing for the particular unit, either by editing the unit directly (not recommended) or by
   using a dropin with `ProtectSystem=no`/`ProtectHome=no`/etc.

2. Make the build directory explicitly accessible using the `ReadWritePaths=` directive, again by tweaking the affected
   unit directly or via a dropin

You may find yourself in a situation when you'll need to do both, since even though you can disable `ProtectSystem=` and
`ProtectHome=` when used explicitly, with `DynamicUser=yes` this won't work and you'll need to resort `ReadWritePaths=`.

A whole another chapter are containers, if your tests utilize them, in which case you have to mount the build dir with
appropriate permissions into the container in the first place and then account for any sandboxing that may take effect.

And to add one head-scratcher from my systemd-adventures, which took a me bit to figure out: our `systemd-homed` test
temporarily overmounts `/home` with a temporary file system, as it, as the name suggests, tests home directories.  This,
however, happens after we copy the build directory into the testbed. So, if your original build directory happens to be
under `/home` (which is usually the case), it'll disappear for the duration of the test, which makes `gcov` complain
about missing `.gcno` files. However, if you check the file system before and after the test, the files are there, as
`/home` is overmounted only for the duration of the test. This is, of course, a weird corner case, but software testing
is full of such weird corner cases.

### Using `_exit()`

In some situations the code might use `_exit()` instead of the regular `exit()` - this is usually the case in forked off
processes. The annoying thing is that `_exit()` skips the exit handlers, mainly the one that updates the coverage
counters, so we effectively lose all coverage collected up to that point.

One particularly easy (but not very pretty) solution is to use a macro to replace all `_exit()` calls with our own exit
function that updates the coverage counters just before exiting. The main benefit of this method is that it is
completely self-contained, as both the macro and the function are in a separate header file which can be pulled in only
when we're building a coverage-aware build, thus requiring no if-def-ery or other noisy code changes:

```c
#pragma once

void __gcov_dump(void);
void _exit(int);

static inline _Noreturn void _coverage__exit(int status) {
        __gcov_dump();
        _exit(status);
}
#define _exit(x) _coverage__exit(x)
```

Including this in the build is then a matter of adding `-include coverage.h` to the compiler command line, properly
wrapped in checks that make sure we do this only when building with coverage enabled.

### `exec*()` syscalls

Another source of potential headache are some more *exotic* syscalls from the `exec()` family. `gcov` provides a wrapper
for most of them[^4], which dumps coverage just before the call, but a couple of syscalls are left out.

The solution is pretty similar to the previous `_exit()` case - we just create the wrappers ourselves, use a macro to
replace all *problematic* calls with our own interceptor, and include the final header file in the build:

```c
#pragma once

void __gcov_dump(void);
void __gcov_reset(void);

int execveat(int, const char *, char * const [], char * const [], int);
int execvpe(const char *, char * const [], char * const []);

static inline int _coverage_execveat(
                        int dirfd,
                        const char *pathname,
                        char * const argv[],
                        char * const envp[],
                        int flags) {
        __gcov_dump();
        int r = execveat(dirfd, pathname, argv, envp, flags);
        __gcov_reset();

        return r;
}
#define execveat(d,p,a,e,f) _coverage_execveat(d, p, a, e, f)

static inline int _coverage_execvpe(
                        const char *file,
                        char * const argv[],
                        char * const envp[]) {
        __gcov_dump();
        int r = execvpe(file, argv, envp);
        __gcov_reset();

        return r;
}
#define execvpe(f,a,e) _coverage_execvpe(f, a, e)
```

Here I used `execveat()` and `execvpe()` as an example, since I had to deal with these two syscalls in my systemd
adventures, but the same applies to any other syscall that is missing a gcov wrapper.

### Unhandled signals

If your application is long running and/or you stop it using a signal, be it explicitly (manually) or implicitly (if it
runs in a systemd unit and you issue `systemctl stop`), make sure you handle at least `SIGTERM`. Otherwise you might
find yourself wondering why there is a gaping hole in your coverage even after all tests run successfully.

Let's take this simple example program, that prints `Hello world` and then waits for a signal:

```c
#include <assert.h>
#include <stdio.h>
#include <signal.h>
#include <unistd.h>

int main(int argc, char *argv[])
{
    puts("Hello world");
    pause();

    return 0;
}
```

After building it, we get the expected `.gcno` file:

```bash
$ gcc --coverage -o main main.c
$ ls
main  main.c  main.gcno
```

However, if you run it, and then kill it with `SIGTERM` from another terminal, you'll notice that there are no newly
generated `.gcda` files:

```
$ ./main
Hello world
## Run `pkill -f ./main` from a different terminal
Terminated
$ ls
main  main.c  main.gcno
```

This can be easily remedied by handling `SIGTERM`. It doesn't have to be anything fancy, the signal handler can even be
empty, for example:

```
#include <assert.h>
#include <stdio.h>
#include <signal.h>
#include <unistd.h>

void noop_signal_handler(int) {
    /* Nothing here */
}

int main(int argc, char *argv[])
{
    static const struct sigaction sigterm = {
        .sa_handler = noop_signal_handler,
        .sa_flags = SA_RESTART,
    };

    assert(sigaction(SIGTERM, &sigterm, NULL) >= 0);

    puts("Hello world");
    pause();

    return 0;
}
```

Now, if we build it, run it, and kill it with `SIGTERM` again, we should see an immediate difference in the generated
files:

```
$ ./main
Hello world
## Run `pkill -f ./main` from a different terminal
$ ls
main  main.c  main.gcda  main.gcno
```


In the end, the more complex the project is the more likely it is that you'll hit a test case that's incompatible with
collecting coverage, and it is completely up to you how much time, additional code, and sanity you're willing to
sacrifice to make the coverage reports as accurate as possible.

## What's next?

Generating the coverage reports locally is quite handy, especially when checking if everything works correctly after
fixing one of the aforementioned issues. But to make the reports really useful it is necessary to regenerate them
regularly. The *where*, *how often*, and *how* will change wildly depending on your project and how you run the tests -
for smaller projects it might be feasible to generate the report for each pull/merge request, so you have an immediate
response to how the coverage changes with your patches. For larger projects with larger test suites this might prove to
be next to impossible, especially due to the gcov overhead that might add up quickly.

To make all this easier you can use one of the code coverage services, like [Coveralls](https://coveralls.io) or
[Codecov](https://about.codecov.io/) which provide integrations with existing CI systems that make setting things up a
matter of couple of lines (in case you're already using one of the *common* CI systems). But even if you have an unusual
CI setup, the tooling is quite flexible to account for that as well.

To provide an example - in systemd we use Coveralls[^5], but since our integration tests run in a custom environment,
and take quite a considerable amount of time when running with gcov, we upload the report in a daily cron job using
Coverall's [Universal Coverage Reporter](https://github.com/coverallsapp/coverage-reporter), which, among other formats,
supports the output from the `lcov` commands above.

[^1]: Assuming you didn't use `-fprofile-dir=path` to store the notes files elsewhere
[^2]: acl(5), setfacl(1)
[^3]: [systemd.exec(5), section *Sandboxing*](https://www.freedesktop.org/software/systemd/man/systemd.exec.html#Sandboxing)
[^4]: [libgcc/libgcov-interface.c](https://gcc.gnu.org/git/?p=gcc.git;a=blob;f=libgcc/libgcov-interface.c;h=b2ee930864183b78c8826255183ca86e15e21ded;hb=HEAD#l196)
[^5]: [https://coveralls.io/github/systemd/systemd](https://coveralls.io/github/systemd/systemd)

*[LSM]: Linux Security Modules
*[SUT]: System under test
