cmake_minimum_required(VERSION 3.24)
project(Mscs LANGUAGES CXX)

# ---------------------------------------------------------------------------
# Cross-platform source of truth for Msgcore + TargetCore (LinuxPortPlan.md §7).
#
# Phase status: this is the Phase-0 skeleton. It builds the header-only platform
# layer everywhere and, on Linux, will build libmsgcore.so / libtargetcore.so as the
# shims are implemented (Phases 1-4). On Windows the authoritative build remains the
# (2022).vcxproj/.sln files until Phase 6 signs off CMake-on-MSVC parity; the CMake
# library targets below are guarded so a Windows configure stays a no-op scaffold.
# ---------------------------------------------------------------------------

set(CMAKE_CXX_STANDARD 23)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

if(NOT WIN32)
    add_compile_options(-fvisibility=hidden)
endif()

enable_testing()   # top-level, so `ctest --test-dir build` discovers subdir add_test()s

# The shim layer lives in Msgcore now -- the standalone Platform repository was
# retired on 2026-09-03 and its tree vendored into Msgcore/Platform, so this is the
# one and only copy. It is added HERE, before Msgcore, because the header check at
# the bottom of this file needs p2pplatform even when MSCS_BUILD_LIBS is OFF;
# Msgcore/CMakeLists.txt adds the same directory itself when nothing has, and its
# `if(NOT TARGET p2pplatform)` guard is what makes the two entry points agree.
#
# Do not re-introduce a second copy anywhere in this tree. A quoted include resolves
# relative to the citing file, so two physical p2ptypes.h reach one Linux translation
# unit and #pragma once cannot deduplicate them -- ~40 redefinition errors, invisible
# on Windows because the win-compat forwarders that reach the second copy are
# generated only on Linux.
add_subdirectory(Msgcore/Platform)

# The msgcore / targetcore library targets are enabled on Linux as the port progresses.
# On Windows they are intentionally not built here (vcxproj remains authoritative, §7).
option(MSCS_BUILD_LIBS "Build msgcore/targetcore via CMake (Linux port, Phases 1-4)" OFF)

if(MSCS_BUILD_LIBS)
    # Msgcore builds on BOTH platforms now: Linux (libmsgcore.so via the Platform shims)
    # and, as of the CMake-on-Windows parity work (§7), Windows/MSVC (Msgcore.dll, dynamic
    # MFC). The rest of the stack is still Linux-only under CMake until its Windows
    # increments land (TargetCore needs native IOCP + CNG; the mesh harnesses + TreeFs
    # FUSE frontend follow it). On Windows those remain the authoritative (2022).vcxproj.
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

    # MscsUnitTestsExternal — the TreeFS / replication / blob / P2PeerUtilityHubs suites and
    # the DSP + util harnesses. Added AFTER MscsUnitTests: it includes that directory's
    # MscsTestCommon.cmake and compiles its TestFramework.cpp by path, and its own
    # CMakeLists guards the dspchain and p2putilhubs blocks on those components existing.
    if(EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/TargetCore/CMakeLists.txt AND
       EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/MscsUnitTestsExternal/CMakeLists.txt AND
       EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/MscsUnitTests/MscsTestCommon.cmake)
        add_subdirectory(MscsUnitTestsExternal)
    endif()
endif()

# Phase-0 header self-consistency check (see Msgcore/Platform/checks/).
option(MSCS_BUILD_HEADER_CHECK "Compile a TU that includes every platform header" ON)
if(MSCS_BUILD_HEADER_CHECK)
    add_executable(platform_header_check Msgcore/Platform/checks/header_check.cpp)
    target_link_libraries(platform_header_check PRIVATE p2pplatform)
endif()
