---
layout: post
title:  "Building and installing GNU gettext translation catalogs in Python"
date:   2018-02-07 23:00:00 +0100
---

The [GNU gettext library and tools](https://www.gnu.org/software/gettext/) are the de facto standard for software localization in the open source world. While being mostly transparent to end users, they are probably known to every developer who ever worked on an internationalized application. To refresh your memory, it's the library that provides the `gettext` function that should be used to wrap every localizable message (often shortened to just `_`) and the utilities to extract those messages into `.po` files, build them in turn into `.mo` files and load them back into the application.

In Python, gettext is so clearly *the* way to localize applications that it's packaged as [a module in the standard library](https://docs.python.org/3/library/gettext.html). And the main other library that deals with localization, [Babel](http://babel.pocoo.org/), also "just" builds on gettext. Given this central role, I've always found it surprising that there didn't seem to be a standard Python way to compile message catalogs (from `.po` to `.mo` files) and install them. Distutils/setuptools, the Python packaging and distribution tools, don't provide any support: no argument in the `setup` function, no section in the `setup.cfg` file, no command, nothing.

I recently tried to tackle this problem again in [one of my projects](http://cms-dev.github.io/) and reached the conclusion that extending distutils/setuptools with custom commands would be the simplest solution. I'll present my code here hoping it can help someone else.

I'll start by assuming that extracting messages and translating them is taken care of already (see [the tools that Babel offers](http://babel.pocoo.org/en/latest/setup.html) if it isn't). I thus suppose that there is a `po` directory in the repository (at the top level, i.e., at the same level as `setup.py`) that contains the localized message catalogs named `<locale>.po` (where locale is something like `it`, `ja`, `es_MX`, ...)[^survey]. My goal is to have the `./setup.py install` command, launched in a clean environment, achieve the result of installing the compiled catalogs in `/usr/share/locale/<locale>/LC_MESSAGES/<domain>.mo` (or elsewhere, but still in an appropriate location, if you're using a local prefix, a custom install location, a virtualenv, ...). The domain in the filename is a "codename" for the application.

[^survey]: according a brief survey of a few arbitrary open source desktop applications (namely the first ones that came to my mind) this seems to be a common format

To get to the point, the trick is adding the two following commands to `setup.py`:
```py
from distutils.cmd import Command

from babel.messages.mofile import write_mo
from babel.messages.pofile import read_po


class BuildLocalization(Command):
    description = "build the po files into mo files"

    user_options = [
        ("domain", None, "domain of the translation"),
        ("build-dir=", "d", "directory to compile mofiles to"),
        ("force", "f", "forcibly build everything (ignore file timestamps)")]

    boolean_options = ["force"]

    def initialize_options(self):
        self.domain = None
        self.build_dir = None
        self.force = None

    def finalize_options(self):
        self.set_undefined_options("build",
                                   ("build_base", "build_dir"),
                                   ("force", "force"))
        if self.domain is None:
            self.domain = self.distribution.get_name()

    def run(self):
        # The working directory is the directory of setup.py.
        base_path = "po"
        for filename in os.listdir(base_path):
            locale, ext = os.path.splitext(filename)
            if ext == ".po":
                pofile = os.path.join(base_path, "%s.po" % locale)
                mofile = os.path.join(self.build_dir, "locale", locale,
                                      "LC_MESSAGES", "%s.mo" % self.domain)
                self.make_file(pofile, mofile, self._build, (pofile, mofile))

    def _build(self, pofile_path, mofile_path):
        with io.open(pofile_path, "rb") as pofile:
            catalog = read_po(pofile)
        self.mkpath(os.path.dirname(mofile_path))
        with io.open(mofile_path, "wb") as mofile:
            write_mo(mofile, catalog)


class InstallLocalization(Command):
    description = "install the mo files"

    user_options = [
        ("install-dir=", "d", "directory to install pofiles to"),
        ("build-dir=", "b", "build directory to install mofiles from"),
        ("force", "f", "force installation (overwrite existing files)"),
        ("skip-build", None, "skip the build steps")]

    boolean_options = ["force", "skip-build"]

    def initialize_options(self):
        self.install_dir = None
        self.build_dir = None
        self.force = None
        self.skip_build = None

    def finalize_options(self):
        self.set_undefined_options("build_l10n",
                                   ("build_dir", "build_dir"))
        self.set_undefined_options("install",
                                   ("install_base", "install_dir"),
                                   ("force", "force"),
                                   ("skip_build", "skip_build"))

    def run(self):
        if not self.skip_build:
            self.run_command('build_l10n')
        # The working directory is the directory of setup.py.
        self.copy_tree(os.path.join(self.build_dir, "locale"),
                       os.path.join(self.install_dir, "share", "locale"))
```

The two commands are pretty straightforward. The first one looks for `.po` files and compiles them into `.mo` files which are stored in the `build` directory (by default) already in the format they will later be installed on disk. The second command then simply copies these files as they are to the final install location. Some bookkeping is needed to play nicely with the rest of the distutils framework.

I'm using Babel as a dependency as I'm not crazy enough to do the parsing of the `.po` files and the serialization of the `.mo` files by hand.

Finally, in order for these command to be automatically called when building and installing, the default `build` and `install` commands need to be patched, and all commands need to be provided to the `setup` function. This is done as follows:
```py
from distutils.command.build import build
from distutils.command.install import install


class build_with_l10n(build):
    sub_commands = build.sub_commands + [("build_l10n", None)]


class install_with_l10n(install):
    sub_commands = install.sub_commands + [("install_l10n", None)]


setup(
    ...,
    cmdclass={
        "build_l10n": BuildLocalization,
        "build": build_with_l10n,
        "install_l10n": InstallLocalization,
        "install": install_with_l10n
    },
    ...)
```

Done.

The fine print in all this is that this only works in distutils. Setuptools defines a different `install` command, that does weird stuff to support egg files, and thus doesn't pick up these new commands. I'm still trying to figure out how to fix that, in the meantime one can tell setuptools to revert to the standard distutils behavior using the `--old-and-unmanageable` flag of the `install` command.
