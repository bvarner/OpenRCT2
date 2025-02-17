# CMAKE project for openrct2
cmake_minimum_required(VERSION 3.9)
if (CMAKE_BINARY_DIR STREQUAL CMAKE_SOURCE_DIR)
    message(FATAL_ERROR "Building in-source is not supported! Create a build dir and remove ${CMAKE_SOURCE_DIR}/CMakeCache.txt")
endif()

project(openrct2 CXX)

include(cmake/platform.cmake)
include(CMakeDependentOption)

# vcpkg includes its own copy of duktapeConfig.cmake; only include ours as needed.
if (NOT MSVC)
    include(FindPkgConfig)
    set(CMAKE_PREFIX_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake;${CMAKE_MODULE_PATH}")
endif ()
include(CheckCXXCompilerFlag)
include(GNUInstallDirs)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

set(ROOT_DIR "${CMAKE_CURRENT_LIST_DIR}")
set(CMAKE_MACOSX_RPATH 1)

set(TITLE_SEQUENCE_URL  "https://github.com/OpenRCT2/title-sequences/releases/download/v0.1.2c/title-sequences.zip")
set(TITLE_SEQUENCE_SHA1 "304d13a126c15bf2c86ff13b81a2f2cc1856ac8d")

set(OBJECTS_URL  "https://github.com/OpenRCT2/objects/releases/download/v1.0.16/objects.zip")
set(OBJECTS_SHA1 "ea8be00a7ab5a1e9369277f8f59f487bc588eb6a")

set(REPLAYS_URL  "https://github.com/OpenRCT2/replays/releases/download/v0.0.17/replays.zip")
set(REPLAYS_SHA1 "8DD6D8DBDB1EA50ABCB3A1FB55D2800E13486F44")

option(FORCE32 "Force 32-bit build. It will add `-m32` to compiler flags.")
option(WITH_TESTS "Build tests")
option(PORTABLE "Create a portable build (-rpath=$ORIGIN)" OFF)
option(APPIMAGE "Create an appimage build (-rpath=$ORIGIN/../lib)" OFF)
option(DOWNLOAD_TITLE_SEQUENCES "Download title sequences during installation." ON)
option(DOWNLOAD_OBJECTS "Download objects during installation." ON)
CMAKE_DEPENDENT_OPTION(DOWNLOAD_REPLAYS "Download replays during installation." ON
    "WITH_TESTS" OFF)

# Options
option(STATIC "Create a static build.")
option(USE_MMAP "Use mmap to try loading rct2's data segment into memory.")

option(DISABLE_DISCORD_RPC "Disable Discord-RPC support." OFF)
option(DISABLE_GOOGLE_BENCHMARK "Disable Google Benchmarks support." OFF)
option(DISABLE_HTTP "Disable HTTP support.")
option(DISABLE_NETWORK "Disable multiplayer functionality. Mainly for testing.")
option(DISABLE_TTF "Disable support for TTF provided by freetype2.")
option(ENABLE_LIGHTFX "Enable lighting effects." ON)
option(ENABLE_SCRIPTING "Enable script / plugin support." ON)

option(DISABLE_GUI "Don't build GUI. (Headless only.)")

if (FORCE32)
    set(TARGET_M "-m32")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${TARGET_M}")
    set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} ${TARGET_M}")
endif ()

if (PORTABLE OR WIN32)
    set(CMAKE_INSTALL_LIBDIR "${CMAKE_INSTALL_BINDIR}")
    set(CMAKE_INSTALL_RPATH "$ORIGIN")
endif ()

if (APPIMAGE)
    set(CMAKE_INSTALL_RPATH "$ORIGIN/../lib")
endif ()

# LIST of supported flags, use SET_CHECK_CXX_FLAGS() to apply to target.
# Use ADD_CHECK_CXX_COMPILER_FLAG() to add to list.
set(SUPPORTED_CHECK_CXX_COMPILER_FLAGS "")

include(cmake/ipo.cmake)
list(APPEND IPO_ENABLED_BUILDS RELEASE RELWITHDEBINFO MINSIZEREL)
ipo_enable("${IPO_ENABLED_BUILDS}")

# Describe current version in terms of closest tag
execute_process(
    COMMAND git describe HEAD
    COMMAND sed -E "s/-g.+$//"
    WORKING_DIRECTORY ${ROOT_DIR}
    OUTPUT_VARIABLE OPENRCT2_VERSION_TAG
    OUTPUT_STRIP_TRAILING_WHITESPACE
    ERROR_QUIET
)

# Define current git branch
execute_process(
    COMMAND git rev-parse --abbrev-ref HEAD
    WORKING_DIRECTORY ${ROOT_DIR}
    OUTPUT_VARIABLE OPENRCT2_BRANCH
    OUTPUT_STRIP_TRAILING_WHITESPACE
    ERROR_QUIET
)

# Define short commit hash
execute_process(
    COMMAND git rev-parse --short HEAD
    WORKING_DIRECTORY ${ROOT_DIR}
    OUTPUT_VARIABLE OPENRCT2_COMMIT_SHA1_SHORT
    OUTPUT_STRIP_TRAILING_WHITESPACE
    ERROR_QUIET
)


# Defines
if (USE_MMAP)
    add_definitions(-DUSE_MMAP)
endif ()
if (DISABLE_NETWORK)
    add_definitions(-DDISABLE_NETWORK)
endif ()
if (DISABLE_HTTP)
    add_definitions(-DDISABLE_HTTP)
endif ()
if (DISABLE_TTF)
    add_definitions(-DNO_TTF)
endif ()
if (ENABLE_LIGHTFX)
    add_definitions(-D__ENABLE_LIGHTFX__)
endif ()
if (ENABLE_SCRIPTING)
    add_definitions(-DENABLE_SCRIPTING)
endif ()

if (NOT DISABLE_DISCORD_RPC)
    if(EXISTS "${ROOT_DIR}/discord-rpc")
        # Don't build discord's examples, some of which are in C and do not honour
        # the flags we set for C++. Also we don't use the provided examples.
        set(BUILD_EXAMPLES OFF CACHE BOOL "Build example apps")
        add_subdirectory("${ROOT_DIR}/discord-rpc")
        add_definitions(-D__ENABLE_DISCORD__)
        include_directories("${ROOT_DIR}/discord-rpc/include")
        set(HAVE_DISCORD_RPC TRUE)
        message("Building with discord-rpc support")
    else()
        message("No discord-rpc detected, to enable clone discord-rpc to root directory: ${ROOT_DIR}")
    endif()
endif()

# Copied from https://github.com/opencv/opencv/blob/dcdd6af5a856826fe62c95322145731e702e54c5/cmake/OpenCVDetectCXXCompiler.cmake#L63-L70
if(MSVC64 OR MINGW64)
    set(X86_64 1)
elseif(MINGW OR (MSVC AND NOT CMAKE_CROSSCOMPILING))
    set(X86 1)
elseif(CMAKE_SYSTEM_PROCESSOR MATCHES "amd64.*|x86_64.*|AMD64.*")
    set(X86_64 1)
elseif(CMAKE_SYSTEM_PROCESSOR MATCHES "i686.*|i386.*|x86.*|amd64.*|AMD64.*")
    set(X86 1)
endif()

# Check if a flag exists and add it to the list of compiler (so, not linker) options
function (ADD_CHECK_CXX_COMPILER_FLAG _CXXFLAGS _CACHE_VAR _FLAG)
    CHECK_CXX_COMPILER_FLAG("${_FLAG}" "${_CACHE_VAR}")
    if (${_CACHE_VAR})
        set(SUPPORTED_CHECK_CXX_COMPILER_FLAGS ${SUPPORTED_CHECK_CXX_COMPILER_FLAGS} ${_FLAG} PARENT_SCOPE)
    else ()
        message(STATUS "Unsupported CXXFLAG: ${_FLAG}")
    endif ()
endfunction ()

# Add check flags to a compile TARGET
function (SET_CHECK_CXX_FLAGS _TARGET)
    target_compile_options("${_TARGET}" PRIVATE "${SUPPORTED_CHECK_CXX_COMPILER_FLAGS}")
endfunction()

# Check if a flag exists and add it to the compiler and the linker options
function (ADD_CHECK_CXX_FLAG _CXXFLAGS _CACHE_VAR _FLAG)
    CHECK_CXX_COMPILER_FLAG("${_FLAG}" "${_CACHE_VAR}")
    if (${_CACHE_VAR})
        set(${_CXXFLAGS} "${${_CXXFLAGS}} ${_FLAG}" PARENT_SCOPE)
    else ()
        message(STATUS "Unsupported CXXFLAG: ${_FLAG}")
    endif ()
endfunction ()

if (MSVC)
    # CMAKE does not have a built-in option for setting the CRT, so override the default flags.
    # NOTE: doing it this way avoids a linker warning about one directive overriding another
    set(CMAKE_CXX_FLAGS_DEBUG "/MTd /Zi /Ob0 /Od /RTC1")
    set(CMAKE_CXX_FLAGS_RELEASE "/MT /O2 /Ob2 /DNDEBUG")

    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /utf-8 /permissive- /Zc:externConstexpr /EHsc /WX")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /wd4244") # C4244: 'conversion_type': conversion from 'type1' to 'type2', possible loss of data
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /wd4068") # C4068: unknown pragma

    add_definitions(-D_CRT_SECURE_NO_WARNINGS)
    add_definitions(-D_SCL_SECURE_NO_WARNINGS)
    if (CMAKE_SIZEOF_VOID_P EQUAL 8 AND CMAKE_SYSTEM_PROCESSOR STREQUAL "AMD64")
        add_definitions(-D__SSE4_1__)
        add_definitions(-D__AVX2__)
    endif ()
    add_definitions(-DNOMINMAX)
else ()
    ADD_CHECK_CXX_COMPILER_FLAG(CMAKE_CXX_FLAGS CXX_WARN_SUGGEST_OVERRIDE -Wsuggest-override)
    ADD_CHECK_CXX_COMPILER_FLAG(CMAKE_CXX_FLAGS CXX_WARN_DUPLICATED_COND -Wduplicated-cond)
    ADD_CHECK_CXX_COMPILER_FLAG(CMAKE_CXX_FLAGS CXX_WARN_NON_VIRTUAL_DTOR -Wnon-virtual-dtor)
    ADD_CHECK_CXX_COMPILER_FLAG(CMAKE_CXX_FLAGS CXX_WARN_MISSING_VARIABLE_DECLARATIONS -Wmissing-variable-declarations)
    ADD_CHECK_CXX_COMPILER_FLAG(CMAKE_CXX_FLAGS CXX_WARN_DUPLICATED_BRANCHES -Wduplicated-branches)
    ADD_CHECK_CXX_COMPILER_FLAG(CMAKE_CXX_FLAGS CXX_WARN_RESTRICT -Wrestrict)
    ADD_CHECK_CXX_COMPILER_FLAG(CMAKE_CXX_FLAGS CXX_WARN_MISSING_FIELD_INITIALIZERS -Wmissing-field-initializers)
    ADD_CHECK_CXX_COMPILER_FLAG(CMAKE_CXX_FLAGS CXX_WARN_UNREACHABLE_CODE_BREAK -Wunreachable-code-break)
    ADD_CHECK_CXX_COMPILER_FLAG(CMAKE_CXX_FLAGS CXX_WARN_LOGICAL_OP -Wlogical-op)
    ADD_CHECK_CXX_COMPILER_FLAG(CMAKE_CXX_FLAGS CXX_WARN_RANGE_LOOP_ANALYSIS -Wrange-loop-analysis)
    ADD_CHECK_CXX_COMPILER_FLAG(CMAKE_CXX_FLAGS CXX_WARN_TAUTOLOGICAL_ZERO_COMPARE -Wtautological-unsigned-zero-compare)
    ADD_CHECK_CXX_COMPILER_FLAG(CMAKE_CXX_FLAGS CXX_WNO_CLOBBERED -Wno-clobbered)
    # Disabled due to problems compiling OpenSSL on macOS.
    # ADD_CHECK_CXX_COMPILER_FLAG(CMAKE_CXX_FLAGS CXX_WARN_DOCUMENTATION -Wdocumentation)

    # Items below are not supported by ICC
    if (NOT MINGW)
        # Do not enable for MinGW, as its headers contain redundant declarations of builtin functions
        ADD_CHECK_CXX_COMPILER_FLAG(CMAKE_CXX_FLAGS CXX_WARN_REDUNDANT_DECLS -Wredundant-decls)

        # Currently used MinGW w/GCC 7.2 doesn't provide sane error messages for this
        ADD_CHECK_CXX_COMPILER_FLAG(CMAKE_CXX_FLAGS CXX_WARN_NULL_DEREFERENCE -Wnull-dereference)
    endif ()
    # These have no workarounds available when building with LTO
    if (NOT IPO_BUILD_ENABLED)
        ADD_CHECK_CXX_COMPILER_FLAG(CMAKE_CXX_FLAGS CXX_WARN_SUGGEST_FINAL_TYPES -Wsuggest-final-types)
        ADD_CHECK_CXX_COMPILER_FLAG(CMAKE_CXX_FLAGS CXX_WARN_SUGGEST_FINAL_METHODS -Wsuggest-final-methods)
    endif ()
    ADD_CHECK_CXX_COMPILER_FLAG(CMAKE_CXX_FLAGS CXX_WARN_IGNORED_QUALIFIERS -Wignored-qualifiers)

    # -Wstrict-overflow is only active when -fstrict-overflow is enabled, but -fstrict-overflow
    # is enabled on -O2, -O3, -Os. This should help catch bugs locally before they reach Travis
    # As of 2a435bf -Wstrict-overflow=1 passes, but higher values do not.
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fstrict-overflow")
    ADD_CHECK_CXX_COMPILER_FLAG(CMAKE_CXX_FLAGS CXX_WARN_STRICT_OVERFLOW -Wstrict-overflow=1)

    # Compiler flags
    set(DEBUG_LEVEL 0 CACHE STRING "Select debug level for compilation. Use value in range 0–3.")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fstrict-aliasing -Werror -Wundef -Wmissing-declarations -Winit-self -Wall -Wextra -Wshadow")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-unknown-pragmas -Wno-missing-braces -Wno-comment -Wnonnull -Wno-unused-parameter")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DDEBUG=${DEBUG_LEVEL}")

    if(APPLE)
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-error=objc-method-access")
    endif()

    # On mingw all code is already PIC, this will avoid compiler error on redefining this option
    if (NOT MINGW)
        set(CMAKE_POSITION_INDEPENDENT_CODE ON)
    endif ()

    if (APPLE AND NOT USE_MMAP)
        set(CMAKE_POSITION_INDEPENDENT_CODE OFF)
    else ()
        set(CMAKE_POSITION_INDEPENDENT_CODE ON)
    endif ()
endif ()

if (CXX_WARN_SUGGEST_FINAL_TYPES)
    # Disable -Wsuggest-final-types via pragmas where due.
   add_definitions(-D__WARN_SUGGEST_FINAL_TYPES__)
endif ()

if (CXX_WARN_SUGGEST_FINAL_METHODS)
    # Disable -Wsuggest-final-methods via pragmas where due.
   add_definitions(-D__WARN_SUGGEST_FINAL_METHODS__)
endif ()

# Include sub-projects
include("${ROOT_DIR}/src/openrct2/CMakeLists.txt" NO_POLICY_SCOPE)
include("${ROOT_DIR}/src/openrct2-cli/CMakeLists.txt" NO_POLICY_SCOPE)
if(NOT DISABLE_GUI)
    include("${ROOT_DIR}/src/openrct2-ui/CMakeLists.txt" NO_POLICY_SCOPE)
endif()

# g2
add_custom_command(
    OUTPUT g2.dat
    COMMAND ./openrct2-cli sprite build \"${CMAKE_BINARY_DIR}/g2.dat\" \"${ROOT_DIR}/resources/g2/sprites.json\"
    WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
)
add_custom_target(g2 DEPENDS ${PROJECT} g2.dat)

# Include tests
if (WITH_TESTS)
    enable_testing()
    if (UNIX AND (NOT USE_MMAP))
        include("${ROOT_DIR}/test/testpaint/CMakeLists.txt" NO_POLICY_SCOPE)
    endif ()
    include("${ROOT_DIR}/test/tests/CMakeLists.txt" NO_POLICY_SCOPE)
endif ()

# Install
# Don't recurse, grab all *.txt and *.md files
file(GLOB DOC_FILES "${ROOT_DIR}/distribution/*.txt")
list(APPEND DOC_FILES "${ROOT_DIR}/contributors.md"
                      "${ROOT_DIR}/licence.txt"
                      "${ROOT_DIR}/distribution/scripting.md"
                      "${ROOT_DIR}/distribution/openrct2.d.ts")

# CMake does not allow specifying a dependency chain which includes built-in
# targets, like `install`, so we have to trick it and execute dependency ourselves.
install(CODE "execute_process(COMMAND \"${CMAKE_COMMAND}\" --build \"${CMAKE_CURRENT_BINARY_DIR}\" --target g2)")
if (DOWNLOAD_TITLE_SEQUENCES)
    # If openrct2.parkseq or data/sequence/ exists, assume all the title sequences are already present
    install(CODE
        "if (EXISTS \"\$ENV{DESTDIR}${CMAKE_INSTALL_FULL_DATADIR}/${PROJECT}/sequence/openrct2.parkseq\" OR EXISTS ${CMAKE_SOURCE_DIR}/data/sequence/)\n\
            message(\"Using cached title sequences\")\n\
        else () \n\
            file(DOWNLOAD ${TITLE_SEQUENCE_URL} \$ENV{DESTDIR}${CMAKE_INSTALL_FULL_DATADIR}/${PROJECT}/sequence/title-sequences.zip EXPECTED_HASH SHA1=${TITLE_SEQUENCE_SHA1} SHOW_PROGRESS)\n\
            execute_process(COMMAND \"${CMAKE_COMMAND}\" -E chdir \$ENV{DESTDIR}${CMAKE_INSTALL_FULL_DATADIR}/${PROJECT}/sequence/ \"${CMAKE_COMMAND}\" -E tar xf title-sequences.zip)\n\
            file(REMOVE \$ENV{DESTDIR}${CMAKE_INSTALL_FULL_DATADIR}/${PROJECT}/sequence/title-sequences.zip)\n\
        endif ()")
endif ()
if (DOWNLOAD_OBJECTS)
    # If rct2.wtrcyan.json or data/object/ exists, assume all the objects are already present
    install(CODE
        "if (EXISTS \"\$ENV{DESTDIR}${CMAKE_INSTALL_FULL_DATADIR}/${PROJECT}/object/rct2/water/rct2.wtrcyan.json\" OR EXISTS ${CMAKE_SOURCE_DIR}/data/object/)\n\
            message(\"Using cached objects\")\n\
        else () \n\
            file(DOWNLOAD ${OBJECTS_URL} \$ENV{DESTDIR}${CMAKE_INSTALL_FULL_DATADIR}/${PROJECT}/object/objects.zip EXPECTED_HASH SHA1=${OBJECTS_SHA1} SHOW_PROGRESS)\n\
            execute_process(COMMAND \"${CMAKE_COMMAND}\" -E chdir \$ENV{DESTDIR}${CMAKE_INSTALL_FULL_DATADIR}/${PROJECT}/object/ \"${CMAKE_COMMAND}\" -E tar xf objects.zip)\n\
            file(REMOVE \$ENV{DESTDIR}${CMAKE_INSTALL_FULL_DATADIR}/${PROJECT}/object/objects.zip)\n\
        endif ()")
endif ()
if (DOWNLOAD_REPLAYS)
    install(CODE
        "if (EXISTS \${CMAKE_CURRENT_BINARY_DIR}/testdata/replays/)\n\
            message(\"Using cached replays\")\n\
        else () \n\
	    file(DOWNLOAD ${REPLAYS_URL} \${CMAKE_CURRENT_BINARY_DIR}/testdata/replays/replays.zip EXPECTED_HASH SHA1=${REPLAYS_SHA1} SHOW_PROGRESS)\n\
            execute_process(COMMAND \"${CMAKE_COMMAND}\" -E chdir \${CMAKE_CURRENT_BINARY_DIR}/testdata/replays/ \"${CMAKE_COMMAND}\" -E tar xf replays.zip)\n\
            file(REMOVE \${CMAKE_CURRENT_BINARY_DIR}/testdata/replays/replays.zip)\n\
        endif ()")
endif ()
install(TARGETS "libopenrct2" LIBRARY DESTINATION "${CMAKE_INSTALL_LIBDIR}"
                              ARCHIVE DESTINATION "${CMAKE_INSTALL_LIBDIR}")
if(NOT DISABLE_GUI)
    install(TARGETS "openrct2" RUNTIME DESTINATION "${CMAKE_INSTALL_BINDIR}")
endif()
install(TARGETS "openrct2-cli" OPTIONAL RUNTIME DESTINATION "${CMAKE_INSTALL_BINDIR}")
install(FILES "${CMAKE_CURRENT_BINARY_DIR}/g2.dat" DESTINATION "${CMAKE_INSTALL_DATADIR}/openrct2")
install(DIRECTORY "data/" DESTINATION "${CMAKE_INSTALL_DATADIR}/openrct2")
install(FILES ${DOC_FILES} DESTINATION "${CMAKE_INSTALL_DOCDIR}")
install(FILES "distribution/linux/openrct2.appdata.xml" DESTINATION "${CMAKE_INSTALL_DATAROOTDIR}/metainfo")
if (NOT DISABLE_GUI)
	install(FILES "resources/logo/icon_x16.png" DESTINATION "${CMAKE_INSTALL_DATAROOTDIR}/icons/hicolor/16x16/apps" RENAME "openrct2.png")
	install(FILES "resources/logo/icon_x24.png" DESTINATION "${CMAKE_INSTALL_DATAROOTDIR}/icons/hicolor/24x24/apps" RENAME "openrct2.png")
	install(FILES "resources/logo/icon_x32.png" DESTINATION "${CMAKE_INSTALL_DATAROOTDIR}/icons/hicolor/32x32/apps" RENAME "openrct2.png")
	install(FILES "resources/logo/icon_x48.png" DESTINATION "${CMAKE_INSTALL_DATAROOTDIR}/icons/hicolor/48x48/apps" RENAME "openrct2.png")
	install(FILES "resources/logo/icon_x64.png" DESTINATION "${CMAKE_INSTALL_DATAROOTDIR}/icons/hicolor/64x64/apps" RENAME "openrct2.png")
	install(FILES "resources/logo/icon_x96.png" DESTINATION "${CMAKE_INSTALL_DATAROOTDIR}/icons/hicolor/96x96/apps" RENAME "openrct2.png")
	install(FILES "resources/logo/icon_x128.png" DESTINATION "${CMAKE_INSTALL_DATAROOTDIR}/icons/hicolor/128x128/apps" RENAME "openrct2.png")
	install(FILES "resources/logo/icon_x256.png" DESTINATION "${CMAKE_INSTALL_DATAROOTDIR}/icons/hicolor/256x256/apps" RENAME "openrct2.png")
	install(FILES "resources/logo/icon_flag.svg" DESTINATION "${CMAKE_INSTALL_DATAROOTDIR}/icons/hicolor/scalable/apps" RENAME "openrct2.svg")
	install(FILES "distribution/linux/openrct2.desktop" DESTINATION "${CMAKE_INSTALL_DATAROOTDIR}/applications")
	install(FILES "distribution/linux/openrct2-savegame.desktop" DESTINATION "${CMAKE_INSTALL_DATAROOTDIR}/applications")
	install(FILES "distribution/linux/openrct2-scenario.desktop" DESTINATION "${CMAKE_INSTALL_DATAROOTDIR}/applications")
	install(FILES "distribution/linux/openrct2-uri.desktop" DESTINATION "${CMAKE_INSTALL_DATAROOTDIR}/applications")
endif()
install(FILES "distribution/linux/openrct2-mimeinfo.xml" DESTINATION "${CMAKE_INSTALL_DATAROOTDIR}/mime/packages/" RENAME "openrct2.xml")
install(DIRECTORY "distribution/man/" DESTINATION "${CMAKE_INSTALL_MANDIR}/man6" FILES_MATCHING PATTERN "*.6")
