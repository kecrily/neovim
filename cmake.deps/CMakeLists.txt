# This is not meant to be included by the top-level.
cmake_minimum_required (VERSION 3.10)
project(NVIM_DEPS C)

# Needed for: check_c_compiler_flag()
include(CheckCCompilerFlag)

# Point CMake at any custom modules we may ship
list(APPEND CMAKE_MODULE_PATH "${PROJECT_SOURCE_DIR}/cmake" "${PROJECT_SOURCE_DIR}/../cmake")

# In Windows/MSVC CMAKE_BUILD_TYPE changes the paths/linking of the build
# recipes (libuv, msgpack), make sure it is set
if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE Release)
endif()

set(DEFAULT_MAKE_CFLAGS CFLAGS+=-g)

check_c_compiler_flag(-Og HAS_OG_FLAG)
if(HAS_OG_FLAG)
  set(DEFAULT_MAKE_CFLAGS CFLAGS+=-Og ${DEFAULT_MAKE_CFLAGS})
endif()

if(CMAKE_SYSTEM_NAME MATCHES "OpenBSD")
  # pkg-config 29.2 has a bug on OpenBSD which causes it to drop any paths that
  # *contain* system include paths. To avoid this, we prefix what would be
  # "/usr/include" as "/_usr/include".
  # This check is also performed in the root CMakeLists.txt
  # https://github.com/neovim/neovim/pull/14745#issuecomment-860201794
  set(DEPS_INSTALL_DIR "${CMAKE_BINARY_DIR}/_usr" CACHE PATH "Dependencies install directory.")
else()
  set(DEPS_INSTALL_DIR "${CMAKE_BINARY_DIR}/usr" CACHE PATH "Dependencies install directory.")
endif()

set(DEPS_BIN_DIR "${DEPS_INSTALL_DIR}/bin" CACHE PATH "Dependencies binary install directory.")
set(DEPS_LIB_DIR "${DEPS_INSTALL_DIR}/lib" CACHE PATH "Dependencies library install directory.")
set(DEPS_BUILD_DIR "${CMAKE_BINARY_DIR}/build" CACHE PATH "Dependencies build directory.")
set(DEPS_DOWNLOAD_DIR "${DEPS_BUILD_DIR}/downloads" CACHE PATH "Dependencies download directory.")

option(USE_BUNDLED "Use bundled dependencies." ON)

option(USE_BUNDLED_UNIBILIUM "Use the bundled unibilium." ${USE_BUNDLED})
option(USE_BUNDLED_LIBTERMKEY "Use the bundled libtermkey." ${USE_BUNDLED})
option(USE_BUNDLED_LIBVTERM "Use the bundled libvterm." ${USE_BUNDLED})
option(USE_BUNDLED_LIBUV "Use the bundled libuv." ${USE_BUNDLED})
option(USE_BUNDLED_MSGPACK "Use the bundled msgpack." ${USE_BUNDLED})
option(USE_BUNDLED_LUAJIT "Use the bundled version of luajit." ${USE_BUNDLED})
option(USE_BUNDLED_LUAROCKS "Use the bundled version of luarocks." ${USE_BUNDLED})
option(USE_BUNDLED_LUV "Use the bundled version of luv." ${USE_BUNDLED})
#XXX(tarruda): Lua is only used for debugging the functional test client, no
# build it unless explicitly requested
option(USE_BUNDLED_LUA "Use the bundled version of lua." OFF)
option(USE_BUNDLED_TS_PARSERS "Use the bundled treesitter parsers." ${USE_BUNDLED})
option(USE_BUNDLED_TS "Use the bundled treesitter runtime." ${USE_BUNDLED})

if(USE_BUNDLED AND MSVC)
  option(USE_BUNDLED_GETTEXT "Use the bundled version of gettext." ON)
  option(USE_BUNDLED_LIBICONV "Use the bundled version of libiconv." ON)
else()
  option(USE_BUNDLED_GETTEXT "Use the bundled version of gettext." OFF)
  option(USE_BUNDLED_LIBICONV "Use the bundled version of libiconv." OFF)
endif()

option(USE_EXISTING_SRC_DIR "Skip download of deps sources in case of existing source directory." OFF)

if(WIN32)
  find_package(Git)
  if(NOT Git_FOUND)
    message(FATAL_ERROR "Git is required to apply patches for Windows.")
  endif()
endif()

if(UNIX)
  find_program(MAKE_PRG NAMES gmake make)
  if(MAKE_PRG)
    execute_process(
      COMMAND "${MAKE_PRG}" --version
      OUTPUT_VARIABLE MAKE_VERSION_INFO)
    if(NOT "${OUTPUT_VARIABLE}" MATCHES ".*GNU.*")
      unset(MAKE_PRG)
    endif()
  endif()
  if(NOT MAKE_PRG)
    message(FATAL_ERROR "GNU Make is required to build the dependencies.")
  else()
    message(STATUS "Found GNU Make at ${MAKE_PRG}")
  endif()
endif()

# When using make, use the $(MAKE) variable to avoid warning about the job
# server.
if(CMAKE_GENERATOR MATCHES "Makefiles")
  set(MAKE_PRG "$(MAKE)")
endif()

if(MINGW AND CMAKE_GENERATOR MATCHES "Ninja")
  find_program(MAKE_PRG NAMES mingw32-make)
  if(NOT MAKE_PRG)
    message(FATAL_ERROR "GNU Make for mingw32 is required to build the dependencies.")
  else()
    message(STATUS "Found GNU Make for mingw32: ${MAKE_PRG}")
  endif()
endif()

if(CMAKE_C_COMPILER_ARG1)
  set(DEPS_C_COMPILER "${CMAKE_C_COMPILER} ${CMAKE_C_COMPILER_ARG1}")
else()
  set(DEPS_C_COMPILER "${CMAKE_C_COMPILER}")
endif()

if(CMAKE_CXX_COMPILER)
  set(DEPS_CXX_COMPILER "${CMAKE_CXX_COMPILER}")
endif()

if(CMAKE_OSX_SYSROOT)
  set(DEPS_C_COMPILER "${DEPS_C_COMPILER} -isysroot${CMAKE_OSX_SYSROOT}")
  if(DEPS_CXX_COMPILER)
    set(DEPS_CXX_COMPILER "${DEPS_CXX_COMPILER} -isysroot${CMAKE_OSX_SYSROOT}")
  endif()
endif()

if(CMAKE_OSX_ARCHITECTURES)
  string(REPLACE ";" "|" CMAKE_OSX_ARCHITECTURES_ALT_SEP "${CMAKE_OSX_ARCHITECTURES}")
  # The LuaJIT build does not like being passed multiple `-arch` flags
  # so we handle a universal build the old-fashioned way.
  set(LUAJIT_C_COMPILER "${DEPS_C_COMPILER}")
  foreach(ARCH IN LISTS CMAKE_OSX_ARCHITECTURES)
    set(DEPS_C_COMPILER "${DEPS_C_COMPILER} -arch ${ARCH}")
    if(DEPS_CXX_COMPILER)
      set(DEPS_CXX_COMPILER "${DEPS_CXX_COMPILER} -arch ${ARCH}")
    endif()
  endforeach()
endif()

# If the macOS deployment target is not set manually (via $MACOSX_DEPLOYMENT_TARGET),
# fall back to local system version. Needs to be done here and in top-level CMakeLists.txt.
if(CMAKE_SYSTEM_NAME STREQUAL "Darwin")
  if(NOT CMAKE_OSX_DEPLOYMENT_TARGET)
    execute_process(COMMAND sw_vers -productVersion
                    OUTPUT_VARIABLE MACOS_VERSION
                    OUTPUT_STRIP_TRAILING_WHITESPACE)
    set(CMAKE_OSX_DEPLOYMENT_TARGET "${MACOS_VERSION}")
  endif()
  message("-- Using deployment target ${CMAKE_OSX_DEPLOYMENT_TARGET}")
endif()

set(HOSTDEPS_INSTALL_DIR "${DEPS_INSTALL_DIR}")
set(HOSTDEPS_BIN_DIR "${DEPS_BIN_DIR}")
set(HOSTDEPS_LIB_DIR "${DEPS_LIB_DIR}")
set(HOSTDEPS_C_COMPILER "${DEPS_C_COMPILER}")
set(HOSTDEPS_CXX_COMPILER "${DEPS_CXX_COMPILER}")

include(ExternalProject)

set(LIBUV_URL https://github.com/libuv/libuv/archive/v1.44.2.tar.gz)
set(LIBUV_SHA256 e6e2ba8b4c349a4182a33370bb9be5e23c51b32efb9b9e209d0e8556b73a48da)

set(MSGPACK_URL https://github.com/msgpack/msgpack-c/releases/download/c-4.0.0/msgpack-c-4.0.0.tar.gz)
set(MSGPACK_SHA256 420fe35e7572f2a168d17e660ef981a589c9cbe77faa25eb34a520e1fcc032c8)

# https://github.com/LuaJIT/LuaJIT/tree/v2.1
set(LUAJIT_URL https://github.com/LuaJIT/LuaJIT/archive/50936d784474747b4569d988767f1b5bab8bb6d0.tar.gz)
set(LUAJIT_SHA256 4d44e4709130b031c1c2c81cf5c102dfce877bf454409dabba03249e18870e66)

set(LUA_URL https://www.lua.org/ftp/lua-5.1.5.tar.gz)
set(LUA_SHA256 2640fc56a795f29d28ef15e13c34a47e223960b0240e8cb0a82d9b0738695333)

set(LUAROCKS_URL https://github.com/luarocks/luarocks/archive/v3.8.0.tar.gz)
set(LUAROCKS_SHA256 ab6612ca9ab87c6984871d2712d05525775e8b50172701a0a1cabddf76de2be7)

set(UNIBILIUM_URL https://github.com/neovim/unibilium/archive/92d929f.tar.gz)
set(UNIBILIUM_SHA256 29815283c654277ef77a3adcc8840db79ddbb20a0f0b0c8f648bd8cd49a02e4b)

set(LIBTERMKEY_URL https://www.leonerd.org.uk/code/libtermkey/libtermkey-0.22.tar.gz)
set(LIBTERMKEY_SHA256 6945bd3c4aaa83da83d80a045c5563da4edd7d0374c62c0d35aec09eb3014600)

set(LIBVTERM_URL https://www.leonerd.org.uk/code/libvterm/libvterm-0.1.4.tar.gz)
set(LIBVTERM_SHA256 bc70349e95559c667672fc8c55b9527d9db9ada0fb80a3beda533418d782d3dd)

set(LUV_VERSION 1.44.2-0)
set(LUV_URL https://github.com/luvit/luv/archive/1.44.2-0.tar.gz)
set(LUV_SHA256 44ccda27035bfe683e6325a2a93f2c254be1eb76bde6efc2bd37c56a7af7b00a)

set(LUA_COMPAT53_URL https://github.com/keplerproject/lua-compat-5.3/archive/v0.9.tar.gz)
set(LUA_COMPAT53_SHA256 ad05540d2d96a48725bb79a1def35cf6652a4e2ec26376e2617c8ce2baa6f416)

# cat.exe curl.exe curl-ca-bundle.crt diff.exe tee.exe xxd.exe
set(WINTOOLS_URL https://github.com/neovim/deps/raw/d66e306abf5b846484b4f2adffd896bce7e065d2/opt/win32tools.zip)
set(WINTOOLS_SHA256 2fb2f8d69070b3f16e029913fb95008e6be33893d77fc358012396c275a0fdb7)

set(WINGUI_URL https://github.com/equalsraf/neovim-qt/releases/download/v0.2.16.1/neovim-qt.zip)
set(WINGUI_SHA256 ddb4492db03da407703fb0ab271c4eb060250d1a7d71200e2b3b981cb0de59de)

set(WIN32YANK_X86_URL https://github.com/equalsraf/win32yank/releases/download/v0.0.4/win32yank-x86.zip)
set(WIN32YANK_X86_SHA256 62f34e5a46c5d4a7b3f3b512e1ff7b77fedd432f42581cbe825233a996eed62c)
set(WIN32YANK_X86_64_URL https://github.com/equalsraf/win32yank/releases/download/v0.0.4/win32yank-x64.zip)
set(WIN32YANK_X86_64_SHA256 33a747a92da60fb65e668edbf7661d3d902411a2d545fe9dc08623cecd142a20)

set(GETTEXT_URL https://ftp.gnu.org/pub/gnu/gettext/gettext-0.20.1.tar.gz)
set(GETTEXT_SHA256 66415634c6e8c3fa8b71362879ec7575e27da43da562c798a8a2f223e6e47f5c)

set(LIBICONV_URL https://ftp.gnu.org/pub/gnu/libiconv/libiconv-1.15.tar.gz)
set(LIBICONV_SHA256 ccf536620a45458d26ba83887a983b96827001e92a13847b45e4925cc8913178)

set(TREESITTER_C_URL https://github.com/tree-sitter/tree-sitter-c/archive/v0.20.1.tar.gz)
set(TREESITTER_C_SHA256 ffcc2ef0eded59ad1acec9aec4f9b0c7dd209fc1a85d85f8b0e81298e3dddcc2)

set(TREESITTER_URL https://github.com/tree-sitter/tree-sitter/archive/1f1b1eb4501ed0a2d195d37f7de15f72aa10acd0.tar.gz)
set(TREESITTER_SHA256 a324bdb7ff507cd5157a36b908224d60ba638f8e3363070dd51aa383c7cbd790)

if(USE_BUNDLED_UNIBILIUM)
  include(BuildUnibilium)
endif()

if(USE_BUNDLED_LIBTERMKEY)
  include(BuildLibtermkey)
  if(USE_BUNDLED_UNIBILIUM)
    add_dependencies(libtermkey unibilium)
  endif()
endif()

if(USE_BUNDLED_LIBVTERM)
  include(BuildLibvterm)
endif()

if(USE_BUNDLED_LIBUV)
  include(BuildLibuv)
endif()

if(USE_BUNDLED_MSGPACK)
  include(BuildMsgpack)
endif()

if(USE_BUNDLED_LUAJIT)
  include(BuildLuajit)
endif()

if(USE_BUNDLED_LUA)
  include(BuildLua)
endif()

if(USE_BUNDLED_LUAROCKS)
  include(BuildLuarocks)
endif()

if(USE_BUNDLED_LUV)
  include(BuildLuv)
endif()

if(USE_BUNDLED_GETTEXT)
  include(BuildGettext)
endif()

if(USE_BUNDLED_LIBICONV)
  include(BuildLibiconv)
endif()

if(USE_BUNDLED_TS_PARSERS)
  include(BuildTreesitterParsers)
endif()

if(USE_BUNDLED_TS)
  include(BuildTreesitter)
endif()

if(WIN32)
  include(GetBinaryDeps)

  GetBinaryDep(TARGET wintools
    INSTALL_COMMAND ${CMAKE_COMMAND} -E copy_directory . ${DEPS_INSTALL_DIR}/bin)

  GetBinaryDep(TARGET wingui
    INSTALL_COMMAND ${CMAKE_COMMAND} -E copy_directory bin ${DEPS_INSTALL_DIR}/bin
      COMMAND ${CMAKE_COMMAND} -E copy_directory share ${DEPS_INSTALL_DIR}/share)

  include(TargetArch)
  GetBinaryDep(TARGET "win32yank_${TARGET_ARCH}"
    INSTALL_COMMAND ${CMAKE_COMMAND} -E copy win32yank.exe ${DEPS_INSTALL_DIR}/bin)

  if("${TARGET_ARCH}" STREQUAL "X86_64")
    set(TARGET_ARCH x64)
  elseif(TARGET_ARCH STREQUAL "X86")
    set(TARGET_ARCH ia32)
  endif()
endif()

# clean-shared-libraries removes ${DEPS_INSTALL_DIR}/lib/nvim/parser/c.dll,
# resulting in MSVC build failure in CI.
if (MSVC)
  set(ALL_DEPS ${THIRD_PARTY_DEPS})
else()
  add_custom_target(clean-shared-libraries
    COMMAND ${CMAKE_COMMAND}
      -DREMOVE_FILE_GLOB=${DEPS_INSTALL_DIR}/lib/${CMAKE_SHARED_LIBRARY_PREFIX}*${CMAKE_SHARED_LIBRARY_SUFFIX}*
      -P ${PROJECT_SOURCE_DIR}/cmake/RemoveFiles.cmake
    DEPENDS ${THIRD_PARTY_DEPS}
  )
  set(ALL_DEPS clean-shared-libraries)
endif()

# TODO(justinmk): does anyone use this target?
add_custom_target(third-party ALL
  COMMAND ${CMAKE_COMMAND} -E touch .third-party
  DEPENDS ${ALL_DEPS}
)
