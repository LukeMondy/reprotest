Command-line examples
=====================

The easiest way to run reprotest is via our presets:

::

    # Build the current directory in a null server (/tmp)
    $ reprotest auto .
    $ reprotest auto . -- null -d # for more verbose output

    # Build the given Debian source package in an schroot
    # See https://wiki.debian.org/sbuild for instructions on setting that up.
    $ reprotest auto reprotest_0.3.3.dsc -- schroot unstable-amd64-sbuild

Currently, we only support this for Debian packages, but are keen on
adding more. If we don't have knowledge on how to build your file or
directory, you can send a patch to us on adding this intelligence - see
the reprotest.presets python module, and adapt the existing logic.

In the meantime, you can use the more advanced CLI to build arbitrary
things. This takes two mandatory arguments, the build command to run and
the build artifact file/pattern to test after running the build. For
example:

::

    $ reprotest 'python3 setup.py bdist' 'dist/*.tar.gz'

When using this from a shell:

If the build command has spaces, you will need to quote them, e.g.
``reprotest "debuild -b -uc -us" [..]``.

If you want to use several build artifact patterns, or if you want to
use shell wildcards as a pattern, you will also need to quote them, e.g.
``reprotest [..] "*.tar.gz *.tar.xz"``.

If your build artifacts have spaces in their names, you will need to
quote these twice, e.g. ``'"a file with spaces.gz"'`` for a single
artifact or ``'"dir 1"/* "dir 2"/*'`` for multiple patterns.

To get more help for the CLI, including documentation on optional
arguments and what they do, run:

::

    $ reprotest --help


Running in a virtual server
===========================

You can also run the build inside what is called a "virtual server".
This could be a container, a chroot, etc. You run them like this:

::

    $ reprotest 'python3 setup.py bdist_wheel' 'dist/*.whl' qemu    /path/to/qemu.img
    $ reprotest 'debuild -b -uc -us'           '../*.deb'   schroot unstable-amd64

There are different server types available. See ``--help`` for a list of
them, which appears near the top, in the "virtual\_server\_args" part of
the "positional arguments" section.

For each virtual server (e.g. "schroot"), you see which extra arguments
it supports:

::

    $ reprotest --help schroot

When running builds inside a virtual server, you will probably have to
give extra commands, in order to set up your build dependencies inside
the virtual server. For example, to take you through what the "Debian
directory" preset would look like, if we ran it via the advanced CLI:

::

    # "Debian directory" preset
    $ reprotest auto . -- schroot unstable-amd64-sbuild
    # In the advanced CLI, this is equivalent to roughly:
    $ reprotest \
        --testbed-init 'apt-get -y --no-install-recommends install \
                        util-linux disorderfs 2>/dev/null; \
                        test -c /dev/fuse || mknod -m 666 /dev/fuse c 10 229' \
        'PATH=/sbin:/usr/sbin:$PATH apt-get -y --no-install-recommends build-dep ./; \
         dpkg-buildpackage -uc -us -b' \
        '../*.deb' \
        -- \
        schroot unstable-amd64-sbuild

The ``--testbed-init`` argument is needed to set up basic tools, which
reprotest needs in order to make the variations in the first place. This
should be the same regardless of what package is being built, but might
differ depending on what virtual\_server is being used.

Next, we have the build\_command. For our Debian directory, we install
build-dependencies using apt-get, then we run the actual build command
itself using dpkg-buildpackage(1).

Then, we have the artifact pattern. For reproducibility, we're only
interested in the binary packages.

Finally, we specify that this is to take place in the "schroot"
virtual\_server with arguments "unstable-amd64-sbuild".

Of course, all of this is a burden to remember, if you must run the same
thing many times. So that is why adding new presets for new files would
be good.

Here is a more complex example. It tells reprotest to store the build products
into ``./artifacts`` to analyse later; and also tweaks the "Debian dsc" preset
so that it uses our `experimental toolchain
<https://wiki.debian.org/ReproducibleBuilds/ExperimentalToolchain>`__.

::

    $ reprotest --store-dir=artifacts \
        --auto-preset-expr '_.prepend.testbed_init("apt-get install -y wget 2>/dev/null; \
            echo deb http://reproducible.alioth.debian.org/debian/ ./ >> /etc/apt/sources.list; \
            wget -q -O- https://reproducible.alioth.debian.org/reproducible.asc | apt-key add -; \
            apt-get update; apt-get upgrade -y 2>/dev/null; ")' \
        auto ./bash_4.4-4.0~reproducible1.dsc \
        -- \
        schroot unstable-amd64-sbuild

(Yes, this could be a lot nicer to achieve; we're working on it.)


Config File
===========

You can also give options to reprotest via a config file. This is a
time-saving measure similar to ``auto`` presets; the difference is that
these are more suited for local builds that are suited to your personal
purposes. (You may use both presets and config files in the same build.)

The config file has one section, basics, and the same options as the
CLI, except there's no ``dont_vary`` option, and there are
``build_command`` and ``artifact`` fields. If ``build_command`` and/or
``artifact`` are set in the config file, reprotest can be run without
passing those as command-line arguments. Command-line arguments always
override config file options.

Reprotest by default loads ``./.reprotestrc``, or you can tell it to
load another file with the ``--config-file`` command line option.

A sample config file is below.

::

    [basics]
    build_command = setup.py sdist
    artifact = dist/reprotest-0.2.tar.gz
    source_root = reprotest/
    variations =
      environment
      build_path
      home
      kernel
      locales
      exec_path
      time
      timezone
      umask


Known bugs
==========

The "time" variation uses **faketime** which *sometimes* causes weird and
hard-to-diagnose problems. In the past, this has included:

- builds taking an infinite amount of time; though this should be fixed in
  recent versions of reprotest.

- builds with implausibly huge differences caused by ./configure scripts
  producing different results with and without faketime. This still affects
  bash and probably certain other packages using autotools.

If you see a difference that you really think should not be there, try passing
``--dont-vary time`` to reprotest, and/or check our results on
https://tests.reproducible-builds.org/ which use a different (more reliable)
mechanism to vary the system time.
