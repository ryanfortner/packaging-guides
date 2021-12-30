# Introduction

This document shows how to create a Debian package for an existing Python library.

Before doing this, it is a good idea to check if there isn't an existing Debian package
for the library to be packaged.

For Debian: https://www.debian.org/distrib/packages

For Ubuntu: https://packages.ubuntu.com/

Most of the Python libraries are already packaged, usually in this format:
* python-library-name-x.y.z-a (for Python2)
* python3-library-name-x.y.z-a (for Python2)

In here, x.y.z is the **upstream** version number (i.e. the library version number) and
"a" is the package version number. There can be more than one package for a given **upstream**
version, for example, given an **upstrea** version of 1.0 a package version 1.0-1 can be created,
and released. Later, a bug is discovered in the package itself (but not in the **upstream**),
so a new fixed package is released with version 1.0-2.

This document does not explain the process of pusblishing a new package to a Debian distribution,
it simply explains how to create it.

For example, it shows only how to create the Python3 version of the package, most real world
packages would have the Python2 version as well.

It also doesn't necessarily follow all the best or accepted practices, this is simply intended to
be a quick start guide.

This document shows how to package the existing ``flask-jwt-extended`` library into a Debian
pacakge.

# Requirements

* A Debian based distribution (Debian, Ubuntu, etc.).

# Fetch the upstream tarball

For a Python library published on PyPi, it is possible to download the tarball from the website
directly: https://pypi.org/project/Flask-JWT-Extended/3.8.1/#files

An alternative way is to get it using ``pip``:

```bash
$ pip3 download flask-jwt-extended --dest . --no-binary :all: --no-deps
```

A quick explanation of the options:
* ``--no-binary :all:`` will download the source package (generally as a tarball), instead of the
  binary version
* ``--no-deps`` prevents the dependencies to be pulled
* ``--dest .`` downloads the tarball in the current folder

For more information, see the [pip download documentation][1].

The current folder should now contain a file named ``Flask-JWT-Extended-3.18.0.tar.gz``.

Debian requires the name of the upstream to be in lowercase, so the file needs to be renamed:

```bash
$ mv Flask-JWT-Extended-3.8.1.tar.gz flask-jwt-extended-3.8.1.tar.gz
```

For more information, see the section ``2.6. Package name and version`` of the
[Debian New Maintainers' Guide][2].

# How to choose which upstream version to package?

There is no easy answer to this question.

In this case, as per the ``setup.py``, ``flask-jwt-extended`` depends on:
* Flask
* PyJWT

On a Ubuntu platform, there are pre-existing packages for these dependencies:
* python3-flask
* python3-jwt

It is important to check the version of these packages:

```bash
$ apt install -y python3-flask python3-jwt
$ python3
>>> import flask
>>> flask.__file__ # To be sure that this is imported from the system package, not a user-local version
'/usr/lib/python3/dist-packages/flask/__init__.py'
>>> flask.__version__
'0.12.2'
>>> import jwt
>>> jwt.__file__
'/usr/lib/python3/dist-packages/jwt/__init__.py'
>>> jwt.__version__
'1.5.3'
>>> 
```

By checking each released version of ``flask-jwt-extended`` it appears that the latest one that can
satisfy these dependencies is version 3.8.1.

# Create the GIT repository

It is almost always recommended to keep a separate git repository for the packaging.

An application or library could be packaged for many operating systems, and it's not a good idea to
keep all the OS specific packaging code along with the **upstream** source code.

If this was the case, any bug in the packaging code would need a new **upstream** version, even if
the **upstream** doesn't change at all.

There are a few exceptions though, and these are called [native packages][3].

To create a new git repository for the package, there is a very helpful tool called
[gbp (git buildpackage)][4].

To do this:

```bash
$ mkdir flask-jwt-extended-3.8.1
$ cd flask-jwt-extended-3.8.1
$ git init
```
The ``gbp import-orig`` and ``debmake`` commands can be used to initialize everything:

```bash
$ gbp import-orig -u 3.8.1 --pristine-tar --sign-tags --no-interactive ../flask-jwt-extended-3.8.1.tar.gz
$ debmake -b":python3"
```

The ``gbp import-orig`` command creates two additional branches, and one tag:
* ``upstream`` branch, that contains the upstream source code
* ``pristine-tar`` branch (because of the ``--pristine-tar`` option), that contains information to
  rebuild a pristine version of the original upstream tarball.
* ``upstream/3.8.1`` tag, that contains the upstream source for this version

The ``master`` branch also contains the upstream contents.

Explanation of some of the options:
* ``--pristine-tar`` is used to create the ``pristine-tar`` branch, that can be used to rebuild the
  upstream tarballs. The branch does not contain the full tarballs, only delta files. Recreating an
  upstream tarball can be done like this:
  ``$ pristine-tar checkout flask-jwt-extended_3.8.1.orig.tar.gz``. To list all the available tars:
  ``$ pristine-tar list``. See the [manual for pristine-tar][5].
* ``--sign-tags`` will GPG-sign the created tags. See signing [commits and tags with Git][6].
* ``-u 3.8.1`` is the upstream version

Once done, the packaging specific files can be generated with ``debmake``:

```bash
$ debmake -b":python3"
E: unknown python version.  check setup.py.
```

This fails, as ``debmake`` is not able to determine the python version that can be used.

The trick for that is to add it temporarily to the ``setup.py``:

```bash
$ # Make sure python3 is specified in setup.py, for debmake.
$ mv setup.py setup.py.old
$ echo "#!/usr/bin/env python3" >> setup.py
$ cat setup.py.old >> setup.p
$ # Create debian folder with defaults.
$ debmake -b":python3"
$ # Reverting setup.py change.
$ mv setup.py.old setup.py
```

It's a good idea to commit the work at this point.

# Editing the debian files

This document won't explain each of the files and options, as there could probably be an entire
book for this.

The edited files are attached in this gist.

The best source of information is the [Debian New Maintainers' Guide][7].

A few important things to note:

* In the ``rules`` file, the ``dh_auto_test`` command is disabled, because this particular upstream
  tarball does not include the test scripts - they are generally provided, so this is normally not
  needed.

# Create the Debian package

This can be done with a single command:

```bash
$ gbp buildpackage --git-pristine-tar
```

The generated files are created in the parent directory.

The command fails if there are uncommitted changes, so during developement mode, the
``--git-ignore-new`` option can be used to circumvent this.

# Test the Debian package

The package should be locally installed and tested, before being published.

A simple test could be:

```bash
$ apt install ./python3-flask-jwt-extended_3.8.1-1_all.deb
$ python3
>>> import flask_jwt_extended
>>> flask_jwt_extended.__version__
'3.8.1'
>>> flask_jwt_extended.__file__
'/usr/lib/python3/dist-packages/flask_jwt_extended/__init__.py'
```

We may want to list the files inside the package:

```bash
$ dpkg -c python3-flask-jwt-extended_3.8.1-1_all.deb 
drwxr-xr-x root/root         0 2019-03-21 00:00 ./
drwxr-xr-x root/root         0 2019-03-21 00:00 ./usr/
drwxr-xr-x root/root         0 2019-03-21 00:00 ./usr/lib/
drwxr-xr-x root/root         0 2019-03-21 00:00 ./usr/lib/python3/
drwxr-xr-x root/root         0 2019-03-21 00:00 ./usr/lib/python3/dist-packages/
drwxr-xr-x root/root         0 2019-03-21 00:00 ./usr/lib/python3/dist-packages/Flask_JWT_Extended-3.8.1.egg-info/
-rw-r--r-- root/root      1041 2019-03-21 00:00 ./usr/lib/python3/dist-packages/Flask_JWT_Extended-3.8.1.egg-info/PKG-INFO
-rw-r--r-- root/root         1 2019-03-21 00:00 ./usr/lib/python3/dist-packages/Flask_JWT_Extended-3.8.1.egg-info/dependency_links.txt
-rw-r--r-- root/root         1 2019-03-21 00:00 ./usr/lib/python3/dist-packages/Flask_JWT_Extended-3.8.1.egg-info/not-zip-safe
-rw-r--r-- root/root        34 2019-03-21 00:00 ./usr/lib/python3/dist-packages/Flask_JWT_Extended-3.8.1.egg-info/requires.txt
-rw-r--r-- root/root        19 2019-03-21 00:00 ./usr/lib/python3/dist-packages/Flask_JWT_Extended-3.8.1.egg-info/top_level.txt
drwxr-xr-x root/root         0 2019-03-21 00:00 ./usr/lib/python3/dist-packages/flask_jwt_extended/
-rw-r--r-- root/root       430 2019-03-21 00:00 ./usr/lib/python3/dist-packages/flask_jwt_extended/__init__.py
-rw-r--r-- root/root      8287 2019-03-21 00:00 ./usr/lib/python3/dist-packages/flask_jwt_extended/config.py
-rw-r--r-- root/root      3292 2019-03-21 00:00 ./usr/lib/python3/dist-packages/flask_jwt_extended/default_callbacks.py
-rw-r--r-- root/root      1552 2019-03-21 00:00 ./usr/lib/python3/dist-packages/flask_jwt_extended/exceptions.py
-rw-r--r-- root/root     17316 2019-03-21 00:00 ./usr/lib/python3/dist-packages/flask_jwt_extended/jwt_manager.py
-rw-r--r-- root/root      5389 2019-03-21 00:00 ./usr/lib/python3/dist-packages/flask_jwt_extended/tokens.py
-rw-r--r-- root/root     13608 2019-03-21 00:00 ./usr/lib/python3/dist-packages/flask_jwt_extended/utils.py
-rw-r--r-- root/root      7080 2019-03-21 00:00 ./usr/lib/python3/dist-packages/flask_jwt_extended/view_decorators.py
drwxr-xr-x root/root         0 2019-03-21 00:00 ./usr/share/
drwxr-xr-x root/root         0 2019-03-21 00:00 ./usr/share/doc/
drwxr-xr-x root/root         0 2019-03-21 00:00 ./usr/share/doc/python3-flask-jwt-extended/
-rw-r--r-- root/root       185 2019-03-21 00:00 ./usr/share/doc/python3-flask-jwt-extended/README.Debian
-rw-r--r-- root/root       152 2019-03-21 00:00 ./usr/share/doc/python3-flask-jwt-extended/changelog.Debian.gz
-rw-r--r-- root/root      1458 2019-03-21 00:00 ./usr/share/doc/python3-flask-jwt-extended/copyright
```

Once satisfied, the package can be uninstalled normally with `apt purge python3-flask-jwt-extended`.

This is just a simple test, much more thorough testing is needed before publishing the package
in a production environment or publicly.

# Conclusion

This document shows how to create a simple Debian package for a Python library.

There is much more to this, such as patching the upstream files when they are not suitable
for packaging, how to create a Python2 version of the package, how to publish the package
publicly or to a private PPA or repository, etc.

[1]: https://pip.pypa.io/en/stable/reference/pip_download/
[2]: https://www.debian.org/doc/manuals/maint-guide/first.en.html
[3]: https://wiki.debian.org/DebianMentorsFaq#What_is_the_difference_between_a_native_Debian_package_and_a_non-native_package.3F
[4]: https://honk.sigxcpu.org/projects/git-buildpackage/manual-html/gbp.html
[5]: https://manpages.debian.org/stretch/pristine-tar/pristine-tar.1.en.html
[6]: https://git-scm.com/book/en/v2/Git-Tools-Signing-Your-Work
[7]: https://www.debian.org/doc/manuals/maint-guide/
