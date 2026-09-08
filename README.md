## Old Rubies, meet new macOS, Ubuntu, and Arch

Trying to run an old, unsupported Ruby? You're in the right place.

This is a collection of ruby-build definitions for local dev on macOS, Ubuntu, and Arch
(including Omarchy). Not for production use.

Available definitions:

| Version | OpenSSL | Bundler |
|---------|---------|---------|
| 1.8.7-p374 | 1.0.2u | 1.17.3 |
| 1.9.3-p551 | 1.0.2u | 1.17.3 |
| 2.0.0-p648 | 1.0.2u | 1.17.3 |
| 2.1.10 | 1.0.2u | 1.17.3 |
| 2.2.10 | 1.0.2u | 1.17.3 |
| 2.3.3 | 1.0.2u | 1.17.3 |
| 2.3.8 | 1.0.2u | 1.17.3 |
| 2.4.10 | 1.1.1w | 1.17.3 and 2.3.27 |
| 2.5.9 | 1.1.1w | 1.17.3 and 2.3.27 |
| 2.6.10 | 1.1.1w | 1.17.3 and 2.4.22 |
| 2.7.8 | 1.1.1w | 1.17.3 and 2.4.22 |

2.0.0-p648, 2.1.10, 2.2.10, 2.4.10, and 2.6.10 are built and tested on Ubuntu Resolute
(26.04) only, on both amd64 and arm64. They may well work elsewhere, but nothing checks
that; the others are tested on Ubuntu Noble and Arch, and on macOS by hand.

Modern OpenSSL dropped the APIs these Rubies need, so each definition brings its own. On
macOS the OpenSSL 1.0 builds come from [basecamp/homebrew-dev](https://github.com/basecamp/homebrew-dev);
everywhere else OpenSSL is compiled from source into the Ruby's own prefix, so nothing
lands system-wide and nothing is shared between versions.

### Optimization flags

Every definition builds with `-O3 -fno-strict-overflow`. Both halves matter:

**`-O3`** — leaving `RUBY_CFLAGS` empty is not "use configure's default". ruby-build exports
it as `CFLAGS`, which supersedes configure's own `optflags` in the compile line, so an empty
value builds at `-O0`. That's 2–3.5× slower. Confusingly, `RbConfig::CONFIG["optflags"]`
still reports `-O3` in that case; time a build rather than believing it.

**`-fno-strict-overflow`** — these sources predate the compilers building them, and their
fixnum overflow checks assume signed overflow wraps. That's undefined behaviour, and GCC
exploits it from `-O2` up. Build 1.8.7 at `-O3` without this flag and it compiles cleanly,
runs, loads every stdlib — and evaluates `2**64` to `0`, typed `Fixnum`. Silent wrong
arithmetic. `test/build` asserts against exactly this, so the trap can't come back.

`-march=native` is deliberately **not** used. It measured ~12% slower than plain `-O3` on
Zen 4, worst on numeric loops (−16% integer, −21% float), and microbenchmarks are the
friendliest case it gets. It would also make binaries non-portable between machines for no
gain.

**1.9.3 caps `_FORTIFY_SOURCE` at 2 under GCC.** Ubuntu's GCC raises it to 3 automatically
whenever optimization is on, and level 3's object-size inference trips on 1.9.3 — the build
aborts with `*** buffer overflow detected ***`. This isn't a hardening regression: fortify
does nothing without optimization, so while these were building at `-O0` there was none at
all. Only 1.9.3 needs the cap.

### Prerequisites

**macOS:** Xcode command line tools and [Homebrew](https://brew.sh).

**Ubuntu:**

```bash
sudo apt-get install -y autoconf bison build-essential curl git libdb-dev libffi-dev \
  libgdbm-dev libgmp-dev libncurses5-dev libreadline-dev libssl-dev libyaml-dev \
  patch uuid-dev zlib1g-dev
```

**Arch / Omarchy:**

```bash
sudo pacman -S --needed autoconf base-devel bison git gmp libffi libyaml openssl patch readline zlib
```

### mise

```bash
git clone https://github.com/basecamp/ruby-dev

export RUBY_BUILD_DEFINITIONS="$PWD/ruby-dev"
mise install ruby@1.8.7
```

`RUBY_BUILD_DEFINITIONS` must be an absolute path and must be set for every command that
builds a Ruby — mise shells out to ruby-build, which reads it from the environment. See
[the warning below](#a-warning-that-applies-to-all-of-the-above); a bad path fails silently.

Fuzzy versions resolve to the definitions here, so `ruby@1.8.7` gets you `1.8.7-p374` and
`ruby@2.3` gets you `2.3.8`.

To pin a project to one:

```bash
export RUBY_BUILD_DEFINITIONS="/absolute/path/to/ruby-dev"
mise use ruby@1.8.7-p374
```

### rbenv

Same idea — point `RUBY_BUILD_DEFINITIONS` at the checkout:

```bash
git clone https://github.com/basecamp/ruby-dev

RUBY_BUILD_DEFINITIONS="$PWD/ruby-dev" rbenv install 1.8.7-p374
```

Or skip the environment variable and hand ruby-build the definition file directly:

```bash
git clone https://github.com/basecamp/ruby-dev
cd ruby-dev
rbenv install ./1.8.7-p374
```

Or grab a single definition without cloning:

```bash
curl -O https://raw.githubusercontent.com/basecamp/ruby-dev/main/1.8.7-p374
rbenv install ./1.8.7-p374
```

(Note the leading `./` - this is a path to a specific build definition for ruby-build, not a version specific for it to look for among its built-in definitions.)

### ruby-build on its own

You don't need a version manager. ruby-build installs a Ruby into any prefix you name:

```bash
git clone https://github.com/basecamp/ruby-dev
cd ruby-dev

ruby-build ./1.8.7-p374 ~/.rubies/1.8.7-p374
~/.rubies/1.8.7-p374/bin/ruby --version
```

`RUBY_BUILD_DEFINITIONS` works here too, and is the better choice when something else is
invoking ruby-build on your behalf:

```bash
RUBY_BUILD_DEFINITIONS="$PWD" ruby-build 1.8.7-p374 ~/.rubies/1.8.7-p374
```

This is also the form to use with chruby or any other manager that just wants a directory
of Rubies — build into its search path (`~/.rubies` for chruby) and it'll pick them up.

### Portable tarballs

A definition can also be built into a tarball that runs from wherever it is extracted, the
way [jdx/ruby](https://github.com/jdx/ruby) does for current Rubies:

```bash
bin/portable 2.7.8 ubuntu-resolute-arm64   # -> dist/ruby-2.7.8.ubuntu-resolute-arm64.tar.gz
bin/portable 2.7.8 ubuntu-resolute-amd64
```

Extract it anywhere and run `bin/ruby`; nothing needs to be installed on the host beyond
glibc. Ruby is configured with `--enable-load-relative`, so it finds its own prefix from
the executable, and OpenSSL, zlib, libyaml, and libffi are compiled in statically. Their
headers and static libraries ship inside the prefix and rbconfig is rewritten to point at
them, so native gems still build after relocation (given a compiler on the host). OpenSSL
uses the host's `/etc/ssl/certs`; a host without one falls back to a bundled CA file.

Everything version-specific — source URL and checksum, OpenSSL version, compiler flags,
configure options, which bundlers to install — is read from the ruby-build definition, so
the two builds can't drift apart. Extensions that would drag in a library nothing
guarantees (gdbm, readline, tk) are left out; `require "readline"` still works via reline
in 2.7.

One caveat comes from RubyGems 3.1, the version 2.7 ships: it doesn't know about
load-relative prefixes, so executables it installs get the absolute path of wherever the
tree lives at the time. The stubs in the tarball are rewritten to be relative, and
`gem install` on the extracted tree works normally, but gems installed after extracting
tie the tree to that location — move it again and reinstall them.

The build runs in the platform's Docker image. The result is then extracted at a random
path inside a *fresh* Ubuntu image with a compiler and nothing else — no `-dev` packages,
no `ca-certificates` — and has to load every bundled extension, fetch an https URL, and
compile a native gem. A tarball built on Resolute runs on Resolute or anything with a
newer glibc; to reach older distributions, build on the oldest one you need to support.

### A warning that applies to all of the above

Whichever tool you use, the definitions are only found if ruby-build can actually see them:

- **`RUBY_BUILD_DEFINITIONS` must be an absolute path.** Build tools run from temporary
  working directories, so a relative path usually resolves to nowhere.
- **A wrong path fails silently.** ruby-build appends its own bundled definition directory
  to the search list and takes the first match, so a typo'd or unexpanded path doesn't
  error — it quietly builds the *upstream* definition instead, which is exactly the one
  that fails on a modern compiler. Confusing compiler errors are the usual symptom.
- **`"~/path/to/ruby-dev"` does not expand.** The tilde is literal inside double quotes.
  Use `"$HOME/path/to/ruby-dev"` or leave it unquoted.

If a build fails in a way that looks nothing like the notes in this repo, check that the
definition was actually picked up before debugging the compiler error.

### CI

There's no cloud CI here. Run `bin/ci` before merging, and it signs off the commit for you
on success:

```bash
bin/ci              # lint + the full build matrix, then gh signoff
bin/ci --lint       # lint only, no Docker — seconds, good for a quick check
bin/ci arch         # lint + Arch only
bin/ci arch 2.7.8   # lint + a single build
```

Only a full run signs off. Anything narrower reports its results and explicitly declines to
sign, because a green tick that covered one platform is worse than no tick.

The lint pass is cheap and catches the two mistakes that otherwise surface ten minutes into
a Docker build: a syntax error in a definition (ruby-build sources these, so a stray quote
is a build failure), and an `install_package` URL with no `#sha256` (ruby-build silently
skips verification when the checksum is missing).

Sign-off needs the extension:

```bash
gh extension install basecamp/gh-signoff
```

### Testing

`test/build` builds definitions in throwaway Docker containers, so a clean-machine build
is checked without touching your own toolchain. `bin/ci` runs it for you; use it directly
when you want a specific slice.

```bash
test/build arch 1.8.7-p374            # one version on one platform
test/build ubuntu-noble all           # every version on one platform
test/build ubuntu-resolute-arm64 all  # a platform on a specific architecture
test/build all                        # everything
```

Platforms are `ubuntu-noble`, `ubuntu-resolute-amd64`, `ubuntu-resolute-arm64`, and
`arch`. A definition that only targets some of them says so in a `# platforms:` comment on
its first line, and `all` respects that; naming a platform and a version explicitly always
runs the pair, so anything can still be tried anywhere.

Builds run concurrently, so results stream in out of order and a sorted summary with any
failure logs is printed at the end. Tune the load with `JOBS` (containers at a time,
defaults to cores/4) and `MAKE_JOBS` (`make -j` inside each, defaults to 4):

```bash
JOBS=4 MAKE_JOBS=8 test/build all
```
