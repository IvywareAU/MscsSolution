# MscsSolution

**The parent CMake solution for the MSCS component repositories.** Two files — `CMakeLists.txt`
and `CMakePresets.json` — and nothing else. It builds nothing on its own; it is the thing the
component repositories assemble *into*.

## Why this repository exists

The MSCS components are separate repositories, and the top-level build that joins them was not
one of them. It lived only in the working tree on one developer's machine, which meant CI could
not configure the project at all: `TargetCore`'s `solution-build.yml` had to be handed a
`solution_repo` that did not exist anywhere, so it had never been dispatched, and the automatic
workflow compiled 4 of 30 translation units. A green tick meant the repository's wiring was
intact and nothing more.

This repository is the missing piece. With it, a runner can check out the component
repositories by name and get the same build a developer gets.

## The layout it expects

Clone this, then place the components inside it:

```
<this repo>/
├── CMakeLists.txt          <- here
├── CMakePresets.json       <- here
├── Msgcore/                <- IvywareAU/Msgcore       (required)
├── TargetCore/             <- IvywareAU/TargetCore    (optional)
└── MscsUnitTests/          <- IvywareAU/MscsUnitTests (optional)
```

`Msgcore` is the only unconditional one, and it is unconditional even with `MSCS_BUILD_LIBS`
off: the platform shim layer lives inside it, at `Msgcore/Platform/`, so `add_subdirectory` is
not guarded and `platform_header_check` compiles `Msgcore/Platform/checks/header_check.cpp`.
Everything else is guarded by `EXISTS`, so a partial checkout configures and simply builds
less. That is deliberate: a Msgcore-only checkout is a supported way to verify the port.

The build file also carries `EXISTS` guards for components that are not published here and
are not planned to be. On a checkout of the repositories above, those branches do not fire.

## Build

```bash
cmake --preset windows-msvc        # or linux-gcc-debug, linux-clang-debug,
cmake --build --preset windows-msvc-debug
ctest --test-dir build/windows-msvc -C Debug --output-on-failure
```

Sanitizer presets: `linux-gcc-asan`, `linux-gcc-tsan`.

Do not use `ctest --preset` in automation. It exports a hardcoded `...\Debug` PATH, which is
wrong for `--config Release` and useless to anything invoking `ctest` directly. The tests carry
their own loader path; a bare `ctest --test-dir` is the supported form and is what CI runs, on
purpose, as the regression guard for that.

## The one thing to know before editing

**These two files exist twice.** The authoritative editing copy is the one in the developer's
MSCS working tree, where the components sit as sibling directories inside a private repository
that holds the design and planning documents. This repository carries a copy so that CI — and
anyone without that private tree — can build.

That duplication is a real cost and it is written down here rather than discovered later: **a
change to the build made in one place and not the other produces a CI run that verifies a
different build from the one developers use**, which is the failure mode most likely to waste a
day. Until the two are collapsed into one, treat a change to either file as requiring both.

The clean fix is to move the private planning documents out of the parent working tree so that
this repository can *be* it rather than mirror it. That is a working-tree reorganisation, not a
build change, and it has not been done.

## Licence

Copyright 2026 Khrustal & Mann, MELBOURNE, VICTORIA, AUSTRALIA, 3000.

Licensed under the Apache License, Version 2.0. See [`LICENSE`](LICENSE) for the full text,
and [`NOTICE`](NOTICE) for attribution.
