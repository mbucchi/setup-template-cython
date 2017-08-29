## setup-template-cython

Setuptools-based `setup.py` template for Cython projects

### Introduction

Setuptools[[1]][setuptools] has become the tool of choice[[2]][packaging] for packaging Python projects, yet not much documentation is available on how to use setuptools in Cython projects. As of this writing (Cython 0.25), Cython's [official packaging instructions](http://cython.readthedocs.io/en/latest/src/reference/compilation.html#distributing-cython-modules) are based on distutils.

For packages to be distributed (especially through [PyPI](https://pypi.python.org/pypi)), setuptools is preferable, since the documentation on distributing packages[[3]][distributing] assumes that is what the developer uses. Also, [setuptools adds dependency resolution](http://stackoverflow.com/questions/25337706/setuptools-vs-distutils-why-is-distutils-still-a-thing) (over distutils), which is an essential feature of `pip`.

This very minimal project documents the author's best guess at current best practices for the packaging and distribution of Cython projects, by piecing together information from various sources. Possible corrections, if any, are welcome.

This is intended as a template [`setup.py`](setup.py) for new Cython projects. For completeness, a minimal Cython-based example library is included, containing examples of things such as absolute cimports, subpackages, [NumPyDoc](https://github.com/numpy/numpy/blob/master/doc/HOWTO_DOCUMENT.rst.txt) style docstrings, and using [memoryviews](http://cython.readthedocs.io/en/latest/src/userguide/memoryviews.html) for passing arrays (for the last two, see [compute.pyx](mylibrary/compute.pyx)). The example in the [test/](test/) subdirectory demonstrates usage of the example library after it is installed.

A pruned-down version of setup.py for pure Python projects, called [`setup-purepython.py`](setup-purepython.py), is also provided for comparison.

This project is placed under The Unlicense so as to allow its use anywhere. Code snippets based on StackOverflow answers are credited by providing the original links.

This is similar to [simple-cython-example](https://github.com/thearn/simple-cython-example), but different. Here the focus is on numerical scientific projects, where a custom Cython extension (containing all-new code) can bring a large speedup.

The aim is to help open-sourcing such extensions in a manner that lets others effortlessly compile them (also in practice), thus advancing the openness and repeatability of science.


### Features

Our [`setup.py`](setup.py) features the following:

 - The most important fields of `setup()`
   - If this is all you need, [simple-cython-example](https://github.com/thearn/simple-cython-example) is much cleaner.
   - If this is all you need, and you somehow ended up here even though your project is pure Python, [PyPA's sampleproject](https://github.com/pypa/sampleproject/blob/master/setup.py) (as mentioned in [[4]][setup-py]) has more detail on this.
 - How to get `absolute_import` working in a Cython project
   - For compatibility with both Python 3 and Python 2.7 (with `from __future__ import absolute_import`)
   - For scientific developers used to Python 2.7, this is perhaps the only tricky part in getting custom Cython code to play nicely with Python 3. (As noted elsewhere[[a]](http://www.python3statement.org/)[[b]](http://python-3-for-scientists.readthedocs.io/en/latest/), it is time to move to Python 3.)
 - How to automatically grab `__version__` from `mylibrary/__init__.py` (using AST; no import or regexes), so that you can declare your package version [OnceAndOnlyOnce](http://wiki.c2.com/?OnceAndOnlyOnce) (based on [[5]][getversion])
 - Hopefully appropriate compiler and linker flags for math and non-math modules on `x86_64`, in production and debug configurations.
   - Also compiler and linker flags for OpenMP, to support `cython.parallel.prange`.
 - How to make `setup.py` pick up non-package data files, such as your documentation and usage examples (based on [[6]][datafolder])
 - How to make `setup.py` pick up data files inside your Python packages
 - How to enforce that `setup.py` is running under a given minimum Python version ([considered harmful](http://stackoverflow.com/a/1093331), but if duck-checking for individual features is not an option for a reason or another) (based on [[7]][enforcing])
 - Disabling `zip_safe`. Having `zip_safe` enabled (which will in practice happen by default) is a bad idea for Cython projects, because:
   - Cython (as of this writing, version 0.25) will not see `.pxd` headers inside installed `.egg` files. Thus other libraries cannot `cimport` modules from yours if it has `zip_safe` set.
   - At (Python-level) `import` time, the OS's dynamic library loader usually needs to have the `.so` unzipped (from the `.egg`) to a temporary directory anyway.


### Usage

See the [setuptools manual](http://setuptools.readthedocs.io/en/latest/setuptools.html#command-reference). Perhaps the most useful commands are:

```bash
python setup.py build_ext
python setup.py build    # copies .py files in the Python packages
python setup.py install  # will automatically "build" and "bdist" first
python setup.py sdist
```

Substitute `python2` or `python3` for `python` if needed.

For `build_ext`, the switch `--inplace` may be useful for one-file throwaway projects, but packages to be installed are generally much better off by letting setuptools create a `build/` subdirectory.

For `install`, the switch `--user` may be useful. As can, alternatively, running the command through `sudo`, depending on your installation.

#### Packaging data files

From the viewpoint of Python packaging, data files in your project come in two varieties:

 - **Non-package data files** are files in the project that are to be distributed, but do not belong to any Python package. Typically, this means end-user documentation and usage examples.
 - **Package data** are data files inside Python packages. Typically, these files can be binary data needed by your library (to be loaded at runtime via the `pkg_resources` API), or developer documentation on a particular package.

Non-package data files arguably have no natural binary-install location, unless they are specified with an absolute target path. Any `data_files` specified with relative paths [will install directly under *prefix*](https://stackoverflow.com/questions/24727709/i-dont-understand-python-manifest-in#comment46482024_24727824). **Importantly**, Python environments in different operating systems may set their default *prefix* differently.

For example, it seems that on Linux Mint, each Python package gets its own prefix, whereas Mac OS uses one common prefix for all Python packages. Thus, on Mac OS, if the `setuptools` prefix is set to `/usr/local` (which is typical), then `setuptools` will try to install e.g. the `data_files` specified as `test/*` into `/usr/local/test/*`, which will fail (for good reason).

For non-package data files, it is better to package them only into the source distribution (sdist). On what gets included into the sdist by default, refer to [the documentation](https://docs.python.org/3/distutils/sourcedist.html). GitHub users specifically note that `README.txt` gets included, but `README.md` does not.

The consensus seems to be that the `package_data` option of `setup()` [behaves unintuitively](http://blog.codekills.net/2011/07/15/lies,-more-lies-and-python-packaging-documentation-on--package_data-/). For a long time, it was meant [only for binary distributions and installation](https://stackoverflow.com/questions/7522250/how-to-include-package-data-with-setuptools-distribute), and was ignored for the sdist. However, the documentation says that [in Python 3.1+](https://docs.python.org/3/distutils/setupscript.html#installing-package-data) (and [also in 2.7](https://docs.python.org/2/distutils/setupscript.html#installing-package-data)), all files specified as `package_data` will be included also into the sdist, **but only if no manifest template is provided**.

The manifest template, often also called `MANIFEST.in`, is an optional, separate configuration file for `setuptools` that controls the contents of the sdist. It can be used to include additional files (both package and non-package data), and to exclude any undesired files that would be included in the sdist by default. Historically, this was the way to control the contents of the sdist; now, confusingly, it can also be used to include files in binary distributions and installation (see below).

#### `data_files` vs. `package_data` vs. `MANIFEST.in`?

**`data_files`**:

 - Meant for non-package data files.
 - Will cause `setuptools` to try to install the files, which may cause problems depending on the configuration of the Python environment (specifically *prefix*). Mac OS users beware.
 - May have some limited use, if an absolute target path for installation is applicable.

**`package_data`**:

 - Historically, main way to control binary distributions and installation. Now also controls sdist **if you don't provide** a `MANIFEST.in`.
 - Meant for package data only. Non-package data files (such as `README.md`!) need a different mechanism.
 - The paths to the data files are specified as relative to each package in question. See the example in [`setup.py`](setup.py) of this template project.

**`MANIFEST.in`**:

 - Historically, main way to control sdist. Now also controls binary distributions and installation, **if you set the option** `include_package_data` in your parameters to `setup()`.
 - If this file is present, it overrides `package_data` for sdist.
 - If `include_package_data` is set, any package data included by `MANIFEST.in` will also get binary-packaged and installed. Any non-package data files will be ignored. On the sdist, the option `include_package_data` has no effect.
 - All files included by `MANIFEST.in` will in any case get included into the sdist.
 - The paths to the data files are specified as relative to the directory `setup.py` and `MANIFEST.in` reside in.

**`package_data` + `MANIFEST.in`**:

 - For creating source and binary distributions completely independently of each other. Be careful.
 - Files specified as `package_data` are included into binary distributions and installation.
 - Files included by `MANIFEST.in` are included into the sdist.
 - The `setup()` option `include_package_data` **must not** be set.


#### Format of `MANIFEST.in`

For an overview, see this [quick explanation](https://stackoverflow.com/questions/24727709/i-dont-understand-python-manifest-in). For available commands, see the (very short) [documentation](https://docs.python.org/3/distutils/commandref.html#sdist-cmd).

Simple example `MANIFEST.in`:

```
include *.md
include doc/*.txt
exclude test/testing_an_idea.py
```

In this example, the argument on each line is a shellglob. In `MANIFEST.in`, relative paths start from the directory where `setup.py` and `MANIFEST.in` are located.

The set of files included by `MANIFEST.in` can also be [marked for installation](http://blog.cykerway.com/posts/2016/10/14/install-package-data-with-setuptools.html) (which implies also binary distribution), via setting `include_package_data = True` in the parameters to `setup()`. This requires that any files to be installed reside inside a Python package, so that they will have a location to install into. This can be used e.g. for binary data files that your library uses internally.

When `include_package_data` is enabled, everything that was included by `MANIFEST.in` and is possible to install will be installed. As far as binary distribution and installation are concerned, any non-package data files included by `MANIFEST.in` will be simply ignored. The sdist will be processed as normal; the `include_package_data` setting only concerns binary distribution and installation.

#### Linux binaries

As noted in the [Python packaging guide](https://packaging.python.org/distributing/#platform-wheels), PyPI accepts platform wheels (platform-specific binary distributions) for Linux only if they conform to [the `manylinux1` ABI](https://www.python.org/dev/peps/pep-0513/), so running `python setup.py bdist_wheel` on an arbitrary development machine is generally not very useful for the purposes of distribution.

For the adventurous, PyPA provides [instructions](https://github.com/pypa/manylinux) along with a Docker image.

For the less adventurous, just make an sdist and upload that; scientific Linux users are likely not scared by an automatic compilation step.

#### Uninstalling

Sometimes it is useful to uninstall the installed copy of your package, such as during the development and testing of the install step for a new version.

Whereas `setuptools` does not know how to uninstall packages, `pip` does. This applies also to `setuptools`-based packages not installed by `pip` itself.

For example, to uninstall this template project (if you have installed it):

```bash
pip uninstall setup-template-cython
```

The package name is the `name` argument provided to `setup()` in `setup.py`.

Note that, when you invoke the command, if the current working directory of your terminal has a subdirectory with the same name as the package to be uninstalled (here `setup-template-cython`), its presence will mask the package, which is probably not what you want.

If you have installed several versions of the package manually, the above command will uninstall only the most recent version. In this case, invoke the command several times until it reports that setup-template-cython is not installed.

Note that `pip` will automatically check also the user directory of the current user (packages installed with `python setup.py install --user`) for the package to uninstall; there is no need to specify any options for that.

Substitute `pip2` or `pip3` for `pip` as needed; run through `sudo` if needed.

To check whether your default `pip` manages your Python 2 or Python 3 packages, use `pip --version`.


### Notes

 0. This project assumes the end user will have Cython installed, which is likely the case for people writing and interacting with numerics code in Python. Indeed, our [`setup.py`](setup.py) has Cython set as a requirement, and hence the eventual `pip install mylibrary` will pull in Cython if it is not installed.  

    The Cython extensions are always compiled using Cython. Or in other words, regular-end-user-friendly logic for conditionally compiling only the Cython-generated C source, or the original Cython source, has **not** been included. If this is needed, see [this StackOverflow discussion](http://stackoverflow.com/questions/4505747/how-should-i-[5]-a-python-package-that-contains-cython-code) for hints. See also item 2 below.  

    The generated C source files, however, are included in the resulting distribution (both sdist and bdist).

 1. In Cython projects, it is preferable to always use absolute module paths when `absolute_import` is in use, even if the module to be cimported is located in the same directory (as the module that is doing the cimport). This allows using the same module paths for imports and cimports.  
 
    The reason for this recommendation is that the relative variant (`from . cimport foo`), although in line with [PEP 328](https://www.python.org/dev/peps/pep-0328/), is difficult to get to work properly with Cython's include path.  

    Our [`setup.py`](setup.py) adds `.`, the top-level directory containing `setup.py`, to Cython's include path, but does not add any of its subdirectories. This makes the cimports with absolute module paths work correctly[[8]][packagehierarchy] (also when pointing to the library being compiled), assuming `mylibrary` lives in a `mylibrary/` subdirectory of the top-level directory that contains `setup.py`. See the included example.

 2. Historically, it was common practice in `setup.py` to import Cython's replacement for distutils' `build_ext`, in order to make `setup()` recognize `.pyx` source files.  

    Instead, we let setuptools keep its `build_ext`, and call `cythonize()` explicitly in the invocation of `setup()`. As of this writing, this is the approach given in [Cython's documentation](http://cython.readthedocs.io/en/latest/src/reference/compilation.html#compiling-with-distutils), albeit it refers to distutils instead of setuptools.  

    This gives us some additional bonuses:
      - Cython extensions can be compiled in debug mode (for use with [cygdb](http://cython.readthedocs.io/en/latest/src/userguide/debugging.html)).
      - We get `make`-like dependency resolution; a `.pyx` source file is automatically re-cythonized, if a `.pxd` file it cimports, changes.
      - We get the nice `[1/4] Cythonizing mylibrary/main.pyx` progress messages when `setup.py` runs, whenever Cython detects it needs to compile `.pyx` sources to C.
      - By requiring Cython, there is no need to store the generated C source files in version control; they are not meant to be directly human-editable.

    The setuptools documentation gives advice that, depending on interpretation, [may be in conflict with this](https://setuptools.readthedocs.io/en/latest/setuptools.html#distributing-extensions-compiled-with-pyrex) (considering Cython is based on Pyrex). We do not import Cython's replacement for `build_ext`, but following the Cython documentation, we do import `cythonize` and call it explicitly.  

    Because we do this, setuptools sees only C sources, so we miss setuptools' [automatic switching](https://setuptools.readthedocs.io/en/latest/setuptools.html#distributing-extensions-compiled-with-pyrex) of Cython and C compilation depending on whether Cython is installed (see the source code for `setuptools.extension.Extension`). Our approach requires having Cython installed even if the generated C sources are up to date (in which case the Cython compilation step will no-op, skipping to the C compilation step).  

    Note that as a side effect, `cythonize()` will run even if the command-line options given to `setup.py` are nonsense (or more commonly, contain a typo), since it runs first, before control even passes to `setup()`. (I.e., don't go grab your coffee until the build starts compiling the generated C sources.)  

    For better or worse, the chosen approach favors Cython's own mechanism for handling `.pyx` sources over the one provided by setuptools.

 3. Old versions of Cython may choke on the `cythonize()` options `include_path` and/or `gdb_debug`. If `setup.py` gives mysterious errors that can be traced back to these, try upgrading your Cython installation.  

    Note that `pip install cython --upgrade` gives you the latest version. (You may need to add `--user`, or run it through `sudo`, depending on your installation.)

 4. Using setuptools with Cython projects needs `setuptools >= 18.0`, to correctly support Cython in `requires`[[9]][setuptools18].  

    In practice this is not a limitation, as `18.0` is already a very old version (`35.0` being current at the time of this writing). In the unlikely event that it is necessary to support versions of setuptools even older than `18.0`, it is possible[[9]][setuptools18] to use [setuptools-cython](https://pypi.python.org/pypi/setuptools_cython/) from PyPI. (This package is not needed if `setuptools >= 18.0`.)

 5. If you are familiar with distutils, but new to setuptools, see [the list of new and changed keywords](https://setuptools.readthedocs.io/en/latest/setuptools.html#new-and-changed-setup-keywords) in the setuptools documentation.


### Distributing your package

If you choose to release your package for distribution:

 0. See the [distributing](https://packaging.python.org/distributing/) section of the packaging guide, and especially the subsection on [uploading to PyPI](https://packaging.python.org/distributing/#uploading-your-project-to-pypi). A short summary of key terms can also be found in the Python documentation on [distributing Python modules](https://docs.python.org/3/distributing/index.html).  

    Especially if your package has dependencies, it is important to get at least an sdist onto PyPI to make the package easy to install (via `pip install`).  

      - Also, keep in mind that outside managed environments such as Anaconda, `pip` is the preferred way for installing scientific Python packages, even though having multiple package managers on the same system could be considered harmful. Scientific packages are relatively rapidly gaining new features, thus making access to the latest release crucial.  

        (Debian-based Linux distributions avoid conflicts between the two sets of managed files by making `sudo pip install` install to `/usr/local`, while the system `apt-get` installs to `/usr`. This does not, however, prevent breakage caused by overrides (loading a newer version from `/usr/local`), if it happens that some Python package is not fully backward-compatible. A proper solution requires [one of the virtualenv tools](http://stackoverflow.com/questions/41573587/what-is-the-difference-between-venv-pyvenv-pyenv-virtualenv-virtualenvwrappe) at the user end.)  

 1. Be sure to use `twine upload`, **not** ~~`python -m setup upload`~~, since the latter may transmit your password in plaintext.  

    ~~Before the first upload of a new project, use `twine register`.~~ As of August 2017, pre-registration of new packages is no longer needed or supported; just proceed to upload. See [new instructions](https://packaging.python.org/guides/migrating-to-pypi-org/#uploading).  

    [Official instructions for twine](https://pypi.python.org/pypi/twine).

 2. Generally speaking, it is [a good idea to disregard old advice](http://stackoverflow.com/a/14753678) on Python packaging. By 2020 when Python 2.7 support ends, that probably includes this document.  

    For example, keep in mind that `pip` has replaced `ez_setup`, and nowadays `pip` (in practice) comes with Python.  

    Many Python distribution tools have been sidelined by history, or merged back into the supported ones (see [the StackOverflow answer already linked above](http://stackoverflow.com/a/14753678)). Distutils and setuptools remain, nowadays [fulfilling different roles](http://stackoverflow.com/a/40176290).


### Compatibility

Tested on Linux Mint, Python 2.7 and 3.4.

On Mac OS, the `data_files` approach used in the example will not work, because all Python packages share the same prefix.

Not tested on Windows (please send feedback, e.g. by opening an issue).

#### Platform-specific notes

On **Linux Mint**:

 - The package installs into a subdirectory of the base install location, with a name following the format `setup_template_cython-0.1.4-py3.4-linux-x86_64.egg`. The `mylibrary` and `test` subdirectories appear under that, as does this README.
 - with `python3 setup.py install --user`, the base install location is `$HOME/.local/lib/python3.4/site-packages/`.
 - with `sudo python3 setup.py install`, the base install location is `/usr/local/lib/python3.4/dist-packages/`.

Then, in Python, `import mylibrary` will import the library. The `test` subdirectory of the project is harmless; `import test` will still import Python's own `test` module.


### License

[The Unlicense](LICENSE.md)

[setuptools]: https://packaging.python.org/key_projects/#easy-install
[packaging]: https://packaging.python.org/current/#packaging-tool-recommendations
[distributing]: https://packaging.python.org/distributing/
[setup-py]: https://packaging.python.org/distributing/#setup-py
[getversion]: http://stackoverflow.com/questions/2058802/how-can-i-get-the-version-defined-in-setup-py-setuptools-in-my-package
[datafolder]: http://stackoverflow.com/questions/13628979/setuptools-how-to-make-package-contain-extra-data-folder-and-all-folders-inside
[enforcing]: http://stackoverflow.com/questions/19534896/enforcing-python-version-in-setup-py
[packagehierarchy]: https://github.com/cython/cython/wiki/PackageHierarchy
[setuptools18]: http://stackoverflow.com/a/27420487

