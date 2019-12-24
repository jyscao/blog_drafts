<!-- # Packaging a Module for Lua Game Engine in Guix -->
<!-- # Packaging the Nuklear GUI Library for the LÖVE Game Engine in Guix -->

# Guix Packaging by Example



A key design principle behind [GNU Guix](http://guix.gnu.org/) as a
package manager is its extensibility, i.e. allowing users to easily
write their own package definitions. Thus this post aims to serve as a
starting reference for doing precisely that.

The package I'll be using for this demonstration is
[LÖVE-Nuklear](https://github.com/keharriso/love-nuklear/),
which provide bindings to the immediate mode GUI library,
[Nuklear](https://github.com/Immediate-Mode-UI/Nuklear) for the Lua game
engine, [LÖVE](https://love2d.org/).



## Load Required Modules

All Guix packages are defined in source files written in Guile Scheme,
with each source corresponding to a single module that contains packages
related along a common theme. For example, libraries for specific
programming languages like Golang or Rust, enduser applications for
music or games, implementations of particular protocols like FTP or
IPFS, and so on.

My example module shall only contain a single package, but in general
there can be many. To begin, we first load the modules required to
define the package:

```
(define-module (love-nuklear)
               #:use-module (guix packages)
               #:use-module (guix git-download)
               #:use-module (guix build-system cmake)
               #:use-module (guix licenses)
               #:use-module (gnu packages lua))
```

All package definitions require at least 4 modules, which are all
located in the `guix` namespace:

1. `packages`, this module exports the `<package>` record type, of which
your package definition shall become an instance of

2. source code retrieval module, typically `download` (for downloading
sources over HTTP(S)) or `git-download` (as shall be seen in this
example); other methods such as `hg-download` and `svn-download` are
also available

3. build-system, such as `gnu` (GNU Make being its core), `meson` with
its `ninja` backend for C/C++, `dune` (for OCaml), `cmake` (used for
this example), etc.

4. `licenses`, which is self-explanatory

Of course in practice, the vast majority of software contain
dependencies that must also be loaded. And if these dependencies are
already packaged, then they can be found under the `gnu packages`
namespace. For my example of `love-nuklear`, the only dependency is the
`luajit` package from the `lua` module.



## Fill Basic Info & Metadata

Now it's time to define the actual package, starting with its basic
information and metadata:

```
(define-public love-nuklear
 (let ((version "v2.6")
       (commit "fef4e00a602efb16c57ae962850b6e7a01f7a29a"))
  (package
   (name "love-nuklear")
   (version (git-version version "+4commits" commit))
   ;; ...
   ;; ...
   ;; ...
   (synopsis "Lightweight immediate mode GUI for LÖVE games")
   (description "LÖVE is a Lua framework for making 2D games.  Nuklear
is a minimal state immediate mode graphical user interface toolkit.  This
package is the Nuklear bindings for LÖVE created by Kevin Harrison.")
   (home-page "https://github.com/keharriso/love-nuklear/")
   (license expat))))
```

A few points worth mentioning:

* Guix packages are defined via `define-public` by convention. This
eliminates having to manually declare the package definiton for `export`
as part of the module's public interface.

* The `version` field can be any string, so a simple `"v2.6"` string
literal would be equally valid in the above. But since I'll be using
`git-download`, I wanted to provide maximum versioning info for the sake
of reproduciblity. Thus the `git-version` procedure was used, which
takes 3 arguments: `version`, `revision`, `commit` hash, and evaluates
to an informative yet succinct string to be used as the version ID of
the package.

* If the software being packaged is multi-licensed, the `license` field
can be set to a list representing such



## Fetch Source

As briefly mentioned earlier, although Guix supports several source
fetching methods, in practice you'll mostly be using either `url-fetch`
from the `download` module, or `git-fetch` from the `git-download`
module. Whenever possible, you should prefer the former, since it is
more efficient.

For LÖVE-Nuklear however, `git-fetch` was needed because the source
code of [Nuklear](https://github.com/Immediate-Mode-UI/Nuklear), which
is not bundled in the tarball release, but is instead tracked as a
git-submodule, is required to build the final shared object. So to do
this, we use `git-fetch` with `(recursive? #t)`, as shown below:

```
   (source (origin
            (method git-fetch)
            (uri (git-reference
                  (url "https://github.com/keharriso/love-nuklear/")
                  (commit commit)
                  (recursive? #t)))
            ;; NOTE: the HEAD of the Nuklear git-submodule is at commit
            ;; "adc52d710fe3c87194b99f540c53e82eb75c2521" of Oct 1 2019
            (file-name (git-file-name name version))
            (sha256
             (base32
              "15qmy8mfwkxy2x9rmxs6f9cyvjvwwj6yf78bs863xmc56dmjzzbn"))))
```

Before building from the source, Guix checks that the hash of the
downloaded files is the same as that supplied by you, the packager.
When using `url-fetch`, the base32 hash of the project's source will
be directly computed by Guix and displayed for your convenience after
you execute `$ guix download <https://url-package-source>`. When
using `git-fetch` however, you must first `git clone` the repository
(including the submodules if applicable), then invoke `$ guix hash -xr
<path/to/git/source>`, where the `-x` flag tells Guix to ignore VCS
files and `-r` tells Guix to compute the hash recursively.



## Specify Dependencies

Guix differentiates between 3 types of dependencies:

* `native-inputs`: build but not runtime dependencies

* `inputs`: runtime dependencies

* `propagated-inputs`: similar to `inputs`, but can also be useful
for specifying packages that should be installed alongside your main
package. You might want this when for example, header files from another
library are required to compile the package in question, or to gain
access to runtime libraries in languages that lack the facility to
record runtime search paths.

As already stated, the only runtime dependency, and thus `input` of
`love-nuklear` is `luajit`:

```
   (inputs
    `(("luajit" ,luajit)))
```



## Fine-Tune Build Procedure

At the time of writing, Guix provides modules for 30+ [build
systems](https://guix.gnu.org/manual/en/html_node/Build-Systems.html).
From the foundational `gnu-build-system`, which all other build
systems inherit from to one degree or another, to language specific
ones like `cargo-build-system` for Rust, `dune-build-system` for
OCaml, `python-build-system`, etc., to build script generators like
`meson-build-system` and `cmake-build-system`.

Note that in addition to representing the build procedure to be used,
the `build-system` field also implicitly specify dependencies of said
build procedure. Thus it is unnecessary to manually specify these build
dependencies as `native-inputs`.

The following snippet shows the first build procedure that was
successful in installing the `love-nuklear` package:

```
   (build-system cmake-build-system)
   (arguments
    `(#:build-type "Release"
      #:tests? #f
      #:phases
      (modify-phases %standard-phases
       (replace 'install
        (lambda* (#:key outputs #:allow-other-keys)
                 (let* ((out (assoc-ref outputs "out"))
                        (share (string-append out "/share")))
                       (install-file "nuklear.so" share)
                       #t))))))
```

* The `#:build-type` argument is specific to the `cmake-build-system`,
which can also accommodate other build flags typically passed on the
command line to `cmake`.

* All build systems accept the `#:tests?` argument, which indicates
whether tests should be run after your package has been successfully
built (`#t` by default).

* The phases of the build procedure itself are modifiable. And quite
often, one may find oneself needing to do just that, since not all
software fully adhere to standarized build procedures.

* This was the case for LÖVE-Nuklear when I began my attempt in
packaging it. Its CMake file did not generate an install target for the
output `nuklear.so`. Therefore the standard `'install` phase of the
build procedure was replaced by a custom one defined by the `lambda`
expression seen above.

* `out` represents the location of the output directory, which is
obtained by Guix via a `getenv` call under-the-hood during runtime, as
it cannot be known beforehand due to it depending on all the inputs of
the package definition.

In the end though, I decided it was better to make the change
upstream, i.e. add an install target to the CMake build procedure of
LÖVE-Nuklear. After this change was kindly merged by its author,
`love-nuklear` was then able to be built and installed smoothly with
zero modifications to the standard phases of the `cmake-build-system`,
resulting in the cleaner snippet shown below:

```
   (build-system cmake-build-system)
   (arguments
    `(#:build-type "Release"
      #:tests? #f))
```

But just keep in mind that, in order to produce a successful package
build, sometimes the only option is to modify the standard phases of
one's build system.



## Test Package Definition

Once the first draft of your package definition is complete, you are
then ready to test out its correctness by attempting to build it
locally.

To do so, just run `$ guix build -K --file=<path/to/package/def>`.
Make sure to add a line at the end of your package module to actually
evaluate the package definiton (`love-nuklear` for my example), as
the `--file` flag tells Guix to build the package that the source
within `<path/to/package/def>` evaluates to. The `-K` flag is short for
`--keep-failed`, thus failed partial build results will be left in your
`/tmp` directory instead of being removed.

Another source of useful debugging information is the build log, which
in addition to being printed to STDIN on each build, is also written to
an appropriate directory identified by its build hash in `/var/guix/`.

Through the iterative process of fixing errors reported in the build
log, and occasionally inspecting the contents of failed partial builds,
I was able to quickly bring the package definition of LÖVE-Nuklear
to a successful state, whose constituent components are exactly the
snippets already showcased earlier. But for viewing convenience, here's
the finalized package definiton in its totality:

**TODO: show full `love-nuklear` definition gist**



## Contribute to Guix

If you'd like to submit your package definition upstream,
i.e. have it added to the GNU Guix ecosystem, then it's
important to run `$ guix lint` before [submitting it to the
project](http://guix.gnu.org/contribute/) as a patch. This command
performs useful checks on your package definition, ensuring it conforms
to the GNU Guix standard. All available checkers can be listed with `$
guix lint --list-checkers`. Here are what some of them do:

* Provide basic validations, such as checking the formatting of your
package's synopsis and description, the existence of the project's
home-page URL, and validity of the license(s)

* Offer technical suggestions, such as the appropriateness of the
categorizations of your specified inputs, the success or failure of
compiling a package to its derivation (which of course should succeed if
you were able to run `guix build` successfully)

Lastly, I'm pleased to inform that my package definition for
LÖVE-Nuklear has been accepted into the GNU Guix project. So if you'd
like to check out this cool GUI module for the LÖVE game engine on Guix
yourself, then it's as simple as:

1. `$ guix pull` to get the latest updates for Guix
2. `$ guix package --install love-nuklear`
