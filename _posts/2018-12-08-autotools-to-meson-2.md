---
layout: post
title:  "Migrating from Autotools to Meson, part 2: more of the how"
date:   2018-12-08 15:34 Europe/London
comments: true
categories: [software development]
---
Reading back over [this post](/software development/2018/11/16/autotools-to-meson.html), I realise
I covered a fair bit of the why, but not much of the how. So I thought I'd
revisit the topic and expand.

## Initial setup

The first thing to realise about Meson is that unlike autotools, no executable
part of the build system is distributed as part of a package. There is no
configure script; but more importantly, no esoteric bootstrapping process to
_create_ the configure script. The second thing to realise is that separation
of source & build directories is the norm, not optional, "advanced" behaviour.

Let's take a look at a tiny C++ hello-world package. From the root, the layout
would look something like this:

```
./meson.build
./src/meson.build
./src/hellow.cpp
```

(Strictly speaking you don't need a _src_ subdirectory, but as I said before,
packaging software for use by others is an untaught art. In any real-world
project, it won't be long before the root folder grows things like a licence,
a readme file, etc., and you want to start adding assets which aren't source
code.)

The top-level _meson.build_ just defines the project, and includes the _src_
subdirectory:

```
project('hellow', 'cpp', license: 'GPL3+', version: '1')
subdir('src')
```

_src/meson.build_ defines the executable we want to build:

```
executable('hellow', 'hellow.cpp', install: true)
```

_src/hellow.cpp_ should be self-explanatory. :)

### Building

So if none of that is actually runnable, how do we produce our executable?
Simple: invoke Meson to create a build directory, then invoke Ninja from that
directory to build the project.

From inside the project root:

```console
phil@hue meson-hellow $ meson setup . build

The Meson build system
Version: 0.47.2
Source dir: /home/phil/Projects/meson-hellow
Build dir: /home/phil/Projects/meson-hellow/build
Build type: native build
Project name: hellow
Project version: 1
Native C++ compiler: c++ (gcc 8.2.1 "c++ (GCC) 8.2.1 20181105 (Red Hat 8.2.1-5)")
Build machine cpu family: x86_64
Build machine cpu: x86_64
Build targets in project: 1
Found ninja-1.8.2 at /usr/bin/ninja

phil@hue meson-hellow $ cd build/

phil@hue build $ ninja

[2/2] Linking target src/hellow.

phil@hue build $ ./src/hellow 

Hello, world!
```

## The build directory: configuration & options

The first line of the above is the actual Meson invocation:
_meson setup \<source directory\> \<build directory\>_. This produces a build
directory, from where you can inspect & alter the build configuration, set any
project-specific compile-time options, build & install the project, and produce
tarballs for distribution.

Modifying the build configuration _after_ the initial set-up may seem
counter-intuitive if, like me, you're used to having to clean everything and
re-run a configure script to change anything. Meson, however, automatically
inserts enough magic into its Ninja rules to regenerate everything required on
the fly. As far as I can tell, in theory, you only need to re-run _meson setup_
if you want to change compiler, or generate something other than Ninja files
for building (XCode & Visual Studio are supported; I haven't tried them).

To list available options and their settings, just run _meson configure_
from the build directory. To change them, use the -D argument; such as
```console
meson configure -Dprefix=/tmp
```

The range of built-in options is quite impressive. Most importantly to me,
with my distribution-friendly-developer hat on, the full set of installation
paths seems to be present and correct, with sane defaults (_bindir_, _libdir_,
_datadir_, etc., all relative to _prefix_ as nature intended).

### Project-specific options

Anyone who's tried to add a custom configure-time option to an autotools
project will know that doing so is a mess. Separate macros for defining the
option, and formatting its help string; a "type system" consisting of booleans
and strings; the necessity of falling back on shell script to actually do
anything meaningful based on the values of options.

With Meson, you can simply drop a *meson_options.txt* in the project root,
define the options you want to add (including their types, descriptions, and
default values), then use *get_option()* from your _meson.build_ files to
do things based on their values.

### Preprocessor definitions

A common thing to want to do in C/C++ projects is define the values of custom
preprocessor symbols at build time, based on the host system, build
environment, or the values of user-specified options. Thankfully this, too,
is simpler and less error-prone than with autotools, though the approach isn't
totally alien.

Within your _meson.build_ files, you can call *configuration_data()* to create
a configuration data object; this then has an array of helpful methods,
including _set()_ and *set_quoted()*. This configuration data object can then
be given as an argument to *configure_file()*, to auto-generate a header file
representing the configuration. Include that header file from your source - job
done.

## A note on imperative, declarative, and domain-specific languages

If you remember, in [part 1](/software development/2018/11/16/autotools-to-meson.html),
I said that sane people shouldn't want their build system to use a full-on
imperative programming language; that they should want it to be declarative.
That may not sit well with all the talk of "creating objects" and "calling
methods" in the previous paragraph. So what's going on?

Well, Meson's syntax seems to sit somewhere between purely declarative, and
imperative. If you don't need any project-specific options, or to modify your
code's behaviour based on the build environment, then you can get away with
declaring the project and its artefacts (executables, shared objects, etc.) in
what looks very much like a simple series of statements, bereft of logic.
Once you start doing those things, you will encounter functions, objects, _if_
statements, etc., but these are all part of a domain-specific language whose
sole focus is defining build systems. It's not quite declarative, but it's
intentionally minimalistic, and restrictive of anything outside its remit.

### ... and the importance of documenting them

It also has a very good manual. Here's
[the official page on configuration](https://mesonbuild.com/Configuration.html),
and the full reference for
[configuration\_data()](https://mesonbuild.com/Reference-manual.html#configuration_data),
the [configuration data object](https://mesonbuild.com/Reference-manual.html#configuration-data-object),
and [configure\_file()](https://mesonbuild.com/Reference-manual.html#configure_file).

## Installation and distribution

Installation and distribution duties are performed through Ninja; hence, from
a build directory.

```console
phil@hue meson-hellow $ meson setup . build -Dprefix=/home/phil/foo

# ... omitted for clarity ...

phil@hue meson-hellow $ cd build/
phil@hue build $ ninja install

[2/3] Installing files.
Installing src/hellow to /home/phil/foo/bin

phil@hue build $ find /home/phil/foo/

/home/phil/foo/
/home/phil/foo/bin
/home/phil/foo/bin/hellow

phil@hue build $ /home/phil/foo/bin/hellow

Hello, world!
```

(I don't know if I mentioned that you can set options directly from the
_meson setup_ command. Well, you can.)

```console
phil@hue build $ ninja dist

# ... shortened for clarity ...

[0/1] Creating source packages
Testing distribution package /home/phil/Projects/meson-hellow/build/meson-dist/hellow-1.tar.xz.
[0/1] Installing files.
Installing src/hellow to /tmp/tmpi2fmjgyp/usr/local/bin
Distribution package /home/phil/Projects/meson-hellow/build/meson-dist/hellow-1.tar.xz tested.

phil@hue build $ find meson-dist/

meson-dist/
meson-dist/hellow-1.tar.xz
meson-dist/hellow-1.tar.xz.sha256sum
```

Nice. Note that not only has it produced a tarball, it's also tested it, by
automatically unpacking it, building & installing in a temporary prefix. It's
also given me a SHA-256 checksum as a little extra.

## Bonus: cross-compilation

I also said previously that cross-compiling for Windows from Linux was trivial,
then proceeded to present no evidence whatsoever. Well, here goes:

```console
phil@hue meson-hellow $ meson setup . w64-build --cross-file /usr/share/mingw/toolchain-mingw64.meson

The Meson build system
Version: 0.47.2
Source dir: /home/phil/Projects/meson-hellow
Build dir: /home/phil/Projects/meson-hellow/w64-build
Build type: cross build
Project name: hellow
Project version: 1
Native C++ compiler: c++ (gcc 8.2.1 "c++ (GCC) 8.2.1 20181105 (Red Hat 8.2.1-5)")
Cross C++ compiler: /usr/bin/x86_64-w64-mingw32-g++ (gcc 7.3.0)
Host machine cpu family: x86_64
Host machine cpu: x86_64
Target machine cpu family: x86_64
Target machine cpu: x86_64
Build machine cpu family: x86_64
Build machine cpu: x86_64
Build targets in project: 1
Found ninja-1.8.2 at /usr/bin/ninja

phil@hue meson-hellow $ cd w64-build/

phil@hue w64-build $ ninja

[2/2] Linking target src/hellow.exe.

phil@hue w64-build $ wine ./src/hellow.exe

Hello, world!
```

Yep - just provide the _\-\-cross-file_ argument to _meson setup_, and away you
go. Note the "Cross C++ compiler" line in the output, the compilation to
hellow.exe, and the usage of [Wine](https://www.winehq.org/) to run it.

In the interests of full disclosure: I'm using Fedora 28 at the moment. The
[MinGW-based](http://www.mingw.org/) compiler itself is from the
_mingw64-gcc-c++_ package, and the Meson cross file from one of its
dependencies, _mingw64-filesystem_. That said, it doesn't look like defining
your own cross files for other, more esoteric systems is rocket science (at
least, not for the kind of people who are likely to attempt it); the file
itself isn't very long.

## The End

Thanks for reading. Have I convinced you yet?
