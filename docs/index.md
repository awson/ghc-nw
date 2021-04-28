## General Information

This is a [GHC](https://www.haskell.org/ghc) 8.10.5 (8.10 branch as of
Apr 11, 2021) 64-bit Windows distro, based on and targeting Microsoft
Visual C runtime and native Windows SDK.

### How it differs from stock Windows GHC distro

This distro is based on the stock LLVM (clang, lld, llvm-lib) tools and
my own set of tools --- haskell DLL exporter, haskell DLL symbol
loader/locator (for interactive/TH GHC), merge objects tool and
LLVM-based llvm-windres, all the tools are written in C++ and are
blazingly fast.

It produces the code that is binary compatible and statically linkable
with native Microsoft Visual C (dynamic CRT) and Windows SDK libraries.

This distro fully supports dynamic linking of *Haskell* libraries and
programs (meet 14kb helloworld exe). The compiler we ship is itself
dynamic.

The stock distro doesn't support this because of:

-   Windows DLL support in GHC is horribly bitrotted and has some never
    resolved issues;
-   The DLL format has a limit of max 64k exported symbols, and large
    Haskell packages (and first of all, GHC itself) may want to export
    more.

For convenience, we also ship statically linked GHC (`ghc-static.exe`,
to use it tell cabal `--with-ghc=ghc-static.exe`), which is analogous
to the stock GHC and has exactly the same workflow. And this workflow is
much faster than the full one because it doesn't create dynamic
versions of packages.

All stock GHC tools are position *dependent* executables and produce
position *dependent* code that must be loaded into lower 2GB address
space. Modern security standards consider this a disadvantage. Btw,
Microsoft tools *won't even let you* create a DLL satisfying this
condition.

Our tools are position independent executables and generate position
independent code that can be loaded into arbitrary addresses.

### Some other advantages

LLD linker is orders of magnitude faster than GNU ld when linking object
files with a large number of sections (which is the case when
`split-sections` is enabled).

When creating pre-linked object files for static GHC ("ghci
libraries") for large packages our tools are much (twice or more)
faster than GNU ld.

When using only static compile workflow using static GHC, we observed
that even at compilation stage only, our GHC is considerably faster
(~9%) on a long-to-compile package (haskell-src-exts) and slightly
slower (~2%) on a medium-to-compile package (vector). Perhaps the LLVM
machine-code assembler is faster than GNU gas. Or perhaps my
measurements are completely bogus (although I tried several times and
got similar results each time).

### Disadvantages

The biggest one is that the Windows GHC ecosystem is centered around GNU
tools, so some packages that have important non-Haskell parts (C code
etc) may need to be modified to work with native Microsoft CRT. The
patches provided by this project are mainly related to this.

If anyone who dares to try my toolset finds a package that don't
compile out of the box and isn't covered by patches provided, I would
be happy to develop the patches required to make the package work. Or I
have the patches ready but forgot to publish them.

Please, use [project's issue
tracker](https://github.com/awson/ghc-nw/issues) for feedback.

While the static workflow should be at least on par with stock GHC,
there exists one known issue with the dynamic workflow related to
handling foreign imported addresses (`foreign import ccall "&foo" :: Ptr a`).

Microsoft and/or Miscrosoft ABI-targeted LLVM tools have no Mingw
runtime pseudo-relocs support (automatic linking of *data*), and if
"foo" resides in a DLL and we need to access it from outside of this
DLL, we really need to declare `foreign import ccall "&__imp_foo" :: Ptr (Ptr a)` instead.

I have some developments in the pipeline to address this issue in
varying degrees of generality, including full port of runtime
pseudo-relocs support to MS world.

But for runtime pseudo-relocs to work either all the code must reside in
the lower 2GBs, or all the relevant codegens must be aware of runtime
pseudo-relocs and generate the appropriate code (any DSO which we can't
prove is local must be accessed through variable stub).

Mingw GCC and clang with Mingw target do this by default, but neither
clang with MSVC target nor any other Visual C compatible compiler nor
GHC itself do this.

Its possible to introduce a separate switch to tell clang (and any other
LLVM-based tool, e.g. GHC llvm backend) to do it even for the MSVC
target (I have such a patch), and teach GHC native codegen to do this,
but still all other tools in the MS world, notably Visual C itself, and
any other compatible compiler *won't do this*, i.e. linking any
third-party binary objects would be extremely dangerous.

But it happens very rarely in a wild, so it may not be necessary to
solve this problem in general, we may well get away with per-case
workarounds at the moment.

### Installation

First, install Visual Studio 2019 "Desktop development with C++"
workload (see e.g. `install_vs.bat` in distro directory).

Then just unpack the distro to any location, let the location be named
'selected_location' then add 'selected_location\ghc-8.10.5\bin'
(and also `selected_location\ghc-8.10.5\xtra\bin` is you want to
use extra utilities, such as Happy, Alex etc) to you PATH.

Long ago I tested if things work when 'selected_location' contain
spaces and that worked, but a lot happened since then and it might be
better to not put spaces in it.

We ship no ghci executable, so please use `ghc --interactive`
instead.

For your convenience I include all the necessary stock LLVM tools into
the distro, but you can use your own if want so.

There exist 2 versions of each distro: "core" (GHC +
cabal/alex/happy/stack) and "full", which also contains a (large) set of
prebuilt packages and useful utilities, and even the entire Agda
compiler (from the current git HEAD).

[DOWNLOAD!](https://github.com/awson/ghc-nw/releases/tag/0.0.2)

### IMPORTANT

I didn't bother to introduce different platform name for my toolset,
reusing "mingw32" for it, so you *CAN'T* use exactly the same version
of the stock GHC on the same machine without special care: either you
have to set different package database and package directories locations
using Cabal/Stack switches, or you need to switch between the toolsets
using directory links or whatever.

You can only use `cabal.exe` and `stack.exe` from this distro to work with
our toolset. The stock `cabal.exe` and `stack.exe` *ARE INCOMPATIBLE* with
our toolset. And our `cabal.exe` and `stack.exe` *ARE INCOMPATIBLE* with
stock GHCs.

I build everything using Cabal and it worked well, I almost use no
stack, so there may be problems. But since *almost all* of modifications
are in the Cabal *library*, which powers both tools, I expect Stack to
be fine (if not, please let me know!).

I haven't bothered to modify RTS linking scheme (a known, long standing
issue), so all package DLLs are linked against nodebug, nolog, threaded
RTS Dll (`HSrts-1.0.1_thr-ghc8.10.5.dll`).

Thus if you ever want to build a dynamic executable (or DLL), e.g, doing
`ghc -dynamic --make Foo`, *YOU SHALL ADD the `-threaded` flag to
that*, i.e. do `ghc -dynamic -threaded --make Foo`. Otherwise you
application will segfault immediately.

If you manually build a dynamic executable or DLL, you must ensure your
application has access to all the DLLs it depends on.

Cabal automatically manages this when building Haskell packages. It just
adds all the necessary paths to the environment when running GHC.

Our toolset executables and DLLs use manually crafted driver executables
and a couple of manifest files.

Your experience may be different, but the easiest way is to do what
cabal does: add necessary paths to your PATH env variable.

All system package DLLs are accessible via
`selected_location\ghc-8.10.5\lib\x86_64-windows-ghc-8.10.5`.

If you want (like me) to create a large globally accessible package repo
(e.g. using `cabal v1-install`) --- it's default location is
`%APPDATA%\cabal\x86_64-windows-ghc-8.10.5` where all the package
DLLs sit.

Cabal `v2-build` (`v2-install`) puts anything package-related (including
DLL) into the directory of that package. The default location for
package files in the global store is
`%APPDATA%\cabal\store\ghc-8.10.5\package_id\lib`, where
`package_id` is a composition of package name version and hash.

If you deploy your executables to another machine, the simples way is to
just copy all the necessary DLLs to the same location.

### Some Details

I provide NO SOURCES at the moment other than the patches from this
project.

I'll try to release the patches that generally improve GHC and could be
incorporated into the main GHC codebase. But I have neither time nor
energy to clean up and release most of MSVC-specific patches.
