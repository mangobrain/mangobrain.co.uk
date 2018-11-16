---
layout: post
title:  "Migrating from Autotools to Meson: why, and how, to do it"
date:   2018-11-16 14:28 Europe/London
comments: true
categories: [software development]
---
In recent years, it seems everyone wants to learn to code. This is great, but
unfortunately, only only one piece of the puzzle. If you're coding in a
traditional compiled language like C or C++, one obvious thing to consider is
how that code actually gets compiled. It's not an exciting topic, but one of
many more boring aspects of software development which, if you want to progress
from simply writing code for personal use to releasing real-world software
packages, you will need to think about at some point. Especially if you want
to release that software as open source.

Given the kind of audience who would willingly read something with this piece's
title, I may be preaching to the choir; but the first time you decide
release something as open source, you subject yourself to a world of technical
pain. (Disclaimer: other forms of pain may be involved.) All of a sudden, you
go from a world in which simply being able to produce binaries on your own
development machine is good enough, to a world in which *other people*
(shudder) are expected to be able to go from code to running program on *their*
machines, preferably without you having to get personally involved. This means
having some automated way of sanity-checking the environment, locating required
tools and dependencies, invoking the compiler & linker to actually build the
damn thing, then installing it; all whilst respecting end-user preferences for
things like optimisation level, install path, the turning on or off of any
optional compile-time features, and so on. This is, in general, a hard problem;
and for many years, the de facto solution has been [The Autotools](https://www.gnu.org/software/automake/manual/automake.html#Autotools-Introduction)
\[sic\].

These tools are very much a product of their time. They were born to solve
the kind of problems developers faced during the [UNIX wars](https://en.wikipedia.org/wiki/Unix_wars):
the presence, location, and level of
standards compliance in compilers, linkers, standard libraries, system calls,
and even basic shell utilities varied widely. Nothing could be relied upon;
everything had to be checked. Couple this with a very UNIX-ish philosophy -
make every tool do one thing, and do it well - and you can see how it ended up
the way it did: autoconf to generate "configure" scripts, which do the grunt
work of locating, sanity-checking, and detecting quirks of the tools which will
be required; automake to generate - from a mixture of template input and
"configure" output - Makefiles; and libtool, to handle all the extra fiddly
intricacies involved in consuming or (brace yourself) *producing* shared
objects. They do work, but even setting aside their inherently unwieldy nature,
the years have not been kind. [M4](https://www.gnu.org/software/m4/m4.html) has
the kind of syntax only a mother could love; "configure" scripts mix supplied
macros, third-party macros, and raw shell script code with gay abandon; nobody
seems to be able to quite agree on whether or not libtool is a good idea, even
before you take into consideration that to consume a shared object requires
metadata - such as compiler & linker flags which may need to be applied to both
the library and binaries which use it - which it doesn't install (for which
you need [pkg-config](https://www.freedesktop.org/wiki/Software/pkg-config/),
a kind of honorary member of The Autotools, but which people also can't all
just agree to use without reservations).

Fast forward two-and-a-bit decades, and things have changed somewhat. For a
start, there are probably far fewer platforms you care about running your code
on: in most cases, Linux, OS X, and maybe Windows pretty much cover it. There
are fewer toolchains in widespread use: for C and C++, you probably only really
care about GCC and CLang. If you're really weird, you may be in the unenviable
position of not only wanting Windows builds, but wanting those builds to be
built by (ahem) Visual Studio, rather than the venerable
[MinGW](http://www.mingw.org/). People - sane ones, at least - have realised
that if you want standardised behaviour, then having components of your build
system written in full-fat imperative programming languages - especially ones
as flaky as shell script - is a bad idea, and that maybe it should be
declarative.

Enter [Meson](https://mesonbuild.com/). Now, I could sit here and write a list
of all the ways in which it is better than The Autotools. Or I could show you.

[Infector](https://github.com/mangobrain/Infector) is one of my pet projects.
As per its GitHub description, it's a variant of [Ataxx](https://en.wikipedia.org/wiki/Ataxx),
written in C++. It also happens to use the GTK+ GUI toolkit, have a Czech
translation, and is packaged on [Flathub](https://flathub.org/apps/details/uk.co.mangobrain.Infector)
(Flatpak being something I'd like to cover in a future article). All of which
serves to illustrate that it is not some first-year Hello World program, but a
piece of real software with real dependencies.

Instead of this:

```m4
AC_PREREQ([2.69])

dnl # Initialise autoconf and automake
dnl # Also specify automake version requirements (again, the version used
dnl # development will suffice for now).
AC_INIT([infector],[0.4],[mangobrain@googlemail.com])
AC_CONFIG_SRCDIR([src/infector.cxx])
AC_CONFIG_MACRO_DIR([acinclude])
AC_CONFIG_AUX_DIR([build-aux])

AM_INIT_AUTOMAKE([1.15 silent-rules])
AM_SILENT_RULES([yes])

dnl # Infector is written in C++, of course.  :)
AC_PROG_CXX
AC_PROG_CXXCPP
AC_LANG([C++])

dnl # Make config.h from config.h.in.  (`autoheader' generates the latter.)
AC_CONFIG_HEADERS([config.h])

dnl # Output the command-line options given to this script
AC_DEFINE_UNQUOTED([CONFIGURE_OPTS],["$ac_configure_args"],[Options given to ./configure])

dnl # Use intltool for i18n.
IT_PROG_INTLTOOL([0.51])

dnl # Infector uses gtkmm.
PKG_PROG_PKG_CONFIG
PKG_CHECK_MODULES([GTKMM], [gtkmm-3.0])

dnl # Are we building on MinGW?
AC_MSG_CHECKING(host os)
AC_CANONICAL_HOST
AC_MSG_RESULT($host_os)
win=false
case "$host_os" in
	mingw*)
		win=true
		LIBS="-lwsock32 -lws2_32 ${LIBS}"
		CXXFLAGS="-mthreads -Wl,-subsystem,windows ${CXXFLAGS}"
		AC_DEFINE([MINGW],[1],[Define if building for Windows (MinGW)])
		AC_DEFINE([_WIN32_WINNT],[0x0502],[Define if building for Windows])
		AC_DEFINE([WINVER],[0x0502],[Define if building for Windows])
		;;
	*)
		;;
esac
AM_CONDITIONAL(MINGW, test "x$win" = "xtrue")

dnl # Sed is used to parse infector.desktop.in
AC_PROG_SED

dnl # Variables necessary for i18n.
AC_SUBST([GETTEXT_PACKAGE], [infector])
AC_DEFINE_UNQUOTED([GETTEXT_PACKAGE], ["$GETTEXT_PACKAGE"], [gettext package name])

dnl # Available translations
ALL_LINGUAS="cs"
AM_GLIB_GNU_GETTEXT

dnl # All done.  Spit out the result.
AC_CONFIG_FILES([
	Makefile
	src/Makefile
	data/Makefile
	po/Makefile.in
])
dnl	doc/Makefile
AC_OUTPUT
```

We have this:
```
project('infector', 'cpp', license: 'GPL3+', version: '0.7')
i18n = import('i18n')
subdir('src')
subdir('data')
subdir('po')
```

Isn't that much better?

Obviously there's a bit more to it than that. Previously, I had to jump through
all kinds of internationalisation-related hoops: intltool's lack (at the time)
of official support for translating .desktop files, for example. Or my desire
to dynamically create icon files of different sizes from the SVG original, on
the fly, at installation time, simply because it was possible. (Given
autoconf's nature as a glorified M4-to-shell-script translator, coupled with
make's willingness to execute direct shell commands when processing targets,
The Autotools let you commit no end of sins.)

I won't paste the full contents of every other file here, as this article is
already too long, but I will let the following comparisons make the rest of my
argument for me.

- The data directory - icons & desktop file installation
  - I did cheat a little bit here and just commit the pre-generated icon files
    directly. But what I had before *was* undeniably hideous.
  - [Before](https://github.com/mangobrain/Infector/blob/8949b6431c789928496e16341f2cd097b0d5db22/data/Makefile.am)
  - [After](https://github.com/mangobrain/Infector/blob/925390c40ac5c6eba243c008b3c324523450af04/data/meson.build)
- The src directory
  - The "after" here may, at first sight, look worse. But don't be fooled.
    A lot of work which was previously being done directly in the top-level
    configure script - looking for dependencies, honouring the native language
    support option, adding additional libraries for Windows builds - is done
    here, in the place where we actually write the configuration header and
    build the source, where it should be. Without a single line of shell script
    in sight.
  - [Before](https://github.com/mangobrain/Infector/blob/8949b6431c789928496e16341f2cd097b0d5db22/src/Makefile.am)
  - [After](https://github.com/mangobrain/Infector/blob/925390c40ac5c6eba243c008b3c324523450af04/src/meson.build)

Everything is cleaner, more succint, and far less fragile. All the previously
unwritten rules to which one had to adhere if one wanted one's packages to
correctly build and install in non-standard locations, based on the "standard"
set of command line options supported by configure scripts, are gone; replaced
instead by genuinely standard command line options (to the meson binary) and
a standard way of getting their values.

To use it, you invoke meson, which produces - in a separate directory tree -
a [ninja](https://ninja-build.org/) build file. This is both faster and safer
than Make, and forces you to support separate source & build directories, which
is a Good Thing. On a distribution like Fedora, which provides pre-built
packages of MinGW compilers targetting Windows, cross-compiling Windows
binaries from a Linux host is trivial.

Don't take my word for it. Have a browse around the Meson documentation, use
the Infector repository as a working example if you wish, and just Do It.

Please?
