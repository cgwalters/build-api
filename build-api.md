Build API
=========

Version: 0.3

Note: the upstream for this document is now:
https://github.com/cgwalters/build-api

Motivations
-----------

There are at present in the FOSS world a number of different
"meta-build" systems that are designed to build (compose) multiple
components - prominent examples are "rpm" and "deb" for self-hosting
builds, "buildroot" and "Yocto" for cross builds, "jhbuild" and
"kdesrc-build" for developer fastbuilds, etc.

It makes sense to drive - as much as possible - build knowledge and
metadata upstream, rather than have it be reproduced in these multiple
"downstream" consumers.

What's wrong with ./configure && make && make install DESTDIR=/foo?
Shouldn't we just encourage people to use the autotools, and keep spec
files, bitbake recipies etc. as they are?

While the autotools are good, we don't need *every* feature they have,
and we don't want to force everyone to use them.

Thus, we want to create a well-defined interface that anyone using an
alternative build system should implement.  A primary goal of the
Build API here is that if one chooses to use waf, scons, or just use a
raw Makefile, you have a clear path to plug in to any of the multiple
meta-build systems listed above.

Also, a main concern is that many project consumers want to split
between a "foo" package that only has the runtime bits, and a
"foo-devel" which has C library headers, etc.  This logic can live
much more sanely upstream than reproduced over and over in every
downstream consumer.

Build API
---------

In a freshly unpacked source tree, there MUST be an
architecture-indepenent script named either 'autogen', 'autogen.sh',
or 'configure'.  If 'configure' exists, it is preferred over the other
two.  If configure does not exist, the 'autogen' script is executed
with no arguments in the source directory, and it MUST generate an
executable script named 'configure'.

After the 'configure' script is run, 'make' will be invoked on the
generated 'Makefile', and then an install target will be invoked.  The
following details each script:

autogen (or "autogen.sh")
-------------------------

If present, this file is an executable script which bootstraps the
build system.  It will be executed before configure, if the configure
script is not present.  For example, in autotools based systems, this
will run a tool such as "autoreconf".

This tool SHOULD NOT automatically run ./configure.  If for historical
reasons it does, then it MUST honor an environment variable
NOCONFIGURE, which if set, suppresses the ./configure run.

configure
---------

This MUST be an executable file.  It MUST take the following arguments:

* `--prefix=PATH`
* `--libdir=PATH`
* `CFLAGS=FLAGS`
* `CXXFLAGS=FLAGS`

Example: 

	./configure --prefix=/opt/build/ --libdir=/opt/build/lib64 CFLAGS='-ggdb -O2'

It SHOULD take the following arguments:

* `--build=HOST`
* `--host=HOST`

The semantics of these arguments are as defined by autoconf.  See
"info autoconf", under the section "Hosts and Cross Compilation".

The configure script MUST either ignore unknown options that start
with `--enable-` and `--disable-`, or accept an argument `--help`
which prints valid options.  The rationale for this is the same as
autotools, it allows one to have global effects without special casing
each configure script over time.

When the configure script is run, it MUST generate a file named
`Makefile` in the same working directory.

The following Build API variables may be used to declare how this
project supports builds.  These variables will be found in the
configure script by substring match on a line-by-line basis.

* `buildapi-variable-no-builddir`: Declares that this project only works when srcdir == builddir.

* `buildapi-variable-require-builddir`: Declares that this project
   ONLY works when srcdir != builddir.  Avoid this if you can.

* `buildapi-variable-supports-runtime-devel`: Declares support for the
   buildapi-install-runtime and buildapi-install-devel Makefile
   targets.

Makefile
--------

This MUST be a Makefile, as consumable by an implementation of "make"
such as GNU Make.  It MUST have the following targets:

* `all`: Build all of the software.  This variable SHOULD accept a make variable
   called `BUILDAPI_JOBS`.  The `BUILDAPI_JOBS` variable specifies roughly how
   many concurrent processes or threads the build system should use.  It
   is however acceptable for the build system to invoke 
   `sysconf (_SC_NPROCESSORS_ONLN)`
   or a wrapper thereof as a basis as well.

* `install`:
   This target MUST take an argument variable `DESTDIR` which
   specifies an additional file path prefix before the install path. Example:

	make install DESTDIR=/opt/build-daemon/work/pkgname/root

It SHOULD support these individual build targets:
  
* `buildapi-install-runtime`: Install binaries, libraries, and data
   files intended to be present on all systems.

* `buildapi-install-devel`: Install header files, library symbolic
   links, and data files necessary for other software to consume your
   software.  This target MUST take an argument variable "DESTDIR"
   which specifies an additional file path prefix before the install
   path.

* `.NOTPARALLEL`: This pseudo-target is taken from GNU Make.  Use this
   to forcibly disable parallel builds if your `Makefile` is not
   compatible with them.  Obviously, you should try to avoid using
   this.  One alternative is to progressively, isolate the parts of
   your build which aren't parallelizable into a separate
   `Makefile.notparallel` (and use `.NOTPARALLEL`, then and
   recursively invoke `make -f Makefile.notparallel`).
