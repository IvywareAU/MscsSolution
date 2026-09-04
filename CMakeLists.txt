cmake_minimum_required(VERSION 3.24)
project(Mscs LANGUAGES CXX)

# ---------------------------------------------------------------------------
# Cross-platform source of truth for Msgcore + TargetCore (the Linux port plan §7).
#
# Phase status: THIS IS NO LONGER A PHASE-0 SKELETON, and the paragraph that said it was
# outlived the work it described. It claimed the library targets were guarded so that a
# Windows configure stayed a no-op -- they are not, and have not been since Phase 6's
# CMake-on-Windows parity landed (the Linux port progress log, "CMake-on-Windows parity (§7)",
# increments 1-7). Both platforms build here now:
#
#   Linux   libmsgcore.so + libtargetcore.so (Platform/ shims, io_uring, OpenSSL);
#           ctest 48/48 via the linux-gcc-debug preset trio, Debug and Release,
#           measured 2026-08-22.
#   Windows Msgcore.dll + TargetCore.dll as MFC extension DLLs (CMAKE_MFC_FLAG=2, .rc
#           compiled), native IOCP + CNG rather than the io_uring/OpenSSL pair;
#           ctest 45/45 via `cmake --preset windows-msvc` -> build -> `ctest --preset`,
#           Debug and Release, measured 2026-08-22.
#
#           These two lines said 15/15 and 12/12 until 2026-08-22. Both were counts
#           taken when the comment was written and neither was re-taken as tests were
#           added, which is the failure mode this whole comment block exists to prevent:
#           a number in prose has no build step to make it wrong out loud.
#
# WHAT THE (2022).vcxproj/.sln STILL DO THAT THIS FILE DOES NOT, because "parity" here
# means parity for the two cores and their test suite, not for the Windows product:
#
#   * CONFIGURATIONS -- CLOSED 2026-08-22, and this bullet is kept rather than deleted
#     because of what closing it found. It used to read: "This file builds one shape,
#     SHARED and x64. It cannot produce the DebugLib/ReleaseLib static archives at all
#     ... and there is no 32-bit preset." All three clauses are now false.
#     windows-msvc-x86 builds Win32 and windows-msvc-lib builds msgcore_static /
#     targetcore_static (MSCS_BUILD_STATIC_LIBS). Neither had ever been built, and
#     each was hiding a defect that had been shipping:
#       - Win32: the FIRST suite ever run against a 32-bit build failed 3 of 45 in both
#         configurations -- memory corruption in the vect block arithmetic, where
#         P2PmsgVect_SizeofItem sized the name at 27 bytes and P2PmsgVect_InitItem wrote
#         127 into it. x64 was never wrong only because both rules answered 127 there.
#       - Static: P2PCngCrypto.h guarded its export macro `_WIN32 && !TargetCore_STATIC`,
#         so a Windows static build fell through to the GCC arm and MSVC was handed
#         __attribute__((visibility("default"))). The vcxproj's OWN DebugLib/ReleaseLib
#         configurations define TargetCore_STATIC, so 4 of the 8 configurations the .sln
#         declares could not compile that header either.
#     Both are fixed and Win32 measures 45/45 in Debug and Release. Not claimed: no test
#     links against either static archive yet.
#   * STAGING -- SETTLED 2026-08-22, and settled as "never" rather than "not yet".
#     Directory.Build.props puts import libraries in
#     $(WDMSCS_LIB)\$(Platform)\$(Configuration) and a CustomBuildStep copies the DLLs to
#     bin\<Config><Arch>. Every downstream consumer resolves against that layout. This file
#     installs into a GNUInstallDirs prefix instead and feeds that tree NOTHING, because
#     the split between the two build systems is product vs verification rather than
#     Windows vs Linux. The full argument, including why running the vcxproj's own copy
#     script is not sufficient, is in the block below headed "WHY THERE IS NO STAGING
#     STEP HERE". It is written down because an MSCS_STAGE_TO_BIN option existed for a
#     few hours that day and the reasoning for deleting it is worth more than the option.
#   * WHOLE LAYERS. 120 .vcxproj against 13 CMakeLists.txt: the facades and their COM
#     layers, MsgcoreMFC/P2PmsgMFC, Chartboard, TreeFsGUI/P2PeerFsGUI, P2PmsgPython, and
#     both _UseExamples trees have no CMake at all and are built only from the .sln.
#
# So the vcxproj remain authoritative for the Windows PRODUCT, and this file is
# authoritative for Linux and at parity for the cores + MscsUnitTests on both.
# ---------------------------------------------------------------------------

set(CMAKE_CXX_STANDARD 23)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

if(NOT WIN32)
    add_compile_options(-fvisibility=hidden)
endif()

enable_testing()   # top-level, so `ctest --test-dir build` discovers subdir add_test()s

# ---------------------------------------------------------------------------
# Runtime loader path — for EVERY test registered in the CALLING directory.
#
# msgcore and targetcore are the only SHARED targets involved, and this tree sets
# no unified CMAKE_RUNTIME_OUTPUT_DIRECTORY, so each lands in its own per-target
# build directory and never beside the test exe. A ctest run therefore only works
# if something puts those directories on the loader path first; without it a test
# fails at 0xc0000135 (STATUS_DLL_NOT_FOUND) BEFORE ENTERING MAIN, which looks
# exactly like a code regression and is not one.
#
# IT LIVES HERE, AT THE ROOT, RATHER THAN IN MscsUnitTests/MscsTestCommon.cmake,
# AND THE MOVE WAS PAID FOR BY THE SECOND OCCURRENCE. That file defined it and is
# included by MscsUnitTests and MscsUnitTestsExternal, so those two were covered
# and nothing else could be: P2PeerWeb is its own component repository, is added
# BEFORE MscsUnitTests, and is guarded only by its own existence, so it cannot
# include a file that may not be in the checkout. On 2026-09-04 all eleven
# p2pweb_w* tests were failing under a bare `ctest` for exactly that reason -- the
# same eleven-test, same-status-code failure MscsTestCommon.cmake's own comment
# records from 2026-08-13, in a second directory, because the FIX was shared and
# the PLACE TO PUT IT was not. A function defined at the root is visible to every
# add_subdirectory below it, so there is still exactly one copy and there is no
# longer a component that cannot reach it.
#
# Two things this has to get right:
#   * MULTI-CONFIG. The Visual Studio generator appends $<CONFIG> to the output
#     directory and that is not known at configure time, so the value must be the
#     generator expression $<TARGET_FILE_DIR:...> -- never a literal "Debug".
#   * The VARIABLE NAME is per-platform: PATH on Windows, LD_LIBRARY_PATH on Linux.
#     Linux already resolves the .so through the build-tree RPATH, so it is
#     belt-and-braces there -- it is what would notice if RPATH were ever disabled.
#
# It sets ENVIRONMENT_MODIFICATION, which is a DIFFERENT ctest property from
# ENVIRONMENT: a test that already sets its own ENVIRONMENT (p2p_fuzzhttp sets the
# sanitiser options) keeps it untouched.
#
# WHAT DEFEATS ALL OF THIS, and did (2026-08-13): a COPY of msgcore.dll or
# targetcore.dll left sitting in the test exes' own output directory. Windows
# searches the application directory BEFORE it looks at PATH, so such a copy wins
# over the entry prepended here and every test in the directory silently runs
# against it. Two stale copies were found next to the exes and deleted; while they
# were there a rebuilt library was NOT what ctest exercised, and a fix could be
# verified green without ever having been loaded. Nothing here puts them there --
# do not add a POST_BUILD copy step to "help". If a test needs a DLL, this loader
# path is the mechanism; a copy beside the exe is a stale result waiting to
# happen. check_repo_invariants.py fails the build if a POST_BUILD step appears.
#
# Call this ONCE, at the END of the including CMakeLists.txt: it reads the
# directory's accumulated TESTS property, so a new add_test() is covered
# automatically instead of failing the day somebody forgets the boilerplate.
#
# It is a no-op when the directory registered no tests, and REFUSES rather than
# generating an unusable path when the two library targets are not in this
# configure at all -- a genex naming a target that does not exist is a generate-
# time error, and a silent skip here is the failure this whole block is about.
# ---------------------------------------------------------------------------
function(_mscs_apply_loader_path)
    if(WIN32)
        set(_ldvar PATH)
    else()
        set(_ldvar LD_LIBRARY_PATH)
    endif()
    get_property(_tests DIRECTORY PROPERTY TESTS)
    if(NOT _tests)
        return()
    endif()
    if(NOT TARGET msgcore OR NOT TARGET targetcore)
        message(FATAL_ERROR
            "_mscs_apply_loader_path() called from "
            "${CMAKE_CURRENT_SOURCE_DIR} with tests registered but without "
            "msgcore/targetcore in this configure. Those tests would fail at "
            "STATUS_DLL_NOT_FOUND before entering main; guard the call, or do "
            "not register the tests.")
    endif()
    set_tests_properties(${_tests} PROPERTIES ENVIRONMENT_MODIFICATION
        "${_ldvar}=path_list_prepend:$<TARGET_FILE_DIR:msgcore>;${_ldvar}=path_list_prepend:$<TARGET_FILE_DIR:targetcore>")
endfunction()

# The shim layer lives in Msgcore now -- the standalone Platform/ repository was
# retired on 2026-09-03 and MSCS/Platform/ deleted with it, so this is the one and
# only copy in the tree. Added HERE, before Msgcore, because the header check at the
# bottom of this file needs p2pplatform even when MSCS_BUILD_LIBS is OFF;
# Msgcore/CMakeLists.txt adds the same directory itself when nothing has, and its
# `if(NOT TARGET p2pplatform)` guard is what makes the two entry points agree.
#
# Do not re-introduce a second copy anywhere in this tree. A quoted include resolves
# relative to the citing file, so two physical p2ptypes.h reach one Linux translation
# unit and #pragma once cannot deduplicate them -- ~40 redefinition errors, invisible
# on Windows because the win-compat forwarders that reach the second copy are
# generated only on Linux.
add_subdirectory(Msgcore/Platform)

# OFF BY DEFAULT, AND THAT DEFAULT IS A TRAP WORTH KNOWING. With it off, a bare
# `cmake -S . -B build` configures and builds cleanly while producing neither library --
# only p2pplatform and the header check below -- so it looks like a working build of
# nothing. Every preset in CMakePresets.json sets it ON; if you are not using one, pass
# -DMSCS_BUILD_LIBS=ON.
option(MSCS_BUILD_LIBS "Build the msgcore/targetcore stack (both platforms; presets set this ON)" OFF)

# The static half of the CONFIGURATIONS gap named above. The cores' vcxproj carry
# DebugLib/ReleaseLib -- StaticLibrary, UseOfMfc still Dynamic, and Msgcore_STATIC +
# TargetCore_STATIC defined, which is the arm Msgcore.h/TargetCore.h already have for
# "no dllimport, nothing to import". With this ON, msgcore_static and targetcore_static
# are added ALONGSIDE the shared pair rather than replacing them, because a CMake config
# and a CMake target are orthogonal where an MSBuild configuration is not: one build tree
# can hold both shapes, which the .sln cannot. OFF by default so the ordinary build does
# not compile the whole core twice; the windows-msvc-lib preset turns it on.
option(MSCS_BUILD_STATIC_LIBS "Also build msgcore_static/targetcore_static (the vcxproj DebugLib/ReleaseLib shape)" OFF)

# ---------------------------------------------------------------------------
# WHY THERE IS NO STAGING STEP HERE, and why adding one is the wrong instinct.
#
# The vcxproj deploys: a CustomBuildStep runs vsutils/IW_CustomBuildStep.bat to copy each
# DLL into bin\<Config><Arch>, and Directory.Build.props points the LINKER at
# $(WDMSCS_LIB)\$(Platform)\$(Configuration) for the import library. Roughly 197 project
# references resolve against that layout. This file deliberately does none of it.
#
# An MSCS_STAGE_TO_BIN option existed briefly on 2026-08-22 and was REMOVED the same day,
# so if you are about to add it back, this is the argument you are answering:
#
#   * The line between the two build systems is PRODUCT vs VERIFICATION, not Windows vs
#     Linux. The vcxproj owns the Windows product and its delivery -- all 120 projects,
#     including the MFC UI, the COM facades and both _UseExamples trees, none of which
#     have any CMake. This file owns verification on BOTH platforms (the two cores,
#     p2pplatform and the 45-test suite) plus the Linux product. Verification has no
#     business writing into a delivery tree.
#   * bin\<Config><Arch> is ONE path with, if staging existed, TWO possible producers and
#     nothing in the file recording which one made it. Last build wins, silently. That is
#     the same shadowing hazard that once had ctest measuring a stale DLL sitting beside
#     the test exes.
#   * The scopes do not match anyway. Staging from here would fill the 13 targets' worth of
#     that directory and leave the other ~107 to the .sln, making bin\ a mixture of two
#     builds with no provenance.
#   * Running the same .bat is necessary but NOT sufficient, which is the part that looks
#     like it should just work: the import library is not copied by any script in the
#     vcxproj either -- the linker writes it to $(WDMSCS_LIB) directly. A faithful stage
#     would have to reproduce a linker output path as well, by a different mechanism.
#
# So a CMake build feeds nothing. `cmake --install` into a GNUInstallDirs prefix is how an
# artifact leaves this tree, and TargetCore/install-check gates that path.
# ---------------------------------------------------------------------------

if(MSCS_BUILD_LIBS)
    # Msgcore builds on BOTH platforms: Linux (libmsgcore.so via the Platform shims) and,
    # since the CMake-on-Windows parity work (§7), Windows/MSVC (Msgcore.dll, dynamic MFC).
    # The sentence that used to follow -- "the rest of the stack is still Linux-only under
    # CMake" -- was contradicted three lines below it by the time anyone read it: TargetCore,
    # TreeFs, DspChain and the MscsUnitTests suites all configure on both now. What stays
    # one-sided is named at each guard: the FUSE spike and the transport harnesses that need
    # PTYs on Linux, the WinFsp frontend and com232/com0com on Windows.
    add_subdirectory(Msgcore)

    # TargetCore builds on both platforms now (Windows/MSVC: native IOCP + CNG, §7).
    # Guard by existence so a Msgcore-only checkout (e.g. Phase-1 verification) still configures.
    if(EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/TargetCore/CMakeLists.txt)
        add_subdirectory(TargetCore)
    endif()

    # TreeFs (P2P FileSystem core, portable C++17 over the msgcore C API) — BOTH platforms.
    # Needs only Msgcore, so it builds in a Msgcore-only checkout. Guard by existence since
    # TreeFs is its own component repo. Added BEFORE MscsUnitTests (unit_suite links treefs).
    # The FUSE spike (MSCS_BUILD_FUSE) stays Linux-only inside TreeFs/CMakeLists.txt.
    if(EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/TreeFs/CMakeLists.txt)
        add_subdirectory(TreeFs)
    endif()

    # DspChain (DSP signal-chain over the TargetCore framework, p2p_SignalChain.md) —
    # BOTH platforms. Needs targetcore + msgcore, so it is added AFTER them and BEFORE
    # MscsUnitTests (the dsp_* harnesses link dspchain). Guard by existence since DspChain
    # is its own component repo.
    if(EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/DspChain/CMakeLists.txt)
        add_subdirectory(DspChain)
    endif()

    # P2PeerFs (mountable live-hub filesystem, p2p_PeerFs.md) — design-validation
    # spikes for now. Added AFTER DspChain + targetcore (spike 1 links dspchain,
    # spike 2 links targetcore). Guard by existence since P2PeerFs is its own dir.
    if(EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/P2PeerFs/CMakeLists.txt)
        add_subdirectory(P2PeerFs)
    endif()

    # P2PeerUtilityHubs (tree-moving utility hubs, p2p_PeerUtilityHubs.md) — BOTH
    # platforms. Needs targetcore + msgcore + treefs, so it is added AFTER them and
    # BEFORE MscsUnitTests (the util_* harnesses link p2putilhubs). Guard by existence.
    if(EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/P2PeerUtilityHubs/CMakeLists.txt)
        add_subdirectory(P2PeerUtilityHubs)
    endif()

    # P2PeerWeb (HTTP/HTTPS front door whose every logical block is a hub,
    # p2p_WebServer.md). Needs targetcore + msgcore, so it is added AFTER them.
    # LINUX IS THE RELEASE TARGET (§3.7) and there is deliberately no vcxproj, so
    # unlike DspChain this file is the ONLY build definition it has. A Windows
    # configure still builds it, as verification rather than as a product: the
    # framework-free half (p2pweb_core -- envelope, HTTP parser, platform floor)
    # compiles and its suite runs on both toolchains, which is what keeps the
    # parsers §9 calls the hostile surface under test on every box.
    if(EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/P2PeerWeb/CMakeLists.txt)
        add_subdirectory(P2PeerWeb)
    endif()

    # MscsUnitTests regression suites — BOTH platforms now (§7). The CMakeLists builds the
    # portable suites (unit_suite / u8 smokes / daemon harness / golden) under CMake+MSVC and
    # guards the Linux-transport mesh harnesses (dmx/wsa/pipe/232/mix) + the bash-script tests
    # (golden_utf16_bytes / alex_test) to Linux internally.
    #
    # Since the 2026-08-14 split it needs ONLY targetcore + msgcore + p2pplatform. Everything
    # that needed treefs, p2putilhubs or dspchain is in MscsUnitTestsExternal below.
    if(EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/TargetCore/CMakeLists.txt AND
       EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/MscsUnitTests/CMakeLists.txt)
        add_subdirectory(MscsUnitTests)
    endif()

    # MscsUnitTestsExternal is NOT configured by this tree, and its absence is a
    # decision rather than an oversight.  Those suites - TreeFS, replication, blob,
    # P2PeerUtilityHubs, plus the DSP and util harnesses - cover components outside
    # the msgcore/targetcore core, and configuring them drags TreeFs, DspChain and
    # P2PeerUtilityHubs into every build of this one.  They are built from their own
    # directory, not from here, and there is deliberately no switch to turn them on.
    #
    # SO THE ctest TOTAL FROM THIS TREE IS A CORE TOTAL.  Compared against a figure
    # taken when those suites were configured it will look 20 tests short, and they
    # were never registered rather than lost.  The security label is unaffected -
    # all of it lives in MscsUnitTests - which is the property the sanitiser gates
    # depend on.
endif()

# Phase-0 header self-consistency check (see Msgcore/Platform/checks/).
option(MSCS_BUILD_HEADER_CHECK "Compile a TU that includes every platform header" ON)
if(MSCS_BUILD_HEADER_CHECK)
    add_executable(platform_header_check Msgcore/Platform/checks/header_check.cpp)
    target_link_libraries(platform_header_check PRIVATE p2pplatform)
endif()
