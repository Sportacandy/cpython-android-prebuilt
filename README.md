# cpython-android-prebuilt

Prebuilt `libpython3.x.so` + standard library, cross-compiled for Android,
published as downloadable GitHub Release assets so any project can embed a
Python interpreter in a native Android app without setting up its own
cross-compile toolchain.

This is pure build automation. It doesn't add, remove, or patch anything in
CPython itself — it just runs CPython's own official Android build support
(`Android/android.py`, added in CPython 3.13 per
[PEP 738](https://peps.python.org/pep-0738/)) on a Linux CI runner (that
script only runs on Linux/macOS — not Windows) and republishes the result.

## Why this exists

Embedding CPython (via the standard C embedding API — `Py_Initialize` and
friends) into an existing native Android app is a legitimate, Google-Play-
policy-compliant way to run Python code — including code fetched at runtime
— without hitting Android's restriction on executing downloaded *native*
binaries: the interpreter itself is bundled inside the signed app (not
downloaded), and it only ever *interprets* `.py`/`.pyc` data, never treats it
as directly-executable machine code. See Google's own Device and Network
Abuse policy: *"code that runs in a virtual machine or an interpreter... Apps
or third-party code, like SDKs, with interpreted languages (JavaScript,
Python, Lua, etc.) loaded at run time must not allow potential violations of
Google Play policies."*

The first consumer of this is
[Vivace](https://github.com/Sportacandy/vivace), which uses it to run
yt-dlp's own official pure-Python release (a `.pyz` zipapp — data, not an
executable) for YouTube URL resolution on Android, where spawning a
downloaded native `yt-dlp`/`ffmpeg` binary as a subprocess is not possible.

## What gets built

For each target (a GNU triple, matching what `android.py` itself accepts —
**not** the Gradle-style ABI name):

| Target                  | Gradle ABI name |
|--------------------------|------------------|
| `aarch64-linux-android`  | `arm64-v8a`      |
| `x86_64-linux-android`   | `x86_64` (emulators) |

Each release asset is a plain `python-{version}-{target}.tar.gz` (exactly
what `android.py package` produces, untouched). Verified against a real
build's actual contents (this doc was wrong about the layout once already —
don't trust it blindly, `tar -tzf` a release asset yourself if in doubt):

```
prefix/include/python3.x/   # Python.h and friends, for embedding
prefix/lib/libpython3.x.so
prefix/lib/libpython3.so    # unversioned copy of the same library
prefix/lib/python3.x/       # the standard library, as a plain directory
                             # tree (not pre-zipped) -- includes CPython's
                             # own _android_support.py bootstrap module,
                             # and the full test/ suite (you'll likely want
                             # to strip that when repackaging into your own
                             # app's assets)
prefix/lib/libssl.so, libcrypto.so, libsqlite3.so, ...
prefix/lib/pkgconfig/
README.md, android.py, android-env.sh   # copies of CPython's own Android/
                                         # directory, for reference
testbed/                                # CPython's own Gradle/Kotlin sample
                                         # app exercising this build -- a
                                         # useful embedding reference, not
                                         # something you need to ship
```

The `.so` files' NEEDED entries were confirmed (via the workflow's own
"Verify libpython has no shared libc++ dependency" step) to be only plain
bionic libraries (`libc.so`, `libm.so`, `libdl.so`, `liblog.so`) -- no
dependency on `libc++_shared.so`, so linking this into an app built with a
*different* NDK version than this workflow's own pinned one is safe (see
that step's own comment for the reasoning).

Minimum Android API level: **24** (CPython 3.14's own default; see
`Android/android-env.sh` in the CPython source for the current value if the
pinned version is bumped).

## Using a release in your own project

1. Download the `.tar.gz` for your target triple from the
   [Releases](https://github.com/Sportacandy/cpython-android-prebuilt/releases)
   page and extract it.
2. Link `prefix/lib/libpython3.x.so` into your native app; add
   `prefix/include/python3.x/` to your include path.
3. Bundle `prefix/lib/python3.x/` (the stdlib) as an asset your app extracts
   to a private, writable directory at first run (optionally stripping
   `test/`, `idlelib/`, `tkinter/` first to save space), and point
   `PyConfig.pythonpath_env` / `Py_SetPath` at it before
   `Py_InitializeFromConfig`.
4. See <https://docs.python.org/3/using/android.html> for the embedding API
   itself, and `testbed/app/src/main/c/main_activity.c` in a release asset
   for CPython's own reference embedding code — this repo only produces the
   prebuilt library, not app-side glue code.

## Building a new version

Trigger the **Build CPython for Android** workflow manually (Actions tab →
"Run workflow"), giving it a CPython git tag (default: the pinned "latest
stable" tag in `.github/workflows/build.yml`). Pushing a `v*` tag on this
repo also builds and publishes a release automatically.

## License

The workflow/scripts in this repo are MIT-licensed (see `LICENSE`). The
*contents* of the published tarballs are CPython itself, unmodified, under
the [PSF License](https://docs.python.org/3/license.html) — anyone
redistributing those tarballs (or apps that embed them) needs to comply with
that license, not this repo's MIT one.
